# CLAUDE.md - Blog Pipeline Notes

## Stack
- **SSG**: Hugo (PaperMod theme), built via `hugomods/hugo:exts` Docker image
- **Hosting**: nginx:alpine container in K8s namespace `blog` on xeon2socket
- **CDN/Tunnel**: Cloudflare Tunnel (cloudflared sidecar in same namespace)
- **Domain**: debene.dev (Cloudflare DNS, zone ID: `74b3f2beeed32cf30a06fffbe7f92400`)

## Build & Deploy Pipeline
1. Edit content in `/Users/felipe/.openclaw/workspace/blog-debene-dev/`
2. `rsync -az --delete` to `xeon2socket:~/blog-debene-dev/`
3. Build on MacStudio: `docker build --platform linux/amd64 -t blog-debene-dev:latest .`
   - Dockerfile uses `hugomods/hugo:exts` (NOT a pinned version — `exts-0.156.0` doesn't exist on Docker Hub)
4. Import to containerd on xeon2socket: `docker save blog-debene-dev:latest | ssh xeon2socket "sudo ctr -n k8s.io images import -"`
5. `kubectl rollout restart deploy/blog -n blog`
6. **Purge Cloudflare cache** after deploy (dashboard or API) — HTML gets cached by CF CDN

## Lessons Learned

### Cloudflare Cache
- Cloudflare caches HTML aggressively. After deploys, old content persists until cache expires or is purged.
- `curl -sI https://debene.dev/ | grep cf-cache` to check cache status.
- API token in `.env` (`CLOUDFLARE_API_TOKEN`) does NOT have cache purge permission — purge via dashboard or update token permissions.
- **TODO**: Add `Cache-Control: no-cache` for HTML in nginx.conf to prevent stale pages.

### Hugo / PaperMod
- `hugo.yaml` is the config file (not config.toml).
- `disableSpecial1stPost: true` — all posts render uniformly (no hero/featured post).
- No `paginate` set — defaults to Hugo's 10 (fine for now with 6 posts).
- Content lives in `content/posts/<slug>/index.md` with images in `content/posts/<slug>/images/`.
- **Don't put extra `---` after front matter** — it generates a `<hr>` at the top of the post.
- Hugo minifies output (`--minify` flag) — grep on raw HTML can be tricky with missing quotes.

### Container / K8s
- Image `blog-debene-dev:latest` with `imagePullPolicy: IfNotPresent` — must import to containerd on the node where pod runs.
- xeon2socket only has `ctr` and `crictl` — no docker/buildah/nerdctl. Build on MacStudio and pipe via `docker save | ssh ... ctr import`.
- The containerd namespace must be `k8s.io`: `sudo ctr -n k8s.io images import -`
- Pod runs on xeon2socket (no nodeSelector, just happens to schedule there).

### Ads
- Google AdSense configured via `googleAdsense` param in hugo.yaml.
- Ad partial in `layouts/partials/ads.html`, included via `extend_footer.html`.

### Content
- 6 posts as of 2026-02-25: aix-power8, elephants-cant-dance, powerpc, ibook-g4, instagram-recipe, harvester-salt.
- Cover images use Hugo's image processing (srcset with `_hu_` variants auto-generated).
