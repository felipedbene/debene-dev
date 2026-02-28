---
title: ".NET 8 on IBM POWER8: What Microsoft Won't Ship"
date: 2026-02-27T00:00:00-06:00
draft: true
author: "Felipe De Bene"
description: "Building the .NET 8 SDK entirely from source on an IBM POWER8 running Gentoo Linux — 7 patches, 160 threads, and a date overflow bug that proves even Microsoft can't escape Y2K."
tags: ["POWER8", "ppc64le", ".NET", "dotnet", "source-build", "Gentoo", "IBM", "homelab", "alternative-computing", "Jellyfin"]
categories: ["Hardware", "Software Engineering"]
cover:
    image: "images/p8-machine.png"
    alt: "IBM S822 POWER8 server standing in a Chicago apartment"
    relative: true
ShowToc: true
TocOpen: false
---

## You Need a .NET SDK to Build a .NET SDK

That's the first thing you learn when you try to compile .NET from source. It's a beautifully circular problem — like needing a car to drive to the dealership where you're buying your first car.

Microsoft ships pre-built SDKs for x86_64, ARM64, and s390x. But POWER? IBM's legendary architecture that runs half the world's banking systems? Sorry — you're on your own.

This is the story of how I built the .NET 8 SDK from source on an IBM POWER8 server running Gentoo Linux — and then compiled and ran Jellyfin on it. It took 7 patches, about 2 hours of compile time, and one delightful discovery: **.NET's own build system can't handle dates after 2025.**

## The Machine

{{< figure src="images/p8-machine.png" alt="IBM S822 POWER8 server" caption="The IBM S822 — 20 cores, 160 threads of POWER8 goodness, casually standing in my Chicago apartment." >}}

| Spec | Value |
|------|-------|
| Model | IBM S822 (8335-GCA) |
| CPU | Dual POWER8, 20 cores / 160 threads (SMT8) @ 3.49 GHz |
| RAM | 128 GB |
| Storage | 444 GB SSD |
| OS | Gentoo Linux ppc64le, kernel 6.17.7 |

I bought this machine to run Kubernetes workloads — yes, really, K8s 1.35 runs great on POWER8. It already hosts WordPress, Strapi, and PostgreSQL in containers. The next project: **Jellyfin for CPU-only video transcoding**. With 160 threads, I wanted to see how POWER8's brute-force parallelism compares to modern x86 hardware with Quick Sync or NVENC.

But Jellyfin is written in C#. And C# means .NET. And .NET doesn't exist for POWER8.

*Challenge accepted.*

## The Bootstrap Problem 🐢

.NET uses a "Virtual Monolithic Repository" (VMR) at [github.com/dotnet/dotnet](https://github.com/dotnet/dotnet) that contains the entire SDK source. The build system is sophisticated — it compiles 26 repositories in dependency order: Arcade → Roslyn → CoreCLR → ASP.NET Core → ... → the final SDK installer.

But it needs an existing .NET SDK to start. It's turtles all the way down.

**Fedora 39** was my lifeline. It was the last distribution to ship .NET 8 on ppc64le — Fedora dropped POWER8 support starting with Fedora 38, keeping only POWER9 and newer. From their build infrastructure, I extracted two critical pieces:

1. **`dotnet-sdk-8.0.110`** — the bootstrap SDK binary (extracted from RPM)
2. **`Private.SourceBuilt.Artifacts`** — 358 MB of pre-compiled NuGet packages from Fedora's previous build

Both extracted directly from Fedora RPMs using `rpm2cpio`. My early Docker-based approach kept getting OOM-killed on the 128 GB machine, which tells you something about .NET's build appetite. Sometimes the old tools work best.

## The 7 Patches

What follows is the part Microsoft doesn't document: their "build from source" process isn't quite as portable as advertised. Each patch addresses a real incompatibility, and each one cost me anywhere from 10 minutes to 2 hours of debugging.

### Patch 1: The Compiler That Can't Compile Itself

The first build phase, `source-build-reference-packages` (SBRP), tries to recompile reference assemblies for older Roslyn versions from source. The problem? The bootstrap SDK's Roslyn (8.0.110) generates code with abstract members (`GetNodeSlot`, `GetCachedSlot`) that don't exist in the Roslyn 3.7.0 API surface.

**4,144 CS0534 errors.** Every single `SyntaxNode` subclass "does not implement inherited abstract member `GetNodeSlot(int)`."

The fix was surgical: skip the entire SBRP phase. The pre-existing 388 reference packages bundled in the VMR are sufficient. After studying Fedora's build spec, I confirmed they do the same thing — they never recompile these from scratch either. It's always been bootstrapped from previous builds.

```xml
<!-- repo-projects/dotnet.proj -->
<!-- SKIPPED: bootstrap SDK incompatible with SBRP recompilation -->
<!-- <RepositoryReference Include="source-build-reference-packages" /> -->
```

### Patch 2: The Hidden NuGet Feed

The inner build's `NuGet.config` doesn't include the Fedora artifacts directory as a package source. So when the build needs `ilasm` and `ildasm` — the IL assembler and disassembler, critical tools for building reference assemblies — NuGet can't find them.

The packages were sitting right there in `/tmp/sba-customize/`. The build just didn't know to look there.

One line added to `NuGet.config`. Hours of head-scratching.

### Patch 3: The Y2K-lite Bug 🐛

This one deserves its own section because it's genuinely funny.

The Arcade SDK — Microsoft's shared build infrastructure used by all .NET repos — generates assembly `FileVersion` numbers using a formula based on the current date:

```
DateStamp  = YY × 1000 + MM × 50 + DD
Revision   = (DateStamp - VersionBaseShortDate) × 100 + BuildRevision
```

On February 27, 2026, this produces a revision of **70,227**. The maximum allowed value for an assembly version field? **65,534**.

The C# compiler dutifully emits warning CS7035: *"The specified version string does not conform to the recommended format."* And Arcade's build configuration promotes all warnings to errors via `/warnaserror`.

**Microsoft's own build system cannot handle dates after approximately mid-2025.** The irony of a tech company shipping a date-dependent build system that breaks within two years is... chef's kiss.

I fixed it by hardcoding the build date to `20240227.1` in the Arcade SDK's `Version.BeforeCommonTargets.targets`. It's ugly, but it works. This bug affects anyone trying to source-build .NET 8 today.

### Patch 4: The API Police

The `azure-activedirectory-identitymodel-extensions` package (inside `source-build-externals`) runs API compatibility validation between its `net8.0` and `netstandard2.0` targets. With our date-shifted build from Patch 3, the validation catches phantom "breaking changes" that don't actually exist — they're artifacts of the version number manipulation.

I disabled `EnablePackageValidation` globally by patching the Arcade SDK's `SourceBuildArcadeBuild.targets` to pass `/p:EnablePackageValidation=false` to all inner builds. Sorry, API police.

### Patch 5: The Private Feed

MSBuild's source code references a private Azure DevOps NuGet feed (`BuildXL`) that requires authentication:

```
https://pkgs.dev.azure.com/ms/BuildXL/_packaging/BuildXL/nuget/v3/index.json
```

Presumably fine for Microsoft engineers. Returns 401 Unauthorized for the rest of us.

One `sed` command to remove the feed from `NuGet.config`. Moving on.

### Patch 6: The Missing App Host

Two repos — `roslyn-analyzers` and `vstest` — include test projects that require an "application host" binary for the `fedora.39-ppc64le` runtime identifier. This binary doesn't exist because Microsoft doesn't build app hosts for POWER.

Set `<UseAppHost>false</UseAppHost>` on test projects nobody will ever run on POWER8 anyway.

### Patch 7: The Version Shuffle

The Arcade SDK (`8.0.0-beta.24463.3`) depends on `Microsoft.DotNet.SourceBuild.Tasks` version `>= 8.0.0-beta.24463.3`. The source tree ships version `24570.5`. NuGet's "approximate best match" resolution triggers warning NU1603, which — say it with me — gets promoted to an error by `/warnaserror`.

Copy the package file with the expected version number. Three seconds of work after thirty minutes of diagnosis.

## The Build

With all patches applied, I launched the build and watched my Grafana dashboard light up:

{{< figure src="images/grafana-full-build.png" alt="Grafana dashboard showing CPU, threads, and memory during the full .NET build" caption="The full build timeline — every spike tells a story. The massive plateau in the middle is CoreCLR compilation." >}}

Here's what 26 repositories compiling in sequence looks like on a POWER8:

| Phase | Duration | Notes |
|-------|----------|-------|
| Arcade | 66s | Build infrastructure |
| command-line-api | 38s | |
| sourcelink | 46s | |
| diagnostics | 25s | |
| emsdk, xliff-tasks, cecil, symreader | ~2 min | Small repos |
| source-build-externals | 2 min | After patches 3 & 4 |
| **CoreCLR Runtime** | **61 min** | **JIT compiler, GC, native crypto libs — C++ heavy** |
| Roslyn | 16 min | The C# compiler compiling itself |
| MSBuild | 4 min | |
| ASP.NET Core | 8 min | |
| razor, deployment-tools, format | ~2 min | |
| NuGet client | 3.5 min | |
| F# | 38 min | Parser generation + compiler |
| vstest, SDK, Aspire, installer | ~5 min | Final assembly |
| **Total** | **~2 hours** | |

The CoreCLR build was the highlight. Watching 160 threads max out on C++ compilation of the JIT compiler, the garbage collector, and the native cryptography libraries — on hardware Microsoft has never tested — was genuinely thrilling.

## The Result

```
$ dotnet --info
.NET SDK:
 Version:           8.0.110
 Commit:            87a66bb3d1

Runtime Environment:
 OS Name:     gentoo
 OS Version:  2.18
 OS Platform: Linux
 RID:         gentoo.2.18-ppc64le

Host:
  Version:      8.0.10
  Architecture: ppc64le

.NET SDKs installed:
  8.0.110 [/home/felipe/dotnet-sdk-power8/sdk]

.NET runtimes installed:
  Microsoft.AspNetCore.App 8.0.10
  Microsoft.NETCore.App 8.0.10
```

{{< figure src="images/dotnet-info-power8.png" alt="Terminal showing dotnet --info on POWER8" caption="dotnet --info on ppc64le. The four most satisfying characters: 8.0.110." >}}

A fully functional .NET 8 SDK — 107 MB of runtime, compiler, and tools — compiled from source, running natively on IBM POWER8. No emulation, no containers, no compromises.

## But Wait — We Came Here for Jellyfin

Remember the whole point of this exercise? With the SDK ready, building Jellyfin was almost anticlimactic:

```bash
$ git clone --depth 1 --branch v10.9.11 https://github.com/jellyfin/jellyfin.git
$ cd jellyfin
$ dotnet publish Jellyfin.Server --configuration Release --no-self-contained \
    -o /tmp/jellyfin-build /p:TreatWarningsAsErrors=false
```

One small hiccup: [SkiaSharp](https://github.com/mono/SkiaSharp) (the image processing library) doesn't ship native binaries for ppc64le. Jellyfin gracefully falls back to `NullImageEncoder` — no thumbnails, but the server runs fine. A quick patch to `CoreAppHost.cs` to avoid the static constructor crash, and:

```
[INF] Main: Jellyfin version: 10.9.11
[INF] Main: Operating system: Gentoo Linux
[INF] Main: Architecture: Ppc64le
[INF] Main: Processor count: 160
[INF] Main: Startup complete 0:00:08.7579676
```

**Jellyfin 10.9.11, running natively on IBM POWER8, serving a 122-movie library over NFS from a ZFS array.** Startup time: 8.7 seconds.

From `dotnet: command not found` to a working media server — in about 6 hours. Not bad for hardware that was never supposed to run any of this.

## What I Learned

1. **"Source build" doesn't mean "from scratch."** Every .NET source build bootstraps from a previous build's artifacts. Fedora's ppc64le support exists because someone, at some point, cross-compiled the initial bootstrap. It's turtles all the way down.

2. **Version pinning is the real enemy.** Five of the seven patches dealt with version mismatches or overly strict validation. The build system is optimized for Microsoft's CI/CD pipeline, not for outsiders bootstrapping on exotic hardware.

3. **The date bug is real and affects everyone.** Anyone trying to source-build .NET 8 after mid-2025 will hit the CS7035 overflow. This should probably be filed upstream.

4. **POWER8 is surprisingly capable.** 160 threads of SMT8 parallelism makes short work of large C++ codebases. The CoreCLR build took 61 minutes — respectable for a 2014-era processor architecture.

5. **The homelab is the ultimate test lab.** No cloud instance would have let me iterate this fast. Having physical access to the machine, a Grafana dashboard on another screen, and tmux sessions running on the metal made debugging a conversation, not a deployment pipeline.

## What's Next

- **Transcoding benchmarks** — The whole reason we're here. 160 POWER8 threads vs x86 hardware with Quick Sync/NVENC for software video transcoding.
- **SkiaSharp on ppc64le** — Compiling the Skia native library would restore thumbnail generation. Another boss fight for another day.
- **Upstream contributions** — Filing issues for the date overflow bug and documenting the ppc64le build process.
- **.NET 9 and beyond** — The patches should be similar. The bootstrap artifacts from this build can seed the next one.

## Resources

- **Build scripts & patches:** [github.com/felipedbene/dotnet-power8](https://github.com/felipedbene/dotnet-power8)
- **.NET VMR:** [github.com/dotnet/dotnet](https://github.com/dotnet/dotnet)
- **Fedora .NET spec:** [src.fedoraproject.org/rpms/dotnet8.0](https://src.fedoraproject.org/rpms/dotnet8.0)

---

*Built with 7 patches, 160 threads, and an unreasonable amount of stubbornness. All because I wanted to watch movies on a mainframe.*

*— Felipe De Bene, Chicago, February 2026*
