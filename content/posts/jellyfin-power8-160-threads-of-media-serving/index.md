---
title: "Jellyfin on POWER8: 160 Threads of Media Serving"
date: 2026-02-28T17:00:00-06:00
draft: false
tags: ["power8", "ppc64le", "jellyfin", "dotnet", "homelab", "fedora"]
categories: ["Alternative Computing"]
description: "Running Jellyfin 10.11 on an IBM POWER8 server with .NET 10 — a tale of 9 build attempts, zero Dart VMs, and one very stubborn lobster."
---

*This is Part 2 of the POWER8 series. Part 1: [What Microsoft Won't Ship: .NET on POWER8](/posts/dotnet-power8-what-microsoft-wont-ship/)*

---

## Previously, on "Things That Shouldn't Work But Do"

[Last time](/posts/dotnet-power8-what-microsoft-wont-ship/), I compiled the entire .NET 8 SDK from source on an IBM POWER8 — a dual-socket beast with 20 cores and 160 threads of SMT8 goodness. It took 6 hours, 7 patches, and one crisis of faith.

The natural next question: **what do you actually run on it?**

The answer, obviously, is Jellyfin. Because nothing says "responsible use of enterprise hardware" like streaming movies from a $30,000 server that IBM designed for running databases.

## The Setup

The POWER8 got an upgrade since Part 1:

| Before | After |
|--------|-------|
| Gentoo Linux | Fedora 43 Server |
| .NET 8 (hand-compiled, 6 hours) | .NET 10 (`dnf install`, 3 seconds) |
| Suffering | Slightly less suffering |

That Fedora migration is a story in itself — 5 installation attempts, a fight with OPAL firmware over XFS boot partitions, a kickstart file that crashed Anaconda, and a manual `grub2-install --no-nvram` from a chroot. But we got there.

The important bit: **Fedora 43 ships .NET 10.0.103 for ppc64le out of the box.** What took me 6 hours to compile from source on Gentoo takes `dnf install dotnet-sdk-10.0` and a coffee break on Fedora. Red Hat does what Microsoft won't.

## Building Jellyfin: A Comedy in Nine Acts

Jellyfin's official Docker image supports `amd64` and `arm64`. Not `ppc64le`. So we build from source. How hard could it be?

**Nine attempts hard.**

### Act 1-3: The Sass Saga

Jellyfin's web UI uses webpack with `sass-embedded` for stylesheet compilation. `sass-embedded` bundles a native Dart VM binary. Dart doesn't support ppc64le.

```
Error: Embedded Dart Sass couldn't find the embedded compiler executable.
Please make sure the optional dependency sass-embedded-linux-ppc64 is installed.
```

The fix: rip out `sass-embedded` and replace it with the pure JavaScript `sass` compiler:

```dockerfile
RUN npm ci --no-audit --unsafe-perm && \
    rm -rf node_modules/sass-embedded node_modules/sass-embedded-* && \
    npm install sass@^1.77.0 --save-dev && \
    sed -i "s/'sass-loader'/{loader: 'sass-loader', options: {implementation: require('sass')}}/" webpack.common.js && \
    npm run build:production
```

Three attempts to get this right — `npm uninstall` doesn't actually remove it from the lockfile, and the `sass-loader` has a resolution order that prefers `sass-embedded` over `sass`. The `rm -rf` + webpack config patch was the nuclear option that worked.

Webpack then proceeded to chew through the entire Jellyfin web UI on POWER8 in about 6.5 minutes, peaking at 10.5 GB of RAM and 560% CPU. JavaScript was not designed for this architecture, but POWER8 doesn't care.

### Act 4-5: The .NET Version Dance

Jellyfin 10.11's `global.json` specifies:

```json
{
    "sdk": {
        "version": "9.0.0",
        "rollForward": "latestMinor"
    }
}
```

`latestMinor` means "any 9.x SDK is fine." It does **not** mean "10.x is fine too." Fedora 43 only ships .NET 10. Fix:

```dockerfile
RUN sed -i 's/"latestMinor"/"latestMajor"/' global.json
```

Then `dotnet publish` tried to download `Microsoft.NETCore.App.Host.linux-ppc64le` from NuGet. Which doesn't exist. Because Microsoft doesn't publish it. The familiar refrain of this series.

```dockerfile
RUN dotnet publish Jellyfin.Server \
    --self-contained false \
    -p:UseAppHost=false
```

`UseAppHost=false` tells the build to skip the native host binary and use `dotnet jellyfin.dll` as the entrypoint instead. Same trick from Part 1.

### Act 6-8: SkiaSharp, My Old Nemesis

With the build succeeding, the container started... and immediately crashed:

```
System.DllNotFoundException: libSkiaSharp
```

SkiaSharp wraps Google's Skia graphics library for .NET. It ships native binaries for x64, arm64, and a few others. Not ppc64le. The static constructor tries to load the native library and throws before the `IsNativeLibAvailable()` check can run.

Attempt 1: Delete the SkiaSharp DLLs → crashes because Jellyfin.Server references the managed assembly.

Attempt 2: Create a stub `libSkiaSharp.so` with `gcc -shared` → crashes because SkiaSharp validates the native library version.

Attempt 3: Surgery.

```dockerfile
# Remove SkiaSharp from the build entirely
RUN sed -i '/Jellyfin.Drawing.Skia/d' Jellyfin.Server/Jellyfin.Server.csproj && \
    sed -i '/using Jellyfin.Drawing.Skia/d' Jellyfin.Server/CoreAppHost.cs && \
    sed -i 's/bool useSkiaEncoder = SkiaEncoder.IsNativeLibAvailable();/bool useSkiaEncoder = false;/' Jellyfin.Server/CoreAppHost.cs && \
    sed -i 's/? typeof(SkiaEncoder)/? typeof(NullImageEncoder)/' Jellyfin.Server/CoreAppHost.cs
```

Remove the project reference, remove the using directive, hardcode `useSkiaEncoder = false`, and replace the type reference. Jellyfin has a `NullImageEncoder` fallback that works fine — you lose thumbnail generation, but playback is completely unaffected.

### Act 9: It Lives

```
[INF] Emby.Server.Implementations.ApplicationHost: Core startup complete
[INF] Main: Startup complete 0:00:19.4247792
```

19 seconds from cold start to serving media. On POWER8. Running .NET 10. Inside a container. From source.

```bash
$ curl http://localhost:8096/health
Healthy
```

## The Final Dockerfile

Three-stage build, all Fedora 43 native:

1. **Web stage**: Node.js 22 + webpack with JS sass
2. **Server stage**: .NET 10 SDK + patched source (no Skia, latestMajor rollforward)
3. **Runtime stage**: ASP.NET 10 runtime + ffmpeg-free

The whole thing builds in about 20 minutes on POWER8, with webpack being the bottleneck.

It's published on GHCR:

```bash
podman pull ghcr.io/felipedbene/jellyfin-power8:latest

podman run -d --name jellyfin -p 8096:8096 \
  -e DOTNET_ROLL_FORWARD=LatestMajor \
  -v ~/jellyfin-config:/config:Z \
  -v ~/jellyfin-cache:/cache:Z \
  -v /path/to/media:/media:ro \
  ghcr.io/felipedbene/jellyfin-power8:latest
```

## What Works

- **Media playback**: Everything. Movies, TV shows, music, home videos.
- **Transcoding**: CPU-only (no GPU on POWER8), but 160 threads of POWER8 handle software transcoding. FFmpeg detects VDPAU, VAAPI, and Vulkan capabilities even without a GPU — future-proofing for when someone inevitably plugs a GPU into this thing.
- **Web UI**: Full Jellyfin interface, all features minus image processing.
- **Client compatibility**: Fire TV, Android, iOS, web — all work. The 10.11 server version is compatible with current clients (unlike the 10.9 we had before on Gentoo).
- **NFS media**: 22TB of media served over NFS from a ZFS pool on another machine.

## What Doesn't

- **Thumbnails/image processing**: No SkiaSharp means no server-side image manipulation. Clients handle their own image scaling.
- **Hardware transcoding**: POWER8 has no GPU. Software transcoding only. It's fine.

## The Bigger Picture

This POWER8 S822 was designed to run enterprise databases and middleware. It has 160 hardware threads, 128GB of ECC RAM, and redundant everything. Using it as a media server is absurd.

But it also demonstrates something real: **the software ecosystem is the bottleneck, not the hardware.** POWER8 can run anything that compiles for ppc64le. The problems aren't performance — they're binary dependencies that only ship for x86 and ARM:

- Dart VM (sass-embedded): no ppc64le
- SkiaSharp/Skia: no ppc64le  
- .NET apphost: no ppc64le on NuGet
- Docker base images: most only ship amd64/arm64

Each one has a workaround. Pure JS sass instead of Dart. NullImageEncoder instead of Skia. `UseAppHost=false` for the apphost. Fedora as the base image instead of Alpine/Debian.

The pattern: when a native binary doesn't exist for your architecture, find the managed/interpreted fallback. It's usually slower but always works.

## Source

- **Jellyfin container**: [github.com/felipedbene/jellyfin-power8](https://github.com/felipedbene/jellyfin-power8)
- **.NET 8 build guide**: [github.com/felipedbene/dotnet-power8](https://github.com/felipedbene/dotnet-power8)
- **Part 1**: [What Microsoft Won't Ship](/posts/dotnet-power8-what-microsoft-wont-ship/)

---

*Built on an IBM S822 (8335-GCA) — Dual POWER8, 160 threads, 128GB RAM, Fedora 43. The machine that refuses to retire.*
