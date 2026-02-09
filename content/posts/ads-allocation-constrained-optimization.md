---
title: "Ads Allocation in Feed via Constrained Optimization"
date: 2024-06-20
description: "A paper review on LinkedIn's approach to blending organic and ad content in feeds"
tags: ["ads-allocation", "optimization", "paper-review"]
---

> **Paper Review**: *Ads Allocation in Feed via Constrained Optimization* (LinkedIn, 2020)

## The Problem
Ad allocation is typically the final step of a recommender system. At this stage, organic content and ads are mixed together and delivered to users. This paper focuses on improving this final allocation process. It addresses two key subproblems. First, how should we assign comparable values to organic content and ads, given that they are usually ranked separately and optimized for different objectives? To rank them together, their scores need to be aligned on the same scale. Second, once these items share a common ranking score, how can we allocate them across available slots in a way that maximizes the total value of the final layout?

This paper focuses specifically on addressing the second subproblem, and the application is theLinkedIn feed.

## Background

The authors begin by outlining a typical large-scale feed recommender system, which is usually composed of two independently optimized ranking pipelines: one for organic content and one for ads. Organic items are the primary driver of user engagement, as users interact with them based on intrinsic interest in the content or its creator. Accordingly, the objective of organic ranking systems is to maximize engagement-related signals such as clicks, dwell time, or other user actions. Ads, in contrast, are the main source of monetization, and ad ranking systems are typically optimized for revenue. A common objective for ads ranking is expected revenue per impression, often modeled as bid × pCTR (predicted click-through rate).

A key challenge is that engagement metrics from organic ranking and revenue-based metrics from ads ranking are not directly comparable, as they live on different scales and represent fundamentally different utilities. For organic content, the system cares primarily about predicted user engagement, whereas for ads, the critical metric is expected revenue. This mismatch motivates the need for a dedicated blending layer.

The blending layer takes the two separately ranked lists—organic items and ads—and produces a single unified feed that balances engagement and revenue. The paper emphasizes several practical requirements for this layer: it must operate under very low latency, respect the relative ordering produced by the upstream organic and ads rankers, and be easy to adapt and iterate on as business objectives evolve. While a fully unified ranking approach—jointly ranking organic and ads from the start—might appear conceptually simpler, the authors note that this is often impractical in real-world systems. In large tech companies, organic ranking and ads ranking are typically owned by different teams, optimized against different north-star metrics. Introducing a single unified objective would significantly increase system complexity and coordination costs.

For these reasons, the paper advocates for a two-stage architecture with a lightweight, post-ranking blending step, and formulates ads allocation in the feed as a constrained optimization problem within this blending layer.

![Overview to rank organic content and ads together](../../images/posts/ads-allocation-constrained-optimization_1.png)

## My Takeaway

## My Questions
