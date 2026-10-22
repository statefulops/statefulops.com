---
title: "Everything on StatefulOps — where to start"
date: 2026-03-25T08:04:57
lastmod: 2026-04-26T10:00:34
aliases:
  - /2026/03/25/start-here/
description: "Not sure where to start? Here’s everything on StatefulOps — three series covering Kubernetes fundamentals, GitOps with ArgoCD, and Kubernetes observability. 27 posts, written for engineers who’d rather read something…"
---

If you’ve landed here from a search or a shared link and you’re not sure where to start — this is the post for you.

StatefulOps covers the stuff that actually matters when you’re running production Kubernetes: getting the fundamentals solid, wiring up GitOps so deployments stop being scary, and building the observability layer so you know what’s happening at 2am when things go sideways. Three series, 27 posts, all written for engineers who’d rather read something honest than wade through another docs-flavoured tutorial.

Here’s everything, in the order I’d read it if I were starting fresh.

---

## Series 1 — Kubernetes fundamentals

The ground floor. If you’re new to Kubernetes or you’ve been using it for a while but feel like you’re cargo-culting your way through YAML, start here. We go from cluster architecture all the way through storage, networking, and resource management — the kind of stuff that turns “it works on my machine” into “it works on the cluster, and I can tell you exactly why.”

→ [Kubernetes Overview](/series/kubernetes-fundamentals/) — start here

---

## Series 2 — GitOps with ArgoCD

Once you’ve got a cluster you understand, you want deployments that don’t terrify you. This series covers GitOps from first principles through to running it at scale across multiple clusters and teams. ArgoCD is the tool, but the ideas apply everywhere.

→ [GitOps Overview](/series/gitops-with-argocd/) — start here

---

## Series 3 — Kubernetes observability

You can’t fix what you can’t see. This series covers the full observability stack: metrics, logs, traces, and dashboards. By the end you’ll have the foundation to actually understand what your cluster is doing instead of just hoping for the best.

→ [Observability Overview](/series/kubernetes-observability/) — start here

---

## What this site is

I’m a platform engineer working with Kubernetes at scale — hundreds of clusters, multi-cloud, the works. StatefulOps is where I write up the things I wish were written down when I was figuring them out. No fluff, no sponsored content, no “10 Kubernetes tips you won’t believe” energy.

If something’s useful, share it. If something’s wrong, tell me.
