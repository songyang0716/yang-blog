---
title: "Whole Page Optimization with Global Constraints"
date: 2026-02-20
description: "A paper review on optimizing Amazon video homepage layouts with global content diversity constraints"
tags: ["optimization", "recommender system" ,"paper-review"]
---

> **Paper Review**: *Whole Page Optimization with Global Constraints*

## 1. The Problem
The Amazon Video homepage serves as the main entry point for users to discover content. The home page is personalized and organized as a sequence of widgets (carousels), each displaying multiple items such as movies or TV shows. Optimizing the ordering of these widgets requires balancing user relevance and content diversity, while also satisfying platform-level business constraints. This paper proposes the first unified framework that jointly models relevance, diversity, and global business constraints. And finally the author propose the corresponding greedy algorithm to achieve these goals. 


## 2. Paper Content
### Background
The Amazon Video homepage is composed of multiple widgets, each representing a themed group of content. Given a large set of candidate widgets and limited page space, the platform must decide which widgets to show and in what order. This widget-ranking problem aims to balance user relevance, content diversity across the page, and platform-level business constraints.

![Amazon video homepage](../../images/posts/whole_page_optimization_1.png)

*An example of the Amazon Video homepage, where each row represents a widget containing a distinct type of content. The paper studies how to select which widgets to display and determine their optimal ordering for each user.*

This paper addresses several key questions:
1.	What are the optimization objectives?
2.	Which metrics are used to represent these objectives?
3.	How are the objectives and constraints formulated mathematically?
4.	How is the resulting optimization problem translated into a practical, production-ready serving algorithm?

### 2.1 Formulation
The authors formulate the widget-ranking task as a constrained optimization problem. The decision variable is $C_n^u $, which represents the widget (carousel) selection for page request $u$. The objective is to maximize the scoring function aggregated over all page requests within a given time period:

$$
\arg\max_{\{ u \in \mathcal{U}^t\}}
\sum_{u \in \mathcal{U}^t} f\left(C_n^u \mid u, \mathbf{w}\right)
$$

subject to average diversity-related constraints on the selected widgets:

$$
\sum_{u \in \mathcal{U}^t} g\left(C_n^u\right) \le 0.
$$

The authors model the homepage as an ordered list of widgets (carousels), denoted by a page configuration $C_n = (c_1, \dots, c_n)$. The overall page-level score is defined as the sum of the incremental utility contributed by each widget:

$\displaystyle f(C_n \mid u, \mathbf{w}) = \sum_{i=1}^n f\left(c_i \mid C_{i-1}, u, \mathbf{w}\right)$

where $C_{i-1}$ represents the widgets already placed above position $i$. This formulation captures the idea that the value of a widget depends not only on its own relevance, but also on the widgets that have already been shown.

The value of a widget (i.e. $f$ ) can be defined in different ways, such as whether the user streams any content from the widget (binary conversion), total streaming time. In the paper, the authors focus on a binary conversion signal and model it in a regression framework.

To account for diversity, the score of each widget explicitly depends on previously selected widgets through diversity features. Specifically, widget interactions are modeled via category-level coverage: as more widgets from the same category are shown, the incremental gain of displaying another widget from that category diminishes. This diminishing-return property induces submodularity and encourages diversity across the page.

The final scoring function for a widget is modeled as a linear function over concatenated relevance and diversity features:

$\displaystyle f\left(c_i \mid C_{i-1}, u, \mathbf{w}\right)
= \big[ \phi(c_i, u)^\top , \mathbf{u} \times \Delta \rho(c_i, C_{i-1})^\top \big] \mathbf{w}$

where $\phi(c_i, u)$ captures the contextual relevance of the widget, and $\Delta \rho(c_i, C_{i-1})$ measures the additional diversity gain contributed by placing $c_i$ given the existing widgets on the page.

At a high level, this formulation decomposes page optimization into additive, widget-level contributions while naturally capturing interaction effects and diminishing returns. A key benefit of this approach is that content diversity is incorporated directly into the value model as explicit features, where the marginal gain of displaying a widget decreases as more widgets from the same categories have already been shown. This structure is particularly well suited for greedy serving algorithms: the submodular property ensures that the greedy solution remains close to the globally optimal widget ordering.


### 2.2 Serving Algorithm


### 2.3 Experimentation



## 3. My Takeaway



## 4. My Questions
1. For the scoring function f, would it possible to use a blockbox ML model to fit it rather than the linear regression ?

