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

2. **Dynamic Optimization**
    The causal estimation approach helps determine relevance weights that, on average, achieve a desired relevance level at the request level. However, this raises a deeper question: does simply hitting a relevance target actually maximize long-term revenue from job seekers? Here, long-term revenue can be understood as future ad revenue driven by continued engagement—such as clicks, applications, or other monetizable interactions that result from a seeker’s ongoing use of the platform.

    This naturally leads to the second approach proposed in the paper: a reinforcement learning framework. In this setup, the relevance weight is treated as the action, while the state includes signals such as bids, relevance scores, and other job posting and seeker attributes. The goal of the RL framework is to learn an optimal policy for selecting relevance weights that maximizes long-term value rather than short-term outcomes.

    $$\sigma^{(m+1)}(x)=\arg\max_{\sigma(x)}\{\text{Gain}(x;\sigma(x))+\delta\mathbb{E}[V^{(m)}(x')\mid x,\sigma(x)]\}$$

    Here $\sigma(x)$ represents the policy, which determines the relevance weight to choose given the current state x. The state vector includes signals such as bids, estimated relevance scores, and other attributes related to the job seeker and job postings.



## Auction Design

The authors point out that the greedy approach, sorting jobs by a single ranking score and assigning them to slots from top to bottom is not optimal when multiple positions are involved. Instead, they propose using position-aware scores, where each job listing has a different score for each possible slot. Formally, a job $j$ has a score $s\_{ijk} = S(b\_{ij}\cdot \pi\_{ijk},\; w\_i \cdot \mu\_{ijk})$ when shown in position $k$.

Determining the optimal assignment of jobs to slots now requires solving a bipartite matching problem, which has a time complexity of $O(n^3)$. While there are approximation methods that can reduce this cost, the approach is still more expensive than greedy sorting. Through simulation, the authors show that using bipartite matching improves the average relevance of the search results compared to the greedy baseline.

![Comparison of relevance score over greedy and bipartite matching](../../images/posts/jobs-marketplace-graph_2.png)


## My Takeaway

- The causal estimation framework is relatively straightforward to implement and can work with a wide range of relevance metrics. This makes it more practical than framing the problem as a constrained optimization problem, where the decision variable must be explicitly tied to the relevance constraint. Rather than assuming a parametric relationship upfront, the framework learns this relationship directly from randomized experiments and extrapolates from observed data.

- While bipartite matching outperforms the greedy approach in terms of relevance, it comes with additional costs. Implementing this method requires building models to estimate position-aware pCTR and eRelevance for every candidate across all slots, which introduces significant engineering complexity. Moreover, it makes incorporating custom business rules more challenging—for example, constraints like limiting the number of consecutive promoted jobs. Such rules are easy to enforce in greedy ranking but much harder to integrate into a bipartite matching framework.

- The causal estimation approach reminds me of heterogeneous treatment effect estimation, where a similar goal is to understand how different values of Seeker-Weight affect relevance outcomes. In principle, this estimation could be pushed to a more granular level—for example, by modeling the baseline parameter $z_g$ as a function of job seeker attributes, such as profile information or past behavior, rather than relying solely on coarse user segments.

## My Questions

- Given the complexity of training reinforcement learning models in a live marketplace, it is reasonable to question how well the RL approach performs in practice. Job marketplaces are highly dynamic: bids evolve over time, and job campaigns actively compete with one another. The paper does not explicitly model these competitive interactions in the RL formulation, which raises questions about how robust the learned policies would be under real-world dynamics.
 
- How does adding the relevance term avoid hurting short-term platform revenue?

- What relevance metric should the platform actually track to guide these decisions?
