---
title: "Trading off Relevance and Revenue in the Jobs Marketplace: Estimation, Optimization and Auction Design"
date: 2026-01-10
description: "A paper review on balancing relevance and revenue in LinkedIn's jobs marketplace"
tags: ["ads-allocation", "optimization", "auction", "paper-review"]
---

> **Paper Review**: *Trading off Relevance and Revenue in the Jobs Marketplace: Estimation, Optimization and Auction Design*

## The Problem
When a job seeker searches on LinkedIn, the platform displays a ranked list of job postings. This paper focuses on the allocation stage, where the ranking must trade off short-term monetization against long-term marketplace health. The authors introduce two methods to improve relevance while preserving revenue: integrating seeker preference signals into the auction score and using position-aware auctions to better optimize multi-slot assignments.

## Background
If you rank jobs purely based on revenue, you end up favoring higher bidders of the promoted jobs, which can push less relevant jobs to the top. That might look good in the short term, but it eventually hurts user satisfaction and engagement. On the flip side, if you rank only by match quality, you risk leaving money on the table. The core challenge this paper tackles is finding the right balance between these two forces — short-term monetization and long-term user happiness — since both are critical to building a healthy, sustainable marketplace

A common way to compute the ranking score is:

$$s_{ij} = S(b_{ij}, \hat{\pi}_{ij})$$

where $\hat{\pi}\_{ij}$ is the estimated click-through rate of job $j$ for seeker $i$, and $b\_{ij}$ is the bid value. For example, S function could be just a multiplication of the two variable, which returns the expected revenue per impression level, and this ranking will prioritize the revenue.

The authors point out two main limitations of this ranking function:

1. Job seekers can have very different quality experiences. For example, in smaller cities or highly specialized job markets, there may be only a limited number of employers. Using a single global ranking formula in these cases fails to explicitly account for job relevance, which can lead to irrelevant promoted jobs being shown to seekers.

2. The predicted click-through rate is position-agnostic. It does not account for the fact that the same job can attract very different levels of attention depending on where it appears in the ranking. A job shown at the top of the page naturally receives more clicks than one shown further down. By greedily sorting candidates based on this position-unaware score, the system can end up with suboptimal job layouts across multiple slots.

To address these two issues, the authors introduce two improvements. First, they add an explicit relevance component to the ranking score to better capture how well a job $j$ matches a particular seeker $i$. Second, they propose position-aware auction scores, where the score of a job also depends on the position $k$ at which it is shown, acknowledging that a job’s attractiveness varies across different slots on the page.

## Relevance-Revenue Tradeoff

The authors introduce an explicit relevance term, $\hat{\mu}\_{ij}$, into the ranking score. This model-based estimate captures how relevant job $j$ is to seeker $i$. It represents the quality of the match from the seeker’s perspective, capturing how well a job aligns with the seeker’s preferences and qualifications. (The paper does not go into detail about how this relevance model is built. In practice, this would likely rely on richer signals such as user profiles, and their current job position's descriptions, potentially combined with human or LLM-based evaluation for fine-tuning.)

The augmented ranking score is:

$$s\_{ij} = S\big(b\_{ij} \cdot \hat{\pi}\_{ij},\; w\_i \cdot \hat{\mu}\_{ij}\big)$$

where $w\_i$ is the seeker-weight controlling the tradeoff between monetization and relevance. 
With this relevance signal, the key question becomes: how much weight should each seeker's relevance receive? The authors describe two approaches for estimating these seeker-weights.

1. **Causal Estimation**

   The authors model how relevance responds to changes in Seeker-Weight using a simple parametric function:

   $$Rel_g = F_g(\text{Seeker-Weight}) = z_g \cdot \text{Seeker-Weight}^{\alpha}$$

   This equation describes the relationship between the Seeker-Weight and the observed relevance metric for each user segment $g$. The parameter $z_g$ is learned separately for each segment and captures baseline relevance, while $\alpha$ is a globally shared elasticity parameter that models diminishing returns.

    The basic idea is to fit this response curve using randomized experiments, where different Seeker-Weight values are assigned to different cohorts. Once the parameters are learned, the model can be inverted: given a desired relevance outcome, the corresponding Seeker-Weight can be computed directly. This allows the platform to personalize relevance weights across user segments, helping maintain a target level of relevance for each group. The authors do not specify a concrete relevance metric or target in the paper; instead, relevance can be defined in whatever way best fits the platform’s goals—for example, average job relevance across the page or a weighted version that emphasizes top-ranked results.

   ![Causal estimation curve](../../images/posts/jobs-marketplace-graph_1.png)
   
   *After adding the relevance term, the overall seeker job relevance dispersion reduces, showing more consistent relevance across different user segments.*

2. Dynamic Optimization

## Auction Design

## Comments

 - If overall relevance dispersion is reduced, how does this not hurt short-term platform revenue?
 - what kind of relevance metric should the platform actually track?

