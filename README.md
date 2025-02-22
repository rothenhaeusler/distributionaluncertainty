
# Calibrated inference: inference that accounts for both sampling and distributional uncertainty

This package provides functions for calibrating linear models to account
for both sampling and distributional uncertainty. In addition, helper
functions are provided to simplify simulating from the distributional
perturbation model. Details on the theoretical background can be found
in [Jeong and Rothenhaeusler (2022)](https://arxiv.org/abs/2202.11886).

## How to install

1.  The [devtools](https://github.com/hadley/devtools) package has to be
    installed. You can install it using `install.packages("devtools")`.
2.  The latest development version can then be installied using
    `devtools::install_github("rothenhaeusler/calinf")`.

## Usage

First, one has to set a distributional seed. Then, one can draw from the
perturbed distribution.

``` r
d_seed <- distributional_seed(n = 1000, delta = 10)
# Sample from perturbed distribution 
# drnorm is the equivalent of rnorm for distributional perturbations
x <- drnorm(d_seed)

par(mfrow = c(1,2))
# In the qqplot there are clear deviations from the standard Gaussian distribution
qqnorm(x, main = "Normal Q-Q plot for drnorm")
# Compare to standard normal
qqnorm(rnorm(1000), main = "Normal Q-Q plot for rnorm")
```

<img src="README_files/figure-gfm/unnamed-chunk-2-1.png" style="display: block; margin: auto;" />

One can also draw from a multivariate perturbed distribution using one
distributional seed.

``` r
# Set distributional seed
d_seed <- distributional_seed(n = 1000, delta = 2)
# Draw from perturbed model
x <- drnorm(d_seed)
y <- drnorm(d_seed)
```

## Looking behind the curtain: a worked out example

**Case 1:** We consider the setting where the data scientist wants to
estimate the direct causal effect of some variable X1 on a target
variable Y based on a sample from a perturbed distribution, where the
strength of the perturbation is unknown to the data scientist. On
observational data, this is often done by computing regression-adjusted
estimators for different choices of adjustment sets that contain X1. Our
method uses variability between these estimators to estimate the
perturbation strength and performs calibrated inference for the direct
causal effect of X1 on Y.

``` r
delta <- 2

# Sample from a multivariate perturbation model stated in the paper
sample_data <-  function(){
  d_seed <- distributional_seed(n=500, delta=delta)
  X <- replicate(5, drnorm(d_seed))
  X[,2] <- X[,2] + X[,3]
  X[,1] <- X[,1] + 0.5 * X[,2] + X[,4]
  Y <- X[,1] + 0.5 * X[,2] + X[,3] + X[,5] + drnorm(d_seed)
  df <- as.data.frame(cbind(X,Y))
  colnames(df) =  c("X1", "X2", "X3", "X4", "X5", "Y")
  return(df)
}
```

``` r
# Create a list of linear regression formulas
formulas <- list(Y ~ X1 + X2, 
                 Y ~ X1 + X2 + X3,
                 Y ~ X1 + X2 + X4,
                 Y ~ X1 + X2 + X5,
                 Y ~ X1 + X2 + X3 + X4,
                 Y ~ X1 + X2 + X3 + X5,
                 Y ~ X1 + X2 + X4 + X5,
                 Y ~ X1 + X2 + X3 + X4 + X5)
```

``` r
# Apply calibrated inference
calm(formulas, data = sample_data(), target = "X1")
```

    ## 
    ## Quantification of both distributional and sampling uncertainty
    ## 
    ##    Estimate Std. Error Pr(>|z|)
    ## X1   1.1643      0.098        0
    ## 
    ## hat delta = 1.932455

``` r
# In the following, we evaluate the coverage of standard (sampling) confidence intervals
evaluate_sampling_ci <- function(i, k = 1){
  df <- sample_data() 
  # constructing sampling confidence intervals with nominal coverage .95
  lm.coef <- summary(lm(formulas[[k]], data = df))$coefficients["X1", 1:2]
  ci <- c(lm.coef[1] - qnorm(0.975) * lm.coef[2], lm.coef[1] + qnorm(0.975) * lm.coef[2])
}
confidence_intervals <- sapply(1:1000, evaluate_sampling_ci)
# For delta = 2, the actual coverage is approximately .65, which is far from the target coverage
mean(confidence_intervals[1,] <= 1 & 1 <= confidence_intervals[2,])
```

    ## [1] 0.669

``` r
# In the following, we evaluate the coverage of distributional confidence intervals
evaluate_distributional_ci <- function(i){
  df <- sample_data() 
  # constructing distributional confidence intervals with nominal coverage .95
  # Here, we use background knowledge about the population mean of Z1...Z4.
  invisible(capture.output(calm.coef <- calm(formulas, data = df, target = "X1")))
  tquantile <- qt(p=.975, df=length(formulas))
  ci <- c(calm.coef[1]-tquantile*calm.coef[2],calm.coef[1]+tquantile*calm.coef[2])
}
confidence_intervals <- sapply(1:1000, evaluate_distributional_ci)
# For delta = 2, the actual coverage is close to the nominal coverage .95
mean(confidence_intervals[1,] <= 1 & 1 <= confidence_intervals[2,])
```

    ## [1] 0.942

**Case 2:** We consider the setting where the data scientist wants to
estimate E\[X\] based on a sample from a perturbed distribution, where
the strength of the perturbation is unknown to the data scientist. We
assume that the population mean of Z1,…,Z4 is known to the data
scientist through background knowledge. This can be used to estimate the
perturbation strength, that means calibrate inference for E\[X\].

``` r
delta <- 10

# Sample from a multivariate perturbation model
sample_data <-  function(){
  d_seed <- distributional_seed(n=100,delta=delta)
  X <- drnorm(d_seed,mean=2,sd=2)
  Z1 <- drnorm(d_seed,mean=2.2,sd=2)
  Z2 <- drnorm(d_seed,mean=3.2,sd=3)
  Z3 <- drnorm(d_seed,mean=5.3,sd=4)
  Z4 <- drnorm(d_seed,mean=1,sd=2)
  df <- as.data.frame(cbind(X,Z1,Z2,Z3,Z4))
  return(df)
}
```

``` r
# In the following, we evaluate the coverage of standard (sampling) confidence intervals
evaluate_sampling_ci <- function(df){
  df <- sample_data() 
  # constructing sampling confidence intervals with nominal coverage .95
  sigmasq = var(df$X)/sqrt(100)
  tquantile <- qt(p=.975,df=999)
  ci <- c(mean(df$X)-tquantile*sqrt(sigmasq),mean(df$X)+tquantile*sqrt(sigmasq))
}
confidence_intervals <- sapply(1:1000,evaluate_sampling_ci)
# for delta=10, the actual coverage is approximately .45, which is far from the target coverage
mean(confidence_intervals[1,] <= 2 & 2 <= confidence_intervals[2,])
```

    ## [1] 0.47

``` r
# In the following, we evaluate the coverage of distributional confidence intervals
evaluate_distributional_ci <- function(i){
  df <- sample_data() 
  # constructing distributional confidence intervals with nominal coverage .95
  # Here, we use background knowledge about the population mean of Z1...Z4.
  sigmasq = var(df$X)*(mean(df$Z1-2.2)^2/var(df$Z1) + mean(df$Z2-3.2)^2/var(df$Z2) + mean(df$Z3-5.3)^2/var(df$Z3) + mean(df$Z4-1)^2/var(df$Z4))/4
  tquantile <- qt(p=.975,df=4)
  ci <- c(mean(df$X)-tquantile*sqrt(sigmasq),mean(df$X)+tquantile*sqrt(sigmasq))
}
confidence_intervals <- sapply(1:1000,evaluate_distributional_ci)
# For delta=10, the actual coverage is close to the nominal coverage .95
mean(confidence_intervals[1,] <= 2 & 2 <= confidence_intervals[2,])
```

    ## [1] 0.943
