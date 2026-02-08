---
title: "Trading off Relevance and Revenue in the Jobs Marketplace: Estimation, Optimization and Auction Design"
date: 2026-01-10
description: "A paper review on balancing relevance and revenue in jobs marketplace"
tags: ["ads-allocation", "optimization", "paper-review"]
---

> **Paper Review**: *Trading off Relevance and Revenue in the Jobs Marketplace: Estimation, Optimization and Auction Design*

## The Problem
When a job seeker searches on LinkedIn, the platform displays a ranked list of job postings. This paper focuses on the allocation stage, where the ranking must trade off short-term monetization against long-term marketplace health. The authors introduce two methods to improve relevance while preserving revenue: integrating seeker preference signals into the auction score and using position-aware auctions to better optimize multi-slot assignments.

## Background
If you rank jobs purely based on revenue, you end up favoring higher bidders of the promoted jobs, which can push less relevant jobs to the top. That might look good in the short term, but it eventually hurts user satisfaction and engagement. On the flip side, if you rank only by match quality, you risk leaving money on the table. The core challenge this paper tackles is finding the right balance between these two forces — short-term monetization and long-term user happiness — since both are critical to building a healthy, sustainable marketplace

A common way to compute the ranking score is:
$$
s_{ij} = S(b_{ij} \times \hat{\pi}_{ij})
$$
where $\hat{\pi}_{ij}$ is the estimated click-through rate of job $j$ for seeker $i$, and $b_{ij}$ is the bid value.

## Innovations

## Results


## Takeaways


## Comments



