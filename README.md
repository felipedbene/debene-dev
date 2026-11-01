# debene.dev

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE) ![Hugo](https://img.shields.io/badge/Hugo-static%20site-ff4088.svg) [![Build and deploy](https://github.com/felipedbene/debene-dev/actions/workflows/deploy.yml/badge.svg)](https://github.com/felipedbene/debene-dev/actions/workflows/deploy.yml)

Read it live at [debene.dev](https://debene.dev).

Personal blog built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, hosted on **GitHub Pages**.

**✨ Fully automated:** push to `main` → GitHub Actions builds the site with Hugo → GitHub Pages serves it.

## Stack

```
GitHub push to main
        │
        ▼
GitHub Actions (.github/workflows/deploy.yml)
  hugo build --gc --minify  →  upload-pages-artifact
        │
        ▼
GitHub Pages (custom domain: debene.dev, HTTPS enforced)
        │
        ▼
     Internet
```

- **Generator**: Hugo `0.162.1` extended (pinned in the workflow via `HUGO_VERSION`)
- **Theme**: PaperMod (git submodule under `themes/PaperMod`)
- **Hosting**: GitHub Pages, deployed from Actions (`build_type: workflow`)
- **Domain**: [debene.dev](https://debene.dev), set in the repo's Pages settings

No servers, containers, or secrets are needed.

## Deployment pipeline

`.github/workflows/deploy.yml` runs on every push to `main`, and can also be started by hand with `workflow_dispatch`:

**`build` job** (`ubuntu-latest`)
1. Checkout with submodules and full history (Hugo uses Git info for `lastmod`).
2. `actions/configure-pages`.
3. Install Dart Sass and Hugo (versions pinned in the workflow `env`).
4. Restore the Hugo cache, then run `hugo build --gc --minify`, then save the cache.
5. `actions/upload-pages-artifact` uploads `./public`.

**`deploy` job**
1. `actions/deploy-pages` publishes the artifact to the `github-pages` environment.

> **Note:** the site URL comes from `baseURL` in `hugo.yaml` (`https://debene.dev/`). Don't pass
> `configure-pages`' `base_url` to Hugo. It follows the Pages "Enforce HTTPS" setting and can come
> back as `http://`. Every image and asset URL would then be `http://`, and browsers block them as
> mixed content.

## Local development

```bash
# Install Hugo (extended)
brew install hugo          # macOS
sudo dnf install hugo      # Fedora

# Clone with submodules (theme)
git clone --recurse-submodules https://github.com/felipedbene/debene-dev.git
cd debene-dev
# (already cloned? run: git submodule update --init)

# Dev server with drafts
hugo server -D

# Production build (same as CI)
hugo build --gc --minify
```

## Writing posts

**Create a new post:**
```bash
hugo new content/posts/my-post-slug/index.md
```

**Directory structure (page bundles):**
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

## Deploy (automatic)

```bash
git add .
git commit -m "New post: My Awesome Article"
git push origin main

# Watch the run
gh run watch -R felipedbene/debene-dev
```

## Monitoring & troubleshooting

```bash
# Recent deploy runs
gh run list -R felipedbene/debene-dev --workflow deploy.yml

# Logs from the last failed run
gh run view -R felipedbene/debene-dev --log-failed

# Pages configuration (custom domain, HTTPS, build type)
gh api repos/felipedbene/debene-dev/pages
```

- **Hugo build failed:** run `hugo build --gc --minify` locally to reproduce it.
- **Theme missing / blank pages locally:** run `git submodule update --init`.
- **Images not loading:** check that the generated URLs are `https://`. `https_enforced` should be
  `true` in the Pages config, and the workflow must not override `baseURL`.

## About

Written from a basement in Chicago, where an IBM POWER8 server runs Gentoo, AIX runs in KVM, and a
Kubernetes cluster ties it all together. The blog itself takes the boring route: static files on
GitHub Pages. 🚀

## License

The site's code and configuration are released under the [MIT License](LICENSE).
Blog post prose and images remain © Felipe De Bene.
