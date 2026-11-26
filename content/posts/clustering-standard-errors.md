---
title: "When Should You Adjust Standard Errors for Clustering?"
date: 2022-01-01
description: "A paper review on why clustering is a design question, not a data question, and what that means for experiments"
tags: ["experimentation", "causal-inference", "statistics", "paper-review"]
---

> **Paper Review**: *When Should You Adjust Standard Errors for Clustering?* (Abadie, Athey, Imbens & Wooldridge). [NBER Working Paper 24003](https://www.nber.org/system/files/working_papers/w24003/w24003.pdf)

## 1. The Problem
Whenever we run a regression or analyze an A/B test, there is a recurring question: should the standard errors be adjusted for clustering? And if so, at what level: user, household, city, state?

The conventional answer goes something like this: if the residuals are correlated within groups, cluster; when in doubt, cluster at the most aggregate level you can afford, since clustering "does no harm" in large samples. The authors argue that this reasoning is flawed. If within-group correlation were the justification, it would be just as easy to justify clustering a wage regression by age cohort as by state, and it would be hard to explain why researchers usually do *not* cluster in randomized experiments, or why clustering still matters after adding cluster fixed effects.

The paper answers two questions:
1. When should we adjust standard errors for clustering, and when should we not?
2. If we do, what is the right level of clustering?

The core message is that **clustering is a design problem, not a data problem**. Whether to cluster depends on how the sample was drawn and how the treatment was assigned, not on whether the clustered standard errors happen to look different.

## 2. Paper Content
### 2.1 Two Sources of Clustering
The authors identify exactly two reasons to adjust standard errors for clustering:

- **Sampling design.** The sample was drawn in two stages: first a subset of clusters is randomly sampled from a population of clusters, then units are randomly sampled within the selected clusters. If we want to make statements about the broader population, including clusters we never observed, we need to account for that first stage.
- **Experimental design.** The treatment is assigned at the cluster level (or with cluster-correlated probabilities) rather than independently to each unit.

To decide which case applies, a researcher has to be explicit about two things: (i) how the units in the sample were selected, and in particular whether there are clusters in the population of interest that are not represented in the sample; and (ii) how units were assigned to treatment, and whether that assignment was clustered. If either the sampling or the assignment varies systematically with groups, clustering is in general justified.

### 2.2 Two Common Misconceptions
Before the formal results, the paper reviews the two standard variance estimators. For a linear regression $Y_i = \beta^\top X_i + \varepsilon_i$, the Eicker-Huber-White (EHW) heteroskedasticity-robust variance is:

$$
\hat{\mathbb{V}}\_{\text{EHW}}(\hat{\beta}) = (\mathbf{X}^\top \mathbf{X})^{-1} \left( \sum_{i=1}^{N} (Y_i - \hat{\beta}^\top X_i)^2 X_i X_i^\top \right) (\mathbf{X}^\top \mathbf{X})^{-1}
$$

The Liang-Zeger (LZ) cluster-robust variance, which is what most software reports as "clustered standard errors", sums the score within each cluster $c$ before taking the outer product:

$$
\hat{\mathbb{V}}\_{\text{LZ}}(\hat{\beta}) = (\mathbf{X}^\top \mathbf{X})^{-1} \left( \sum_{c=1}^{C} \Big( \sum_{i: C_i = c} \hat{\varepsilon}\_i X_i \Big) \Big( \sum_{i: C_i = c} \hat{\varepsilon}\_i X_i \Big)^\top \right) (\mathbf{X}^\top \mathbf{X})^{-1}
$$

With these two estimators in mind, the authors take apart two widely held beliefs.

**Misconception 1: Clustering matters only if both the residuals and the regressors are correlated within clusters.** This is wrong. Within-cluster correlation of residuals and regressors is neither necessary nor sufficient for the clustering adjustment to matter. The adjustment can change the standard errors substantially even when both residuals and regressors are uncorrelated within clusters.

**Misconception 2: If clustering matters, one should cluster.** This is also wrong. The common view is that clustering is harmless in large samples, so one should cluster at the highest feasible level: if it matters, good; if not, no loss. The authors argue that the decision must rest on substantive knowledge of the design, not on whether the adjustment changes the numbers. The data can tell you *whether* clustering matters, but only partially whether you *should* cluster.

The authors illustrate this with a simulation. A large population is divided into 100 clusters, units are sampled at random, the treatment is randomly assigned at the unit level, and treatment effects differ across clusters. All 100 clusters appear in the sample. The EHW and LZ standard errors come out very different, yet **both are correct, for different estimands**:

- The **EHW** standard errors assume the sample is randomly drawn from the population of interest, which here consists of exactly these 100 clusters.
- The **LZ** standard errors implicitly assume that there are *other* clusters in the population of interest beyond the 100 in the sample.

If instead the population had 1,000 clusters, of which 100 were randomly drawn before sampling units within them, the LZ standard errors would be correct and the EHW standard errors would be too small. Put simply: if you only want to say something about the clusters you have, EHW is enough; if you want to generalize to clusters you have not seen, you need LZ. This hidden assumption, that unobserved clusters exist in the population of interest, is the critical and usually unstated premise behind the model-based case for clustering.

### 2.3 Formal Setup: Potential Outcomes with Design-Based Uncertainty
The formal framework is where the paper really shines. It uses the Rubin potential outcomes framework and treats the population as fixed.

There is a sequence of populations indexed by $n$. Population $n$ has $M_n$ units, partitioned into $C_n$ clusters, with $M_{cn}$ units in cluster $c$. Each unit has two potential outcomes, $Y_{in}(0)$ and $Y_{in}(1)$. The estimand is the population average treatment effect:

$$
\tau_n = \frac{1}{M_n} \sum_{i=1}^{M_n} \big( Y_{in}(1) - Y_{in}(0) \big) = \overline{Y}\_n(1) - \overline{Y}\_n(0)
$$

which can also be written as a weighted average of the cluster-level effects $\tau_{cn}$, with weights proportional to the cluster sizes $M_{cn}$.

A subtle but important point: the potential outcomes, and therefore the treatment-specific residuals $\varepsilon_{in}(w) = Y_{in}(w) - \overline{Y}\_n(w)$, are **not random**. They are fixed attributes of the population. All the randomness comes from two places:

1. **Sampling**: which units end up in our sample.
2. **Assignment**: which sampled units receive the treatment.

A correct standard error must capture both sources of variation. It answers the question: had we drawn a different sample and assigned treatment differently, how different would the estimate be?

**Sampling process.** Let $R_{in} \in \{0, 1\}$ indicate whether unit $i$ is sampled. Sampling happens in two stages: each cluster is selected with probability $P_{Cn}$, and then each unit in a selected cluster is sampled with probability $P_{Un}$. Formally:

$$
\Pr(R_{in} = 1) = P_{Cn} P_{Un}, \qquad \Pr(R_{in} = 1 \mid R_{jn} = 1, C_{in} = C_{jn}) = P_{Un}
$$

$P_{Cn} = 1$ means every cluster is included, which is equivalent to simple random sampling of units. $P_{Cn}$ close to zero is the scenario the LZ estimator was built for: we only observe a few clusters out of a very large population of clusters.

**Assignment process.** Assignment also has two stages. First, each cluster $c$ draws its own treatment probability $q_{cn} \in [0, 1]$ from a distribution with mean $\mu_n = 1/2$ and variance $\sigma_n^2$. Second, each unit in the cluster is treated independently with probability $q_{cn}$. The variance $\sigma_n^2$ is the key parameter:

- $\sigma_n^2 = 0$: every cluster has the same probability, so this is ordinary unit-level randomization.
- $\sigma_n^2 = 1/4$: $q_{cn} \in \{0, 1\}$, so all units in a cluster share the same assignment. This is a pure cluster-randomized experiment.

Both moments in the assumption follow directly from this two-stage process. Marginally, each unit is treated with probability equal to the average of the $q_{cn}$, which is $\mu_n = 1/2$. But two units in the *same* cluster share the same $q_{cn}$, so

$$
\Pr(W_{in} = 1 \mid W_{jn} = 1, C_{in} = C_{jn}) = \frac{\mathbb{E}[q_{cn}^2]}{\mathbb{E}[q_{cn}]} = \frac{\mu_n^2 + \sigma_n^2}{\mu_n} = \frac{1}{2} + 2\sigma_n^2
$$

In other words, whenever clusters have different treatment probabilities, assignments within the same cluster become dependent. That dependence is exactly what "clustered assignment" means.

### 2.4 Main Results
The estimator is the ordinary least squares coefficient from regressing $Y_{in}$ on a treatment indicator $W_{in}$, which is just the difference in means $\hat{\tau} = \overline{Y}\_1 - \overline{Y}\_0$. The authors approximate $\sqrt{N_n}(\hat{\tau} - \tau_n)$ by a sample average, derive its exact variance under the design above, and compare it with what the EHW and LZ estimators converge to. This yields two clean corollaries.

**When clustering is not needed (Corollary 1).** Standard errors need to account for clustering *unless* one of the following holds:
1. There is no clustering in sampling ($P_{Cn} = 1$) **and** no clustering in assignment ($\sigma^2 = 0$); or
2. There is no treatment effect heterogeneity ($Y_i(1) - Y_i(0) = \tau$ for all units) **and** no clustering in assignment ($\sigma^2 = 0$).

**When the LZ variance is approximately correct (Corollary 2).** The LZ estimator is accurate if one of these holds:
1. There is no treatment effect heterogeneity;
2. $P_{Cn}$ is close to zero, meaning we observe only a few clusters out of a large population of clusters; or
3. $P_{Un}$ is close to zero, so at most one unit is sampled per cluster (in which case clustering adjustments do not matter anyway).

Outside these cases, **the LZ estimator is conservative**: it overstates the variance. The authors note that the first condition (homogeneous effects) almost never holds in practice. The third is easy to check from the data, by looking at the number of sampled units per cluster. The second cannot be checked from the data at all; it requires knowing how the sample was generated and whether the population of interest contains clusters that are missing from the sample.

When all clusters in the population are observed and treatment varies *within* clusters, the authors propose a tighter **cluster-adjusted (CA)** variance that subtracts the overly conservative part from LZ:

$$
\hat{\mathbb{V}}\_{\text{CA}}(\hat{\tau}) = \hat{\mathbb{V}}\_{\text{LZ}}(\hat{\tau}) - \frac{1}{N^2} \sum_{c=1}^{C} N_c^2 (\hat{\tau}\_c - \hat{\tau})^2
$$

where $\hat{\tau}\_c = \overline{Y}\_{c1} - \overline{Y}\_{c0}$ is the within-cluster difference in means. If the assignment is perfectly correlated within clusters, $\hat{\tau}\_c$ cannot be computed, and there is no general improvement over LZ.

### 2.5 The Fixed Effects Case
A common follow-up question is whether we still need clustered standard errors once cluster fixed effects are included, i.e. in the regression

$$
Y_i = \alpha_{C_i} + \tau W_i + \dot{\varepsilon}\_i
$$

First, a practical note: if the assignment is perfectly correlated within clusters ($\sigma^2 = 1/4$), the treatment indicator is collinear with the cluster dummies and $\tau$ cannot be estimated. Fixed effects at the cluster level therefore require some within-cluster variation in treatment.

The answer to the question itself is that clustering can still matter, but the conditions shift:

- **Without fixed effects**, cluster if either (i) the sampling is clustered ($P_{Cn} < 1$) and treatment effects are heterogeneous, or (ii) the assignment is clustered ($\sigma^2 > 0$).
- **With fixed effects**, cluster if either (i) the sampling is clustered ($P_{Cn} < 1$) and treatment effects are heterogeneous, or (ii) the assignment is clustered ($\sigma^2 > 0$) **and** treatment effects are heterogeneous.

In other words, once cluster fixed effects are included, treatment effect heterogeneity becomes a requirement for the clustering adjustment to matter. If the effect is homogeneous, there is no need to cluster.

## 3. My Takeaway
The practical recipe from this paper is short:

1. **Ask whether the sampling is clustered.** Are there clusters in the population you care about that are not in your sample?
2. **Ask whether the assignment is clustered.** Was treatment assigned to groups, or with group-dependent probabilities?
3. **If both answers are no, do not cluster**, even if clustering would change the standard errors.
4. **If either answer is yes, cluster**, keeping in mind that LZ is conservative when you observe most of the clusters in the population and effects are heterogeneous.

What I appreciate most is the shift in perspective. For a long time, I treated clustering as a diagnostic exercise: check whether residuals look correlated within groups, and cluster if they do. This paper replaces that with a design-based question that only the researcher can answer: *where does the randomness in my estimate actually come from?*

This maps directly onto online experimentation. In one use case I worked on, treatment was assigned at the cluster level, so every unit in a cluster received the same variant. That is clustered assignment with $\sigma^2 = 1/4$, and by Corollary 1 the standard errors must account for it. Since the assignment is perfectly correlated within clusters, the CA refinement is not available, and the LZ cluster-robust estimator is the right tool, even if it may be somewhat conservative. Conversely, for a standard user-level A/B test, where users are randomized independently, there is no need to cluster by city, platform, or any other grouping, however tempting it may look.

Revisiting this paper later, the question that stayed with me was more fundamental: what actually makes a set of observations i.i.d.? Aronow and Miller's *Foundations of Agnostic Statistics* offers a helpful framing. Independence fails when units are intertwined (for example, members of the same household), and identical distribution fails when the population drifts over time (for example, survey opinions changing during data collection). In the latter case, our conclusions are about a *pseudopopulation* that aggregates the different distributions, rather than a single clean population. In practice, i.i.d. is almost always an assumption we make rather than something we can verify. This paper's contribution is to make that assumption explicit, by tying it to the sampling and assignment design instead of leaving it implicit in the choice of standard errors.

## 4. Further Reading
- [OLS with clustered standard errors vs. multilevel modeling](https://stats.stackexchange.com/questions/38419/ols-with-clustered-standard-errors-vs-multilevel-modeling-when-the-main-interes/38430#38430): a good discussion of cluster-robust standard errors versus linear mixed effect models.
- [Abadie, Athey, Imbens & Wooldridge, *Sampling-based vs. Design-based Uncertainty in Regression Analysis*](https://www.nber.org/papers/w22449): a companion paper on the distinction between sampling-based and design-based uncertainty.
