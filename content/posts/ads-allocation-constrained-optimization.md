---
title: "Ads Allocation in Feed via Constrained Optimization"
date: 2024-06-20
description: "A paper review on LinkedIn's approach to blending organic and ad content in feeds"
tags: ["ads-allocation", "optimization", "paper-review"]
---

> **Paper Review**: *Ads Allocation in Feed via Constrained Optimization* (LinkedIn, 2020)

## The Problem
Ad allocation is typically the final step of a recommender system. At this stage, organic content and ads are mixed together and delivered to users. This paper focuses on improving this final allocation process. This step alwasy involves two subproblems. 

First, how should we assign comparable values to organic content and ads, given that they are usually ranked separately and optimized for different objectives? To rank them together, their scores need to be aligned on the same scale. 

Second, once these items share a common ranking score, how can we allocate them across available slots in a way that maximizes the total value of the final layout?

This paper focuses specifically on the second subproblem, applying it to the LinkedIn feed.

## Paper Content
### Background
The authors begin by outlining a typical large-scale feed recommender system, which is usually composed of two independently optimized ranking pipelines: one for organic content and one for ads. Organic items are the primary driver of user engagement, as users interact with them based on intrinsic interest in the content or its creator. Accordingly, the objective of organic ranking systems is to maximize engagement-related signals such as clicks, dwell time, or other user actions. Ads, in contrast, are the main source of monetization, and ad ranking systems are typically optimized for revenue. A common objective for ads ranking is expected revenue per impression, often modeled as bid × pCTR (predicted click-through rate).

A key challenge is that engagement metrics from organic ranking and revenue-based metrics from ads ranking are not directly comparable, as they live on different scales and represent fundamentally different utilities. For organic content, the system cares primarily about predicted user engagement, whereas for ads, the critical metric is expected revenue. This mismatch motivates the need for a dedicated blending layer.

The blending layer takes the two separately ranked lists—organic items and ads—and produces a single unified feed that balances engagement and revenue. The paper emphasizes several practical requirements for this layer: it must operate under very low latency, respect the relative ordering produced by the upstream organic and ads rankers, and be easy to adapt and iterate on as business objectives evolve. While a fully unified ranking approach—jointly ranking organic and ads from the start—might appear conceptually simpler, the authors note that this is often impractical in real-world systems. In large tech companies, organic ranking and ads ranking are typically owned by different teams, optimized against different north-star metrics. Introducing a single unified objective would significantly increase system complexity and coordination costs.

For these reasons, the paper advocates for a two-stage architecture with a lightweight, post-ranking blending step, and formulates ads allocation in the feed as a constrained optimization problem within this blending layer.

![Overview to rank organic content and ads together](../../images/posts/ads-allocation-constrained-optimization_1.png)

The authors also introduce business guardrails to prevent unintended member experiences. These guardrails take the form of heuristic constraints, such as (1) a top-slot constraint, which restricts ads from appearing within the first x positions of the feed, and (2) a minimum-gap constraint, which enforces a minimum separation between two consecutive ads.

### Formulation and Algorithm

Next, the authors frame the problem as a constrained optimization task. For each slot $i$, they introduce a binary decision variable $x_i$ that indicates whether an ad is placed in that slot. The objective is to maximize the total revenue per request, subject to a constraint that total user engagement is greater than or equal to a constant $C$.

The constant $C$ is a business decision and can be chosen as a fraction of the maximum achievable engagement—for example, the engagement level when no ads are shown. To solve this problem online as new requests arrive, the authors apply a Lagrangian dual formulation to derive an “optimal” primal serving policy. Finally, they incorporate the two business guardrails discussed earlier into the serving plan to ensure the final allocation satisfies platform requirements.

![Constrained Optimization Framing per Request](../../images/posts/ads-allocation-constrained-optimization_2.png)
In the above formulation, the objective is to maximize revenue across all impressions, spanning multiple requests and multiple slots per request. The term $\frac{w}{2}\lVert x \rVert^2$ is added to enforce strong convexity, which makes the conversion from the dual problem back to the primal problem more tractable.

After solving the Lagrangian dual, the optimal policy is to place an ad in slot $i$ whenever $r_i^a + \alpha u_i^a - \alpha u_i^o > 0.$. Here, $\alpha$ is the dual variable associated with the engagement constraint and is determined by the choice of $C$. In simpler terms, if the revenue from showing an ad plus its engagement value—scaled by $\alpha$ exceeds that of the organic candidate, the platform will choose to display the ad. Prior to this stage, both organic and ad candidates have already been ranked separately, so the “next” candidate from each list is already determined when this comparison is made.

The authors propose three ways to determine the value of $\alpha$:
1. Solving the dual formulation using historical data.
2. Running online A/B experiments to empirically estimate the optimal $\alpha$.
3. Using offline replay simulations to derive an appropriate value.

An interesting observation highlighted in the paper is the **Gap Effect**. The authors note that an ad’s click-through rate (CTR) is strongly correlated with the distance from the previous ad. When two ads appear close to each other, the second ad tends to receive fewer clicked, leading to a noticeable drop in CTR.

To account for this effect, the authors introduce a simple linear correction to the ad’s pCTR based on its gap from the previous ad. With this adjustment, the decision rule for favoring an ad over an organic result becomes:

$$
\theta_d \bigl(r_i^a + \alpha u_i^a\bigr) - \alpha u_i^o > 0.
$$

Here, $\theta_d$ represents the gap-dependent correction factor applied to the ad, capturing the reduced pCTR of ads shown too close together.

![Gap Effect](../../images/posts/ads-allocation-constrained-optimization_3.png)

When two ads appear closer together, the click-through rate of the second ad drops more sharply. This effect is quite pronounced when the gap is small and gradually fades as the gap increases, becoming negligible once the gap exceeds 5 slots.

Below is the complete serving plan, built on the optimal policy (derived from Lagrangian dual) and extended with the two business guardrails and the gap-effect correction.

![Serving Plan](../../images/posts/ads-allocation-constrained-optimization_4.png)

### Experimentation

## My Takeaway

From the paper, this appears to be one of the first industry works to openly discuss ad–organic allocation practices in internet feed systems. The proposed serving plan is relatively simple to implement, and the framework for reasoning about tradeoffs between revenue and engagement is clear and intuitive.

### Is Hard Rule Optimal?

Mixed ranking (organic + ads) often involves various hard business rules, such as restricting ads from appearing in the first position, enforcing a minimum gap between ads, or applying show-time gap constraints. From a technical perspective, these discrete hard rules are clearly not optimal, as they significantly limit the search space of the algorithm. From a business perspective, these rules are often defined early in the lifecycle of the product; as the business evolves, whether they are still appropriate for the current stage needs to be re-evaluated.

For example, applying the same set of hard rules to both ad-sensitive users and ad-insensitive users is not optimal. Under the same ad frequency, ad-sensitive users tend to have lower conversion rates and experience a larger negative impact on retention (relative to ad-insensitive users). If we instead show more ads to ad-insensitive users while reducing ads for ad-sensitive users—keeping the total number of ads roughly constant—the overall engagement utility and revenue utility should, in theory, be improved.

In practice, this usually requires defining a set of rules and corresponding computation methods that can evaluate the utility under all possible scenarios permitted by the hard constraints, and then incorporating these utilities into the overall value of the ranked list.

This also points to a broader optimization direction in recommender and advertising systems: personalization. Even when the high-level strategy or hyperparameters are shared across all users, there is often significant room to personalize these policies for different user segments.

### Is the global constraint enough ?

The original paper places the constraint at the total impression level, which may align with platform-level preferences. However, one might ask whether applying constraints at the user level which ensuring a more consistent minimum threshold of experience quality across users would be more appropriate.

## My Questions

What are effective ways to define engagement utility? Is a single engagement metric sufficient to capture the tradeoffs in user experience? And how should short-term engagement signals be connected to long-term user satisfaction and retention?