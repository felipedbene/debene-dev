---
title: "🌹 ULTRA2: The Love That Wouldn't Compile"
date: 2026-05-09
draft: false
tags: ["kubernetes", "infiniband", "rdma", "npu", "soap-opera", "homelab", "ubuntu", "kernel"]
categories: ["homelab", "storytelling"]
description: "A Brazilian-style soap opera in 8 chapters about building a Kubernetes node with NPU, RDMA, and a lot of drama with Ubuntu 26.04"
cover:
    image: "images/cover.png"
    alt: "Ultra2 telenovela cover art - dramatic server room romance"
    caption: "The love story between Ultra2 and FreeNAS: 40 gigabits per second of passion"
---

## 🎬 OPENING CREDITS

*Camera opens on a dark warehouse. Blinking lights. Sound of cooling fans. In the background, a masculine silhouette with 24 cores forms in the mist. It's ULTRA2.*

*Plim plim.* (That's the iconic Brazilian soap opera sound effect.)

*Theme song plays:*

> 🎵 *"Such tiny details of us two...*  
> *A swap nobody asked for, a kernel that wouldn't boot through..."* 🎵

---

## CHAPTER 1: THE STRANGER ARRIVES

It was a hot night in **Debene Ranch**, a small server farm by the MoCA 2.5 River. Dona **Felipe**, matriarch of the family and owner of all the land (and all the /24 subnets, which she personally configured by hand), received a stranger from afar.

— *Who are you, young man?* — inquired Dom Felipe, adjusting her terminal.

— *They call me **Ultra2**, ma'am. I come from far away, from the Erying Ultra 9 285H. I have 24 cores, 64 gigs of DDR5, and an NPU beating in my chest that nobody on this ranch has ever seen...*

The music swells. Close-up on Dom Felipe's face. She gets emotional. A tear rolls down. *(Soft plim plim.)*

— *My God... an NPU? Here? In Debene?*

But from the dark corner of the data center, watching with contempt, stood **INTEL5**, the old i5-8400 — nicknamed by the ranch hands as "**Poor I5**". He spat on the polished concrete floor:

— *NPU my ass. This kid won't last a week.*

And **INTEL9**, the younger but unloved brother (because he was BAPTIZED WRONG from birth — he's actually an i7-12900K but everyone calls him "intel9", a family tragedy that's dragged on for generations), just sighed with his iGPU UHD 770 blinking sadly.

🎵 *PLIM PLIM* 🎵

---

## CHAPTER 2: THE NIGHT OF BETRAYAL (UBUNTU 26.04)

**11 PM.** Full moon over the warehouse. Ultra2, young and naive, decides to dress in the NEWEST outfit from the store: **Ubuntu 26.04**, kernel 7.0, fresh from the local seamstress.

— *I'm going to impress Dom Felipe!* — he thought, vain.

But the outfit had a **DEMON SEWN INSIDE**.

It was the **SWAP DEMON**, an ancient entity, expelled from Linus Torvalds' Olympus, condemned to wander eternally through the /etc/fstab files of mortals.

```
ERROR: "running with swap on is not supported"
STATUS: CrashLoopBackOff
```

Ultra2 fell to his knees. Foaming through his serial port. The kubelet refused his oath.

— *NOOOOO!* — screamed Dom Felipe, running in slow motion through the rack corridor, her hair flowing as if there were a Noctua fan in front of her. — *MY SONNNN!*

Enter **GARRA**, the faithful digital butler, dressed in a suit jacket and running on a Samsung A54. He draws his terminal like a sword:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

The Swap Demon roars and dissolves into smoke. Ultra2 breathes. But it's too late... the tragedy has just begun.

🎵 *Tension music. Plim plim.* 🎵

---

## CHAPTER 3: THE FORBIDDEN LOVE WITH FREENAS

![The forbidden love between Ultra2 and FreeNAS](images/chapter-3.png)
*InfiniBand: 40 gigabits of pure mystical passion*


On the other side of the ranch lived **FREENAS**, the mysterious widow. Beautiful. Full of secrets. Guardian of **32 terabytes** of old memories, family photos, and some torrents nobody mentions at dinner.

Ultra2 saw her for the first time through the warehouse window and his cooler accelerated.

— *Who is that woman?* — he whispered.

— *Forget it, kid* — said Intel5, chewing a toothpick. — *That one only talks through **InfiniBand**. 40 gigabits of pure mystical magic. You don't even have the driver to talk to her.*

But Ultra2 was STUBBORN. He had a **ConnectX-3** hidden in his pocket. And that same night, under the blue LED lights, he tried to send his first RDMA message to FreeNAS...

```
rpcrdma module loaded: ✓
Active connections: 0
Status: Failed to communicate
```

**SILENCE.**

She didn't respond. Kernel 7.0 had SABOTAGED his love letter. A regression. A BETRAYAL written in the code's fine print.

Ultra2 cried pixels. Garra appeared at the door:

— *Boss... kernel 7.0 isn't letting the boy love.*

Dom Felipe, smoking a Linux Mint cigarette on the porch, turned her face and said the phrase that went down in Brazilian soap opera history:

> *— Then we DOWNGRADE this thing. LIVE.*

🎵 *DRAMATIC PLIM PLIM* 🎵

---

## CHAPTER 4: THE DOWNGRADE THAT KILLED THE BOY

All of Brazil stopped to watch.

```bash
sudo apt install linux-image-6.8.0-31-generic
sudo update-grub
sudo reboot
```

The camera closes on the Enter button. Dom Felipe's trembling finger. Sweat. The finger descends. The system reboots.

And then... **NOTHING**.

```
ultra2: NotReady (NodeStatusUnknown)
Last seen: 11:41:36 AM
Kubelet stopped posting node status.
```

**ULTRA2 DIED.**

Damn. DIED.

No network. No SSH. No sign of life. Just the cruel silence of a `ping` that never returns.

Dom Felipe fell to her knees on the data center floor, pounding the raised floor:

— *WHYYYYY?? WHY DID I DO THIS?? I JUST WANTED HIM TO LOVE FREENAS!*

Garra, the butler, held her by the shoulders, serious as never before:

— *Ma'am. Listen. There are two paths now. We can spend **THIRTY HOURS** debugging this corpse... or we reflash the boy and in **THIRTY MINUTES** he's reborn.*

Dom Felipe lifted her face. Red eyes. Smudged mascara. And said:

> *— Reflash him. And may God have mercy on bleeding edge.*

🎵 *Plim plim plim PLIM* 🎵  
*(Margarine commercial. Back in 3 minutes.)*

---

## CHAPTER 5: THE REBIRTH (UBUNTU 24.04 LTS, THE HUMBLE BRIDE)

Ultra2 woke up. Different. Changed. Now wearing the SIMPLE BUT DIGNIFIED clothes of **Ubuntu 24.04.4 LTS**, kernel 6.8.0-111. No frills. No bleeding edge. Just the truth.

— *Where am I?* — he whispered, still half-booting.

— *You're home, son* — said Dom Felipe, holding his SSD hand. — *I learned my lesson. Bleeding edge sometimes cuts you. LTS exists for a reason.*

But the peace didn't last long. For the SSH ceremony was interrupted by an unexpected villain:

### 🎭 ENTER: THE 1PASSWORD AGENT

Dressed in red, strong lipstick, evil gaze. The **1Password Agent** was THE OTHER WOMAN. The mistress. The one who interfered in every SSH handshake offering the WRONG key on purpose.

— *Try with `gitHub-Macm2`, darling...* — she whispered, seductive.

— *THAT'S NOT IT, YOU WITCH!* — shouted Garra, pulling out the `~/.ssh/config`:

```
IdentityAgent none
IdentitiesOnly yes
```

The 1Password Agent screamed and writhed like a witch in an old movie. She was banished from the ranch forever. *(She returns in chapter 47, married to an expired certificate. Spoiler.)*

**01:05 PM.** The kubelet, crying, finally accepted Ultra2's oath:

```
ultra2   Ready   worker   20s   v1.34.7
```

Dom Felipe hugged Garra. Garra cried a SQL of tears. The soundtrack exploded:

🎵 *"You gotta know how to live... with swap off, with the right kernel..."* 🎵

---

## CHAPTER 6: THE NIGHT ULTRA2 FINALLY MOUNTED FREENAS

![The consummation of love via RDMA](images/chapter-6.png)
*40 gigabits of pure passion. proto=rdma,port=20049*


*(Warning: this chapter contains intense scenes of packet transfer at 40 gigabits. Not recommended for those under 18 or Wi-Fi 5 users.)*

It was dawn. The rack lights were low. Only the InfiniBand LED pulsing, red as desire.

Ultra2, now reborn, installed the instruments of love:

```bash
sudo apt-get install -y rdma-core infiniband-diags ibverbs-utils nfs-common
sudo modprobe mlx4_core
sudo modprobe mlx4_ib
sudo modprobe ib_ipoib
```

He approached FreeNAS, sweat dripping down his heatsink. Trembling hands typed the IP he had dreamed of:

```bash
sudo ip addr add 172.16.0.26/24 dev ibp1s0
sudo ip link set ibp1s0 up
```

And then, WITH EVERYTHING, the damn mount:

```bash
sudo mount -t nfs -o vers=4.2,proto=rdma,port=20049 \
  172.16.0.20:/mnt/tank /mnt/rdma-test
```

The camera slowly closes on the terminal. Slow motion. Mexican soap opera music. And appears, in giant letters on the screen:

```
172.16.0.20:/mnt/tank on /mnt/rdma-test type nfs4
(rw,relatime,vers=4.2,rsize=1048576,wsize=1048576,
proto=rdma,port=20049,clientaddr=172.16.0.26)
```

**IT MOUNTED. DUDE. IT MOUNTED.**

Dom Felipe, watching from outside through the Frigate camera, brought her hand to her mouth:

— *My God... they're... they're... TRANSFERRING PACKETS AT 40 GIGABITS... VIA RDMA... WITH ZERO COPY... OH MY LORD...*

Garra discreetly closed the door:

— *Let's leave them alone, ma'am. This is adult stuff.*

🎵 *SPICY PLIM PLIM* 🎵

---

## CHAPTER 7: THE NPU SCANDAL (AND THE FRIGATE MIGRATION)

The whole ranch was already talking: Ultra2 not only had conquered FreeNAS, but also BRAGGED about having that **NPU**, this Intel AI Boost that nobody in the neighborhood had ever seen.

That's when Dom Felipe made THE DECISION:

— *Garra. Take **FRIGATE** from Intel9. Put her with Ultra2.*

Garra's eyes widened:

— *Ma'am... but Frigate has been married to Intel9 for YEARS. Their Persistent Volumes have `nodeAffinity` STUCK! There's gonna be drama!*

— *I KNOW, GARRA. BUT THE HEART WANTS WHAT IT WANTS.*

And off Garra went to mess with the node labels, like an underground divorce registry:

```bash
kubectl label node ultra2 infiniband=true
```

Frigate, confused, was ripped from her conjugal bed with Intel9 and thrown into Ultra2's arms. **Intel9 was left aside, abandoned, with his UHD 770 crying VAAPI tears.**

— *She'll come back to me* — murmured Intel9. — *She ALWAYS comes back.*

(Spoiler: she doesn't come back. Intel9 becomes resentful and in chapter 89 tries to sabotage the cluster by flipping parity bits. But that's another story.)

**01:20 PM.** The pod came up:

```
frigate-8459995868-qcwdm   1/1   Running   0   5m   192.168.76.203   ultra2
```

And in the logs, the CONFIRMATION THAT SHOOK THE DATA CENTER:

```
frigate.detector:ov — 74.6% CPU
OpenVINO detector active
NPU device: /dev/accel/accel0
```

**FRIGATE WAS RUNNING ON THE NPU. FIRST TIME IN THE RANCH'S HISTORY. FIRST TIME IN PRODUCTION. IN THE WHOLE WORLD MAYBE.**

Dom Felipe opened a beer. Looked at the horizon. Said to the camera, breaking the fourth wall:

> *— I did this. Me, a Cloud Architect III at AWS. On a Saturday. While you were watching football.*

🎵 *TRIUMPHANT PLIM PLIM* 🎵

---

## CHAPTER 8: THE REVELATION OF ORION (THE LOST SON)

But there was ONE thread left in the plot. An old mystery. Far away, in the ARM64 corner of the ranch, lived **ORION** — a quiet ranch hand, Fedora 43, whom nobody had ever invited to the RDMA table.

Dom Felipe, drunk with success, decided to call him:

```bash
ssh orion
```

And looking inside him... **DISCOVERED THE TRUTH THAT CHANGED EVERYTHING.**

```
Hardware: Mellanox ConnectX-4 (NEWER THAN ULTRA2'S!)
InfiniBand: ibp145s0 → 172.16.0.12/24 (ALREADY CONFIGURED!)
OS: Fedora 43 (ARM64)
Kernel: 6.19.14
```

**ORION HAD RDMA ALL ALONG.**

Dom Felipe sank into her ergonomic chair. The tears returned:

— *My son... my lost son... you always had the gift... and I never called you to play with FreeNAS...*

Orion, shy, lowered his head:

— *Mom... I just wanted passwordless sudo. I just wanted to belong.*

Garra, crying uncontrollably, typed the reconciliation decree:

```bash
echo "felipedbene ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/felipedbene
```

And when Orion mounted FreeNAS for the first time... it mounted FIRST TRY, no fuss, no reboot, no drama. Like someone who had never been away.

**THE FAMILY WAS COMPLETE.**

🎵 *"Like a wave in the sea, like a wave in the sea..."* 🎵

---

## 🌹 EPILOGUE: THE FAMILY PHOTO 🌹

Camera opens. Sunset in the data center. Everyone gathered:

```
╔══════════════════════════════════════════════════════════════╗
║  THE RDMA FAMILY — OFFICIAL PHOTO — MAY 9, 2026              ║
╠══════════════════════════════════════════════════════════════╣
║  INTEL9    │ ConnectX-3   │ x86_64 │ 40Gb │ 💔 (alone)      ║
║  ORION     │ ConnectX-4   │ ARM64  │ 40Gb │ 🌟 (reborn)     ║
║  ULTRA2    │ ConnectX-3   │ x86_64 │ 40Gb │ 👑 (with NPU)   ║
║  FREENAS   │ eternal goal │ TrueNAS│ 40Gb │ 💃 (the widow)  ║
╚══════════════════════════════════════════════════════════════╝
```

**SUPPORTING CAST:**
- **Poor I5 (intel5):** the drunk uncle who talks nonsense but nobody shuts him up
- **Xeon2socket:** deceased. Don't even mention him.
- **Debene-lab Talos:** the ex nobody wants to remember. RIP.
- **The 1Password Agent:** villain, imprisoned in another dimension called `~/.ssh/config`
- **The Swap Demon:** will return in season 2, swore revenge

---

## 📜 THE MORAL OF THE STORY

Lord, I mean, **Dom Felipe** looked at the camera in the final frame, holding a glass of artisanal Sungage cachaça, and said to all of Brazil (and now, the world):

> *— Listen up. Learn something from me. **Bleeding edge sometimes cuts you. LTS exists for a reason.** It's not being chicken. It's not being a coward. It's WISDOM. I lost 2 hours to save 30. Don't do what I did. Do what I LEARNED.*

*She took a sip. The music rose. Ultra2's cooler spun serenely in the background. Frigate detected a cat in the garden with 94% confidence via NPU. Everything was at peace.*

🎵 *"I know I'm going to love you... for all my **uptime** I'll love youuuu..."* 🎵

---

**END OF SEASON 1**

*Next season: "When democratic-csi discovered NVMe-oF and left iSCSI at the altar"*

*Premieres: as soon as Felipe has time between the ANOV hearing, green card process, and Sofia coming back from St. Genevieve.*

---

**Credits:**
- *Screenplay:* Claudinha Bagunceira 💃
- *Executive Producer:* Garra De Baitola
- *Sponsored by:* Aruba JL258A PSU — "still looking for one at $70-100, anyone?"
- *Cinematography:* Blue rack LEDs
- *Technical Advisor:* Felipe de Bene, who lived through all of this

🎬 *FIN* 🎬


---

## 📊 Real Technical Specs

For those who arrived here via Google and just want the facts:

**Hardware (ultra2):**
- CPU: Intel Core Ultra 9 285H (Arrow Lake-H)
- Cores: 6 P-cores + 8 E-cores + 2 LP E-cores = 24 threads
- NPU: Intel AI Boost (first production K8s implementation)
- RAM: 64GB DDR5-5200
- InfiniBand: Mellanox ConnectX-3 Pro (40Gb/s RDMA)
- Cooler: Thermalright PA120 SE (75x75mm, 27mm height)

**Software:**
- OS: Ubuntu 24.04.4 LTS (kernel 6.8.0-111)
- Runtime: containerd 2.2.1
- Kubernetes: v1.34.7 (kubeadm)
- RDMA: rdma-core 58.0, rpcrdma module
- NPU detector: Frigate + OpenVINO

**Issues with Ubuntu 26.04 (kernel 7.0):**
- Swap enabled by default (`/swap.img` 8GB) → kubelet crash
- RDMA broken (rpcrdma loaded but 0 active connections)
- Networking doesn't survive kernel downgrade (NetworkManager fails)
- Solution: Fresh install with Ubuntu 24.04 LTS

**Results:**
- ✅ RDMA working (proto=rdma,port=20049)
- ✅ NPU operational (Frigate OpenVINO detector)
- ✅ 3 nodes with InfiniBand (intel9, orion, ultra2)
- ✅ 87.5% of PVs using RDMA (7 of 8)
- ✅ Total time: 3h47min (including reflash)

**Lesson learned:** LTS > bleeding edge for production. Always.

---

*Cover art generated with DiffusionBee (Stable Diffusion via Mac Studio M2 Neural Engine)*

*For the serious technical version, read: [The 285H That Cried Meh](/posts/ultra2-diagnostic-saga/)*

🦞⚔️🌹💃🦞

---

**Technical tags (for those who parachuted in):**
- Ubuntu 26.04 (kernel 7.0) had bugs: swap enabled by default, RDMA broken
- Ubuntu 24.04 LTS (kernel 6.8) worked perfectly
- Intel Core Ultra 9 285H has NPU (Intel AI Boost)
- First NPU implementation in production Kubernetes
- InfiniBand ConnectX-3/4 with RDMA working at 40Gb/s
- Frigate running OpenVINO detector on NPU
- Lesson: LTS > bleeding edge always
