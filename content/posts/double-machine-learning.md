---
title: "Double / Debiased Machine Learning"
date: 2022-05-10
description: "A paper review on how orthogonalization and cross-fitting let us use any ML model to estimate causal effects with valid confidence intervals"
tags: ["causal-inference", "statistics", "machine-learning", "paper-review"]
---

> **Paper Review**: *Double/Debiased Machine Learning for Treatment and Structural Parameters* (Chernozhukov, Chetverikov, Demirer, Duflo, Hansen, Newey & Robins, The Econometrics Journal, 2018)

## 1. The Problem
In observational data, estimating a treatment effect usually means adjusting for confounders $X$. When $X$ is high-dimensional, or affects the outcome and the treatment in complicated non-linear ways, machine learning seems like the obvious tool: random forests, boosting, LASSO, and neural nets are all very good at learning $\mathbb{E}[Y \mid X]$.

The problem is that being good at *prediction* is not the same as being good at *estimation*. If we plug an ML model into a regression and read off the treatment coefficient, the estimate is usually biased, converges slowly, and does not come with valid confidence intervals.

Double/Debiased Machine Learning (DML) fixes this. Its goals are to:

1. Eliminate the bias that ML introduces into the treatment effect estimate.
2. Achieve $\sqrt{n}$-consistency, i.e. the estimate converges at the usual parametric rate.
3. Construct valid confidence intervals.

The paper identifies two sources of bias and gives one fix for each:

- Bias from **regularization**, which is removed with **Neyman orthogonality** (a residuals-on-residuals approach, in the spirit of Robinson (1988) and the Frisch–Waugh–Lovell theorem).
- Bias from **overfitting**, which is removed with **sample splitting**, and made efficient with **cross-fitting**.

## 2. Paper Content
### 2.1 The Partially Linear Model
To explain how DML works, the paper uses a partially linear model:

$$
\begin{aligned}
Y &= D\theta_0 + g_0(X) + U, \qquad \mathbb{E}[U \mid X, D] = 0 \\\\
D &= m_0(X) + V, \qquad \mathbb{E}[V \mid X] = 0
\end{aligned}
$$

where:
- $\theta_0$ is the treatment effect we want to estimate.
- $g_0(\cdot)$ is some function mapping $X$ to $Y$, *conditional on* $D$.
- $m_0(\cdot)$ is some function mapping $X$ to $D$, i.e. the propensity (or treatment) model.
- $\eta_0 = (g_0, m_0)$ are called **nuisance parameters**. We do not care about them directly, as long as estimating them gives us a good estimate of $\theta_0$.

Compared to standard OLS, which assumes everything is linear and additive, this set-up lets both $Y$ and $D$ depend on $X$ in arbitrary non-linear and interactive ways. The confounders can interact with each other, just not with the treatment. We still need the standard identification assumptions: unconfoundedness conditional on $X$, positivity, and consistency.

Two caveats are worth keeping in mind:

- **The effect of $D$ on $Y$ is assumed to be additive and linear here.** If the whole point of ML is to relax linearity, why keep it for $D$? The model is used only for illustration. The DML framework itself works with fully interactive, non-linear models with heterogeneous treatment effects (e.g. the interactive regression model for a binary treatment).
- **$\theta_0$ is not necessarily the average treatment effect.** If the true effects are heterogeneous, the regression coefficient is a weighted average of individual effects, with weights proportional to the conditional variance of the treatment, $\text{Var}(D \mid X)$ (see Aronow & Samii, 2016).

### 2.2 The Naive Approach and Why It Fails
The most intuitive way to use ML here is to fit $Y = D\hat{\theta}\_0 + \hat{g}\_0(X) + \hat{U}$ directly. For example:

- Alternate between a random forest that fits $\hat{g}\_0(X)$ on $Y - D\hat{\theta}\_0$, and OLS that fits $\hat{\theta}\_0$ on $Y - \hat{g}\_0(X)$, until convergence; or
- Generate many non-linear transformations and interactions of $X$, and fit one big LASSO regression with $D$ included.

Note that $g_0(X) \neq \mathbb{E}[Y \mid X]$, because $D\theta_0$ is also in the model and $D$ itself depends on $X$. The paper uses $\ell_0(X) = \mathbb{E}[Y \mid X] = \theta_0 m_0(X) + g_0(X)$ for the conditional mean of $Y$.

This naive approach can give very good *predictions* of $Y$, but the estimate of $\theta_0$ is badly off. In the simulation from the lecture slides I followed ($n = 100$, $p = 20$), the distribution of $\theta_0 - \hat{\theta}\_0$ across replications sits clearly away from zero, nowhere near the $N(0, \Sigma)$ it should follow.

To see why, assume the alternating procedure has converged, so $\hat{\theta}\_0$ is the OLS solution given $\hat{g}\_0$ (and, for simplicity, $\mathbb{E}[g_0(X)] = \mathbb{E}[m_0(X)] = 0$):

$$
\hat{\theta}\_0 = \Big(\frac{1}{n}\sum_i D_i^2\Big)^{-1} \frac{1}{n}\sum_i D_i \big(Y_i - \hat{g}\_0(X_i)\big)
$$

Substituting $Y_i = D_i\theta_0 + g_0(X_i) + U_i$ and $D_i = m_0(X_i) + V_i$, the scaled estimation error splits into three terms:

$$
\sqrt{n}(\hat{\theta}\_0 - \theta_0) = \frac{1}{\mathbb{E}[D^2]} \Bigg[ \underbrace{\frac{1}{\sqrt{n}} \sum_i D_i U_i}\_{A} + \underbrace{\frac{1}{\sqrt{n}} \sum_i m_0(X_i) \big(g_0(X_i) - \hat{g}\_0(X_i)\big)}\_{B} + \underbrace{\frac{1}{\sqrt{n}} \sum_i V_i \big(g_0(X_i) - \hat{g}\_0(X_i)\big)}\_{C} \Bigg]
$$

- **Term A** is fine. It is a scaled sample average of mean-zero terms, so by the central limit theorem $A \rightsquigarrow N(0, \sigma_A^2)$.
- **Term B** is the **regularization bias**.
- **Term C** is the **overfitting bias**.

To isolate each, the paper looks at regularization bias assuming sample splitting is already in place, and at overfitting bias assuming orthogonalization is already in place.

### 2.3 Regularization Bias (Term B)
To avoid overfitting complex functional forms, ML methods use regularization: L1/L2 penalties, tree depth limits, early stopping, and so on. Regularization reduces variance, but it also *shrinks* the estimate, so $\hat{g}\_0$ is systematically biased toward simpler functions. In high-dimensional settings, this is the price of learning anything at all.

A useful way to think about it: for $\hat{g}\_0$, the bias does not disappear at the $\sqrt{n}$ rate. The error typically shrinks like

$$
g_0(X_i) - \hat{g}\_0(X_i) \propto n^{-\varphi_g}, \qquad \varphi_g < \tfrac{1}{2}
$$

The terms $m_0(X_i)\big(g_0(X_i) - \hat{g}\_0(X_i)\big)$ do not have mean zero, because the regularization bias in $\hat{g}\_0$ points in a consistent direction and $D_i$ is centered at $m_0(X_i) \neq 0$. So term B is a sum of $n$ non-mean-zero terms divided by $\sqrt{n}$:

$$
B \propto \frac{1}{\sqrt{n}} \cdot n \cdot n^{-\varphi_g} = n^{\frac{1}{2} - \varphi_g} \to \infty
$$

The bias in $\hat{g}\_0$ does shrink, but too slowly: once scaled by $\sqrt{n}$, it blows up. That is why the naive estimator is not $\sqrt{n}$-consistent.

The deeper reason is that the treatment and the outcome share the confounder $X$. Whatever part of $g_0$ the regularized model fails to capture leaks into the estimate of $\theta_0$ through the correlation between $D$ and $X$, i.e. through $m_0(X)$. This is exactly classic omitted variable bias, with the "omitted" part being whatever the ML model shrank away.

### 2.4 Fixing Regularization Bias with Orthogonalization
The fix is to fit two ML models instead of one, and partial out the effect of $X$ from the treatment as well:

1. Fit the **treatment model** $D = \hat{m}\_0(X) + \hat{V}$.
2. Fit the **outcome model** $\hat{g}\_0(X)$, as in the naive approach.
3. Regress $Y - \hat{g}\_0(X)$ on $\hat{V}$.

This is where the name "double" comes from: machine learning is used twice, once for $\hat{g}\_0$ and once for $\hat{m}\_0$. The resulting estimator is

$$
\check{\theta}\_0 = \Big(\frac{1}{n}\sum_i \hat{V}\_i D_i\Big)^{-1} \frac{1}{n}\sum_i \hat{V}\_i \big(Y_i - \hat{g}\_0(X_i)\big)
$$

(The paper's analysis uses $\hat{V}\_i D_i$ in the denominator rather than $\hat{V}\_i^2$. The two give the same asymptotic results since $\mathbb{E}[\hat{V}^2] = \mathbb{E}[\hat{V}D]$ in the limit.)

**Connection to instrumental variables.** This formula has exactly the form of the linear IV estimator $\hat{\beta}\_{\text{IV}} = (Z^\top D)^{-1} Z^\top y$, with the treatment residual $\hat{V}$ playing the role of the instrument. That makes intuitive sense: $\hat{V}$ is the part of the treatment that $X$ cannot explain. It moves $D$ but, by construction, is not driven by the confounders, which is exactly what we ask of an instrument.

**Why does this work?** Redo the decomposition with $\hat{V}\_i$ in place of $D_i$. Terms A and C keep the same form, but term B becomes

$$
B' = \frac{1}{\sqrt{n}} \sum_i \big(m_0(X_i) - \hat{m}\_0(X_i)\big) \big(g_0(X_i) - \hat{g}\_0(X_i)\big)
$$

The bias is now a **product of two estimation errors**, instead of one error multiplied by $m_0(X)$. If $\hat{m}\_0$ converges at rate $n^{-\varphi_m}$ and $\hat{g}\_0$ at rate $n^{-\varphi_g}$, then

$$
B' \propto \frac{1}{\sqrt{n}} \cdot n \cdot n^{-\varphi_m} \cdot n^{-\varphi_g} = n^{\frac{1}{2} - \varphi_m - \varphi_g}
$$

which goes to zero as long as $\varphi_m + \varphi_g > \tfrac{1}{2}$. For example, if both converge at rate $n^{-(1/4 + \epsilon)}$, then $B' \propto n^{-2\epsilon} \to 0$. Both nuisance models can be learned *slowly* (much slower than $\sqrt{n}$), and their errors still multiply into something negligible. This is the central insight of the paper.

### 2.5 The Connection to Frisch–Waugh–Lovell and Robinson
The residuals-on-residuals idea is not new. Suppose we fit $Y = \beta_0 + \beta_1 D + \beta_2 X + U$ by OLS. The **Frisch–Waugh–Lovell (FWL) theorem** says we can recover $\hat{\beta}\_1$ in three steps:

1. Regress $D$ on $X$ and keep the residuals $\hat{V} = D - \hat{D}$.
2. Regress $Y$ on $X$ and keep the residuals $\hat{W} = Y - \hat{Y}$.
3. Regress $\hat{W}$ on $\hat{V}$.

The coefficient on $\hat{V}$ is identical to $\hat{\beta}\_1$ from the full regression. Intuitively, we strip out of $D$ everything $X$ explains, strip out of $Y$ everything $X$ explains, and whatever relationship remains between the two is the effect of $D$ on $Y$.

**Robinson (1988)** made one change: replace the linear regressions in steps 1 and 2 with non-parametric (kernel) regressions, while keeping step 3 linear. DML takes this one step further and allows *any* ML model in steps 1 and 2.

This gives a second DML estimator, the **partialling-out** estimator, which the paper also uses (Section 4):

1. Estimate $D = \hat{m}\_0(X) + \hat{V}$.
2. Estimate $Y = \hat{\ell}\_0(X) + \hat{U}$. Note that $D$ is not used here: $\ell_0(X) = \mathbb{E}[Y \mid X]$, not $g_0(X)$.
3. Regress $\hat{U}$ on $\hat{V}$ using OLS:

$$
\check{\theta}\_0 = \Big(\frac{1}{n}\sum_i \hat{V}\_i \hat{V}\_i\Big)^{-1} \frac{1}{n}\sum_i \hat{V}\_i \big(Y_i - \hat{\ell}\_0(X_i)\big)
$$

When I first read the paper, I was confused about which outcome model to use. If DML is a generalization of FWL, the outcome model should predict $Y$ from $X$ alone, without $D$. That is exactly the partialling-out version. The version in Section 2.4, which uses $\hat{g}\_0$ (fitted conditional on $D$), is the "IV-type" score. Both are valid and give consistent estimates, and the DoubleML package supports both. Its simulations show both recover the true effect.

This also answers another question I had: **is the final step an iterative fit?** For the partialling-out estimator, no. You fit $\hat{m}\_0$ and $\hat{\ell}\_0$ once (on held-out folds, see below), compute the residuals, and run a single OLS of one residual on the other. No alternation between models is needed. The IV-type score needs $\hat{g}\_0$, which depends on $\theta$; in practice it is obtained from $\hat{\ell}\_0$ and $\hat{m}\_0$ using a preliminary estimate of $\theta$, rather than by alternating to convergence.

### 2.6 Overfitting Bias (Term C)
Orthogonalization takes care of term B, but term C,

$$
C = \frac{1}{\sqrt{n}} \sum_i V_i \big(g_0(X_i) - \hat{g}\_0(X_i)\big)
$$

is still a problem if $\hat{g}\_0$ is fitted on the same data used to estimate $\theta_0$.

Overfitting means modeling the noise too closely. To see the effect, (artificially) suppose the fitted model absorbs part of each observation's own residual:

$$
\hat{g}\_0(X_i) = g_0(X_i) + \frac{Y_i - g_0(X_i)}{n^{1/2 - \epsilon}}
$$

The error term contains $Y_i$, and $Y_i$ contains $D_i = m_0(X_i) + V_i$. So the estimation error of $\hat{g}\_0$ at observation $i$ contains $V_i$ itself, and term C picks up a $\frac{1}{\sqrt{n}} \sum_i V_i^2 / n^{1/2 - \epsilon}$ piece:

$$
C \approx -\\, n^{\epsilon} \cdot \text{Var}(V) + \dots \to \infty
$$

(The sign is not important; the magnitude grows without bound.)

When I first read the slides, I was confused by the claim that this bias arises "if $U$ and $V$ are associated", since the model assumes they are not. The example above shows that $U$ and $V$ do not need to be correlated at all. The problem is that the model's error at observation $i$ depends on that observation's own noise, including $V_i$ which enters through $D_i$ in $Y_i$. Evaluated on the same observation, $V_i$ and the model's error are correlated by construction.

### 2.7 Fixing Overfitting with Sample Splitting and Cross-Fitting
The fix is simple: never estimate $\theta_0$ on the same observations used to fit the nuisance models.

**Sample splitting:**
1. Randomly partition the data into two subsets, $I$ and its complement $I^c$.
2. Fit $\hat{g}\_0$ and $\hat{m}\_0$ on $I^c$.
3. Estimate $\check{\theta}\_0$ on $I$, using the models fitted on $I^c$.

Now the error $\text{Err}\_i = g_0(X_i) - \hat{g}\_0(X_i)$ depends only on the auxiliary sample $I^c$, which is independent of $V_i$ for $i \in I$. Conditioning on $I^c$:

$$
\mathbb{E}[C] = \frac{1}{\sqrt{n}} \sum_{i \in I} \mathbb{E}\Big[\mathbb{E}[V_i] \cdot \mathbb{E}[\text{Err}\_i \mid I^c]\Big] = 0
$$

since $\mathbb{E}[V_i] = 0$. The paper shows this term is $o_P(1)$ under weak conditions.

**Cross-fitting.** Plain sample splitting throws away half the data for the final estimate, which costs efficiency and power. Cross-fitting avoids that by swapping roles:

1. Fit $\hat{g}\_{0,1}$ and $\hat{m}\_{0,1}$ on the first subset, and estimate $\check{\theta}\_{0,1}$ on the second.
2. Fit $\hat{g}\_{0,2}$ and $\hat{m}\_{0,2}$ on the second subset, and estimate $\check{\theta}\_{0,2}$ on the first.
3. Average the two to get $\check{\theta}\_0$.

In practice, this generalizes to $K$ folds (e.g. $K = 5$), just like cross-validation.

### 2.8 Neyman Orthogonality and Inference
The paper then formalizes these ideas. Everything is expressed through a **score function** $\psi(W; \theta, \eta)$ such that $\mathbb{E}[\psi(W; \theta_0, \eta_0)] = 0$, where $W = (Y, D, X)$ is the data. For the partially linear model, the partialling-out score is:

$$
\psi(W; \theta, \eta) = \underbrace{\big(D - m(X)\big)}\_{V} \times \underbrace{\big(Y - \ell(X) - \theta(D - m(X))\big)}\_{U}
$$

Each part is a noise term from the model, so the score is just the product of the two residuals.

**Neyman orthogonality** means that the expected score is locally insensitive to small errors in the nuisance parameters:

$$
\partial_r \\, \mathbb{E}\Big[\psi\big(W; \theta_0, \eta_0 + r(\eta - \eta_0)\big)\Big] \Big|\_{r = 0} = 0
$$

For this score, it is easy to check. Perturbing $\ell$ by $\delta_\ell(X)$ changes the expected score by $-\mathbb{E}[V \delta_\ell(X)] = 0$, since $\mathbb{E}[V \mid X] = 0$. Perturbing $m$ by $\delta_m(X)$ changes it by $-\mathbb{E}[U \delta_m(X)] + \theta_0 \mathbb{E}[V \delta_m(X)] - \theta_0 \mathbb{E}[V \delta_m(X)] = 0$. Because the first-order effect is zero, the remaining effect of nuisance errors is second order: the product of errors we saw in term $B'$. The naive score $D(Y - D\theta - g(X))$ does not have this property: perturbing $g$ changes it by $-\mathbb{E}[D \delta_g(X)] = -\mathbb{E}[m_0(X) \delta_g(X)] \neq 0$, which is exactly term B.

**The DML recipe.** Combining the score with cross-fitting:

1. Split the data into $K$ folds.
2. For each fold $k$, fit the nuisance models $\hat{\eta}\_{-k} = (\hat{\ell}, \hat{m})$ on all other folds, using any ML method.
3. Solve $\frac{1}{n} \sum_k \sum_{i \in I_k} \psi(W_i; \check{\theta}\_0, \hat{\eta}\_{-k}) = 0$ for $\check{\theta}\_0$. For the partialling-out score, this is just OLS of $Y$-residuals on $D$-residuals.

**Confidence intervals.** Because the estimator is asymptotically normal,

$$
\sqrt{n}(\check{\theta}\_0 - \theta_0) \rightsquigarrow N(0, \sigma^2), \qquad \sigma^2 = \big(\mathbb{E}[V^2]\big)^{-2} \\, \mathbb{E}[V^2 U^2]
$$

and a 95% confidence interval is $\check{\theta}\_0 \pm 1.96 \\, \hat{\sigma} / \sqrt{n}$, with $\hat{\sigma}^2$ estimated from the cross-fitted residuals. This is the familiar heteroskedasticity-robust sandwich variance from the residual-on-residual regression.

## 3. My Takeaway
DML gives a clean recipe for using ML in causal inference: we can use a broad range of ML or non-parametric methods for the high-dimensional nuisance parameters $\eta_0 = (g_0, m_0)$, and still get a $\sqrt{n}$-consistent, asymptotically normal estimate of the treatment effect with valid confidence intervals. Since it was first released in 2016, it has become one of the standard tools for estimating effects from observational data.

What I find most satisfying is that each of the two ingredients solves exactly one problem:

- **Orthogonalization** turns regularization bias from a first-order error into a *product* of two errors, so both nuisance models only need to be "good enough" (roughly faster than $n^{-1/4}$) rather than perfect.
- **Cross-fitting** breaks the link between a model's errors and the noise in the observations used to estimate $\theta$, so overfitting no longer leaks into the estimate, at no cost in sample size.

It also changed how I think about ML models in this setting. The nuisance models are not the goal; their predictions are only intermediate quantities. A model with great predictive accuracy can still give a badly biased causal estimate if it is plugged into the wrong estimating equation, and the residuals-on-residuals structure, a direct descendant of the Frisch–Waugh–Lovell theorem, is what makes the difference.

Finally, DML does not remove the need for identification assumptions. It handles *estimation* when $X$ is complicated, but if there are unmeasured confounders, no amount of ML on $X$ will fix the bias.

## 4. Further Reading
- [Chernozhukov et al., *Double/Debiased Machine Learning for Treatment and Structural Parameters*](https://arxiv.org/abs/1608.00060): the original paper.
- [Chernozhukov et al., handout](https://scholar.princeton.edu/sites/default/files/bstewart/files/chern.handout.pdf) and [Felton's slides](https://bstewart.scholar.princeton.edu/sites/g/files/toruqf4016/files/bstewart/files/felton.chern_.slides.20190318.pdf) from Princeton's reading group: a gentle walkthrough of the paper.
- [CSC2541 Lecture 7: Double Machine Learning](https://csc2541-2022.github.io/lectures/CSC2541_lecture7_dml.pdf) (University of Toronto): the clearest step-by-step derivation of the error decomposition that I found.
- [DoubleML](https://docs.doubleml.org/): Python and R implementations, with tutorials comparing the partialling-out and IV-type scores.
