
<!-- README.md is generated from README.Rmd. Please edit that file -->

# PI3M

<!-- badges: start -->
<!-- badges: end -->

This `R` package fits our prevalence-incidence mixture model to
interval-censored data to estimate the cumulative disease risk in a
population with a temporarily elevated risk of disease, e.g., risk of
CIN2+ in HPV-positive women.

In longitudinal screening studies it is possible to observe prevalent,
early or late events during follow-up. In our model, early events are
modelled via a competing risks framework, infections either progress to
the disease state or to a (latent) “clearance” state (i.e. viral
clearance) at constant rates. Late events are modelled by adding
background risk to the model. Parameters can depend on individual risk
factors and are estimated with an expectation-maximisation (EM)
algorithm with weakly informative Cauchy priors. More details are given
in the accompanying paper.

There are five main functions in this package:

- `PI3M.fit`: fits our Prevalence-Incidence-Cure model
- `PI3M.predict`: makes predictions from the model
- `PI3M.simulator`: simulates data under user-specified parameter values
  and covariates
- `score.test.gamma`: performs a score test whether the shape parameter
  of the Gamma distribution for progression is equal to one or not.
- `simulator.gamma`: simulates data where progression follows a Gamma
  rather than exponential distribution and so can have a shape parameter
  not equal to one. Does not allow for covariates, yet.

## Installation

You can install the most recent version of `PI3M` from
[GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("kelsikroon/PI3M")
```

## Examples

This is a basic example in a setting without covariates which
illustrates how to simulate data, fit the model, make predictions from
the plot and compare to a non-parametric cumulative incidence curve:

``` r
library(PI3M)
sim.thetas <- c(-5, -1.6, -1.2, -3)
sim.dat <- PI3M.simulator(1000, params = sim.thetas, show_prob = 0.9, interval=3, include.h=T)
head(sim.dat) # view simulated data
#>       left    right z      age    age.std hpv cyt cause     actual
#> 1 24.13198      Inf 0 47.08226 -0.1281307   0   0     3 135.089499
#> 2  0.00000 2.856223 0 33.52999 -0.7261583   1   1     2   2.374809
#> 3 23.75561      Inf 0 66.65570  0.7355958   0   0     3 313.164012
#> 4 24.05088      Inf 0 46.16319 -0.1686869   0   1     3  73.591303
#> 5 23.86395      Inf 0 42.63115 -0.3245471   1   0     3  75.627023
#> 6  0.00000 3.098614 0 63.38731  0.5913699   0   0     2   1.515395
```

``` r

sim.fit <- PI3M.fit(data=sim.dat) # fit model to simulated data
sim.fit$summary # view model fit summary
#>        param theta.hat std.dev   lower   upper
#> h          h   -4.9690  0.1620 -5.2865 -4.6514
#> g0 intercept   -1.5853  0.0820 -1.7460 -1.4245
#> w0 intercept   -1.2907  0.1027 -1.4920 -1.0894
#> p0 intercept   -3.1019  0.1624 -3.4202 -2.7836
```

``` r

sim.predict <- PI3M.predict(data=sim.dat[1,], time.points = seq(0, 15, 0.5), fit=sim.fit)

library(survival) # compare model fit to non-parametric Kaplan-Meier curve 
sim.km.fit <- survfit(Surv(sim.dat$left, sim.dat$right, type='interval2')~1)

# plot PI3M predictions and KM-curve to compare 
plot(sim.predict[[1]]$Time, sim.predict[[1]]$CR, type='l', xlab='time', ylab='CR')
lines(sim.km.fit$time, 1-sim.km.fit$surv, col='blue')
```

<img src="man/figures/README-example-1.png" width="50%" style="display: block; margin: auto;" />

To add covariates to the model we specify them separately for the
progression, clearance, and prevalence parameters. For example if we
wanted to add HPV16 as a covariate for progression and abnormal cytology
as a covariate for prevalence then we would do the following:

``` r

sim.thetas.cov <- c(-5, -1.6, 1, -1.2, -3, 4)
sim.dat2 <- PI3M.simulator(1000, prog_model = "prog ~ hpv", prev_model = "prev ~ cyt", sim.thetas.cov, show_prob = 0.9, interval=3, include.h=T)
head(sim.dat2) # view simulated data
#>       left    right z      age    age.std hpv cyt cause      actual
#> 1 23.90248      Inf 0 67.41109  0.7414240   0   0     3  60.5638419
#> 2  0.00000 2.682506 0 50.22573  0.0100072   0   1     2   0.4176361
#> 3 23.94910      Inf 0 64.15406  0.6028036   0   0     3 103.1641465
#> 4 23.94229      Inf 0 44.24946 -0.2443458   0   0     3 167.6269674
#> 5 23.81414      Inf 0 63.70295  0.5836040   0   0     3 211.6056296
#> 6 21.03203      Inf 0 38.27810 -0.4984900   0   0     3 118.3657120
```

``` r

sim.fit2 <- PI3M.fit(sim.dat2, prog_model = "prog ~ hpv", prev_model = "prev ~ cyt") # fit model to simulated data
sim.fit2$summary # view model fit summary
#>        param theta.hat std.dev   lower   upper
#> h          h   -5.0118  0.1806 -5.3658 -4.6578
#> g0 intercept   -1.6186  0.1134 -1.8409 -1.3963
#> g1       hpv    1.1503  0.1505  0.8553  1.4453
#> w0 intercept   -1.1163  0.1151 -1.3419 -0.8907
#> p0 intercept   -3.1620  0.2141 -3.5816 -2.7423
#> p1       cyt    4.1634  0.2451  3.6830  4.6438
```

``` r

sim.predict2 <- PI3M.predict(data=data.frame(hpv = c(1, 1, 0, 0), cyt=c(1, 0, 1, 0)), 
                              time.points = seq(0, 15, 0.5), fit=sim.fit2)

# compare model fit to non-parametric Kaplan-Meier curve 
sim.km.fit1 <- survfit(Surv(left, right, type='interval2')~1, data = sim.dat2[sim.dat2$hpv==1 & sim.dat2$cyt==1,])
sim.km.fit2 <- survfit(Surv(left, right, type='interval2')~1, data = sim.dat2[sim.dat2$hpv==1 & sim.dat2$cyt==0,])
sim.km.fit3 <- survfit(Surv(left, right, type='interval2')~1, data = sim.dat2[sim.dat2$hpv==0 & sim.dat2$cyt==1,])
sim.km.fit4 <- survfit(Surv(left, right, type='interval2')~1, data = sim.dat2[sim.dat2$hpv==0 & sim.dat2$cyt==0,])
```

``` r
# plot PI3M predictions and KM-curve to compare 
plot(sim.predict2[[1]]$Time, sim.predict2[[1]]$CR, type='l', xlab='time', ylab='CR', ylim=c(0,1), col='darkblue')
lines(sim.km.fit1$time, 1-sim.km.fit1$surv, col='lightblue', lty=2)

lines(sim.predict2[[2]]$Time, sim.predict2[[2]]$CR, col = 'darkgreen')
lines(sim.km.fit2$time, 1-sim.km.fit2$surv, col='darkolivegreen1', lty=2)

lines(sim.predict2[[3]]$Time, sim.predict2[[3]]$CR,col='deeppink2')
lines(sim.km.fit3$time, 1-sim.km.fit3$surv, col='pink1', lty=2)

lines(sim.predict2[[4]]$Time, sim.predict2[[4]]$CR, col='darkorange3')
lines(sim.km.fit4$time, 1-sim.km.fit4$surv, col='orange1', lty=2)

legend("bottomright",
         legend=c("HPV16-pos & abnormal cyt", "HPV16-pos & normal cyt", "HPV16-neg & abnormal cyt", "HPV16-neg & normal cyt"),
         col=c("darkblue", "darkgreen", "deeppink2", "darkorange3"), lty=1, lwd=2, bty = "n", cex=0.75)
```

<img src="man/figures/README-example3-1.png" width="50%" />

## Authors

- **Kelsi R. Kroon** <k.kroon@amsterdamumc.nl>
