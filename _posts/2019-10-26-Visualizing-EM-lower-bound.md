---
layout: post
title: "Visualizing EM lower bound updates"
date: 2019-10-26
tags: maths theory code ipython
image: surfing2.jpg
---

Let's start with a short recap. <span class="marker_underline"><strong>Expectation-Maximization</strong></span> is <span class="marker_underline">an iterative</span> <span class="marker_underline">method</span> <span class="marker_underline">that</span> <span class="marker_underline">performs</span> <span class="marker_underline">clustering</span>. EM maximizes data likelihood by updating current model's parameters with a sequence of E and M steps. What happens during E-steps and M-steps?

<span id="highlight">From practical perspective</span>:
* <strong>E-step</strong>: measure "how much" every data point belongs to each of the clusters, assign a responsibility vector based on that.
* <strong>M-step</strong>: update parameters of the clusters using the distribution of points that belong to it.

<span id="highlight">From theoretical perspective</span>:
* <strong>E-step</strong>: construct a lower bound estimate for data log-likelihood.
* <strong>M-step</strong>: update parameters of the clusters so that we reach maximum of the lower bound estimate.

When it comes to maximizing the lower bound of EM algorithm, many resources show the following illustration (originally taken from C. Bishop):
<br/>
<br/>
<img src="/assets/img/EM/bishop_EM.jpeg" width="50%" style="align:center">


This picture gives a great intuition about how EM updates maximize data log-likelihood in theory, but I've never come across a practical guide on how to derive a similar plot from real data.

<strong>In this tutorial I want to explore how EM updates work on real data</strong>. You will learn how to:
1. derive EM updates (we will use GMM as a model)
2. plot log-likehood and lower bound from some data
3. code everything in python


<br/>
I will show everything through Jupyter Notebook.

<strong><span class='underline_magical'>Step 1: get data and initialize a model</span></strong>. Let's start by importing the libraries that we will need:
```python
import numpy as np
import os
import matplotlib.pyplot as plt
import math
import scipy.stats as stats
import scipy
%matplotlib inline
```


We will try to cluster 100 points that come from two different distributions (I just picked those randomly):

$$X_1 \sim N(0, 0.16)$$
and
$$X_2 \sim N(2, 0.04)$$

```python
x1 = np.random.normal(size = 50, loc = 0, scale = 0.4)
x2 = np.random.normal(size = 50, loc = 2, scale = 0.2) # loc - mean, scale - std
```

We don't have labels in EM, so let's combine x1 and x2 arrays into one x array and plot all points:


```python
x = np.concatenate((x1,x2))

# let's try to visualize our original data
# since it's 1-D data, we'll do a scatter plot but plug in zeros for y values
plt.scatter(x, np.zeros((x.shape[0])), color='black', s=12, marker='o')
```

![png](/../assets/notebooks/EM/output_1.png)


We will use <span class="marker_underline">GMM (Gaussian Mixture Model)</span> as a model to perform EM. Under GMM model we assume that all data was generated by $N$ Gaussian distributions, in our case $N=2$.

Let's set some random initial values for our two Gaussians: <b>means</b> $\mu$, <b>standard deviations</b> $\sigma$ and <b>weights</b> of each Gaussian $\pi$. We will get to why we need the weights $\pi$ in a moment.

```python
mu1 = np.random.rand(1)/4 - 1
mu2 = np.random.rand(1)/4
sigma1 = sigma2 = 0.2

u = np.array([mu1, mu2])
sigma = np.array([sigma1, sigma2])
pi = np.array([0.5, 0.5])
```

We can plot our points together with our initial Gaussians:

```python
points = np.linspace(-1, 3, 1000)
plt.rcParams['figure.figsize']=(9,3)

# plotting PDFs of two normal distributions
for mu, s in zip(u, sigma):
    plt.plot(points, stats.norm.pdf(points, mu, s))

# plotting 1D data
plt.scatter(x, np.zeros((x.shape[0])), color='black', s=12, marker='o')
```

![png](/../assets/notebooks/EM/output_2.png)

<br/>
<span class='underline_magical'><strong>Step 2: plotting data log-likelihood</strong></span>. Data log-likelihood shows how probable it is to obtain the given data under current model parameters. <b><i>The goal of EM algorithm is to maximize data log-likelihood</i></b>:

$$l(\theta) =  \log p(X | \theta) = \sum_{i} \log p(x_i | \theta) \rightarrow \max_{\theta}$$

Here $\theta$ denotes model's parameters and $X = \{ x_1, ... ,x_m \} $ is our data.

Since EM model consists of $N$ "clusters" (in our case 2 Gaussians), we introduce $N$ latent variables $z_i$ that correspond to a cluster number and marginalize $l(\theta)$ wrt $z_i$:

$$l(\theta) =  \sum_{i} \log p(x_i | \theta) = \sum_{i} \log \sum_{z_i} p(x_i, z_i | \theta) = \sum_{i} \log \sum_{z_i} p(x_i | z_i ; \theta) \cdot p(z_i | \theta) $$

In our case of 2 Gaussians:

$$l(\theta) = \sum_{i} \log (\pi_1 N(x_i | \theta_1) + \pi_2 N(x_i | \theta_2))$$

You can see that:

$$ p(z_i | \theta) = \pi_{z_i}$$

$$ p(x_i, z_i | \theta) = N(x_i | \theta_{z_i}) $$

Let's compute data log-likelihood under current model:

```python
# here we calculate data log-likelihood
N1 = scipy.stats.norm(u[0], sigma[0])
N2 = scipy.stats.norm(u[1], sigma[1])

log_likelihood = 0
for i, point in enumerate(x):
    log_likelihood += np.log(pi[0]*N1.pdf(x[i])+pi[1]*N2.pdf(x[i]))

print('log_likelihood = ', log_likelihood)
```

    log_likelihood =  [-1305.22477413]


Now we can try to change one of the model's parameters and see how it affects data log-likelihood. Let's try changing mean of the second Gaussian and plot $l(\mu_2)$ with all other parameters being fixed.

```python
# create a function for calculating log-likelihood
def get_log_likelihood(x_arr, u_arr, sigma_arr, pi_arr):
    N1 = scipy.stats.norm(u_arr[0], sigma_arr[0])
    N2 = scipy.stats.norm(u_arr[1], sigma_arr[1])

    log_likelihood = 0
    for i, point in enumerate(x_arr):
        log_likelihood += np.log(pi_arr[0]*N1.pdf(x_arr[i])+pi_arr[1]*N2.pdf(x_arr[i]))

    return log_likelihood
```


```python
# let's try to visualize data log-likelihood
# to get a 2D plot we'll fix all parameter to their current values
# except for u[1] (mean of the second Gaussian), which we will vary
mu = np.linspace(1, 4, 100)
plt.plot(mu, get_log_likelihood(x, np.array((u[0], mu)), sigma, pi))
plt.xlabel('2nd Gaussian mean')
plt.ylabel('Data log-likelihood')
```

![png](/../assets/notebooks/EM/output_3.png)

Looks like an optimal value for $\mu_2$ is close to 2. What is our current value?
```python
print(u[1])
1.15
```

We can vary any other parameter as well and get more plots like that. Here is std plot:
```python
# let's try fixing everything except for sigma[0] (first Gaussian std)
s = np.linspace(0, 3, 100)
plt.plot(s, get_log_likelihood(x, u, np.array((s, sigma[1])), pi))
plt.xlabel('1st Gaussian std')
plt.ylabel('Data log-likelihood')
```

![png](/../assets/notebooks/EM/output_4.png)


Since we were only varying one parameter at a time, it's very hard to find optimal values for all parameters by this method.


Let's perform series of EM updates.

<br/>
<br/>
<span class='underline_magical'><strong>Step 3: E update</strong></span>. We perform E-step by assigning each point $x_i$ its responsibility vector
$$\begin{bmatrix} r_{i1} \\ r_{i2} \end{bmatrix}$$
, where $r_{ij}$ are probabilities that the corresponding point comes from Gaussian $j$.

In our example:

$$ r_{i1}  = \frac{\pi_1 \cdot N(x_i | \theta_1)}{\pi_1 \cdot N(x_i | \theta_1) + \pi_2 \cdot N(x_i | \theta_2)}$$

$r_{i2}$ calculation is analogous. Let's code this equation:

```python
# here we do E-step: find responsibilities for each data point in x
N1 = scipy.stats.norm(u[0], sigma[0])
N2 = scipy.stats.norm(u[1], sigma[1])

R = np.empty((x.shape[0],2))

for i, point in enumerate(x):
    R[i,0] = pi[0]*N1.pdf(x[i]) / ( pi[0]*N1.pdf(x[i])+pi[1]*N2.pdf(x[i]) )
    R[i,1] = pi[1]*N2.pdf(x[i]) / ( pi[0]*N1.pdf(x[i])+pi[1]*N2.pdf(x[i]) )
```

To visualize how responsibilities were assigned, let's color each point to the color of its highest responsibility Gaussian. So, if for the first point $ r_{11} < r_{12}$ we will color it orange (as the second Gaussian).


```python
# two values of each point's responsibility vector show
# the probability of data point coming from one of two Gaussians
# to visualize how responsibilities were assigned,
# let's plot points in orange / blue depending on which value of their responsibility vector is higher
for i, point in enumerate(x):
    if R[i,0] < R[i,1]:
        col = 'orange'
    else:
        col = 'blue'
    plt.scatter(point, np.zeros((1)), color=col, s=12, marker='o')

# and let's plot our Gaussians as well
points = np.linspace(-1, 3, 1000)

for mu, s in zip(u, sigma):
    plt.plot(points, stats.norm.pdf(points, mu, s))
```


![png](/../assets/notebooks/EM/output_5.png)



The picture above makes a lot of sense! Every point get colored based on which Gaussian is more likely to produce it (i.e. which PDF is higher at that point).

<br/>
<span class='underline_magical'><strong>Step 4: M update</strong></span>. We perform M-step by updating parameters of each Gaussian using each point's responsibility information from E-step. First we will save old parameter values (we will need them for our plots soon!).

```python
# now we are moving to M-step, where we update the parameters
# let's save old parameter values before update
u_old = u
sigma_old = sigma
pi_old = pi
```

Equations for M-step updates are the following:

$$ \pi_j = \frac{1}{N}\sum_{i}r_{ij}$$

$$ \mu_j = \frac{ \sum_{i}r_{ij}x_i }{ \sum_{i}r_{ij} }$$

$$ \sigma_j^2 = \frac{ \sum_{i}r_{ij}(x_i-\mu_j)^2 }{ \sum_{i}r_{ij} }$$


We can get updated parameter values by using these equations:
```python
# now let's perform M-step: update parameter values
# using responsibilities from E-step
pi = np.average(R, axis=0)

u = np.empty(2)
for i in range(2):
    u[i] = np.dot(R[:,i],x) / np.sum(R[:,i])

sigma = np.empty(2)
for i in range(2):
    sigma[i] = no.sqrt( np.dot(R[:,i],(x-u[i])**2) / np.sum(R[:,i]) )
```

Now let's look at our new Gaussians' PDFs

```python
# plot new Gaussians
points = np.linspace(-1, 3, 1000)
plt.rcParams['figure.figsize']=(9,3)

for mu, s in zip(u, sigma):
    plt.plot(points, stats.norm.pdf(points, mu, s))

# plot points
plt.scatter(x, np.zeros((x.shape[0])), color='black', s=12, marker='o')
```

![png](/../assets/notebooks/EM/output_6.png)

Great! It's clear that new model explains our data much better.

But let's get back to the equations and understand why did we get those equations for E-step and M-step.

<br/>
<span class='underline_magical'><strong>Step 5: understanding EM updates from theoretical perspective</strong></span>.

<strong>Why do we break the optimization process into E and M steps?</strong> As we learnt before, the goal of EM algorithm is to maximize data log-likelihood by optimizing model's parameters:

$$l(\theta) =  \sum_{i} \log p(x_i | \theta) = \sum_{i} \log \sum_{z_i} p(x_i, z_i | \theta) \rightarrow \max_{\theta} $$

However, because we introduced latent variable $z_i$ and marginalized $l(\theta)$, now we have summation inside the log function. This makes it very hard to optimize data log-likelihood directly...

What can we do? <span id="highlight">We can find some approximation for $l(\theta)$ that doesn't have a sum inside log and try to optimize that function</span>. To get rid of the log inside the sum we will use <strong>Jensen's inequality</strong>, which allows us to swap log and summation:

In case of a <u>concave</u> function $g$ (like log function)

$$ \mathbb{E}[g(X)] \leqslant	g(\mathbb{E}[X]) $$

We can rewrite this equation for the log function case:

$$ \sum_{i}p(y_i)\log(y_i) \leqslant	\log(\sum_{i}p(y_i)y_i) $$

Then we can introduce some probability distribution $Q_i(z_i)$ as $p(y_i)$ and plug in rest of variables under log into $y_i$ to get lower bound estimate for $l(\theta)$:

$$ l(\theta) = \sum_{i} \log \sum_{z_i} p(x_i, z_i | \theta) = \sum_{i} \log \sum_{z_i} Q_i(z_i) \frac{p(x_i, z_i | \theta)}{Q_i(z_i)} \\
\geqslant \sum_{i} \sum_{z_i} Q_i(z_i) \log \frac{p(x_i, z_i | \theta)}{Q_i(z_i)} $$  

We want inequality to turn into equality, according to Jensen's inequality this happens when:

$$ \frac{p(x_i, z_i | \theta)}{Q_i(z_i)} = const $$  

So we pick
$Q_i(z_i) \propto	p(x_i, z_i | \theta)$:


$$ Q_i(z_i) =  \frac{p(x_i, z_i | \theta)}{ \sum_{j} p(x_i, z_j | \theta) } = \frac{p(x_i, z_i | \theta)}{ p(x_i | \theta) }  = p(z_i | x_i, \theta)$$

In our case,
$p(z_i | x_i, \theta)$
are the responsibilities of each data point. The <strong>lower bound estimate takes the following form</strong>:

$$\mathcal{L}(q, \theta) = \sum_{i} \sum_{z_i} Q_i(z_i) \log \frac{p(x_i, z_i | \theta)}{Q_i(z_i)} =
 \sum_{i} \sum_{z_i} p(z_i | x_i, \theta_{old}) \log \frac{p(x_i, z_i | \theta)}{p(z_i | x_i, \theta_{old})}  $$

At <strong>E-step</strong> we are fixing $Q_i(z_i)$ at our current parameter values to
$p(z_i | x_i, \theta_{old})$
to get lower bound estimate.

At <strong>M-step</strong> we are updating
$p(x_i, z_i | \theta)$
by updating $\theta$ to get maximal value of the lower bound. We find optimal values for each model parameter $\theta_j$ by setting derivatives $\frac{\partial \mathcal{L}(\theta_j)}{\partial \theta_j}=0$.

Now we understand the illustration from the beginning of this post:
<br/>
<img src="/assets/img/EM/bishop_EM.jpeg" width="50%" style="align:center">
* we cannot optimize data log-likehood directly (red plot)
* so we construct a lower bound estimate $\mathcal{L(q, \theta)}$ (blue plot)
* $\mathcal{L(q, \theta)}$ is equal to log-likehood at  $\theta_{old}$
* then we find new parameters $\theta_{new}$ that maximize $\mathcal{L}(q, \theta)$ and construct a new lower bound (green plot)

How can we get this plot from our data?

<br/>
<span class='underline_magical'><strong>Step 6: plotting data log-likelihood and lower bound updates from real data</strong></span>

Let's code lower bound equation and see if it's equals to data log-likelihood at our old parameter values.

```python
# code for lower bound
N1 = scipy.stats.norm(u_old[0], sigma_old[0])
N2 = scipy.stats.norm(u_old[1], sigma_old[1])

lower_bound = 0
for i, point in enumerate(x):
    lower_bound += R[i,0]*np.log(pi_old[0]*N1.pdf(x[i])/R[i,0]) + \
                   R[i,1]*np.log(pi_old[1]*N2.pdf(x[i])/R[i,1] )

print("lower_bound = ", lower_bound)
log_likelihood = get_log_likelihood(x, u_old, sigma_old, pi_old)
print("log_likelihood = ", log_likelihood)
```

    lower_bound    =  [-1305.22477413]
    log_likelihood =  [-1305.22477413]


Perfect match! Now let's code lower bound equation into a function:


```python
def get_lower_bound(x_arr, u_arr, sigma_arr, pi_arr, R):
    N1 = scipy.stats.norm(u_arr[0], sigma_arr[0])
    N2 = scipy.stats.norm(u_arr[1], sigma_arr[1])

    # make it into array because we will be varying 100 values
    # for one of the Gaussian parameters
    lower_bound = np.empty(100)
    for i, point in enumerate(x_arr):

        # to avoid numerical overflow in case of small values
        if (R[i,1]<0.0001):
            lower_bound += R[i,0]*np.log(pi_arr[0]*N1.pdf(x_arr[i])/R[i,0])
        elif (R[i,0]<0.0001):
            lower_bound += R[i,1]*np.log(pi_arr[1]*N2.pdf(x_arr[i])/R[i,1] )

        # standard case
        else:
            lower_bound += R[i,0]*np.log(pi_arr[0]*N1.pdf(x_arr[i])/R[i,0]) + \
                           R[i,1]*np.log(pi_arr[1]*N2.pdf(x_arr[i])/R[i,1])

    return lower_bound
```

Now we can <span class="marker_underline">plot lower bound estimate</span>. We do it by fixing all parameters except for the one that we decide to vary (in this case mean of the second Gaussian):

```python
mu = np.linspace(-2, 5, 100)
plt.plot(mu, get_lower_bound(x, np.array((u_old[0], mu)), sigma, pi, R), '--', color='salmon')
```

![png](/../assets/notebooks/EM/output_7.png)


<strong>Let's plot data log-likelihood and lower bound functions</strong> together on one plot. Again, we fix all parameters except for $\mu_2$ that we use as an argument of those functions. Also, let's plot $\mu_2$ initial value and its value after update together with $l(\mu_2)$ and $\mathcal{L}(\mu_2)$.


```python
mu = np.linspace(-2, 5, 100)
# plot log-likelihood with old parameter values, u2 is varied
plt.plot(mu, get_log_likelihood(x, np.array((u_old[0], mu)), sigma_old, pi_old), color='deeppink')
# plot lower bound estimate with old parameter values, u2 is varied
plt.plot(mu, get_lower_bound(x, np.array((u_old[0], mu)), sigma_old, pi_old, R), '--', color='salmon')
plt.xlabel('2nd Gaussian mean')
plt.ylabel('Data log-likelihood')

# now let's look at how our parameters have been updated
lower_bound = get_lower_bound(x, np.array((u_old[0], mu)), sigma_old, pi_old, R)

# u2_max = u2 value that maximizes lower bound
mu_max = mu[ np.argmax(lower_bound) ]
# log-likelihood value at u2_max
max_l = get_log_likelihood(x, np.array((u_old[0], mu_max)), sigma_old, pi_old)

# log-likelihood value with old parameters
old_l = get_log_likelihood(x, np.array((u_old[0], u_old[1])), sigma_old, pi_old)

# log-likelihood value with new parameters
new_l = get_log_likelihood(x, np.array((u_old[0], u[1])), sigma_old, pi_old)

plt.scatter(u_old[1], old_l, color='black', marker='x', s=64, zorder=10)
plt.scatter(mu_max, max_l, color='black', marker='x', s=64, zorder=10)
plt.scatter(u[1], new_l, color='red', marker='x', s=64, zorder=10)
```

![png](/../assets/notebooks/EM/output_8.png)

We've done it!

Solid pink plot is data log-likelihood, dashed plot is lower bound estimate.

Look how initial $\mu_2$ value (left black cross) was optimized (red cross) to get maximum of the lower bound estimate (right black cross).


<br/>
We can also plot log-likelihood and lower bound functions before and after parameter update.

```python
# now let's add to our plot updated lower bound and log-likelihood functions
# plot previous plot
plt.plot(mu, get_log_likelihood(x, np.array((u_old[0], mu)), sigma_old, pi_old), color='deeppink')
plt.plot(mu, get_lower_bound(x, np.array((u_old[0], mu)), sigma_old, pi_old, R), '--', color='salmon')
plt.scatter(u_old[1], old_l, color='black', marker='x', s=64, zorder=10)
#plt.scatter(mu_max, max_l, color='black', marker='x', s=64)
#plt.scatter(u[1], new_l, color='red', marker='x', s=64)

# plot new functions
plt.plot(mu, get_log_likelihood(x, np.array((u[0], mu)), sigma, pi), color='royalblue')
plt.plot(mu, get_lower_bound(x, np.array((u[0], mu)), sigma, pi, R), '--', color='cornflowerblue')
new_l = get_log_likelihood(x, np.array((u[0], u[1])), sigma, pi)
plt.scatter(u[1], new_l, color='red', marker='x', s=64, zorder=10)

plt.xlabel('2nd Gaussian mean')
plt.ylabel('Data log-likelihood')
```


![png](/../assets/notebooks/EM/output_9.png)

You can find the full code + Jupyter notebook for this tutorial <a href='https://github.com/arazd/Visualizing-EM'>at the github repository</a>.

The end :) thanks for reading!
