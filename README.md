# causalDML
This package implements the Double Machine Learning based methods reviewed in Knaus (2020).
It is also basis of the paper supplementing [notebook](https://mcknaus.github.io/assets/code/Notebook_DML_ALMP_MCK2020.html).

The current version can be installed via devtools:

```R
library(devtools)
install_github(repo="MCKnaus/causalDML")
```

The following code creates synthetic data to showcase a comprehensive analysis based on the package:

```R
library(causalDML)
library(policytree)
library(lmtest)
library(sandwich)

set.seed(1234)
## Generate random data
n = 2000
p = 10
X = matrix(rnorm(n * p), n, p)
W = sample(c("A", "B", "C"), n, replace = TRUE)
Y = X[, 1] + X[, 2] * (W == "B") + X[, 3] * (W == "C") + rnorm(n)

## Create components of ensemble
# General methods
mean = create_method("mean")
forest =  create_method("forest_grf",args=list(tune.parameters = "all"))

# Pscore specific components
ridge_bin = create_method("ridge",args=list(family = "binomial"))
lasso_bin = create_method("lasso",args=list(family = "binomial"))

# Outcome specific components
ols = create_method("ols")
ridge = create_method("ridge")
lasso = create_method("lasso")

## Run the main function that outputs nuisance parameters, APO and ATE
# Can take some minutes, set quiet=F to see progress or remove forest to speed up
DML = causalDML(Y,W,X,ml_w=list(mean,forest,ridge_bin,lasso_bin),
                      ml_y=list(mean,forest,ols,ridge,lasso),quiet=T)

## Show and plot estimated APOs
summary(DML$APO)
plot(DML$APO)

## Show estimated ATEs
summary(DML$ATE)

## Estimate all possible ATETs
for (i in 1:3) {
  APO_atet = APO_dml_atet(Y,DML$APO$m_mat,DML$APO$w_mat,DML$APO$e_mat,DML$APO$cf_mat,treated=i)
  ATET = ATE_dml(APO_atet)
  print(summary(ATET))
}

## Best linear prediction of CATEs along variable X1
for (i in 1:3) {
  temp_ols = lm(DML$ATE$delta[,i] ~ X[, 1])
  print(coeftest(temp_ols, vcov = vcovHC(temp_ols)))
}

## Kernel regression CATEs along variable X1
# Might also take some minutes
for (i in 1:3) {
  temp_kr = kr_cate(DML$ATE$delta[,i],X[, 1])
  plot(temp_kr)
}

## (N)DR-learner for all observations in the sample
# Can take some minutes, set quiet=F to see progress or remove forest to speed up
# Be sure to specify only linear smoothers for tau
ndr = ndr_learner(Y,W,X,ml_w=list(mean,forest,ridge_bin,lasso_bin),
                  ml_y = list(mean,forest,ols,ridge,lasso),
                  ml_tau = list(mean,forest,ols,ridge),quiet=T)

# Plot DR-learner distribution of B-A
hist(ndr$cates[1,,1])

# Plot NDR-learner distribution of B-A
hist(ndr$cates[1,,2])

## (N)DR-learner for out-of-sample prediction in test set
# Can take some minutes, set quiet=F to see progress or remove forest to speed up
# Be sure to specify only linear smoothers for tau
X_test = matrix(rnorm(n/2 * p), n/2, p)
ndr.oos = ndr_oos(Y,W,X,xnew=X_test,ml_w=list(mean,forest,ridge_bin,lasso_bin),
           ml_y = list(mean,forest,ols,ridge,lasso),
           ml_tau = list(mean,forest,ols,ridge),quiet=T)

# Plot DR-learner distribution of B-A
hist(ndr.oos$cates[[1]][,1])

# Plot NDR-learner distribution of B-A
hist(ndr.oos$cates[[1]][,2])

## Policy tree
tree1 = policy_tree(X,DML$APO$gamma,depth=1)
plot(tree1)
tree2 = policy_tree(X,DML$APO$gamma,depth=2)
plot(tree2)
```

## References

Knaus, M. C. (2020). Double machine learning based program evaluation under unconfoundedness, [arXiv](https://arxiv.org/abs/2003.03191)
