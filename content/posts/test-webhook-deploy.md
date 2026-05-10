---
title: "🧪 Webhook Deploy Test"
date: 2026-05-10T18:17:00-05:00
draft: false
tags: ["test", "automation", "k8s"]
categories: ["infrastructure"]
---

## End-to-End Webhook Test

Testing the automated deployment pipeline:

1. ✅ **Create** this post locally
2. ✅ **Commit** to git
3. ✅ **Push** to GitHub
4. 🔄 **Webhook** detects push
5. 🏗️ **Hugo** builds static site
6. 📦 **Deploy** to K8s NFS storage
7. 🌐 **Live** on debene.dev

---

**Test timestamp:** 2026-05-10 18:17 CDT

If you're reading this, the webhook pipeline works! 🚀

**Pipeline components:**
- Git push → GitHub webhook
- K8s webhook pod (orion node)
- Hugo build (v0.152.2)
- NFS deployment
- Multi-arch blog pods (intel9 + orion)

**Expected deploy time:** ~5-10 seconds from push to live.

---

*This post will be deleted after successful test verification.*
