---
title: "Ads Allocation in Feed via Constrained Optimization"
date: 2024-06-20
description: "A paper review on LinkedIn's approach to blending organic and ad content in feeds"
tags: ["ads-allocation", "optimization", "paper-review"]
---

> **Paper**: *Ads Allocation in Feed via Constrained Optimization* (LinkedIn, 2020)

## The Problem

**Blending** merges two ranked lists into one feed:
- **Organic items**: optimized for engagement/user experience
- **Ads**: optimized for relevance + revenue

Two challenges:
1. **Calibration**: Make scores comparable across different models
2. **Allocation**: Decide where ads should go to optimize business value without harming UX

This paper focuses on allocation and treats calibration separately.

## Utility Framework

- **Engagement utility** `u`: Proxy for user value (both organic and ads)  
- **Revenue utility** `r`: Proxy for monetization (ads only)
- **Global constraint** `C`: Minimum acceptable engagement for the whole feed

## Key Innovation: Shadow Bid (α)

Instead of solving a hard 0–1 integer optimization, use a **Lagrange multiplier** `α`:

```
Ad wins if: r_ad + α·u_ad > α·u_org
```

**Intuition**:
- Large `α` → value engagement more → fewer/higher quality ads
- Small `α` → value revenue more → aggressive monetization

## Practical Algorithm

**Greedy merge** of two ranked lists with business constraints:
- **Top slot restriction**: First ad can't appear above position k
- **Min gap**: Two ads must be ≥ g items apart
- **Ad load limits**: Max fraction of ads in feed

## Corrections for Feed Dynamics

### Position Bias
Items higher in feed get more attention. Use position weighting:
```
w_k = 1 / log2(k + 1)
```

### Gap Effect (Ad Fatigue)
Ad CTR depends on distance from previous ad:
```
effective_value = θ(d) · (r_ad + α·u_ad)
```
where `d` = gap to previous ad, `θ(d)` = discount factor

## Evaluation Approach

**Offline**: Replay simulation with metrics:
- **DCR**: Discounted Cumulative Revenue
- **DCE**: Discounted Cumulative Engagement

Sweep `α` to get Pareto frontier between revenue and engagement.

**Online**: A/B test promising `α` values from offline analysis.

## Calibration Strategy

When upstream models change:
- **Isotonic regression**: When score is calibrated probability
- **Thompson Sampling**: Find global calibration factor for composite utility

Goal: Match key metric between new model and baseline.

## Key Insights

1. **Shadow bid `α`** provides intuitive control knob for engagement/revenue tradeoff
2. **Position bias and gap effects** are critical in feeds (independence assumption breaks)
3. **Hard rules** (top-slot, min-gap) are practical but potentially suboptimal
4. **Offline replay + online calibration** workflow keeps system stable

## Further Thoughts

**Can organic items have a "bid"?**  
Yes - estimate long-term value from user signals (e.g., session duration) and derive item value scale. `α` becomes the exchange rate between short-term revenue and long-term user value.

**Are hard rules optimal?**  
No - different users have different ad tolerance. More advanced: model effects directly and let policy adapt by user/context.
