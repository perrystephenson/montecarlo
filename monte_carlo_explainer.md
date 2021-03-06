Monte Carlo
================

After battling through a Monte Carlo example in a poorly designed Excel workbook at uni, I decided to put it all together in R to make sure I know how it really works. This won't cover too much of the **why** for Monte Carlo (the lecture notes cover that pretty well), rather it will cover the mechanics of how you make it work. This is demonstrably possible to do in Excel, but it's much easier to understand programmatically.

The basics
----------

The core idea of Monte Carlo simulation is that you are building a model which represents your beliefs about the world, and then using simulated data generated in accordance with those beliefs to understand the probable outcomes of that model. We'll work through an example of how all of this works, but for now the key points are:

-   This technique does not (can not) use data. It is entirely based on your assumptions and beliefs about the world.
-   This technique lets you encode your beliefs as probability distributions (rather than point estimates) which lets you reason about the uncertainty in your beliefs
-   This technique produces a probability distribution as an output, which lets you understand the range of likely outcomes
-   This technique lets you use numerical approximation methods to quickly estimate the solution to problems that would otherwise require complex analytical solutions, or may have no exact solution
-   This technique makes no attempt to assess the relative sensitivity of the model to each of your assumptions, although performing such analysis would be a trivial extension of the technique.

A worked example
----------------

Let's say we want to estimate the duration and cost of a project. We know some things about the tasks in this project:

1.  There are 4 tasks. Tasks 1 and 2 can be completed at the same time, but task 3 depends on the outputs of tasks 1 and 2, and task 4 relies on the output of task 3.
2.  The cost per day for each task is known.
3.  The minimum, maximum, and most likely time taken to complete each task is able to be estimated based on business knowledge.

Let's start with the third point - the estimation of each task. Based on these three estimated values, we can simplify the problem by making an assumption that the uncertainty in our estimate of the time taken to complete each task follows a triangular distribution. If you're not familiar with probability distributions, this might require further explanation.

### Triangular distributions

A probability distribution represents uncertainty in a value. A triangular distribution looks like this:

    ## Warning: package 'triangle' was built under R version 3.4.2

![](monte_carlo_explainer_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-1-1.png)

In this plot the blue line represents my beliefs about the the values that X might be. I believe that it is between 1 and 5, and that it is most likely to be 2. The red bars are the distribution of observed values when I sampled 10,000 values from this distribution. This is the key bit of Monte Carlo - I can encode my beliefs as a distribution, and then sample from that distribution to model the uncertainty.

Let's take a deeper look at how the triangular distribution works.

The probability density function for a triangular distribution is fairly easy to determine using the rules we all learned about triangle geometry in primary school. Formally though, the distribution is defined as:

$$
\\begin{cases}
    0 & \\text{for } x &lt; a, \\\\
    \\frac{2(x-a)}{(b-a)(c-a)} & \\text{for } a \\le x &lt; c, \\\\\[4pt\]
    \\frac{2}{b-a}             & \\text{for } x = c, \\\\\[4pt\]
    \\frac{2(b-x)}{(b-a)(b-c)} & \\text{for } c &lt; x \\le b, \\\\\[4pt\]
    0 & \\text{for } b &lt; x.
 \\end{cases}
$$

where *a* is the lowest value *x* may take, *b* is the highest value *x* may take, and *c* is your estimation of the most likely value for *x*. We can define this probability distribution as a function in R:

``` r
library(dplyr)
triangle_pdf <- function(x, a, b, c) {
  case_when(
    (x < a) ~ 0,
    (x >= a & x < c) ~ (2*(x-a))/((b-a)*(c-a)),
    (x == c) ~ 2/(b-a),
    (x > c & x <= b) ~ (2*(b-x))/((b-a)*(b-c)),
    (x > b) ~ 0
  )
}
```

Let's plot some values for our function to make sure it's correct:

``` r
plot(triangle_pdf(x = seq(0,6,0.1), a = 1, b = 5, c = 2), type='l')
```

![](monte_carlo_explainer_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-3-1.png)

Good.

Now we need to work out how to sample from this distribution so that we can use these samples in a Monte Carlo. This isn't exactly straight forward - R gives us some ways to generate random numbers but none of these random number generators can draw from our new triangular distribution. In order to do this effectively, we need to first calculate the "cumulative density function" (CDF) for the distribution above. This function is the integral (area under the curve) for the figure above, and it estimates the probability that a point in distribution will be less than or equal to the x. We can calculate this integral numerically to demonstrate how it works.

``` r
X <- seq(from = 0, to = 6, by = 0.01)
prob <- triangle_pdf(X, a = 1, b = 5, c = 2) * 0.01
plot(cumsum(prob), type='l')
```

![](monte_carlo_explainer_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-4-1.png)

Hooray! It looks just like the one on Wikipedia! We can actually calculate this directly using the formula for a cumulative density function:

$$
\\begin{cases}
    0 & \\text{for } x \\leq a, \\\\\[2pt\]
    \\frac{(x-a)^2}{(b-a)(c-a)} & \\text{for } a &lt; x \\leq c, \\\\\[4pt\]
    1-\\frac{(b-x)^2}{(b-a)(b-c)} & \\text{for } c &lt; x &lt; b, \\\\\[4pt\]
    1 & \\text{for } b \\leq x.
  \\end{cases}
$$

And just like with the probability density function, we can implement this in R:

``` r
triangle_cdf <- function(x, a, b, c) {
  case_when(
    (x <= a) ~ 0,
    (x > a & x <= c) ~ ((x-a)^2)/((b-a)*(c-a)),
    (x > c & x < b) ~ 1 - ((b-x)^2)/((b-a)*(b-c)),
    (x >= b) ~ 1
  )
}
```

Just like with the PDF, let's plot some values for our function to make sure it's correct:

``` r
plot(triangle_cdf(x = seq(0,6,0.1), a = 1, b = 5, c = 2), type='l')
```

![](monte_carlo_explainer_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-6-1.png)

Killing it!

Now we can look at why we're bothering with the CDF. R, much like Excel, has a couple of probability density functions built in, which you can sample from. The typical ones you might use are the normal distribution:

``` r
hist(rnorm(100000), breaks=50, col = "red", probability=TRUE)
```

![](monte_carlo_explainer_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-7-1.png)

or the uniform distribution:

``` r
hist(runif(100000), breaks=50, col = "red", probability=TRUE)
```

![](monte_carlo_explainer_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-8-1.png)

In this case, we can use our triangle CDF to transform points the uniform distribution into a triangle distribution! Unfortunately, this requires using the CDF "backwards", i.e. taking the inverse. Logically this looks like the following workflow:

1.  Take a random point from the uniform distribution between 0 and 1
2.  Find the value of X which gives this random point when you call **triangle\_cdf**
3.  Record that value of X and repeat

The values of X can be shown to be *drawn* from the triangle distribution. Let's do this process numerically to demonstrate how it works, before we cheat and look at how to do this analytically.

``` r
inverse_cdf <- function(Y, a, b, c) {
  X_vals <- seq(from = 0, to = 6, by = 0.001)
  CDF_vals <- triangle_cdf(X_vals, a=a, b=b, c=c)
  idx <- rep(NA, length = length(Y))
  for (i in seq_along(Y)) {
    idx[i] <- which.min(abs(CDF_vals - Y[i]))
  }
  return(X_vals[idx])
}
```

``` r
uniform_samples <- runif(10000, min = 0, max = 1)
triangle_samples <- inverse_cdf(uniform_samples, a = 1, b = 5, c = 2)
hist(triangle_samples, breaks=50, col = "red", probability=TRUE)
```

![](monte_carlo_explainer_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-10-1.png)

This is great! By creating our own "inverse CDF" function, we're now able to generate random data from a triange distribution, which is a key requirement for this Monte Carlo simulation. Now that we've shown how it works, we can cheat and use the analytically derived formula:

$$
X = \\begin{cases}
a + \\sqrt{U(b-a)(c-a)} & \\text{ for } 0 &lt; U &lt; F(c) \\\\ & \\\\
b - \\sqrt{(1-U)(b-a)(b-c)} & \\text{ for } F(c) \\le U &lt; 1
\\end{cases}
$$

where *F*(*c*)=(*c* − *a*)/(*b* − *a*), has a triangular distribution with parameters *a*, *b* and *c*.

``` r
triangle_sample <- function(n, a, b, c) {
  
  # Generate some data from the uniform distribution
  U <- runif(n, min = 0, max = 1)
  
  # Use the inverse CDF to find the corresponding point from the triangular distribution
  Fc <- (c-a)/(b-a)
  case_when(
    (U > 0 & U < Fc) ~ a + sqrt(U * (b-a) * (c-a)),
    (U >= Fc & U < 1) ~ b - sqrt((1-U) * (b-a) * (b-c))
  )
}
```

``` r
hist(triangle_sample(100000, a = 1, b = 5, c = 2), breaks=50, col = "red", probability=TRUE)
```

![](monte_carlo_explainer_files/figure-markdown_github-ascii_identifiers/unnamed-chunk-12-1.png)

Finally! Now we can get on with the Monte Carlo simulation.

### The rest of Monte Carlo

With a way to generate data from a triangle distribution, it's pretty straight-forward to build a simulation and estimate some values.
