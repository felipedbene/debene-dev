# debene.dev

Personal blog running on [Hugo](https://gohugo.io/) with the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. Self-hosted on a Kubernetes homelab, served via Cloudflare Tunnel.

## Stack

```
Internet → Cloudflare Tunnel (QUIC) → K8s Pod (cloudflared) → nginx (static files)
```

- **Generator**: Hugo 0.156.0
- **Theme**: PaperMod
- **Container**: nginx:alpine serving pre-built static files
- **Hosting**: Kubernetes (kubeadm, 4-node homelab cluster)
- **CDN/Tunnel**: Cloudflare Tunnel (Zero Trust)
- **Domain**: [debene.dev](https://debene.dev)

## Local Development

```bash
# Install Hugo
brew install hugo

# Clone with submodules (theme)
git clone --recurse-submodules https://github.com/felipedbene/debene-dev.git
cd debene-dev

# Run dev server
hugo server -D

# Build for production
hugo --minify
```

## Deploy

```bash
# Build container image (linux/amd64)
docker build --platform linux/amd64 -t blog-debene-dev:latest -f Dockerfile.simple .

# Export and import to K8s nodes
docker save blog-debene-dev:latest | gzip > /tmp/blog-image.tar.gz
scp /tmp/blog-image.tar.gz <node>:/tmp/
ssh <node> "gunzip -c /tmp/blog-image.tar.gz | sudo ctr -n k8s.io images import -"

# Apply K8s manifests
kubectl apply -f k8s/deployment.yaml
```

## Writing

New posts go in `content/posts/<slug>/index.md` with images in an `images/` subdirectory (Hugo page bundles).

## About

Written from a basement in Chicago where an IBM POWER8 server runs Gentoo, AIX runs in KVM, and a Kubernetes cluster ties it all together.
