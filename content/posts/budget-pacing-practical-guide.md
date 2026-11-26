---
title: "A Practical Guide to Budget Pacing Algorithms in Digital Advertising"
date: 2025-06-12
description: "From the Lagrangian formulation of max delivery and cost cap to the feedback loops and segment-level multipliers used in production"
tags: ["budget-pacing", "advertising", "auction", "optimization"]
---

> This post collects my notes on budget pacing from various resources, combined with what I have learned working on ads systems. The goal is to connect the textbook formulation with what actually runs in production.

## 1. Bidding Products
Before talking about pacing, it helps to be clear about what an advertiser is actually buying. Ad platforms offer a range of bidding products, which differ in their objective, what the system optimizes for, and how the advertiser is charged.

| Bidding Product | Objective | Optimization Goal | Charging Model |
|---|---|---|---|
| **CPC** (Cost Per Click) | Drive traffic to a website or landing page | Maximize clicks | Pay per click |
| **eCPC** (Enhanced CPC) | Increase conversions while keeping control over CPC | Adjust bids dynamically to win clicks more likely to convert | Pay per click (bids adjusted dynamically) |
| **oCPC** (Optimized CPC) | Drive more conversions by optimizing for higher-value clicks | Optimize delivery for clicks likely to convert | Pay per click |
| **CPM** (Cost Per Mille) | Maximize brand awareness and visibility | Maximize impressions and reach | Pay per 1,000 impressions |
| **oCPM** (Optimized CPM) | Drive conversions with impression-based delivery | Optimize impressions for users most likely to act | Pay per 1,000 impressions |
| **CPA** (Cost Per Action) | Acquire conversions efficiently | Maximize conversions at a target cost per action | Pay per completed action |
| **CPV** (Cost Per View) | Maximize video engagement | Optimize for video views | Pay per completed view |
| **CPL** (Cost Per Lead) | Generate high-quality leads | Maximize lead form submissions | Pay per lead |

The progression from CPC to eCPC to oCPC is a good way to see how much control shifts from the advertiser to the platform:

- **CPC**: the advertiser sets the bid, and the platform uses it as is.
- **eCPC**: the advertiser still sets the bid, but the platform adjusts it up or down to win clicks that are more likely to convert.
- **oCPC**: the platform has much more freedom. It no longer follows the advertiser's bid directly; conversions become the main objective, and the bid itself is set to optimize for conversions.

This shift is exactly why pacing algorithms matter. Once the platform is deciding the bid, it is also responsible for spending the budget well.

## 2. Auction Basics: Incentive Compatibility
A mechanism (an allocation rule plus a pricing rule) is **incentive compatible**, or more precisely *dominant-strategy incentive compatible* (DSIC), if for every participant, bidding their true valuation is the best strategy regardless of what everyone else bids.

- With a **single slot**, GSP reduces to the second-price auction, which is DSIC: truthful bidding is a dominant strategy.
- With **multiple slots**, ranking ads by bid × pCTR and charging with **GSP** is *not* DSIC.
- Ranking ads by bid × pCTR and charging with **VCG** *is* DSIC.

DSIC is a much stronger property than a Nash equilibrium. DSIC means truthful bidding is optimal *no matter what others do*; a Nash equilibrium only means no one wants to deviate *given the current strategies of others*.

The VCG mechanism has three attractive properties:

- **Efficiency**: the allocation maximizes social welfare, i.e. the total valuation of the ads shown.
- **Incentive compatibility**: truthful bidding is the dominant strategy, because a bidder's payment depends only on the valuations of the others, not on their own bid.
- **Fairness**: each participant pays exactly the externality they impose on others.

## 3. Max Delivery
In everything below, assume impressions are charged on a CPM basis, the objective is to maximize total clicks, and each auction has a single slot with second-price pricing.

### 3.1 Formulation
**Max delivery** is the most fundamental pacing product: the advertiser sets a budget, and the system tries to get as many clicks as possible within it.

$$
\max_{x_t \in \\{0,1\\}} \sum_{t=1}^{T} x_t \cdot r_t \quad \text{s.t.} \quad \sum_{t=1}^{T} x_t \cdot c_t \le B
$$

where:
- $T$ is the total (predicted) number of auction opportunities in a day.
- $r_t$ is the predicted CTR of the $t$-th auction.
- $c_t$ is the cost of the $t$-th auction. In a second-price auction, this is the highest competing eCPM.
- $x_t$ indicates whether we win the $t$-th auction. Under second-price rules, $x_t = \mathbb{1}\\{b_t > c_t\\}$, where $b_t$ is our bid per impression.

We assume that $\\{r_t\\}$ and $\\{c_t\\}$ are drawn i.i.d. from some unknown distribution, e.g. log-normal.

### 3.2 Solving It with the Lagrangian
Solving this problem directly is hard, so we move to the dual space. The Lagrangian is:

$$
\mathcal{L}(x, \lambda) = \sum_{t=1}^{T} x_t \cdot r_t - \lambda \left( \sum_{t=1}^{T} x_t \cdot c_t - B \right) = \sum_{t=1}^{T} x_t (r_t - \lambda c_t) + \lambda B
$$

For a fixed $\lambda$, maximizing over $x$ is trivial: set $x_t = 1$ whenever $r_t - \lambda c_t > 0$, and $x_t = 0$ otherwise. Plugging this back in gives the dual function:

$$
\mathcal{L}^{\star}(\lambda) = \sum_{t=1}^{T} (r_t - \lambda c_t)\_{+} + \lambda B
$$

where $(z)\_{+} = \max(z, 0)$. The dual problem is then

$$
\min_{\lambda \ge 0} \sum_{t=1}^{T} \left[ (r_t - \lambda c_t)\_{+} + \lambda \cdot \frac{B}{T} \right]
$$

Let $\lambda^{\star}$ be its solution. By the KKT conditions, if $\lambda^{\star} > 0$, the budget constraint is binding: $\sum_t x_t c_t = B$, i.e. the budget is exactly spent.

### 3.3 The Optimal Bid
Here is the elegant part. The optimal rule says: win auction $t$ if and only if $r_t / \lambda^{\star} > c_t$. In a second-price auction, we win exactly when our bid exceeds $c_t$. So if we simply bid

$$
b_t^{\star} = \frac{r_t}{\lambda^{\star}}
$$

we win precisely the auctions we *should* win and lose precisely the ones we should not. The bid itself implements the optimal selection rule. Equivalently, the optimal **bid per click** is a constant:

$$
b_{\text{click}}^{\star} = \frac{1}{\lambda^{\star}}
$$

A few ways to read this:
- The Lagrangian turns the budget constraint into a **price** $\lambda$ on spend.
- The optimal policy only wins impressions whose value per cost $r_t / c_t$ is high enough, with $\lambda$ as the threshold.
- In practice, we implement this by bidding $r_t / \lambda$ and tuning $\lambda$ until the budget constraint is met.

Since the optimal bid per click is constant and the auctions are i.i.d., the expected spend per auction opportunity is also constant. That means **spend should be proportional to auction volume**: the ideal spending curve follows the traffic curve over the day.

### 3.4 From Theory to Production
We never know $\lambda^{\star}$ in advance, because we do not know the future auctions. In a real system, it is found in two steps:

1. **Initialize $\lambda$** by replaying historical auction data offline and finding a value that roughly satisfies the budget constraint.
2. **Adjust $\lambda$ online** (this is the pacing part):
   - Spending too fast → increase $\lambda$ → lower bids → win less.
   - Spending too slow → decrease $\lambda$ → higher bids → win more.

For a long time, I thought of these as two separate things: the Lagrangian gives a nice formula for the initial bid, and then a feedback controller that compares metrics against targets takes over online, more or less unrelated to the math. That is not quite right. **The feedback loop is itself optimizing the dual.**

By the envelope theorem, the gradient of the dual function with respect to $\lambda$ is just the partial derivative of the Lagrangian evaluated at the optimal $x^{\star}(\lambda)$:

$$
\frac{\partial \mathcal{L}^{\star}}{\partial \lambda} = B - \sum_t x_t^{\star}(\lambda) \\, c_t = B - \text{spend}(\lambda)
$$

In other words, the dual gradient *is* the constraint violation: budget minus actual spend. A pacing controller that nudges $\lambda$ up when spend exceeds the target and down when it falls short is doing stochastic (sub)gradient descent on the dual, using observed spend as a noisy gradient. The normalization and PID-style terms we add in production are there to make this gradient step stable.

## 4. Cost Cap
### 4.1 Formulation
**Cost cap** is a product for price-sensitive advertisers. In addition to a budget, the advertiser sets an upper limit $C$ on the average cost per result. For an oCPM daily campaign optimizing for clicks:

$$
\max_{x_t \in \\{0,1\\}} \sum_{t=1}^{T} x_t \cdot r_t \quad \text{s.t.} \quad \sum_{t=1}^{T} x_t \cdot c_t \le B, \quad \frac{\sum_{t} x_t \cdot c_t}{\sum_{t} x_t \cdot r_t} \le C
$$

### 4.2 Solution
The derivation follows the same steps as max delivery, with a second dual variable $\mu$ for the cost constraint (rewritten linearly as $\sum_t x_t c_t \le C \sum_t x_t r_t$). We should win auction $t$ when

$$
r_t - \lambda c_t - \mu c_t + \mu C r_t > 0 \quad \Longleftrightarrow \quad \frac{1 + \mu C}{\lambda + \mu} \cdot r_t > c_t
$$

which again translates directly into an optimal bid per impression:

$$
b_t^{\star} = \frac{1 + \mu^{\star} C}{\lambda^{\star} + \mu^{\star}} \cdot r_t
$$

The bid is now controlled by two dual variables. With a bit of algebra, the bid per click can be written as

$$
b_{\text{click}}^{\star} = \frac{1}{\lambda^{\star} + \mu^{\star}} + \frac{\mu^{\star}}{\lambda^{\star} + \mu^{\star}} \cdot C = \alpha \cdot \frac{1}{\lambda^{\star}} + (1 - \alpha) \cdot C, \qquad \alpha = \frac{\lambda^{\star}}{\lambda^{\star} + \mu^{\star}}
$$

This form is very intuitive. Compared to max delivery, the bid per click is now a **weighted blend** of the max-delivery bid $1/\lambda^{\star}$ and the cost cap $C$. When the cost constraint is not binding ($\mu^{\star} = 0$, so $\alpha = 1$), we are back to max delivery. The tighter the cost constraint, the more the bid is pulled toward the cap. In production, $\alpha$ acts like a knob that balances the two constraints based on how spend and CPA are tracking.

### 4.3 Two Feedback Loops in Production
With two constraints, production systems typically run two feedback loops. One tracks spend against the pacing target:

$$
e_{\text{spend}}(t) = \frac{S(t) - S_{\text{target}}(t)}{S_{\text{target}}(t)}
$$

and one tracks CPA against the cap:

$$
e_{\text{cpa}}(t) = \frac{\text{CPA}(t) - C}{C}
$$

The control logic is:
- Spending too fast → lower the bid.
- Spending too slow → raise the bid.
- CPA too high → lower the bid.
- CPA well below the cap → room to bid more aggressively.

A natural implementation is a bid multiplier that combines both errors:

$$
\log m_t = -k_s \\, e_{\text{spend}}(t) - k_c \\, e_{\text{cpa}}(t)
$$

or, equivalently, two separate multipliers $m_t = m_{\text{spend},t} \cdot m_{\text{cpa},t}$. These are the online counterparts of $\lambda$ and $\mu$.

So in the end, production pacing is really control theory. The Lagrangian tells us the *form* of the optimal bid and gives a good starting point; the controllers then keep it on target.

### 4.4 When the Problem Has No Solution
There is a catch: depending on the quality of the available traffic, the two constraints may simply not be satisfiable at the same time. Raising the bid increases spend, but it also increases CPC; lowering the bid brings CPC down, but spend drops with it. If there is not enough cheap, high-quality traffic, no bid meets both targets.

A few practical ways to handle this:
- Replay historical traffic offline to check whether both targets are achievable before launching.
- Define an explicit priority between the two goals (e.g. the cost cap is a hard limit, spend is best effort).
- Add clipping and guardrails on the multipliers so that neither goal is violated by too much while the controller chases the other.

## 5. Limitations of the Lagrangian Solution
The formulation above is clean, but it rests on assumptions that do not fully hold in practice:

- The decisions $x_t$ are discrete (0 or 1), so the dual solution is only approximately optimal for the original integer problem. With many small auctions, this gap is usually negligible.
- Auctions are stochastic and not truly i.i.d. Traffic volume, pCTR, and clearing prices shift by hour of day, region, query, and more.

The second point leads to the next question.

## 6. Beyond a Single Global λ
### 6.1 What a Global λ Does and Does Not Guarantee
So far, all traffic shares a single $\lambda$. That $\lambda$ defines a threshold rule: an impression is won if and only if

$$
\frac{r_t}{c_t} \ge \lambda
$$

so $\lambda$ is the **minimum value-per-cost threshold** for winning an impression.

Applying the same threshold everywhere does *not* mean that the impressions we win are equally efficient across segments. If region A has better traffic than region B, then

$$
\mathbb{E}\left[\frac{r}{c} \\,\middle|\\, \frac{r}{c} \ge \lambda, z = A\right] \neq \mathbb{E}\left[\frac{r}{c} \\,\middle|\\, \frac{r}{c} \ge \lambda, z = B\right]
$$

and the global $\lambda$ will naturally buy more traffic in region A. This is a good thing, and the differing averages on their own are not a problem. The optimality condition for maximizing total clicks under a budget is about the **margin**, not the average: the marginal value per unit of cost must be equal across every segment that receives spend,

$$
\frac{\partial \\, \text{value}\_z}{\partial \\, \text{spend}\_z} = \lambda^{\star} \quad \text{for all } z \text{ with } \text{spend}\_z > 0
$$

In the idealized model, a global threshold already satisfies this: the last impression we buy in every segment has $r_t / c_t = \lambda^{\star}$.

### 6.2 Why Segment-Level Multipliers Help in Practice
The idealized argument breaks down in real systems, and that is where a global $\lambda$ falls short:

- **The value estimate is biased by segment.** $r_t$ is a model prediction, and pCTR calibration often differs by region, device, or placement. If pCTR is overestimated in region B, the global threshold overbuys there, and its true marginal value per dollar is lower than elsewhere.
- **The value of a click differs by segment.** If the real objective is conversions or revenue, a click in one region may be worth more than a click in another, which pCTR alone does not capture.
- **Supply is not stationary.** Different segments peak at different times of day. A single controller reacts to the aggregate, so it can be too aggressive in one segment and too conservative in another at any given moment.

In these cases, the solution is to give each segment its own effective threshold, $\lambda_z$, and tune those thresholds so that marginal value per dollar ends up equal across segments:

- In segments with better traffic, lower $\lambda_z$ to buy more of it.
- In segments with worse traffic, raise $\lambda_z$ to buy less of it.

### 6.3 A Practical Formula
A common production form is:

$$
b_{t,z} = \frac{r_{t,z}}{\lambda_{\text{global}} \cdot \alpha_z}, \qquad \lambda_z = \lambda_{\text{global}} \cdot \alpha_z
$$

where:
- $r_{t,z}$ is the value of the impression, e.g. pCTR.
- $\lambda_{\text{global}}$ controls **total spend**, i.e. overall budget pacing.
- $\alpha_z$ is a **segment-level adjustment** that makes each region or query type more aggressive or more conservative relative to the rest, correcting for local response differences.

This is often called a *contextual dual approximation*. The two parts are updated separately.

**Global update.** As before, compare total spend with the target:

$$
e_{\text{global}}(t) = \frac{S(t) - S_{\text{target}}(t)}{S_{\text{target}}(t)}, \qquad \lambda_{\text{global}} \leftarrow \lambda_{\text{global}} \cdot \exp(\eta_g \\, e_{\text{global}})
$$

If total spend is too fast, $\lambda_{\text{global}}$ increases and all bids go down; if it is too slow, all bids go up. This loop is responsible for neither overspending nor underspending the budget.

**Segment update.** At the segment level, it is better to look at an efficiency gap than at raw clicks. If the goal is to maximize clicks, a natural metric is the segment's CPC relative to the global CPC:

$$
e_z = \frac{\text{CPC}\_z}{\text{CPC}\_{\text{global}}} - 1, \qquad \alpha_z \leftarrow \alpha_z \cdot \exp(\eta_z \\, e_z)
$$

Be careful with the direction. A segment with high CPC is expensive, so $\alpha_z$ should go up and its bids should go down; a segment with low CPC is cheap, so $\alpha_z$ should go down and its bids should go up. It is also a good idea to keep the $\alpha_z$ normalized (e.g. spend-weighted mean of 1), so that the segment loop redistributes spend while $\lambda_{\text{global}}$ alone controls the total.

One caveat: comparing *average* CPC is a practical proxy. Strictly speaking, the optimality condition is about *marginal* efficiency, so a segment can have a low average CPC while its marginal clicks are already expensive. When possible, bid-perturbation experiments that measure how clicks respond to a small bid change in each segment give a more faithful signal.

### 6.4 How to Choose Segments
The right question is not "how finely can we slice the traffic?" but **"along which dimensions does the same bid, or the same λ, produce systematically different results?"** Look for dimensions where these metrics differ significantly between groups:

- pCTR distribution
- Cost / clearing price distribution
- Win rate response to bid changes
- Clicks per dollar
- Spend velocity
- Traffic volume and supply

If two groups differ a lot on these, they are probably worth separating. Common candidate dimensions to start with:

- Geo / region
- Query intent / category
- Device
- Hour of day / day of week
- User type (new vs. returning, high vs. low intent)
- Publisher / placement
- Advertiser type / campaign type

More segments are not always better. Each segment needs enough traffic for its feedback signal to be reliable; otherwise, the $\alpha_z$ will mostly chase noise.

## 7. My Takeaway
The main lesson for me is that the theory and the production system are much more connected than they first appear. The Lagrangian formulation tells us the *shape* of the optimal bid (value divided by a price on spend, $r_t / \lambda$, or a blend with the cost cap when there are two constraints), and the online feedback loops are simply noisy gradient steps on the dual variables. Once you see the controller as dual optimization, many engineering choices, such as normalizing the error, using multiplicative updates, and splitting the multiplier into global and segment components, become much easier to reason about.

At the same time, the i.i.d. assumption behind a single global $\lambda$ is where theory and reality diverge the most. Real traffic is heterogeneous across regions, times, and queries, and the model's value estimates are not equally accurate everywhere. Segment-level multipliers are a practical way to close that gap, as long as each segment has enough data to support it.
