---
title: "Smart Pacing for Effective Online Ad Campaign Optimization"
date: 2023-04-15
description: "A paper review on Yahoo's control-based budget pacing that balances spending plans and campaign performance"
tags: ["budget-pacing", "advertising", "optimization", "paper-review"]
---

> **Paper Review**: *Smart Pacing for Effective Online Ad Campaign Optimization* (Yahoo, KDD 2015)

## 1. The Problem
An advertiser running a campaign through a demand-side platform (DSP) usually wants several things at once:

- **Reach the delivery and performance goals.** For performance campaigns, the objective is to meet a performance target (e.g. eCPC no more than 2 dollars) while spending as much of the budget as possible.
- **Follow the budget spending plan.** Advertisers expect their ads to be shown smoothly throughout the flight period to reach a wider audience. Two common spending plans are *even pacing* (the same budget per time slot) and *traffic-based pacing* (budget proportional to expected traffic).
- **Reduce creative serving cost.** On top of what the DSP charges, every served impression also incurs a creative serving cost.

Achieving all of these simultaneously is hard. Each campaign has its own budget, spending plan, target audience, performance goal, billing method, and serving cost, and the marketplace itself keeps shifting. The question the paper asks is: **can we smartly decide which inventory the budget should be spent on, so that all the objectives of a campaign are met?**

The authors' answer is a control-based pacing method that learns from both offline and online data and optimizes budget pacing and campaign performance at the same time. It is deployed in a production DSP at Yahoo.

## 2. Paper Content
### 2.1 Background: Bid Modification vs. Probabilistic Throttling
There are two common families of pacing methods in the literature:

- **Bid modification** adjusts the bid price to change the win rate, and thereby the spend.
- **Probabilistic throttling** decides, for each incoming ad request, whether the campaign participates in the auction at all. The probability of participating is called the *pacing rate*.

The authors argue that throttling is better suited to real-time bidding (RTB) for three reasons:

1. The win-rate curve as a function of bid is usually not smooth, so small bid changes can cause large swings in spend. The bid landscape also shifts dramatically over time, which makes bid-based control unstable.
2. Supply-side platforms (SSPs) typically set reserve prices, so pacing fails whenever the controller needs to push the bid below the reserve.
3. Most importantly, **throttling decouples pacing control from bid calculation**. The pacing module can be built independently and combined with any bid optimization module.

The paper also positions itself relative to budget allocation work (e.g. jointly optimizing budget allocation and bidding). Budget allocation decides how much to spend at the campaign level; smart pacing then operates at the impression level to decide *which* requests to spend it on. Like prior work, the paper assumes that the available inventory is much larger than advertiser demand, so the problem is about selecting high-quality inventory rather than competing for scarce supply.

### 2.2 Problem Formulation
The paper focuses on the two most common campaign types:

1. **Branding campaigns**, which aim to spend out the budget and reach a broad audience.
2. **Performance campaigns**, which have explicit performance goals such as eCPC or eCPA.

Let $B$ be the campaign's budget and $G$ its performance goal (if any). A spending plan splits the budget across $K$ time slots, $\mathbf{B} = (B^{(1)}, \dots, B^{(K)})$ with $\sum_t B^{(t)} = B$.

For each ad request $i$, let $r_i$ be the pacing rate, $s_i \in \{0,1\}$ whether the campaign bids (with $\Pr(s_i = 1) = r_i$), $w_i$ whether it wins, $c_i$ the cost, and $q_i$ whether the user responds (e.g. clicks). Then the total cost and the performance of the campaign are

$$
C = \sum_i s_i \times w_i \times c_i, \qquad P = \frac{C}{\sum_i s_i \times w_i \times q_i}
$$

where $P$ is the eCPC when the desired response is a click. With $\mathbf{C} = (C^{(1)}, \dots, C^{(K)})$ denoting the actual spend per slot, a penalty function measures how far the spending pattern deviates from the plan:

$$
\Omega(\mathbf{C}, \mathbf{B}) = \sqrt{\frac{1}{K} \sum_{t=1}^{K} \left( C^{(t)} - B^{(t)} \right)^2}
$$

The three objectives (spend the budget, follow the plan, optimize performance) can conflict, so there are generally multiple Pareto-optimal solutions, i.e. solutions where no objective can be improved without making another worse. In practice, advertisers prioritize them differently depending on the campaign type. And since the system uses probabilistic throttling, the only variable it fully controls at serving time is the pacing rate $r_i$.

**Branding campaigns (no performance goal).** Spending the budget comes first, followed by sticking to the plan; performance is secondary:

$$
\min_{r_i} \; P \quad \text{s.t.} \quad C = B, \;\; \Omega(\mathbf{C}, \mathbf{B}) \le \epsilon
$$

**Performance campaigns.** Meeting the performance goal comes first, and following the spending plan matters least:

$$
\min_{r_i} \; \Omega(\mathbf{C}, \mathbf{B}) \quad \text{s.t.} \quad P \le G, \;\; B - C \le \varepsilon
$$

This framing covers different billing models too. For CPM campaigns, branding advertisers fall under the first problem and performance advertisers under the second. For CPC/CPA campaigns, there is an implicit performance goal that ensures the DSP does not lose money when bidding on the advertiser's behalf, so they also fall under the second problem. In every case, the intuition is the same: **high-responding ad requests should get higher pacing rates.** This improves performance for the advertiser, reduces wasted creative serving cost, and frees up low-quality opportunities for other campaigns.

### 2.3 Solution Overview
Solving these problems exactly in an online setting is impractical, so the authors approximate the solution in three steps:

1. **Response prediction.** Train a model on offline serving logs to estimate $p_i = \Pr(\text{respond} \mid \text{Req}\_i, \text{Ad})$, e.g. a pCTR model. The paper does not innovate here and relies on existing work.
2. **Grouping.** Instead of solving for a pacing rate per request, bucket requests with similar predicted response rates into groups that share a single *group pacing rate*. This shrinks the solution space from one variable per request to one per group.
3. **Online control.** Use a feedback controller to adjust the group pacing rates at the end of every time slot, based on how the campaign actually delivered.

### 2.4 A Layered Representation
Each campaign maintains $L$ layers, one per request group. For each layer, the system stores the average predicted response rate, its priority, its pacing rate, and the campaign's spend on that layer in the latest time slot. Two principles govern the layers:

1. Layers with higher response rates have higher priority.
2. A higher-priority layer never has a lower pacing rate than a lower-priority layer.

At serving time, when an eligible request arrives, the DSP finds the layer the request belongs to, looks up its pacing rate, and bids with that probability, at the price given by an upstream bid optimization module.

### 2.5 Online Pacing Rate Adjustment
Let $\mathbf{p} = (p_1, \dots, p_L)$ be the predicted response rate of each layer. For CPM billing with clicks as the response, the estimated eCPC of each layer is $e_l = \text{CPM} / (1000 \times p_l)$. Given the previous slot's pacing rates $\mathbf{r}^{(t-1)}$ and per-layer spend $\mathbf{c}^{(t-1)}$, the controller computes the new rates $\mathbf{r}^{(t)}$.

#### Campaigns without performance goals
At the end of each time slot, the controller first decides how much to spend in the next slot. Suppose that after $m$ slots the remaining budget is $B_m$. We want to plan the remaining spend $\widehat{C}^{(m+1)}, \dots, \widehat{C}^{(K)}$ so that the budget is fully spent and the penalty $\Omega$ is minimized. With the RMSE penalty above, the solution has a simple closed form: spread the leftover (or deficit) evenly over the remaining slots,

$$
\widehat{C}^{(t)} = B^{(t)} + \frac{B_m - \sum_{t'=m+1}^{K} B^{(t')}}{K - m}
$$

The *residual* $R = \widehat{C}^{(t)} - C^{(t-1)}$, the difference between the planned spend and the most recent actual spend, then drives the adjustment:

- If $R = 0$, keep the pacing rates unchanged.
- If $R > 0$ (we need to **speed up**), raise pacing rates **top-down**, starting from the highest-response layer. Each layer's rate is scaled up in proportion to how much additional spend is needed, capped at 1.0, and the remaining residual is passed down to the next layer.
- If $R < 0$ (we need to **slow down**), lower pacing rates **bottom-up**, starting from the lowest active layer, floored at 0.0.

The update for each layer $l$ assumes its spend scales linearly with its pacing rate:

$$
r_l^{(t)} = \min\left(1.0, \; r_l^{(t-1)} \times \frac{c_l^{(t-1)} + R}{c_l^{(t-1)}}\right), \qquad R \leftarrow R - c_l^{(t-1)} \times \frac{r_l^{(t)} - r_l^{(t-1)}}{r_l^{(t-1)}}
$$

In short: when spending is too fast, cut the low-pCTR layers first; when spending is too slow, open up the high-pCTR layers first. Within each slot, the budget is always invested in the best-performing inventory available under the budget and plan constraints. This greedy strategy approximates the optimal solution to the branding problem in an online setting. Since billing is CPM, shifting spend toward high-pCTR layers directly lowers eCPC.

#### Campaigns with performance goals
With a goal like eCPC ≤ 2 dollars, simply exploiting every request in the current slot that meets the goal is not necessarily optimal, since future traffic and response-rate distributions are hard to predict and can change over time.

The algorithm first runs the no-goal adjustment above to get candidate rates. It then estimates the joint eCPC of layers $i, \dots, L$ under the new rates:

$$
\text{ExpPerf}(\mathbf{c}^{(t-1)}, \mathbf{r}^{(t-1)}, \mathbf{r}^{(t)}, \mathbf{e}, i) = \frac{\sum_{j=i}^{L} c_j^{(t-1)} r_j^{(t)} / r_j^{(t-1)}}{\sum_{j=i}^{L} c_j^{(t-1)} r_j^{(t)} / \left( r_j^{(t-1)} e_j \right)}
$$

The numerator is the expected spend across layers in the coming slot, and the denominator is the expected number of clicks, so the ratio is the expected joint eCPC. If it exceeds the goal, the algorithm walks up from the lowest layer, shutting off layers entirely ($r_l^{(t)} = 0$) until the remaining layers can meet the goal, and partially throttles the boundary layer so that the combined eCPC lands right at the goal.

### 2.6 Number of Layers, Initial and Trial Rates
For a new campaign without delivery data, the system looks up the most similar existing campaigns and estimates a global pacing rate $r_G$ at which the new campaign is expected to spend its budget. The number of layers is then set to $L = \lceil 1 / r_G \rceil$. A moderate number of layers is preferred, because too many layers make per-layer delivery statistics noisy and consume extra bandwidth and memory.

The campaign runs at $r_G$ during the first slot, an *initialization phase* used to collect delivery data. Impressions are then split into $L$ equal-sized layers by predicted response rate to set the layer boundaries. From the next slot on, high-response layers get a pacing rate of 1.0 and low-response layers get 0.0, based on the planned budget.

One nice detail: the layer directly below the lowest active layer is given a small **trial pacing rate**. This keeps collecting delivery data from that layer, so the controller has accurate statistics ready when it needs to speed up. It is a simple form of exploration.

### 2.7 Experiments
The authors evaluate the approach on real campaigns and in offline simulations, looking at all three aspects: performance, budget spending, and spending pattern. On real campaigns, the approach was remarkably effective: compared to the baseline, eCPC dropped by **72%, 67%, and 79%** for the three campaigns tested, while still delivering the budget according to plan.

## 3. My Takeaway
What I like about this paper is how pragmatic it is. Rather than solving a hard multi-objective optimization problem exactly, it reduces the problem to a handful of group pacing rates, orders them by predicted response rate, and lets a simple feedback controller do the rest. The layered structure gives you performance optimization almost for free: under CPM billing, any spend shifted from low-pCTR layers to high-pCTR layers directly lowers eCPC. Decoupling pacing from bidding is also a valuable design principle, since it lets the two components evolve independently.

One caveat worth keeping in mind is the perspective the paper takes. The model is built from the DSP's point of view, where minimizing the advertiser's cost is the right objective. In a fully integrated ad platform, where the same company runs the SSP, the exchange, and the DSP (as with most in-app ad platforms today), aggressively minimizing advertiser cost can hurt platform revenue. As Wu Liangchao points out in his commentary on this paper, a more reasonable objective there is to bring the advertiser's actual cost *close to their expected cost*, rather than as low as possible.

The assumption that inventory far exceeds demand also matters. When inventory is limited relative to advertiser demand, which is common on smaller or more vertical platforms, throttling away low-pCTR requests may leave budgets unspent, and the balance between selectivity and delivery becomes much more delicate.

## 4. My Questions
1. The paper says budget allocation is complementary and happens at the campaign level before pacing. What exactly does budget allocation mean here: splitting an advertiser's budget across campaigns, or across channels and time?
2. The residual $R = \widehat{C}^{(t)} - C^{(t-1)}$ compares the planned spend for the *next* slot with the actual spend in the *previous* slot. If traffic volume differs a lot between the two slots (e.g. day vs. night), this residual mixes up a pacing error with a natural traffic change. It seems to implicitly assume roughly constant traffic per slot, or a spending plan that already follows traffic. Would it be better to normalize by forecasted traffic for the next slot?
