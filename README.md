`{nadir}`
================

*nadir* (noun): nā-dir

> the lowest point.

Fitting with the *minimum loss based estimation*[^1][^2] literature,
`{nadir}` is an implementation of the Super Learner algorithm with
improved support for flexible formula based syntax and that is fond of
functional programming solutions such as closures and currying.

------------------------------------------------------------------------

`{nadir}` implements the Super Learner[^3] algorithm. To quote *the
Guide to SuperLearner*[^4]:

> SuperLearner is an algorithm that uses cross-validation to estimate
> the performance of multiple machine learning models, or the same model
> with different settings. It then creates an optimal weighted average
> of those models, aka an “ensemble”, using the test data performance.
> This approach has been proven to be asymptotically as accurate as the
> best possible prediction algorithm that is tested.

## Why `{nadir}` and why reimplement Super Learner again?

In previous implementations
([`{SuperLearner}`](https://github.com/ecpolley/SuperLearner),
[`{sl3}`](https://github.com/tlverse/sl3/),
[`{mlr3superlearner}`](https://cran.r-project.org/web/packages/mlr3superlearner/mlr3superlearner.pdf)),
support for *flexible formula-based syntax* has been limited, instead
opting for specifying learners as models on an $X$ matrix and $Y$
outcome vector. Many popular R packages such as `lme4` and `mgcv` (for
random effects and generalized additive models) use formulas extensively
to specify models using syntax like `(age | strata)` to specify random
effects on age by strata, or `s(age, income)` to specify a smoothing
term on `age` and `income` simultaneously.

At present, it is difficult to use these kinds of features in
`{SuperLearner}`, `{sl3}` and `{ml3superlearner}`.

For example, it is easy to imagine the Super Learner algorithm being
appealing to modelers fond of random effects based models because they
may want to hedge on the exact nature of the random effects models, not
sure if random intercepts are enough or if random slopes should be
included, etc., and similar other modeling decisions in other
frameworks.

Therefore, the `{nadir}` package takes as its charges to:

- Implement a syntax in which it is easy to specify *different formulas*
  for each of many candidate learners.
- To make it easy to pass new learners to the Super Learner algorithm.

# Installation Instructions

At present, `{nadir}` is only available on GitHub.

``` r
devtools::install_github("ctesta01/nadir")
```

# Demonstration

``` r
library(nadir)

learners <- list(
     glm = lnr_glm,
     rf = lnr_rf,
     glmnet = lnr_glmnet,
     lmer = lnr_lmer
  )

# mtcars example ---
regression_formulas <- c(
  rep(c(mpg ~ cyl + hp), 3), # first three models use same formula
  mpg ~ (1 | cyl) + hp # lme4 uses different language features
  )

# fit a super_learner
sl_model <- super_learner(
  data = mtcars,
  regression_formula = regression_formulas,
  learners = learners)

# produce super_learner predictions
sl_model_predictions <- sl_model(mtcars)
# compare against the predictions from the individual learners
fit_individual_learners <- lapply(1:length(learners), function(i) { learners[[i]](data = mtcars, regression_formula = regression_formulas[[i]]) } )
individual_learners_mse <- lapply(fit_individual_learners, function(fit_learner) { mse(fit_learner(mtcars) - mtcars$mpg) })
names(individual_learners_mse) <- names(learners)

print(paste0("super-learner mse: ", mse(sl_model_predictions - mtcars$mpg)))
```

    ## [1] "super-learner mse: 4.88905625458048"

``` r
individual_learners_mse
```

    ## $glm
    ## [1] 9.124205
    ## 
    ## $rf
    ## [1] 4.848698
    ## 
    ## $glmnet
    ## [1] 9.167678
    ## 
    ## $lmer
    ## [1] 8.744686

``` r
# iris example ---
sl_model <- super_learner(
  data = iris,
  regression_formula = Sepal.Length ~ Sepal.Width + Petal.Length + Petal.Width,
  learners = learners[1:3])

# produce super_learner predictions and compare against the individual learners
sl_model_predictions <- sl_model(iris)
fit_individual_learners <- lapply(learners[1:3], function(learner) { learner(data = iris, regression_formula = Sepal.Length ~ Sepal.Width + Petal.Length + Petal.Width) } )
individual_learners_mse <- lapply(fit_individual_learners, function(fit_learner) { mse(fit_learner(iris) - iris$Sepal.Length) })

print(paste0("super-learner mse: ", mse(sl_model_predictions - iris$Sepal.Length)))
```

    ## [1] "super-learner mse: 0.0806565534004013"

``` r
individual_learners_mse
```

    ## $glm
    ## [1] 0.0963027
    ## 
    ## $rf
    ## [1] 0.04187863
    ## 
    ## $glmnet
    ## [1] 0.2035002

## Coming Down the Pipe

- Reworking some of the internals to use
  - `{future}` and `{future.apply}`
  - `{origami}`
- Investigating if using `formula`s are not performant enough given that
  they store an associated environment inside them.
- Adding support for named `extra_learner_args` and
  `regression_formulas` so that, say, formulas are matched to the
  appropriate learner based off names if names are provided. E.g., it
  would be nice to have the following syntax rather than relying on the
  user to get the indexing right every time.

``` r
  regression_formulas = list(
    .default = Y ~ .,
    gam = Y ~ s(smoothing_term) + ...,
    lme4 = Y ~ (random|effect) + ...
    )
```

[^1]: van der Laan, Mark J. and Dudoit, Sandrine, “Unified
    Cross-Validation Methodology For Selection Among Estimators and a
    General Cross-Validated Adaptive Epsilon-Net Estimator: Finite
    Sample Oracle Inequalities and Examples” (November 2003). U.C.
    Berkeley Division of Biostatistics Working Paper Series. Working
    Paper 130. <https://biostats.bepress.com/ucbbiostat/paper130>

[^2]: Zheng, W., & van der Laan, M. J. (2011). Cross-Validated Targeted
    Minimum-Loss-Based Estimation. In Springer Series in Statistics
    (pp. 459–474). Springer New York.
    <https://doi.org/10.1007/978-1-4419-9782-1_27>

[^3]: van der Laan, M. J., Polley, E. C., & Hubbard, A. E. (2007). Super
    Learner. In Statistical Applications in Genetics and Molecular
    Biology (Vol. 6, Issue 1). Walter de Gruyter GmbH.
    <https://doi.org/10.2202/1544-6115.1309>
    <https://pubmed.ncbi.nlm.nih.gov/17910531/>

[^4]: Guide to `{SuperLearner}`:
    <https://cran.r-project.org/web/packages/SuperLearner/vignettes/Guide-to-SuperLearner.html>
