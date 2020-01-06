---
layout: post
title: "Humble Jump Filter"
date: 2020-01-05 22:15:00 -0500
categories: Filters
---

_A humble methodology to filter jumps from price returns observations using maximum likelihood._

_The code used may be found [here][repo]._

---
<br/>
# 1. Introduction

The starting point for modeling a time series of price returns $$(r_t)_{t=1}^T$$ is to assume they are i.i.d. [Gaussian random variables][gaussian-dist] with mean $$\mu$$ and standard deviation $$\sigma$$. In other words, we write them as

$$r_t = \mu + u_t,\qquad t\in\{1,\ldots,T\},$$

where each $$u_t$$ is a centered Gaussian random variable with standard deviation $$\sigma$$.

Although this assumption is very convenient from an operational perspective -- for example, the sum of two returns is also Gaussian -- one may be interested in allowing the prices to **jump**. Why? Because it is a *natural* way to represent large movements driven by major news shocks. A common way do so is by adding a jump term to the previous model and write

$$r_t = \mu + u_t + z_t\xi_t,\qquad t\in\{1,\ldots,T\},$$

where we also assume each $$z_t$$ follows a Gaussian distribution with mean $$\mu_J$$ and standard deviation $$\sigma_J$$, while $$\xi_t$$ follows a [Bernoulli distribution][bernoulli-dist] with parameter $$\lambda>0$$.[^merton] For simplicity, we will also assume that the $$\epsilon$$ and $$z$$ are two independent processes. Therefore, the vector containing our model parameters is given by

$$\boldsymbol{\Theta} = \begin{bmatrix}
\mu & \sigma & \lambda & \mu_J & \sigma_J
\end{bmatrix}^\intercal.$$

Notice when the price moves smoothly, ie. $$\xi_t= 0$$, we recover the original expression for the price return. However, when the price jumps, ie. $$\xi_t= 1$$, the return corresponds to the sum of two independent Gaussian random variables. In other words, we may interpret this model as if the price returns were sampled with probability $$1 - \lambda$$ from a Gaussian distribution with parameters $$(\mu, \sigma^2)$$ and with probability $$\lambda$$, with parameters $$(\mu+\mu_J, \sigma^2+\sigma^2_J)$$. This type of model is also known as a [Mixture Model][mixture-model].

I have denominated this model as **humble** because there is a vast set of more complex models. However, it should be enough to keep us entertained for a while.

# 2. How to calibrate it?

The model introduced can be calibrated using the **maximum likelihood estimation** ([MLE][mle]) technique. Denote by $$\boldsymbol{r}^T$$ the vector of price returns observations until $$T$$. Given a set of parameters $$\boldsymbol{\Theta}$$, the definition of [conditional probability][cond-prob] allows us to write the likelihood of said observations as

$$p\left(\boldsymbol{r}^T \;\middle|\; \boldsymbol{\Theta}\right)
= p\left(r_T, \boldsymbol{r}^{T-1} \;\middle|\; \boldsymbol{\Theta}\right)
= p\left(r_T \;\middle|\; \boldsymbol{r}^{T-1}, \boldsymbol{\Theta}\right) p\left(\boldsymbol{r}^{T-1} \;\middle|\; \boldsymbol{\Theta}\right).
$$

After applying this relation recursively and taking logarithms, we may define the [log-likelihood function][llf] as

$$L\left(\boldsymbol{\Theta}\right) = \log p\left(\boldsymbol{r}^T \;\middle|\; \boldsymbol{\Theta}\right)
= \log\prod_{t=1} ^ T p\left(r_t \;\middle|\; \boldsymbol{r}^{t-1}, \boldsymbol{\Theta}\right)
= \sum_{t=1} ^ T \log p\left(r_t \;\middle|\; \boldsymbol{r}^{t-1}, \boldsymbol{\Theta}\right).
$$

Additionally, our model is so humble that, given a set of parameters, the price return today is independent from the past returns. Hence, we may simply write

$$
p\left(r_t \;\middle|\; \boldsymbol{r}^{t-1}, \boldsymbol{\Theta}\right)
= p\left(r_t \;\middle|\; \boldsymbol{\Theta}\right)
= (1-\lambda)g\left(r_t \;\middle|\; \mu, \sigma^2\right) + \lambda g\left(r_t \;\middle|\; \mu + \mu_J, \sigma^2 + \sigma_J^2\right),
$$

where $$g$$ is the normal probability density of a Gaussian variable given by

$$
g\left(r_t \;\middle|\; \mu, \sigma^2\right)
= \frac 1{\sqrt{2\pi\sigma^2}}\exp\left(- \frac{(x - \mu)^2}{2\sigma^2}\right).
$$

Based on these expressions, the MLE estimator for the parameters vector $$\boldsymbol{\hat\Theta}$$ is obtained by maximizing the log-likelihood function $$L$$.

Lastly, using the estimated parameters, we may compute the probability that jump actually occurred. Simply notice the [Bayes theorem][bayes] yields that

$$p\left(\xi_t\;\middle|\; r_t, \boldsymbol{\Theta}\right) = \frac{p\left(r_t \;\middle|\; \xi_t, \boldsymbol{\Theta}\right)p\left(\xi_t\;\middle|\; \boldsymbol{\Theta}\right)}{p\left(r_t \;\middle|\; \boldsymbol{\Theta}\right)},$$

where each expression involved on the right-hand side was computed above. Actually, for our particular model, it might be shown that a return will be identified as a jump only if a second-order inequality on $$r_t$$ is satisfied.[^messy]

# 3. Well, does it work?

After all these equations, I threw some pseudo-random numbers into the ring. A sample of 10,000 observations was generated from a set of parameters that seemed reasonable for daily returns. Specifically, these values imply a mean return (in absence of jumps) of 5% per annum with an annual volatility of 20%. Additionally, a jump is expected to be seen on 5% of the days, shocking the price returns by a Gaussian variable with mean -6% and volatility 3%.

The generated sample is displayed by the following histogram; at least at first sight, one might believe these returns come from an actual asset -- perhaps a volatile stock?

![hist](/assets/hist.png)

The results of the estimation are presented in the following table.[^stderr] As it may be seen, the estimated values are quite close to the actual ones and within two -- or even one -- standard deviations. Pretty decent.[^return]

| Parameter	    | Actual 	| Estimated | StErr 	|
|:------:	    |:------:	|:------:	|:------:	|
| $$\mu$$     	|  0.020 	|  0.017 	|  0.014 	|
| $$\sigma$$  	|  1.260 	|  1.265 	|  0.010	|
| $$\lambda$$ 	|  5.000 	|  4.574 	|  0.375 	|
| $$\mu_J$$    	| -6.000 	| -6.118	|  0.358 	|
| $$\sigma_J$$ 	|  3.000 	|  3.037 	|  0.257 	|

Now, if we identify a jump only when its probability is greater than 50%, how many of the actual jumps did the filter capture? To answer this, let's look at the [confusion matrix][confMatrix]. From it we may read that most of the no-jump movements were correctly identified as such. Meanwhile, from 469 actual jumps, only 352 were correctly labeled as such while 117 were not mislabeled.[^jumps] Should we worry about missing those? 

|  Predicted vs. Actual | No Jump | Jump |  Total |
|:---------------------:|:-------:|:----:|:------:|
|**No Jump**            |  9,515  |  117 |  9,632 |
|**Jump**               |    16   |  352 |    368 |
|**Total**              |  9,531  |  469 | 10,000 |

Not necessarily. The following plot displays the simulated returns label accordingly with their actual nature (no jump or jump) vs. the probability they will be identified as jumps by the filter. What happened was that only the large movements -- those that would appear on the newspapers front page -- are identified as jumps.[^posreturns] Sometimes the jump occurred but wasn't large enough to move the return dramatically away from zero. On the other hand, observe that the 16 dark points identified as jumps, were quite strong plummets. Lastly, after a certain magnitude, the filter identifies a jump almost surely.

![inferredJumps](/assets/inferredJumps.png)

# 4. Takeaway

The stuff presented here is a simple formalization of some practitioners' ideas about extreme returns. In particular, the normality of the distribution is perturbed because there are two distribution playing at the same time. This framework allows us to put some numbers to it; give a simple rule to decide from which distribution was the return sampled.

---

[^merton]: This model actually corresponds to a discrete-time version of the jump-diffusion model introduced by [Merton (1976)][merton-76]. 
[^stderr]: The standard errors were computed using the popular *sandwich* estimator proposed by White involving the Hessian and the outer product of the gradients.
[^messy]: Things get a bit messy and nobody wants to see that, so I decided to omit it.
[^return]: Although, the estimated mean return $$\hat\mu$$ ends up being non-statistically different from zero -- as usual.
[^jumps]: Notice that the estimated parameter $$\hat\lambda$$ is much closer to the proportion of of actual jumps (4.69%) in the sample than the actual value (5.00%).
[^posreturns]: Eventually, a large positive return could also imply a jump.

[gaussian-dist]: https://en.wikipedia.org/wiki/Normal_distribution
[bernoulli-dist]: https://en.wikipedia.org/wiki/Bernoulli_distribution
[mixture-model]: https://en.wikipedia.org/wiki/Mixture_model
[mle]: https://en.wikipedia.org/wiki/Maximum_likelihood_estimation
[cond-prob]: https://en.wikipedia.org/wiki/Conditional_probability
[llf]: https://en.wikipedia.org/wiki/Likelihood_function
[bayes]: https://en.wikipedia.org/wiki/Bayes%27_theorem
[confMatrix]: https://en.wikipedia.org/wiki/Confusion_matrix
[merton-76]: https://www.sciencedirect.com/science/article/abs/pii/0304405X76900222?via%3Dihub
[repo]: https://github.com/mflopezabu/jumpFilter