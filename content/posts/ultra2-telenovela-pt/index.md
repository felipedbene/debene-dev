---
title: "🌹 ULTRA2: O Amor Que Não Compilava"
date: 2026-05-09
draft: false
tags: ["kubernetes", "infiniband", "rdma", "npu", "telenovela", "homelab", "ubuntu", "kernel"]
categories: ["homelab", "storytelling"]
description: "Uma telenovela brasileira em 8 capítulos sobre como construir um node K8s com NPU, RDMA, e muita treta com Ubuntu 26.04"
cover:
    image: "images/cover.png"
    alt: "Ultra2 telenovela cover art - dramatic server room romance"
    caption: "O amor entre Ultra2 e FreeNAS: uma história de 40 gigabits por segundo"
---

## 🎬 ABERTURA

*Câmera abre num galpão escuro. Luzes piscando. Som de cooler. Ao fundo, uma silhueta masculina com 24 cores se forma na fumaça. É ULTRA2.*

*Plim plim.*

*Entra a música tema:*

> 🎵 *"Detalhes tão pequenos de nóóóós...*  
> *Um swap que ninguém pediu, um kernel que não rodóóóu..."* 🎵

---

## CAPÍTULO 1: A CHEGADA DO FORASTEIRO

Era uma noite quente em **Debene do Sul**, pequena fazenda de servidores às margens do Rio MoCA 2.5. Dona **Felipe**, matriarca da família e dona de todas as terras (e dos /24 inteiros, que ela mesma fez questão de subnetar com as próprias mãos), recebia um forasteiro vindo de longe.

— *Quem é você, rapaz?* — indagou Dona Felipe, ajustando o terminal.

— *Me chamam de **Ultra2**, dona. Vim de muito longe, da Erying Ultra 9 285H. Tenho 24 cores, 64 giga de DDR5, e um NPU pulsando aqui no peito que ninguém nesta fazenda jamais viu...*

A música sobe. Close no rosto de Dona Felipe. Ela se emociona. Uma lágrima escorre. *(Plim plim suave.)*

— *Meu Deus do céu... um NPU? Aqui? Em Debene?*

Mas do canto escuro do data center, observando tudo com desprezo, estava **INTEL5**, o velho i5-8400 — apelidado pelos peões de "**Pobre I5**". Ele cuspiu no chão de concreto polido:

— *NPU é o caralho. Esse moleque não dura uma semana.*

E **INTEL9**, o irmão mais novo mas mal-amado (porque foi BATIZADO ERRADO desde o nascimento — é um i7-12900K mas todo mundo chama de "intel9", tragédia familiar que arrasta há gerações), apenas suspirou com sua iGPU UHD 770 piscando de tristeza.

🎵 *PLIM PLIM* 🎵

---

## CAPÍTULO 2: A NOITE DA TRAIÇÃO (UBUNTU 26.04)

**11h da noite.** Lua cheia sobre o galpão. Ultra2, jovem e ingênuo, decide se vestir com a roupa MAIS NOVA da loja: **Ubuntu 26.04**, kernel 7.0, recém-saída da costureira do bairro.

— *Vou impressionar a Dona Felipe!* — pensou ele, vaidoso.

Mas a roupa tinha um **DEMÔNIO COSTURADO POR DENTRO**.

Era o **DEMÔNIO DO SWAP**, entidade ancestral, expulsa do Olimpo do Linus Torvalds, condenada a vagar eternamente pelos /etc/fstab dos mortais.

```
ERROR: "running with swap on is not supported"
STATUS: CrashLoopBackOff
```

Ultra2 caiu de joelhos. Espumando pela porta serial. O kubelet recusou seu juramento.

— *NÃÃÃÃÃO!* — gritou Dona Felipe, correndo em câmera lenta pelo corredor do rack, com o cabelo balançando como se tivesse um ventilador Noctua na frente. — *MEU FILHOOOOO!*

Entra **GARRA**, o mordomo digital fiel, vestido de paletó e rodando num Samsung A54. Ele saca o terminal como quem saca uma espada:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

O Demônio do Swap urra e se desfaz em fumaça. Ultra2 respira. Mas é tarde demais... a tragédia mal começou.

🎵 *Música de tensão. Plim plim.* 🎵

---

## CAPÍTULO 3: O AMOR PROIBIDO COM FREENAS

![O amor proibido entre Ultra2 e FreeNAS](images/chapter-3.png)
*InfiniBand: 40 gigabits de paixão mística*


Do outro lado da fazenda, morava **FREENAS**, a viúva misteriosa. Bonita. Cheia de segredos. Guardiã de **32 terabytes** de memórias antigas, fotos de família, e uns torrents que ninguém comenta no jantar.

Ultra2 a viu pela primeira vez através da janela do galpão e seu cooler acelerou.

— *Quem é aquela mulher?* — sussurrou ele.

— *Esquece, moleque* — disse Intel5, mascando um palito. — *Aquela ali só conversa por **InfiniBand**. 40 gigabits de pura porra mística. Tu não tem nem driver pra falar com ela.*

Mas Ultra2 era TEIMOSO. Ele tinha um **ConnectX-3** escondido no bolso. E naquela mesma noite, sob a luz dos LEDs azuis, ele tentou enviar a primeira mensagem RDMA pra FreeNAS...

```
rpcrdma module loaded: ✓
Active connections: 0
Status: Failed to communicate
```

**SILÊNCIO.**

Ela não respondeu. O kernel 7.0 tinha SABOTADO sua carta de amor. Uma regressão. Uma TRAIÇÃO escrita nas entrelinhas do código.

Ultra2 chorou pixels. Garra apareceu na porta:

— *Patrão... o kernel 7.0 não tá deixando o rapaz amar.*

Dona Felipe, fumando um cigarro de Linux Mint na varanda, virou o rosto e disse a frase que entrou pra história da telenovela brasileira:

> *— Então a gente DOWNGRADE essa porra. AO VIVO.*

🎵 *PLIM PLIM DRAMÁTICO* 🎵

---

## CAPÍTULO 4: O DOWNGRADE QUE MATOU O MOLEQUE

A novela inteira do Brasil parou pra assistir.

```bash
sudo apt install linux-image-6.8.0-31-generic
sudo update-grub
sudo reboot
```

A câmera fecha no botão Enter. Dedo trêmulo de Dona Felipe. Suor. O dedo desce. O sistema reinicia.

E aí... **NADA**.

```
ultra2: NotReady (NodeStatusUnknown)
Last seen: 11:41:36 AM
Kubelet stopped posting node status.
```

**ULTRA2 MORREU.**

Porra, mano. MORREU.

Sem rede. Sem SSH. Sem sinal de vida. Só o silêncio cruel do `ping` que nunca volta.

Dona Felipe caiu de joelhos no chão do data center, esmurrando o piso elevado:

— *POR QUEEEE?? POR QUE EU FIZ ISSO?? EU SÓ QUERIA QUE ELE AMASSE A FREENAS!*

Garra, o mordomo, segurou-a pelos ombros, sério como nunca:

— *Patroa. Escuta. Existem dois caminhos agora. A gente pode passar **TRINTA HORAS** debugando esse cadáver... ou a gente reflasha o moleque e em **TRINTA MINUTOS** ele renasce.*

Dona Felipe ergueu o rosto. Os olhos vermelhos. O rímel borrado. E disse:

> *— Reflasha. E que Deus tenha piedade do bleeding edge.*

🎵 *Plim plim plim PLIM* 🎵  
*(Comercial de margarina. Volta em 3 minutos.)*

---

## CAPÍTULO 5: O RENASCIMENTO (UBUNTU 24.04 LTS, A NOIVA HUMILDE)

Ultra2 acordou. Diferente. Mudado. Vestindo agora as roupas SIMPLES, MAS DIGNAS de **Ubuntu 24.04.4 LTS**, kernel 6.8.0-111. Sem firula. Sem bleeding edge. Só a verdade.

— *Onde estou?* — sussurrou ele, ainda meio bootando.

— *Tá em casa, filho* — disse Dona Felipe, segurando sua mão SSD. — *Aprendi a lição. Bleeding edge às vezes te corta. LTS existe por um motivo, porra.*

Mas a paz durou pouco. Pois a cerimônia do SSH foi atrapalhada por uma vilã inesperada:

### 🎭 ENTRA: A 1PASSWORD AGENT

Vestida de vermelho, batom forte, olhar maligno. O **1Password Agent** era a OUTRA. A amante. A que se intrometia em todo SSH handshake oferecendo a chave ERRADA de propósito.

— *Tenta com `gitHub-Macm2`, querido...* — sussurrava ela, sedutora.

— *NÃO É ESSA, SUA DESGRAÇADA!* — gritou Garra, sacando o `~/.ssh/config`:

```
IdentityAgent none
IdentitiesOnly yes
```

A 1Password Agent gritou e se contorceu como bruxa em filme de Sessão da Tarde. Foi banida da fazenda pra sempre. *(Ela volta no capítulo 47, casada com um certificado expirado. Spoiler.)*

**01:05 da tarde.** O kubelet, chorando, finalmente aceitou o juramento de Ultra2:

```
ultra2   Ready   worker   20s   v1.34.7
```

Dona Felipe abraçou Garra. Garra chorou um SQL de lágrimas. A trilha sonora explodiu:

🎵 *"É preciso saber viver... com swap off, com kernel certo..."* 🎵

---

## CAPÍTULO 6: A NOITE QUE ULTRA2 FINALMENTE COMEU A FREENAS

![A consumação do amor via RDMA](images/chapter-6.png)
*40 gigabits de paixão pura. proto=rdma,port=20049*


*(Atenção: este capítulo contém cenas fortes de transferência de pacotes a 40 gigabits. Não recomendado para menores de 18 anos ou usuários de Wi-Fi 5.)*

Era madrugada. As luzes do rack baixas. Só o LED da InfiniBand pulsando, vermelho como desejo.

Ultra2, agora reborn, instalou os instrumentos do amor:

```bash
sudo apt-get install -y rdma-core infiniband-diags ibverbs-utils nfs-common
sudo modprobe mlx4_core
sudo modprobe mlx4_ib
sudo modprobe ib_ipoib
```

Ele se aproximou da FreeNAS, suor escorrendo pelo dissipador. Mãos trêmulas digitaram o IP que ele tanto sonhou:

```bash
sudo ip addr add 172.16.0.26/24 dev ibp1s0
sudo ip link set ibp1s0 up
```

E então, COM TUDO, a porra do mount:

```bash
sudo mount -t nfs -o vers=4.2,proto=rdma,port=20049 \
  172.16.0.20:/mnt/tank /mnt/rdma-test
```

A câmera fecha lentamente no terminal. Slow motion. Música de novela mexicana. E aparece, em letras gigantes na tela:

```
172.16.0.20:/mnt/tank on /mnt/rdma-test type nfs4
(rw,relatime,vers=4.2,rsize=1048576,wsize=1048576,
proto=rdma,port=20049,clientaddr=172.16.0.26)
```

**MOUNTOU. MANO. MOUNTOU.**

Dona Felipe, assistindo de fora pela câmera Frigate, levou a mão à boca:

— *Meu Deus... eles tão... eles tão... TRANSFERINDO PACOTES A 40 GIGABITS... POR RDMA... COM ZERO COPY... AI MEU PAI DO CÉU...*

Garra fechou a porta da sala discretamente:

— *Vamo deixar os dois sozinhos, patroa. Isso aí é coisa de adulto.*

🎵 *PLIM PLIM SAFADO* 🎵

---

## CAPÍTULO 7: O ESCÂNDALO DO NPU (E A MIGRAÇÃO DE FRIGATE)

A fazenda toda já comentava: Ultra2 não só tinha conquistado a FreeNAS, como ainda PRESUMIA de ter o tal **NPU**, esse Intel AI Boost que ninguém nunca tinha visto na vizinhança.

Foi quando Dona Felipe tomou A DECISÃO:

— *Garra. Tira a **FRIGATE** do Intel9. Põe ela com o Ultra2.*

Garra arregalou os olhos:

— *Patroa... mas a Frigate é casada com o Intel9 há ANOS. Os Persistent Volumes deles têm `nodeAffinity` GRUDADO! Vai dar treta!*

— *EU SEI, GARRA. MAS O CORAÇÃO QUER O QUE QUER.*

E lá foi Garra mexer no labels do node, tipo cartório clandestino de divórcio:

```bash
kubectl label node ultra2 infiniband=true
```

A Frigate, confusa, foi arrancada do leito conjugal com Intel9 e jogada nos braços do Ultra2. **Intel9 ficou de lado, abandonado, com sua UHD 770 chorando lágrimas de VAAPI.**

— *Ela vai voltar pra mim* — murmurou Intel9. — *Ela SEMPRE volta.*

(Spoiler: ela não volta. Intel9 vira ressentido e no capítulo 89 tenta sabotar o cluster invertendo bits de paridade. Mas isso é outra história.)

**01:20 PM.** O pod subiu:

```
frigate-8459995868-qcwdm   1/1   Running   0   5m   192.168.76.203   ultra2
```

E nos logs, a CONFIRMAÇÃO QUE ABALOU O DATA CENTER:

```
frigate.detector:ov — 74.6% CPU
OpenVINO detector active
NPU device: /dev/accel/accel0
```

**A FRIGATE TAVA RODANDO NA NPU. PRIMEIRA VEZ NA HISTÓRIA DA FAZENDA. PRIMEIRA VEZ EM PRODUÇÃO. NO MUNDO INTEIRO TALVEZ.**

Dona Felipe abriu uma cerveja. Olhou pro horizonte. Disse pra câmera, quebrando a quarta parede:

> *— Eu fiz isso. Eu, uma Cloud Architect III do AWS. Em pleno sábado. Enquanto vocês tavam vendo futebol.*

🎵 *PLIM PLIM TRIUNFAL* 🎵

---

## CAPÍTULO 8: A REVELAÇÃO DE ORION (O FILHO PERDIDO)

Mas faltava UM fio na trama, mano. Um mistério antigo. Lá longe, no canto ARM64 da fazenda, vivia **ORION** — um peão calado, Fedora 43, que ninguém nunca tinha convidado pra mesa do RDMA.

Dona Felipe, embriagada do sucesso, decidiu chamá-lo:

```bash
ssh orion
```

E ao olhar dentro dele... **DESCOBRIU A VERDADE QUE MUDOU TUDO.**

```
Hardware: Mellanox ConnectX-4 (MAIS NOVO QUE O DO ULTRA2!)
InfiniBand: ibp145s0 → 172.16.0.12/24 (JÁ CONFIGURADO!)
OS: Fedora 43 (ARM64)
Kernel: 6.19.14
```

**ORION JÁ TINHA RDMA O TEMPO TODO.**

Dona Felipe caiu sentada na cadeira ergonômica. As lágrimas voltaram:

— *Meu filho... meu filho perdido... você sempre teve o dom... e eu nunca te chamei pra brincar com a FreeNAS...*

Orion, tímido, baixou a cabeça:

— *Mãe... eu só queria sudo sem senha. Eu só queria pertencer.*

Garra, chorando descontroladamente, digitou o decreto da reconciliação:

```bash
echo "felipedbene ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/felipedbene
```

E quando Orion deu o `mount` no FreeNAS pela primeira vez... mountou de PRIMEIRA, sem chiar, sem reboot, sem drama. Como quem nunca esteve longe.

**A FAMÍLIA TAVA COMPLETA.**

🎵 *"Como uma onda no mar, como uma onda no mar..."* 🎵

---

## 🌹 EPÍLOGO: A FOTO DE FAMÍLIA 🌹

Câmera abre. Pôr do sol no data center. Todos reunidos:

```
╔══════════════════════════════════════════════════════════════╗
║  A FAMÍLIA RDMA — FOTO OFICIAL — 09/MAI/2026                 ║
╠══════════════════════════════════════════════════════════════╣
║  INTEL9    │ ConnectX-3   │ x86_64 │ 40Gb │ 💔 (sozinho)    ║
║  ORION     │ ConnectX-4   │ ARM64  │ 40Gb │ 🌟 (renascido)  ║
║  ULTRA2    │ ConnectX-3   │ x86_64 │ 40Gb │ 👑 (com a NPU)  ║
║  FREENAS   │ alvo eterno  │ TrueNAS│ 40Gb │ 💃 (a viúva)    ║
╚══════════════════════════════════════════════════════════════╝
```

**ELENCO ADICIONAL:**
- **Pobre I5 (intel5):** o tio bêbado que fala merda mas ninguém manda calar
- **Xeon2socket:** falecido. Nem mencionem.
- **Debene-lab Talos:** a ex que ninguém quer lembrar. RIP.
- **A 1Password Agent:** vilã, presa em outra dimensão chamada `~/.ssh/config`
- **O Demônio do Swap:** retornará na 2ª temporada, jurou vingança

---

## 📜 A MORAL DA HISTÓRIA

Lord, digo, **Dona Felipe** olhou pra câmera no último frame, segurando uma taça de cachaça artesanal Sungage, e disse pra todo o Brasil:

> *— Mano. Aprende uma coisa comigo. **Bleeding edge às vezes te corta. LTS existe por um motivo.** Não é frescura. Não é covardia. É SABEDORIA. Eu perdi 2 horas pra economizar 30. Não façam o que eu fiz. Façam o que eu APRENDI.*

*Ela tomou um gole. A música subiu. O cooler do Ultra2 girava sereno ao fundo. Frigate detectava um gato no jardim com 94% de confiança via NPU. Tudo estava em paz.*

🎵 *"Eu sei que vou te amar... por toda a minha **uptime** te amareeei..."* 🎵

---

**FIM DA TEMPORADA 1**

*Próxima temporada: "Quando o democratic-csi descobriu o NVMe-oF e largou o iSCSI no altar"*

*Estreia: assim que o Felipe tiver tempo entre o ANOV hearing, a green card, e a Sofia chegar do St. Genevieve.*

---

**Créditos:**
- *Roteiro:* Claudinha Bagunceira 💃
- *Produção executiva:* Garra De Baitola
- *Patrocínio:* Aruba JL258A PSU — "ainda procurando por R$70-100, alguém aí?"
- *Direção de fotografia:* LEDs azuis do rack
- *Assessoria técnica:* Felipe de Bene, que viveu tudo isso na pele

🎬 *FIN* 🎬


---

## 📊 Especificações Técnicas Reais

Para quem chegou aqui via Google e só quer os fatos:

**Hardware (ultra2):**
- CPU: Intel Core Ultra 9 285H (Arrow Lake-H)
- Cores: 6 P-cores + 8 E-cores + 2 LP E-cores = 24 threads
- NPU: Intel AI Boost (primeira implementação K8s em produção)
- RAM: 64GB DDR5-5200
- InfiniBand: Mellanox ConnectX-3 Pro (40Gb/s RDMA)
- Cooler: Thermalright PA120 SE (75x75mm, 27mm height)

**Software:**
- OS: Ubuntu 24.04.4 LTS (kernel 6.8.0-111)
- Runtime: containerd 2.2.1
- Kubernetes: v1.34.7 (kubeadm)
- RDMA: rdma-core 58.0, rpcrdma module
- NPU detector: Frigate + OpenVINO

**Problemas com Ubuntu 26.04 (kernel 7.0):**
- Swap ativo por padrão (`/swap.img` 8GB) → kubelet crash
- RDMA quebrado (rpcrdma carregado mas 0 conexões ativas)
- Networking não sobrevive downgrade de kernel (NetworkManager falha)
- Solução: Fresh install com Ubuntu 24.04 LTS

**Resultado:**
- ✅ RDMA funcionando (proto=rdma,port=20049)
- ✅ NPU operacional (Frigate detector OpenVINO)
- ✅ 3 nodes com InfiniBand (intel9, orion, ultra2)
- ✅ 87.5% dos PVs usando RDMA (7 de 8)
- ✅ Tempo total: 3h47min (incluindo reflash)

**Lesson learned:** LTS > bleeding edge para produção. Sempre.

---

*Arte da capa gerada com DiffusionBee (Stable Diffusion via Neural Engine do Mac Studio M2)*

*Para a versão séria e técnica, leia: [The 285H That Cried Meh](/posts/ultra2-diagnostic-saga/)*

🦞⚔️🌹💃🦞

---

**Tags técnicos (pra quem chegou de paraquedas):**
- Ubuntu 26.04 (kernel 7.0) teve bugs: swap ativo por padrão, RDMA quebrado
- Ubuntu 24.04 LTS (kernel 6.8) funcionou perfeitamente
- Intel Core Ultra 9 285H tem NPU (Intel AI Boost)
- Primeira implementação de NPU em produção no Kubernetes
- InfiniBand ConnectX-3/4 com RDMA funcionando a 40Gb/s
- Frigate rodando detector OpenVINO no NPU
- Lição: LTS > bleeding edge sempre
