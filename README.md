# debene.dev

Personal blog running on [Hugo](https://gohugo.io/) with the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. Self-hosted on a Kubernetes homelab with automated webhook-based deployment.

**✨ Fully automated:** Push to `main` → Live in ~40 seconds!

## Stack

```
Internet → Cloudflare Tunnel → K8s (nginx pods) → NFS Storage
                                      ↑
                                 Webhook Pod
                                      ↑
                              GitHub Push Event
```

- **Generator**: Hugo 0.152.2+extended (ARM64)
- **Theme**: PaperMod (git submodule)
- **Web Server**: nginx:alpine serving static files (multi-arch: AMD64 + ARM64)
- **Storage**: NFS (TrueNAS SCALE, 40Gb InfiniBand RDMA)
- **Hosting**: Kubernetes homelab (kubeadm, 5-node cluster: x86_64 + ARM64)
- **CDN/Tunnel**: Cloudflare Tunnel (Zero Trust, QUIC protocol)
- **Domain**: [debene.dev](https://debene.dev)

## Architecture

### Deployment Pipeline

**Automated webhook-based deployment** (no GitHub Actions, no Docker builds):

```
┌─────────────┐
│ Git Push    │  Developer pushes to main branch
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ GitHub      │  Webhook fires POST request
│ Webhook     │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────────┐
│ K8s Webhook Pod (orion.debene.dev)  │
│ ┌─────────────────────────────────┐ │
│ │ 1. Git pull (fetch latest)      │ │
│ │ 2. Hugo build (static site gen) │ │
│ │ 3. Copy to NFS (/var/www/html)  │ │
│ │ 4. Fix permissions (www-data)   │ │
│ └─────────────────────────────────┘ │
└──────────────┬──────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ NFS Storage (TrueNAS)                │
│ /mnt/tank/media/www/blog/            │
│ - Mounted via InfiniBand (RDMA)     │
│ - Shared across all nginx pods      │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ Nginx Pods (intel9 + orion)          │
│ - Multi-arch deployment              │
│ - Serve static files from NFS       │
│ - 2 replicas for HA                  │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ Cloudflare Tunnel (cloudflared pod) │
│ - Secure tunnel to Cloudflare edge  │
│ - No open firewall ports             │
│ - QUIC protocol for performance      │
└──────────────┬───────────────────────┘
               │
               ▼
       ┌───────────────┐
       │  Internet     │
       │ debene.dev    │
       └───────────────┘
```

### Why This Architecture?

**No Docker builds, no image registry:**
- Hugo builds static files directly in webhook pod
- Files copied to NFS, served by nginx
- Faster deploys (~40s vs 5+ min with Docker build)
- Simpler infrastructure (no Docker Hub, no image pulls)

**NFS over InfiniBand:**
- 40Gb/s RDMA storage network
- Single source of truth for static files
- All nginx pods serve same content instantly
- No need to update deployments or pull images

**Webhook vs GitHub Actions:**
- Runs inside cluster (no external runners)
- Direct NFS access (no kubectl needed)
- Faster feedback loop
- Lower resource usage

## Deploy Performance

**Typical deploy timeline:**
```
00:00 - Git push to main
00:01 - Webhook triggered (< 1ms response)
00:03 - Git pull complete
00:05 - Hugo build (2-3s for ~200 pages)
00:40 - Deploy complete (copy + permissions)
00:40 - Site live on debene.dev
```

**Real-world example (2026-05-10):**
- Pages: 210 → 215 (+5 new)
- Hugo build: 2.085s
- Total deploy: 40 seconds
- Webhook latency: 587μs

## Local Development

```bash
# Install Hugo (macOS)
brew install hugo

# Clone with submodules (theme)
git clone --recurse-submodules https://github.com/felipedbene/debene-dev.git
cd debene-dev

# Run dev server with drafts
hugo server -D

# Build for production (test locally)
hugo --minify
```

## Writing Posts

**Create new post:**
```bash
hugo new content/posts/my-post-slug/index.md
```

**Directory structure (Page Bundles):**
```
content/posts/my-post-slug/
├── index.md              # Post content
└── images/               # Post images
    ├── cover.png
    └── screenshot.png
```

**Front matter template:**
```yaml
---
title: "Post Title"
date: 2026-05-10T18:00:00-05:00
draft: false
tags: ["tag1", "tag2"]
categories: ["category"]
cover:
  image: "images/cover.png"
  alt: "Alt text"
---

Post content here...
```

## Deploy (Automatic)

**Just push to main:**
```bash
git add .
git commit -m "New post: My Awesome Article"
git push origin main

# Webhook automatically triggers deploy
# Watch logs: kubectl logs -n blog deployment/blog-webhook -f
```

**No manual steps required!** The webhook handles:
1. ✅ Git pull
2. ✅ Hugo build
3. ✅ NFS deployment
4. ✅ Permissions
5. ✅ Live site update

## Webhook Configuration

**GitHub webhook settings:**
- **Payload URL:** `https://debene.dev/hooks/blog-deploy`
- **Content type:** `application/json`
- **Events:** Just push events on `main` branch
- **Secret:** Configured in K8s secret

**Webhook pod spec:**
```yaml
namespace: blog
deployment: blog-webhook
replicas: 1
node: orion.debene.dev (ARM64)
init-container: clone-repo (one-time git clone)
container: webhook-all-in-one (Alpine + Hugo + webhook binary)
```

## Infrastructure Details

**Kubernetes cluster:**
- **Control plane:** zima (10.0.10.100)
- **Workers:** intel5, intel9, orion, ultra2
- **CNI:** Calico (VXLAN)
- **Storage:** Longhorn (replicated) + NFS (media)
- **Load Balancer:** MetalLB
- **Ingress:** Caddy (K8s operator)

**Storage backend:**
- **NAS:** TrueNAS SCALE 25.10.3.1
- **Network:** InfiniBand 40Gb/s (FDR)
- **Protocol:** NFS with RDMA (proto=rdma, port=20049)
- **Mount:** `/mnt/tank/media/www/blog/` → `/var/www/html`

**Blog pods:**
```bash
# 2 nginx replicas for HA
kubectl get pods -n blog -l app=blog
NAME                    READY   STATUS    NODE
blog-c5b7f45f7-88lbb    1/1     Running   intel9.debene.dev
blog-c5b7f45f7-sgxpm    1/1     Running   orion.debene.dev
```

## Monitoring

**Webhook logs:**
```bash
# Watch deploy in real-time
kubectl logs -n blog deployment/blog-webhook -f

# Check last deploy
kubectl logs -n blog deployment/blog-webhook --tail=50
```

**Blog access logs:**
```bash
# Nginx access logs
kubectl logs -n blog -l app=blog --tail=100 | grep -v healthz
```

**Deployment status:**
```bash
# Check all blog resources
kubectl get all -n blog

# Check NFS mount status
kubectl exec -n blog deployment/blog -- df -h /var/www/html
```

## Troubleshooting

**Webhook not triggering:**
```bash
# Check webhook pod logs
kubectl logs -n blog deployment/blog-webhook

# Verify GitHub webhook deliveries (GitHub repo → Settings → Webhooks)

# Test webhook manually
curl -X POST https://debene.dev/hooks/blog-deploy \
  -H "Content-Type: application/json" \
  -d '{"ref":"refs/heads/main"}'
```

**Hugo build failed:**
```bash
# Check for syntax errors in posts
hugo --minify

# Rebuild webhook pod
kubectl rollout restart -n blog deployment/blog-webhook
```

**Site not updating:**
```bash
# Check NFS mount
kubectl exec -n blog deployment/blog -- ls -la /var/www/html

# Check file timestamps
kubectl exec -n blog deployment/blog -- stat /var/www/html/index.html

# Force nginx pods to reload
kubectl rollout restart -n blog deployment/blog
```

## About

Written from a basement in Chicago where an IBM POWER8 server runs Gentoo, AIX runs in KVM, and a Kubernetes cluster ties it all together with InfiniBand storage at 40Gb/s. The blog you're reading was deployed automatically by a webhook pod running on an ARM64 node, served from NFS over RDMA, and delivered via Cloudflare Tunnel — because why not? 🚀
