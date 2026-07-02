---
title: "I put Spotify on Mac OS 9 — over Gopher"
date: 2026-07-01T21:18:00-05:00
draft: false
tags: ["homelab", "kubernetes", "vintage-computing", "macos9", "gopher", "spotify", "librespot"]
categories: ["Projects"]
summary: "Streaming 2026 music on a 1999 Mac via a 1991 protocol. Because the only thing better than making old tech work is making it work wrong."
cover:
  image: "images/hero-now-playing.png"
  alt: "Mac OS 9 desktop showing Netscape Communicator displaying a Gopher menu with 'Now Playing: Budapest by George Ezra', with a media player actively streaming at 44 kHz"
  caption: "Budapest by George Ezra, streaming live on Mac OS 9 via gopher://10.0.100.112:70/1/spot/now"
---

*Or: how a text protocol from 1991 ended up driving a 2020s streaming service on an operating system that shipped on beige plastic, running inside a VM, and why the audio was the only hard part.*

---

## The dumb, wonderful idea

I have a Mac OS 9 machine. Not a museum piece behind glass — a thing I actually poke at, lately as a UTM/QEMU guest on my desk. And I have a homelab Kubernetes cluster in the other room. Somewhere between those two facts, a question formed that I could not un-ask:

**Could I control Spotify from Mac OS 9?**

Not the modern web — OS 9's browsers gave up on that two decades ago. Not the Spotify app — there will never be one. The constraint that made it fun was picking the most gloriously wrong transport imaginable and making it work anyway:

**Gopher.** [RFC 1436](https://datatracker.ietf.org/doc/html/rfc1436). 1991. The protocol the World Wide Web ran over on its way out the door. Menus of text, one item per line, a type character and a selector. No CSS, no JavaScript, no TLS, no images. A G3 can speak it in its sleep — and here's the lovely part: **Netscape Communicator**, the 1997 browser sitting right there on the OS 9 desktop, still speaks Gopher natively. You just type `gopher://…` into the location bar. (Old dedicated clients like TurboGopher work too, but why boot another app when Netscape already does it?)

So: search, playlists, play/pause, now-playing — all driven from gopher menus on a 25-year-old Mac. And it works. I clicked a track called "Construção" in a gopher menu on OS 9 and Chico Buarque came out of the speakers. I may have made a noise.

## The one trick that makes it possible

Gopher is never going to carry a 128 kbps MP3 to a 1999 Mac. So it doesn't.

The whole design is a **split between the control plane and the data plane** — a separation that turns out to map perfectly onto two apps that already exist on OS 9:

- **Netscape's Gopher browser** is the remote control. It only ever sends and receives tiny text menus. Search results, a track's detail page, a ">> Tocar agora" (play now) link. Bytes, not audio.
- **MacAST** — a classic OS 9 MP3 player — is the speaker. You park it once on an HTTP MP3 stream and leave it there. It never talks to Gopher at all.

When you hit ">> Tocar agora" in a gopher menu, Netscape fires a request that ultimately becomes a Spotify `PUT /play`. The audio was *already flowing* to MacAST on a completely separate socket. The two planes never touch. OS 9 drives Spotify without ever asking Gopher to do something Gopher can't.

On the cluster side, that's two pods:

1. **`audio-stream`** — `librespot` (an open-source Spotify Connect client) piping raw PCM into `ffmpeg`, encoding to MP3, served by Icecast. This pod *is* a Spotify Connect device: I "transfer playback" to it from my phone, and it appears in the cluster as a speaker named `gopher-spot`.
2. **`gopher-server`** — `geomyidae` (a tiny gopher daemon) plus a little Rust program that translates gopher selectors into Spotify Web API calls and Spotify JSON back into gopher menus.

Two MetalLB LoadBalancer IPs, two jobs, zero overlap. The Rust program in the middle is stateless and, delightfully, has no idea what music sounds like. Selector in, text menu out.

## The parts that were easy

I built the entire thing in one evening — about three and a half hours, both pods, OAuth, search, playback control, playlists, and 28 tests. Once the control/data split was clear, everything downstream was just plumbing:

- **The gopher server** runs `geomyidae`, which has a lovely convention: drop an executable `index.dcgi` in a directory and it runs it for any unmatched path, handing it the request and reading its stdout as a menu. My Rust binary *is* that dcgi. Every `/spot/*` URL — `/spot/search`, `/spot/track/{id}`, `/spot/play?uri=…` — routes through one `match` statement.
- **The Spotify glue** is boring in the best way: blocking HTTP (no async runtime — the dcgi is a fresh process per request, so async buys nothing), an OAuth refresh token, a handful of `serde` structs. librespot has no local API, so *all* control goes through Spotify's Web API against the `gopher-spot` device.
- **Testing offline** fell out naturally. The router takes a `SpotifyApi` trait object; in tests I hand it a fake with canned responses, so the whole rendering layer is verified with no network and no credentials.

There's exactly one moment of real craft in the text layer, and it's a good one: **MacRoman**. The OS 9 gopher client decodes bytes as Mac OS Roman, not UTF-8. So "Construção" sent as UTF-8 shows up as mojibake on OS 9. The fix is a transcoder at the very last step before stdout: ASCII passes through untouched (including every structural `[ ] | tab newline` byte that Gopher itself parses), and only the accented display characters get remapped to their single MacRoman byte. `ç` becomes `0x8D`, not the two-byte UTF-8 sequence. It's the kind of detail that's invisible when it works and makes the whole thing feel *right* on the old machine.

## The part that ate a whole morning

And then there was the audio. Everything above worked in an evening. Getting sound to actually come out of MacAST took a second sitting, most of a morning, and only two commits to show for it — because it was two separate rabbit holes stacked on top of each other.

**Rabbit hole #1: librespot was silently dead.** Control worked perfectly — search, play command, the device showed up, Spotify acknowledged the play — and *no audio*. MacAST just got "connection refused." I spent a long time convinced it was my plumbing: the pod networking, the LoadBalancer, ffmpeg, the pipe. It was none of those. Around November 2025 Spotify made a server-side change that broke the last released version of librespot (0.6.0): it authenticates fine, accepts the play command, and then produces exactly zero bytes of audio — ["not available in any supported format."](https://github.com/librespot-org/librespot/issues/1623) The fix was to build librespot from its unreleased `dev` branch (0.8.0), which carries the workaround. Auth had been a red herring the entire time; the break was downstream of everything I'd been staring at.

**Rabbit hole #2: `ffmpeg -listen 1` is not a streaming server.** My first audio design was ffmpeg's built-in HTTP listener. It technically serves an MP3 — but it serves *exactly one client*, *only while a track is actively producing audio*, and it drops the connection on every pause and every gap between tracks. Which means MacAST, parked on the stream, got "connection refused" roughly whenever music wasn't already playing — i.e. exactly when you'd go to start some. The real fix was to put a proper streaming server in the pod: **Icecast**, with an always-on silence source as a fallback mount. Now the socket is *always* up — you hear silence when nothing's playing, and the instant a track starts, the live source overrides the fallback and MacAST snaps to the music. Connect once, stay forever.

The lesson I keep relearning: when you bolt a modern SaaS onto vintage clients, the protocol translation — the part that *sounds* hard — is easy. The **stateful media plane** is where the hours go. Text is stateless and forgiving. Audio is a live thing that has to be *up* even when nothing is happening.

## Odds and ends that cost real time

A field guide, so you don't rediscover these the hard way:

- **Spotify's `/v1/search` rejects `limit=20`** with a 400 "Invalid limit" — even though the docs say 0–50. `limit=10` works. No idea why. Ten results is plenty for a gopher menu anyway.
- **Spotify deprecated `localhost` in OAuth redirect URIs.** You have to use the literal loopback `http://127.0.0.1:8888/callback`, and register it exactly.
- **mDNS can't escape a Kubernetes pod.** librespot's normal "pick me from the Connect list" magic uses zeroconf/mDNS — link-local multicast that never crosses the CNI boundary onto my LAN without `hostNetwork` (which I'd banned). So instead I log in once locally, cache the `credentials.json`, mount it as a Secret, and librespot appears as a Connect device *through Spotify's cloud* — outbound HTTPS only, no multicast required.
- **Binding port 70 without root.** Gopher's well-known port is privileged. Rather than run as root, I stamp a Linux file capability (`cap_net_bind_service`) onto the geomyidae binary so an unprivileged user can bind it.
- **A dcgi has no memory.** geomyidae runs a *fresh process per request*, so an in-memory cache would never survive to the next request. The access token, search results, and device list are cached to a tiny file-backed TTL cache in an `emptyDir` instead.

## Why bother?

Because it's the best kind of pointless. Nobody needed this. It connects two things — Spotify's 2020s cloud and a beige-era operating system — that have no business talking, over a protocol both sides had every reason to have forgotten. And when it works, it *really* works: Netscape Communicator browsing gopher playlists, a menu of tracks with accented titles rendering cleanly in MacRoman, and Chico Buarque pouring out of MacAST, all on an OS from 1999 running in a VM in 2026.

The build was lopsided in a way that taught me something: features in an evening, one audio bug across a morning. The old protocol was never the problem. Keeping a live stream *alive* was.

If you've got an old Mac and a spare afternoon: the whole thing is a couple of small pods, one stateless Rust binary, and a stubborn refusal to accept that Gopher is dead. Turns out it just needed a good reason to get up.

---

**Code:** [github.com/felipedbene/gopher-spot](https://github.com/felipedbene/gopher-spot)

*gopher-spot runs LAN-only on a homelab Kubernetes cluster. It does not touch the public internet beyond Spotify's own API. Mac OS Roman forever.*
