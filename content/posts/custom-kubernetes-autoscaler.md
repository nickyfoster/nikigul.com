---
author:
  name: "Nikita Gulyayev"
date: 2026-11-10
title: 'Why We Built a Custom Kubernetes Autoscaler, and the Math Behind It'
series:
- Articles
---

I recently published an article on the [Hydrolix blog](https://hydrolix.io/blog/) about hdx-scaler, the custom Kubernetes autoscaler we built and now run on 200+ production clusters.

**Read the full article:** [Why We Built A Custom Kubernetes Autoscaler, and the Math Behind It](https://hydrolix.io/blog/custom-kubernetes-autoscaler/)

## Overview

In the article, I cover:

- **The problem:** Our intake services were rejecting customer traffic with 429 errors, and on clusters without a request cap the same overload ended in OOM kills. CPU sat at a healthy-looking 35%, so the built-in autoscaler saw no reason to add pods.

- **Why standard autoscaling couldn't save us:** The built-in HPA scales on CPU and memory, which weren't the failure signals. KEDA needs a cluster-wide metrics adapter that we can't take over in customer-owned clusters. The standard VPA reacts to OOMs slowly and applies changes by evicting pods.

- **What we built:** A single Go daemon that scrapes pods directly, treats OOM kills and evictions as events, handles both horizontal and vertical scaling, and makes decisions every 5 seconds on any metric we choose.

- **The math behind it:** EWMA smoothing with bias correction, reservoir sampling, deadbands and cooldowns on the horizontal side; high-water marks, headroom calculations and a prioritized cascade of strategies on the vertical side.

- **Production results:** Over 30 days, roughly 2 million scale-ups and 2 million scale-downs across 200+ clusters, and 37% less reserved memory on one worker fleet.

If you're hitting the limits of CPU-based autoscaling or wondering what it takes to build your own, I hope you find this article useful.
