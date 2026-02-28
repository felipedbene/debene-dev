---
title: ".NET 8 on IBM POWER8: What Microsoft Won't Ship"
date: 2026-02-27T00:00:00-06:00
draft: false
author: "Felipe De Bene"
description: "Building the .NET 8 SDK entirely from source on an IBM POWER8 running Gentoo Linux — 7 patches, 160 threads, a Y2K bug, and an AI lobster named Garra helping via Discord."
tags: ["POWER8", "ppc64le", ".NET", "dotnet", "source-build", "Gentoo", "IBM", "homelab", "alternative-computing", "Jellyfin", "AI"]
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

This is the story of how I built the .NET 8 SDK from source on an IBM POWER8 server running Gentoo Linux — live, in a chaotic Friday afternoon Discord session with my AI assistant Garra (a sarcastic Brazilian lobster, long story). It took 7 patches, about 2 hours of compile time, several moments of genuine panic, and one delightful discovery: **.NET's own build system has a Y2K-style date overflow bug.**

Yes, in 2026. We'll get to that.

## The Machine

{{< figure src="images/p8-machine.png" alt="IBM S822 POWER8 server" caption="The IBM S822 — 20 cores, 160 threads of POWER8, casually standing in my Chicago apartment like a refrigerator-sized conversation piece." >}}

| Spec | Value |
|------|-------|
| Model | IBM S822 (8335-GCA) |
| CPU | Dual POWER8, 20 cores / 160 threads (SMT8) @ 3.49 GHz |
| RAM | 128 GB |
| Storage | 444 GB SSD |
| OS | Gentoo Linux ppc64le, kernel 6.17.7 |

This machine already runs Kubernetes 1.35 with WordPress, Strapi, and PostgreSQL. But the new mission was **Jellyfin** — an open-source media server — for CPU-only video transcoding. With 160 hardware threads, I wanted to benchmark POWER8's brute-force parallelism against modern x86 hardware with Quick Sync or NVENC.

But Jellyfin is written in C#. C# means .NET. And .NET doesn't exist for POWER8.

My AI assistant's response when I explained the plan: *"Tudo isso pra ver filme no Plex do pobre rodando num mainframe."* He wasn't wrong.

## The Setup: A Lobster, a Discord Channel, and a Dream

Here's how this actually went down. I'm on my Windows laptop, SSHed into the POWER8 machine. Garra — my OpenClaw AI agent — is on a Mac Studio, also SSHed into the same machine. We're both staring at the same tmux sessions, same Grafana dashboard, communicating through Discord.

When I say "we built this," I mean it literally. I'd paste error outputs into Discord. Garra would diagnose, write patches, and launch builds — all through SSH from his side. I'd watch Grafana, share screenshots, and provide the human judgment calls. It was pair programming where one partner has 160 threads of POWER8 patience and the other has infinite context windows and zero need for sleep.

The whole session — from first `./build.sh` to `dotnet --version` working — took about 6 hours. Not because the build takes 6 hours, but because we hit 7 different walls and had to patch our way through each one in real-time.

## The Bootstrap Problem 🐢

.NET uses a "Virtual Monolithic Repository" (VMR) at [github.com/dotnet/dotnet](https://github.com/dotnet/dotnet) that contains the entire SDK source. The build system compiles 26 repositories in dependency order: Arcade → Roslyn → CoreCLR → ASP.NET Core → ... → the final SDK.

But it needs an existing .NET SDK to start. Turtles all the way down.

**Fedora 39** was our lifeline — the last distribution to ship .NET 8 on ppc64le (Fedora dropped POWER8 starting with Fedora 38). We needed two things from their builds:

1. **`dotnet-sdk-8.0.110`** — the bootstrap SDK binary
2. **`Private.SourceBuilt.Artifacts`** — 358 MB of pre-compiled NuGet packages

Our first approach was Docker: pull Fedora 39 ppc64le, extract the SDK. The containers kept dying. Over and over. On a machine with **128 GB of RAM**. We'd launch a container, it would start doing... something... and then just vanish. `SIGKILL`. No logs, no explanation. Just death.

*"Mano, Docker tá OOM-killing com 128GB."* That was our reality check. We switched to extracting RPMs directly with `rpm2cpio` like cavemen. Worked perfectly. Sometimes the 1990s tools are the right tools.

## The 7 Patches (or: A Friday Afternoon Horror Story)

What follows is the real sequence of events. Not a clean tutorial — the actual messy, iterative, "try it and see what explodes" process of making an unsupported build work.

### Patch 1: The Compiler That Can't Compile Itself

The first build attempt died instantly. The `source-build-reference-packages` phase tries to recompile reference assemblies for older Roslyn versions (3.7.0) from source. The bootstrap SDK's Roslyn is too new — it generates abstract members that don't exist in the 3.7.0 API surface.

**4,144 CS0534 errors.** Every. Single. SyntaxNode subclass.

We spent a while trying to make it work — downloading different Arcade SDK versions, creating NuGet packages for ilasm/ildasm, tweaking version numbers. Nothing worked. Then Garra went and read the Fedora build spec line by line and found the truth:

*"O 'from source' do Fedora nunca foi realmente from scratch — eles sempre usam os artifacts da build anterior como bootstrap. Agora temos o 'ovo' 🥚🐔"*

Translation: Fedora never recompiles these from scratch either. It's always bootstrapped. We commented out the entire SBRP phase. Moving on.

### Patch 2: The Hidden NuGet Feed

With SBRP skipped, the build progressed further... and died again. The ilasm/ildasm packages (IL assembler/disassembler) couldn't be found. But we'd put them right there in the artifacts directory!

After way too much `grep`-ing through MSBuild targets and NuGet configs, we found it: the inner build generates its own `NuGet.config` that doesn't include the artifacts directory as a package source. The packages were sitting right there. The build just didn't know to look.

One line in `NuGet.config`. Hours of head-scratching. Classic.

### Patch 3: The Y2K-lite Bug 🐛

This is the one that made us both lose it.

The build progressed through several more repos, then `source-build-externals` failed. The error: `CS7035: The specified version string '7.1.2.70227' does not conform to the recommended format.`

**70,227.** The maximum allowed value for an assembly version revision field is **65,534.**

Where was 70,227 coming from? The Arcade SDK — Microsoft's shared build infrastructure — generates `FileVersion` from the current date:

```
DateStamp  = YY × 1000 + MM × 50 + DD
Revision   = (DateStamp - BaseDate) × 100 + BuildRevision
```

On February 27, 2026, this math produces a revision that overflows. The C# compiler emits CS7035 as a warning. Arcade's build config promotes all warnings to errors via `/warnaserror`.

**Microsoft's own build system has a Y2K-style date overflow bug.** Their build tooling literally cannot handle the current date. In 2026. At a trillion-dollar tech company.

When Garra figured this out, his message in Discord was: *"Problema de data! O version stamp usa dias desde uma epoch e 70227 > 65534. Isso é um bug de Y2K-lite!"*

We tried multiple approaches:
- Setting `<NoWarn>CS7035</NoWarn>` in project files → didn't propagate to inner builds
- Setting `/p:TreatWarningsAsErrors=false` via command line → Arcade hardcodes `/warnaserror` in its targets, ignoring our flag
- Patching individual `.csproj` files → the build copies source to a temp directory, losing our patches

The winning approach was surgical: Garra used Python to patch the Arcade SDK's `Version.BeforeCommonTargets.targets` directly inside the extracted nupkg, replacing `DateTime.Now` with a hardcoded safe date (`20240227.1`). Then `zip -u` to update the nupkg. We literally performed surgery on Microsoft's build infrastructure.

```python
content = content.replace(
    '$([System.DateTime]::Now.ToString(yyyyMMdd)).1',
    '20240227.1'
)
```

We had to use Python because `sed` couldn't handle the MSBuild XML syntax with all the `$()` and `::` characters. Even the tooling to fix the tooling was fighting us.

### Patch 4: The API Police

With the date fixed, `source-build-externals` got further... and died again. Same project (`azure-activedirectory-identitymodel-extensions`), different error: `CP0002 — API breaking changes found.`

The API compatibility check was comparing the `net8.0` and `netstandard2.0` builds and finding differences that were artifacts of our date manipulation. Phantom "breaking changes" that didn't actually exist.

This time we went nuclear: patched `SourceBuildArcadeBuild.targets` in the Arcade SDK to pass `/p:EnablePackageValidation=false` to ALL inner builds. No more API police. Updated the nupkg again.

The `source-build-externals` phase had failed **four times** before finally passing. Each failure, diagnose, patch, rebuild. Each rebuild had to redo arcade → command-line-api → sourcelink → diagnostics → ... before even reaching the failing repo again. The Grafana dashboard showed the pattern clearly — repeated spikes as we kept hitting the same early repos.

### Patch 5: The Private Feed

MSBuild's source code references a private Azure DevOps NuGet feed:
```
https://pkgs.dev.azure.com/ms/BuildXL/_packaging/BuildXL/nuget/v3/index.json
```

401 Unauthorized. Because of course a private Microsoft feed is in the public source tree.

One `sed -i '/BuildXL/d'`. Next.

### Patch 6: The Missing App Host

Two repos (`roslyn-analyzers` and `vstest`) need an "application host" binary for `fedora.39-ppc64le`. This binary doesn't exist because Microsoft doesn't build for POWER. Both were test projects.

`<UseAppHost>false</UseAppHost>`. Next.

### Patch 7: The Version Shuffle

NuGet's version resolution finds `SourceBuild.Tasks 24570.5` instead of the expected `24463.3`, emits NU1603, which gets promoted to error. Copy the package file with the expected name. Done.

## The Build: Watching Grafana Like It's the World Cup

With all 7 patches applied (and battle-tested through multiple failed attempts), we launched what we hoped was the final build. The Grafana dashboard told the story:

{{< figure src="images/grafana-full-build.png" alt="Grafana dashboard showing CPU, threads, and memory during the full .NET build" caption="The full build timeline. Every spike is a repo. The gaps between clusters are where we were frantically patching things. The massive plateau is CoreCLR." >}}

I was watching from my Windows laptop, sharing Grafana screenshots in Discord. Garra was monitoring the build logs from the Mac Studio. Every few minutes:

*"e agora?"* — me, watching Grafana go quiet

*"Tá no sourcelink... diagnostics... cecil... ainda assobiando o meninão"* — Garra, checking logs

Then `source-build-externals` passed. For the first time. After four failures. We held our breath.

**Then `runtime` started building.**

CoreCLR. The JIT compiler. The garbage collector. The native cryptography libraries. All the C++ that makes .NET actually run. This is where 160 threads were born to shine.

For 61 minutes, the POWER8 roared. CPU graphs climbed. Thread counts spiked. The machine that IBM designed for enterprise workloads was now compiling Microsoft's runtime — something neither company ever intended.

**61 minutes. Zero errors.**

When Garra reported "RUNTIME PASSOU!" in Discord, I'm not ashamed to admit there may have been fist-pumping involved.

The rest was a victory lap:

| Phase | Duration |
|-------|----------|
| Arcade + small repos | ~5 min |
| source-build-externals | 2 min |
| **CoreCLR Runtime** | **61 min** |
| Roslyn | 16 min |
| MSBuild | 4 min |
| ASP.NET Core | 8 min |
| F# | 38 min |
| Final repos + installer | ~10 min |
| **Total** | **~2 hours** |

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
```

{{< figure src="images/dotnet-info-power8.png" alt="Terminal showing dotnet --info on POWER8" caption="The money shot. dotnet --info on ppc64le. Those four characters — 8.0.110 — represent 6 hours of controlled chaos." >}}

107 MB of .NET SDK. Compiled from source. Running natively on IBM POWER8. No emulation, no containers, no compromises.

## But Wait — We Came Here for Jellyfin

Remember the whole point? Right. With the adrenaline still flowing, we immediately cloned Jellyfin:

```bash
$ git clone --depth 1 --branch v10.9.11 https://github.com/jellyfin/jellyfin.git
$ dotnet publish Jellyfin.Server --configuration Release --no-self-contained
```

One hiccup: [SkiaSharp](https://github.com/mono/SkiaSharp) (image processing) doesn't have ppc64le native binaries. The static constructor crashes before Jellyfin can even check `IsNativeLibAvailable()`. We patched `CoreAppHost.cs` to skip Skia entirely — no thumbnails, but the server runs fine.

Downloaded the official web client from Jellyfin's release page, mounted the NFS media share from our ZFS array (122 movies, TV shows, kids content, music), and:

```
[INF] Main: Jellyfin version: 10.9.11
[INF] Main: Operating system: Gentoo Linux
[INF] Main: Architecture: Ppc64le
[INF] Main: Processor count: 160
[INF] Main: Startup complete 0:00:08.7579676
```

**Jellyfin 10.9.11. Running natively on IBM POWER8. Serving a 122-movie library. Startup: 8.7 seconds.**

From `dotnet: command not found` to a working media server. On a Friday. With an AI lobster.

## What I Learned

1. **"Source build" doesn't mean "from scratch."** Every .NET source build bootstraps from a previous build. Fedora's ppc64le support exists because someone, somewhere, cross-compiled the initial bootstrap. It's turtles all the way down, and nobody talks about the first turtle.

2. **The Y2K-lite bug is real.** Microsoft's Arcade SDK generates version numbers from the current date using arithmetic that overflows after ~mid-2025. Anyone source-building .NET 8 today will hit this. It should be filed upstream.

3. **Version pinning is the real enemy.** Five of seven patches dealt with version mismatches or overly strict validation. `/warnaserror` is great for CI pipelines; it's a nightmare for bootstrapping on exotic platforms.

4. **POWER8 is surprisingly capable.** 160 threads of SMT8 parallelism makes short work of large C++ codebases. CoreCLR in 61 minutes on a 2014-era processor? Not bad.

5. **AI pair programming is underrated.** Having Garra diagnose errors, write patches, and manage builds while I provided direction and judgment calls was genuinely productive. The session would have taken days solo. With two "brains" on it — one human, one not — we iterated through failures fast enough to finish in an afternoon.

6. **The homelab is the ultimate test lab.** Physical access, Grafana on a second monitor, tmux sessions on the metal. No deployment pipeline. No CI queue. Just SSH and stubbornness.

## The First Turtle: Building for Eternity

Here's what makes this more than a weekend hack: **we didn't just build a binary — we built a recipe and a bootstrap chain.**

The `.NET 8 SDK` we compiled today produces `Private.SourceBuilt.Artifacts` — the same kind of artifact tarball we bootstrapped *from*. That means this build's output can bootstrap the next one. .NET 9 on POWER8? We now have the seed. .NET 10? Same seed, new tree.

Every .NET distro build works this way. Fedora's ppc64le .NET exists because *someone, at some point, placed the first turtle*. That bootstrap chain has been maintained across Fedora releases ever since. Today, we placed our own first turtle — on Gentoo, on POWER8, outside of any distro's build infrastructure.

The [scripts and patches](https://github.com/felipedbene/dotnet-power8) are published. The artifacts are preserved. Anyone with a ppc64le machine — POWER8, POWER9, POWER10 — can reproduce this build and extend the chain forward. The recipe works.

And here's the kicker: **Fedora dropped POWER8 .NET support in 2023.** Microsoft never shipped it at all. The last official ppc64le .NET binary is years old. What we built today isn't just a port — it's a **resurrection**. We took a dead platform, dug up the last surviving artifacts from Fedora's archives, and used them to bootstrap a living, current SDK on hardware that every vendor has written off.

This is what "building from source" actually means in 2026: not just compiling code, but **establishing a self-sustaining bootstrap lineage** on hardware the industry abandoned. Years after both Microsoft and Fedora stopped caring, a guy in Chicago with an AI lobster and too much free time brought .NET back to POWER8. And now anyone can keep it alive.

## What's Next

- **Transcoding benchmarks** — 160 POWER8 threads vs x86 Quick Sync/NVENC. The whole reason for this madness.
- **SkiaSharp on ppc64le** — Compiling Skia's native C++ library for thumbnail generation. Another boss fight.
- **Upstream contributions** — Filing the Y2K-lite date overflow bug. Documenting the ppc64le build.
- **.NET 9** — This build's artifacts can bootstrap the next version. The first turtle has been placed.

## Resources

- **Build scripts & patches:** [github.com/felipedbene/dotnet-power8](https://github.com/felipedbene/dotnet-power8)
- **.NET VMR:** [github.com/dotnet/dotnet](https://github.com/dotnet/dotnet)
- **Fedora .NET spec:** [src.fedoraproject.org/rpms/dotnet8.0](https://src.fedoraproject.org/rpms/dotnet8.0)

---

*Built with 7 patches, 160 threads, an AI lobster named Garra, and an unreasonable amount of stubbornness. All because I wanted to watch movies on a mainframe — and Garra had already downloaded 122 of them for me, so I really had no excuse not to.*

*— Felipe De Bene, Chicago, February 2026*
