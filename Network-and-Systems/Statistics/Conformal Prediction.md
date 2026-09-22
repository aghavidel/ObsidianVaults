**Papers:** 
- [A Tutorial on Conformal Prediction](https://arxiv.org/pdf/0706.3188)
- [A Gentle Introduction to Conformal Prediction and Distribution-Free Uncertainty Quantification](https://arxiv.org/pdf/2107.07511)
- [The limits of distribution-free conditional predictive inference](https://arxiv.org/pdf/1903.04684)

This document is the Frankenstein's monster stitching of the above 3 papers, mainly there to help me understand some of the aspects of the theory around conformal prediction methods.

>[!FAQ]- Latex Macros
>- **Probability Measure:** $\newcommand{\prob}[1]{\mathbb{P}\Big\{#1\Big\}}$

# Distribution-Free Prediction

Assume we are given a training dataset consisting of $n$ pairs of feature and label values $(X_i, Y_i)$. For now, consider $X_i \in \mathbb{R}^m$ to be some vector and $Y_i \in \mathbb{R}$ as just some number.
We have been given a previously unseen feature vector $X_{n+1}$ and are asked to construct an interval that we can argue contains the actual label $Y_{n+1}$ with high probability.

We will denote this interval that we aim to create with $\hat{C}(X_{n+1})$. When the true label is contained within this interval, we say that $\hat{C}$ *covers* the label. There are different ways that one can define the probability of coverage for a given prediction interval. For example, one way to define it (which is what people usually think about) is **Conditional Coverage** which basically means that conditioned on the revealed feature vector $X_{n+1}$, give us a set that covers the label with high probability.
Formally, a distribution-free $\alpha$ conditionally covering interval (denoted by $\alpha$-CC) means:
$$
\prob{Y_{n+1} \in \hat{C}(X_{n+1}) \Big | X_{n+1} = x} \geq 1 - \alpha
$$
For all $x$ and probability measure $\mathbb{P}$.

>[!NOTE]- A Technicality
> The formal definition should say something like "for almost all $x$".
> That is to say, as long as the set of values that violate the property have measure-zero (that is to say can be covered by arbitrarily small balls). We won't be making such distinctions in this document though.

An interpretation of the above is as follows: "A doctor trained to treat cancer on some set of patients will succeed in treating it on any unseen patient with probability $1-\alpha$". Hearing this should give some comfort to any unfortunate patient going to this doctor.

However, the above definition is too strong to be practical. In particular, it can be proven that for continuous data and no constraint on its distribution, any prediction interval that guarantees coverage with probability $1 - \alpha$ with only finite samples must have infinite expected length (i.e. basically uninformative).
Intuitively, this isn't difficult to see, as when we say "distribution-free", we are allowing for all sort of nasty distributions with discontinuities. In such distributions, knowledge about points near $x$ tell us nothing about the behavior *at* $x$ (in fact, take cases where the PDF has a delta function exactly on $x$, we'll be in a lot of trouble).

For this reason, conditional coverage in a distribution-free setting is out of reach. For this reason, we should relax our definition. A much more relaxed definition is **Marginal Coverage**, where we expect things to work out only on average.
Formally, a distribution-free $\alpha$ marginally covering interval (denoted by $\alpha$-MC) means:
$$
\prob{Y_{n+1} \in \hat{C}(X_{n+1})} \geq 1 - \alpha
$$
For any measure $\mathbb{P}$.
Going back to the doctor example, a marginally covering doctor guarantees that over a random sample of 100 patients, it is right with probability $1-\alpha$. The trouble here is that if the samples aren't random, there is no guarantee! For example, it could be the case that the doctor predicts with 100 percent accuracy for men but fails completely for women. The only consolation that a woman patient will get for the accuracy of this doctor's practice is that it makes up by being really good for men! (which probably the women aren't thrilled about ...).

## Split Conformal Prediction

Piggybacking from our definition of $\alpha$-MC, we'll discuss the simplest form of conformal prediction.
Assume we have $n = n_1 + n_2$ feature-label pairs. We take $n_1$ samples and fit some predictive model $\hat{\mu}_{n_1}(x)$ (it doesn't matter what it is ...).
Define the prediction residuals over the remaining data as:
$$
R_i = \Big| Y_i - \hat{\mu}_{n_1}(X_i) \Big| \quad n_1+1 \leq i \leq n_2
$$
Sort the residuals in ascending order and let $\hat{q}_{n_1}$ be the $\lceil (1 - \alpha)(n_1+1) \rceil$-th residual in the sorted list.
The Split Conformal Prediction interval is then defined as:
$$
\hat{C}(x) := \Big[\hat{\mu}_{n_1}(x) - \hat{q}_{n_1}, \hat{\mu}_{n_1}(x) + \hat{q}_{n_1} \Big]
$$
