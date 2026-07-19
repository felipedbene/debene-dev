# debene.dev

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE) ![Hugo](https://img.shields.io/badge/Hugo-static%20site-ff4088.svg) [![Deploy Blog](https://github.com/felipedbene/debene-dev/actions/workflows/deploy.yml/badge.svg)](https://github.com/felipedbene/debene-dev/actions/workflows/deploy.yml)

Read it live at [debene.dev](https://debene.dev).

Personal blog running on [Hugo](https://gohugo.io/) with the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. Self-hosted on a Kubernetes homelab, built and deployed by a GitHub Actions workflow.

**✨ Fully automated:** Push to `main` → GitHub Actions builds a container image, rolls out the deployment, and purges the CDN cache.

## Stack

```
Internet → Cloudflare Tunnel → K8s (nginx pod serving the built image)
                                      ↑
                          kubectl rollout restart
                                      ↑
                         GitHub Actions (self-hosted runner)
                                      ↑
                              GitHub Push to main
```

- **Generator**: Hugo `std-0.157.0` (via the `hugomods/hugo:std-0.157.0` build image)
- **Theme**: PaperMod (git submodule)
- **Web Server**: nginx:alpine serving the static site baked into the container image
- **Container build**: `nerdctl build` on a self-hosted homelab runner (containerd, `k8s.io` namespace)
- **Hosting**: Kubernetes homelab (`blog` namespace)
- **CDN/Tunnel**: Cloudflare Tunnel (`cloudflared` deployment in-cluster)
- **Domain**: [debene.dev](https://debene.dev)

## Architecture

### Deployment Pipeline

The pipeline is a **GitHub Actions workflow** (`.github/workflows/deploy.yml`) that runs on a
**self-hosted homelab runner**. On every push to `main` (or a manual `workflow_dispatch`) it builds a
container image, restarts the Kubernetes deployment, and purges the Cloudflare cache.

```
┌─────────────┐
│ Git Push    │  Developer pushes to main branch
└──────┬──────┘
       │
       ▼
┌──────────────────────────────────────────────┐
│ GitHub Actions — job: build-and-deploy       │
│ runs-on: [self-hosted, homelab]              │
│                                              │
│ 1. Checkout (submodules: true, full history) │
│ 2. nerdctl build (Hugo + nginx image)        │
│      tags: blog-debene-dev:latest            │
│            blog-debene-dev:<git-sha>          │
│      into the containerd k8s.io namespace    │
│ 3. Write kubeconfig from KUBE_CONFIG_B64      │
│ 4. kubectl -n blog rollout restart           │
│      deployment/blog                          │
│    kubectl -n blog rollout status (120s)      │
│ 5. Purge Cloudflare cache (purge_everything)  │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ K8s namespace: blog                  │
│ - Deployment/blog (nginx:alpine)     │
│   serves the freshly built image     │
│ - Service/blog (port 80)             │
│ - Deployment/cloudflared (tunnel)    │
└──────────────┬───────────────────────┘
               │
               ▼
┌──────────────────────────────────────┐
│ Cloudflare Tunnel (cloudflared pod) │
│ - Secure tunnel to Cloudflare edge  │
│ - No open firewall ports             │
└──────────────┬───────────────────────┘
               │
               ▼
       ┌───────────────┐
       │  Internet     │
       │ debene.dev    │
       └───────────────┘
```

### How the image is built

`Dockerfile` is a two-stage build:

1. **Builder** — `hugomods/hugo:std-0.157.0` copies the repo in and runs `hugo --minify`.
2. **Runtime** — `nginx:alpine` copies the generated `public/` directory to
   `/usr/share/nginx/html` and adds `nginx.conf`.

The whole site is baked into the image, so `Deployment/blog` serves static files straight from the
container — there is no shared storage volume. Because the deployment uses
`imagePullPolicy: IfNotPresent` with the `:latest` tag, the workflow ships new content with a
`kubectl rollout restart` rather than by changing the image reference.

### Required secrets

The workflow reads three repository secrets:

- `KUBE_CONFIG_B64` — base64-encoded kubeconfig for the cluster.
  Set it with: `base64 < ~/.kube/config | gh secret set KUBE_CONFIG_B64 -R <owner>/<repo>`
- `CF_ZONE_ID` — Cloudflare zone ID for `debene.dev`.
- `CF_API_TOKEN` — Cloudflare API token with cache-purge permission.

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

# GitHub Actions builds the image and rolls out the deployment.
# Watch the run: gh run watch -R felipedbene/debene-dev
```

The workflow handles everything:
1. ✅ Build the Hugo + nginx container image (`nerdctl build`)
2. ✅ Write the kubeconfig from the `KUBE_CONFIG_B64` secret
3. ✅ `kubectl rollout restart` + `rollout status` the `blog` deployment
4. ✅ Purge the Cloudflare cache
5. ✅ Live site update

## Monitoring

**Deploy runs (GitHub Actions):**
```bash
# Watch the latest deploy
gh run watch -R felipedbene/debene-dev

# List recent runs
gh run list -R felipedbene/debene-dev --workflow deploy.yml
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

# Check rollout status
kubectl -n blog rollout status deployment/blog
```

## Troubleshooting

**Deploy workflow failing:**
```bash
# Inspect the last run's logs
gh run view -R felipedbene/debene-dev --log-failed

# Common cause: KUBE_CONFIG_B64 secret missing/expired — the workflow
# errors out early in the "Write kubeconfig from secret" step.
```

**Hugo build failed:**
```bash
# Reproduce locally — the CI build runs the same command
hugo --minify
```

**Site not updating:**
```bash
# Check the deployment picked up the new image
kubectl -n blog rollout restart deployment/blog
kubectl -n blog rollout status deployment/blog

# Confirm the Cloudflare cache was purged (re-run the workflow if needed)
```

## About

Written from a basement in Chicago where an IBM POWER8 server runs Gentoo, AIX runs in KVM, and a
Kubernetes cluster ties it all together. The blog you're reading is built into an nginx container by
a GitHub Actions workflow on a self-hosted homelab runner, rolled out with `kubectl`, and delivered
via Cloudflare Tunnel — because why not? 🚀

## License

The site's code and configuration are released under the [MIT License](LICENSE).
Blog post prose and images remain © Felipe De Bene.
