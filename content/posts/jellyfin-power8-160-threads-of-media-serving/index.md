---
title: "Jellyfin on POWER8: 160 Threads of Media Serving"
date: 2026-02-28T17:00:00-06:00
draft: false
tags: ["power8", "ppc64le", "jellyfin", "dotnet", "homelab", "fedora", "ibm", "alternative-computing"]
categories: ["Alternative Computing"]
series: ["PowerPC Saga"]
description: "Running Jellyfin 10.11 on an IBM POWER8 server with .NET 10 — a tale of 9 build attempts, zero Dart VMs, and one very stubborn lobster."
---

*This is Part 2 of the [POWER8 Saga](#the-power8-saga). Part 1: [What Microsoft Won't Ship: .NET on POWER8](/posts/dotnet-power8-what-microsoft-wont-ship/)*

---

## The Machine

Before we begin — let me reintroduce the star of this show.

The **IBM S822 (8335-GCA)** is a 2U rackmount server that IBM launched in 2014 for enterprise workloads. It was designed to run DB2, WebSphere, and SAP HANA. It cost somewhere north of $30,000 new. I bought it used for the price of a nice dinner.

**The specs that matter:**
- Dual POWER8 processors, 20 cores total
- 160 hardware threads (SMT8 — each core runs 8 simultaneous threads)
- 128 GB ECC DDR3 RAM
- OPAL/Petitboot firmware (open-source, Linux-native)
- PCIe Gen3 slots for days

This thing idles at 400W and sounds like a small aircraft. I love it.

If you're wondering why someone would run a media server on enterprise POWER hardware — you're asking the wrong question. The right question is: **why wouldn't you?**

## Previously, on "Things That Shouldn't Work But Do"

[Last time](/posts/dotnet-power8-what-microsoft-wont-ship/), I compiled the entire .NET 8 SDK from source on this POWER8 — from Fedora bootstrap artifacts, through 7 patches, across 26 repos, over 6 hours of wall-clock compilation. The [full build guide](https://github.com/felipedbene/dotnet-power8) is on GitHub.

The natural next question: **what do you actually run on it?**

## From Gentoo to Fedora: The Great Migration

The POWER8 got an OS upgrade since Part 1:

| Before | After |
|--------|-------|
| Gentoo Linux | Fedora 43 Server |
| .NET 8 (hand-compiled, 6 hours) | .NET 10 (`dnf install`, 3 seconds) |
| Everything compiled from source | Packages, like a normal person |
| Suffering | Slightly less suffering |

The Fedora installation deserves its own war story. POWER8's OPAL firmware uses [Petitboot](https://github.com/open-power/petitboot) as its bootloader — a Linux-based boot environment that's fantastic until you need to install an OS that doesn't quite expect it.

**Five installation attempts:**

1. **kexec from Gentoo** — kernel loaded, dracut hung. No `inst.repo` since the ISO dies when kexec kills the host.
2. **BMC virtual media** — USB 1.0 speeds over IPMI. Loading a 2.9GB ISO at ~1MB/s. Kernel panic during boot.
3. **kexec + HTTP repo** — Anaconda text mode loaded! Configured storage! ... and silently failed because **OPAL firmware doesn't support XFS on /boot**. POWER requires a PReP boot partition + ext4 /boot. Fedora defaults to XFS. Nobody tells you this.
4. **kexec + VNC mode** — Same XFS error, three times, with increasing frustration.
5. **kexec + kickstart via SSH** — The one that worked. SSH'd into the running installer, downloaded a [kickstart file](https://github.com/felipedbene/jellyfin-power8), fixed the bootloader location (`--location=mbr`, not `--location=partition`), set `LANG=en_US.UTF-8` (Anaconda 43 crashes without it — Python 3.14 regression?), and ran `anaconda --kickstart /tmp/ks.cfg`. Then manually ran `grub2-install --no-nvram /dev/nvme0n1p1` from a chroot because the installer crashed before writing the bootloader.

**Lesson learned**: POWER8 partition layout must be: PReP (8MB) + /boot (ext4) + swap + root. No XFS on /boot. No exceptions.

The important payoff: **Fedora 43 ships .NET 10.0.103 for ppc64le out of the box.** What took me 6 hours to compile from source on Gentoo takes `dnf install dotnet-sdk-10.0` and a coffee break on Fedora. Red Hat does what Microsoft won't.

## Building Jellyfin: A Comedy in Nine Acts

Jellyfin's official Docker image supports `amd64` and `arm64`. Not `ppc64le`. So we build from source using the official [jellyfin-packaging](https://github.com/jellyfin/jellyfin-packaging) repo. How hard could it be?

**Nine attempts hard.**

### Act 1-3: The Sass Saga

Jellyfin's web UI uses webpack with `sass-embedded` for stylesheet compilation. `sass-embedded` bundles a native Dart VM binary. **Dart doesn't support ppc64le.** There's no Dart VM, no Dart compiler, no Dart anything for POWER. Compiling Dart from source is a chicken-and-egg problem (you need a Dart SDK to build a Dart SDK).

```
Error: Embedded Dart Sass couldn't find the embedded compiler executable.
Please make sure the optional dependency sass-embedded-linux-ppc64 is installed.
```

The fix: rip out `sass-embedded` and replace it with the pure JavaScript `sass` compiler:

```dockerfile
RUN npm ci --no-audit --unsafe-perm && \
    rm -rf node_modules/sass-embedded node_modules/sass-embedded-* && \
    npm install sass@^1.77.0 --save-dev && \
    sed -i "s/'sass-loader'/{loader: 'sass-loader', options: \
      {implementation: require('sass')}}/" webpack.common.js && \
    npm run build:production
```

Three attempts to get this right — `npm uninstall` doesn't actually remove it from the lockfile, and the `sass-loader` has a resolution order that prefers `sass-embedded` over `sass`. The `rm -rf` + webpack config patch was the nuclear option that worked.

Webpack then proceeded to chew through the entire Jellyfin web UI on POWER8 in **6.5 minutes, peaking at 10.5 GB of RAM and 560% CPU**. A single webpack process was consuming more resources than most people's entire home servers. JavaScript was not designed for this architecture, but 160 threads of POWER8 don't care about your design intentions.

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

`latestMinor` means "any 9.x SDK is fine." It does **not** mean "10.x is fine too." Fedora 43 only ships .NET 10. Same energy as [Part 1's SDK version battles](/posts/dotnet-power8-what-microsoft-wont-ship/).

```dockerfile
RUN sed -i 's/"latestMinor"/"latestMajor"/' global.json
```

Then `dotnet publish` tried to download `Microsoft.NETCore.App.Host.linux-ppc64le` from NuGet. Which doesn't exist. Because Microsoft doesn't publish POWER8 artifacts. **The familiar refrain of this entire series.**

```dockerfile
RUN dotnet publish Jellyfin.Server \
    --self-contained false \
    -p:UseAppHost=false
```

`UseAppHost=false` tells the build to skip the native host binary and use `dotnet jellyfin.dll` as the entrypoint instead. [Same trick from Part 1](https://github.com/felipedbene/dotnet-power8#patches-applied) — we know this dance by now.

### Act 6-8: SkiaSharp, My Old Nemesis

With the build succeeding, the container started... and immediately crashed:

```
System.DllNotFoundException: libSkiaSharp
```

SkiaSharp wraps Google's Skia graphics library for .NET. It ships native binaries for x64, arm64, and a few others. Not ppc64le. We hit [this same issue in Part 1](/posts/dotnet-power8-what-microsoft-wont-ship/) when building Jellyfin 10.9 on Gentoo. Old nemesis, new round.

The twist: Jellyfin's `CoreAppHost.cs` has an `IsNativeLibAvailable()` check — but the SkiaSharp **static constructor** throws before that check can run. Three sub-attempts:

**Attempt 1**: Delete the SkiaSharp DLLs → `FileNotFoundException` because Jellyfin.Server still references the managed assembly at load time.

**Attempt 2**: Create a stub `libSkiaSharp.so` with `gcc -shared -o libSkiaSharp.so stub.c` → crashes because SkiaSharp validates the native library version: *"Supported versions of the native libSkiaSharp library are in the range [116.0, 117.0)."*

**Attempt 3**: Surgery. Remove all references at build time:

```dockerfile
RUN sed -i '/Jellyfin.Drawing.Skia/d' \
      Jellyfin.Server/Jellyfin.Server.csproj && \
    sed -i '/using Jellyfin.Drawing.Skia/d' \
      Jellyfin.Server/CoreAppHost.cs && \
    sed -i 's/bool useSkiaEncoder = SkiaEncoder.IsNativeLibAvailable();/\
      bool useSkiaEncoder = false;/' \
      Jellyfin.Server/CoreAppHost.cs && \
    sed -i 's/? typeof(SkiaEncoder)/\
      ? typeof(NullImageEncoder)/' \
      Jellyfin.Server/CoreAppHost.cs
```

Remove the project reference, remove the using directive, hardcode `useSkiaEncoder = false`, and replace the type reference. Jellyfin has a `NullImageEncoder` fallback — you lose thumbnail generation, but playback is completely unaffected.

### Act 9: It Lives

```
[INF] Core startup complete
[INF] Main: Startup complete 0:00:19.4247792
```

19 seconds from cold start to serving media. On POWER8. Running .NET 10. Inside a Podman container. From source. With NFS media.

```bash
$ curl http://localhost:8096/health
Healthy
```

## The Final Architecture

```
┌─────────────────────────────────────────────────┐
│  IBM S822 (POWER8) — Fedora 43                  │
│  160 threads / 128GB RAM                        │
│                                                 │
│  ┌─────────────────────────────────────────┐    │
│  │  Podman Container                       │    │
│  │  jellyfin-power8:latest                 │    │
│  │                                         │    │
│  │  .NET 10 Runtime (Fedora native)        │    │
│  │  Jellyfin 10.11.0 Server               │    │
│  │  Jellyfin 10.11.0 Web UI               │    │
│  │  ffmpeg-free                            │    │
│  │  NullImageEncoder (no SkiaSharp)        │    │
│  └────────────┬────────────────────────────┘    │
│               │ :8096                           │
│  ┌────────────┴────────────────────────────┐    │
│  │  NFS Mount: /mnt/media                  │    │
│  │  22TB ZFS pool on xeon2socket           │    │
│  │  movies / tv / kids / music / family    │    │
│  └─────────────────────────────────────────┘    │
│                                                 │
│  Also running:                                  │
│  • Prometheus node_exporter (Puppet-managed)    │
│  • .NET 8 SDK (hand-compiled, Part 1)           │
│  • Gentoo on sda (for science)                  │
└─────────────────────────────────────────────────┘
```

Three-stage Dockerfile, all Fedora 43 native:

1. **Web stage**: Node.js 22 + webpack with pure JS sass (no Dart)
2. **Server stage**: .NET 10 SDK + patched source (no Skia, latestMajor rollforward, no apphost)
3. **Runtime stage**: ASP.NET 10 runtime + ffmpeg-free

The whole thing builds in about 20 minutes on POWER8, with webpack being the bottleneck.

Published on GHCR — anyone with a ppc64le machine can pull and run:

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

- **Media playback**: Everything. Movies, TV shows, music, home videos. 22TB library across 5 collections.
- **Transcoding**: CPU-only, but 160 threads of POWER8 make software transcoding viable. FFmpeg detects VDPAU, VAAPI, and Vulkan capabilities even without a GPU — future-proofing for when someone inevitably plugs a GPU into this thing.
- **Web UI**: Full Jellyfin interface, all features minus image processing.
- **Client compatibility**: Fire TV, Android, iOS, web — all work. The 10.11 server version is compatible with current clients (unlike the 10.9.11 I had before on Gentoo, which Fire TV rejected).
- **Puppet integration**: The POWER8 is managed by Foreman/Puppet, with node_exporter feeding metrics to Prometheus/Grafana. Infrastructure as code, even for the weird machines.

## What Doesn't

- **Thumbnails/image processing**: No SkiaSharp = no server-side image manipulation. Clients handle their own image scaling. Not a dealbreaker.
- **Hardware transcoding**: POWER8 has no GPU. Software transcoding only. 160 threads compensate.

## The Pattern: Alternative Architecture Survival Guide

After two rounds of building .NET applications on POWER8, a pattern emerges. This applies to ppc64le, s390x, RISC-V, or any architecture that's not x86/ARM:

| Problem | Pattern | Example |
|---------|---------|---------|
| Native binary doesn't exist | Use managed/interpreted fallback | sass-embedded → sass (JS) |
| Static constructor crashes | Remove dependency, use null implementation | SkiaSharp → NullImageEncoder |
| NuGet package missing | Disable the feature that needs it | apphost → `UseAppHost=false` |
| SDK version pinned | Relax rollforward policy | `latestMinor` → `latestMajor` |
| Base image unavailable | Use distro that supports your arch | Alpine/Debian → Fedora |
| Installer doesn't work | SSH in and do it manually | Anaconda text mode → kickstart via SSH |

**The software ecosystem is the bottleneck, not the hardware.** POWER8 can run anything that compiles for ppc64le. The problems are always binary dependencies that only ship for x86 and ARM.

## The PowerPC Saga

From a $67 iBook to a $30,000 enterprise server — one architecture, four stories:

1. 🍎 **[Resurrecting My iBook G4](/posts/resurrecting-my-ibook-g4/)** — A 20-year dream fulfilled. SSD upgrades, broken power buttons, and a Kubernetes-powered compile farm for a 1.33GHz laptop.
2. 🖥️ **[Cloud Architect Meets PowerPC: The $50 Time Machine](/posts/cloud-architect-meets-powerpc/)** — A PowerMac G5 from a guy named Ray. Big Endian, Open Firmware, and discovering the forgotten world of pre-Intel Apple.
3. ⚡ **[What Microsoft Won't Ship: .NET on POWER8](/posts/dotnet-power8-what-microsoft-wont-ship/)** — Compiling the .NET 8 SDK entirely from source. 6 hours, 7 patches, 26 repos, 160 threads of SMT8.
4. 🎬 **Jellyfin on POWER8: 160 Threads of Media Serving** ← You are here. 9 build attempts, containerized Jellyfin 10.11 with .NET 10 on Fedora 43.
5. 🔜 **Coming soon**: Transcoding benchmarks — 160 POWER8 threads vs x86. The CPU-only showdown.

## Source

- **Jellyfin container**: [github.com/felipedbene/jellyfin-power8](https://github.com/felipedbene/jellyfin-power8)
- **Container image**: [ghcr.io/felipedbene/jellyfin-power8](https://ghcr.io/felipedbene/jellyfin-power8)
- **.NET 8 build guide**: [github.com/felipedbene/dotnet-power8](https://github.com/felipedbene/dotnet-power8)
- **Part 1**: [What Microsoft Won't Ship](/posts/dotnet-power8-what-microsoft-wont-ship/)

---

*Built on an IBM S822 (8335-GCA) — Dual POWER8, 160 threads, 128GB RAM, Fedora 43. The machine that refuses to retire, and the human who refuses to let it.*
