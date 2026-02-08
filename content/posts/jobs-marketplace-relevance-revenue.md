---
title: "Trading off Relevance and Revenue in the Jobs Marketplace: Estimation, Optimization and Auction Design"
date: 2026-01-10
description: "A paper review on balancing relevance and revenue in LinkedIn's jobs marketplace"
tags: ["ads-allocation", "optimization", "paper-review"]
---

> **Paper Review**: *Trading off Relevance and Revenue in the Jobs Marketplace: Estimation, Optimization and Auction Design*

## The Problem
When a job seeker searches on LinkedIn, the platform displays a ranked list of job postings. This paper focuses on the allocation stage, where the ranking must trade off short-term monetization against long-term marketplace health. The authors introduce two methods to improve relevance while preserving revenue: integrating seeker preference signals into the auction score and using position-aware auctions to better optimize multi-slot assignments.

## Background
If you rank jobs purely based on revenue, you end up favoring higher bidders of the promoted jobs, which can push less relevant jobs to the top. That might look good in the short term, but it eventually hurts user satisfaction and engagement. On the flip side, if you rank only by match quality, you risk leaving money on the table. The core challenge this paper tackles is finding the right balance between these two forces — short-term monetization and long-term user happiness — since both are critical to building a healthy, sustainable marketplace

A common way to compute the ranking score is:

$$s_{ij} = S(b_{ij}, \hat{\pi}_{ij})$$

where $\hat{\pi}_{ij}$ is the estimated click-through rate of job $j$ for seeker $i$, and $b_{ij}$ is the bid value. For example, S function could be just a multiplication of the two variable, which returns the expected revenue per impression level, and this ranking will prioritize the revenue.

The authors point out two main limitations of this ranking function:

1. Job seekers can have very different quality experiences. For example, in smaller cities or highly specialized job markets, there may be only a limited number of employers. Using a single global ranking formula in these cases fails to explicitly account for job relevance, which can lead to irrelevant promoted jobs being shown to seekers.

2. The predicted click-through rate is position-agnostic. It does not account for the fact that the same job can attract very different levels of attention depending on where it appears in the ranking. A job shown at the top of the page naturally receives more clicks than one shown further down. By greedily sorting candidates based on this position-unaware score, the system can end up with suboptimal job layouts across multiple slots.

## Innovations

## Results


## Takeaways


## Comments



