Penalized Regression
================

In this section I will use different types of penalized regression to see if it can improve over the original regression models. The fundamental idea is to avoid large coefficients by penalizing them. The purpose is to decrease the variance, which is high if small differences in the input generate great changes in the output (which happens when some coefficients are large). Conceptually having large coefficients means that the model picks up the noise in the data, which of course leads to overfitting. The best performing logistic regression models I created in the previous section had low coefficients so I don't expect particular improvements but let's try!

Note: to understand better how penalized logistic regression works, I found here a very good explanation <https://eight2late.wordpress.com/2017/07/11/a-gentle-introduction-to-logistic-regression-and-lasso-regularisation-using-r/>

#### Model 1 - Penalized Regression

The first model is developed with the caret package and the algorithm "plr" which uses the forward stepwise selection procedure to determine the variables to include in the model. This method was found useful by Google researchers for the detection of gene interactions in common diseases (<https://www.ncbi.nlm.nih.gov/pubmed/17429103>). The goal is to minimize the score:

*D* + *c**p* \* *d**f*
 where D deviance df degrees of freedom cp complexity parameter.

This is obtained using a quadratic penalization on the coefficients. The minimizing criterion is:

−*l**o**g* − *l**i**k**e**l**i**h**o**o**d* + *λ* ∗ ∥*β*∥<sup>2</sup>
 The last term is what prevents the coefficients from growing too much.

The tuning parameters are:

-   cp: if cp = "aic" or cp = "bic", these are converted to cp = 2 or cp = log(sample size), respectively.
-   lambda (L2 Penalty)

After performing a search through different values of lambda and trying both values for cp, the best performance is with lambda = 1 and cp = bic, which gives logloss 0.4234673. The Official Score is 0.3533, the best so far. Interestingly if I plot the importance of the variables for this model I get fairly similar results to the random forest model in the previous section, which confirms my first impression. Now 2 different models and exploratory plots are in general agreement while the simple logistic regression isn't.

``` r
library(caret)
set.seed(111)
tunegrid <- expand.grid(cp = "bic", lambda = seq(0.0001, 1, length = 10))
modelp1 <- train(heart_disease_present~., data = training[,2:15], method = "plr", 
            tuneGrid=tunegrid, trControl = trainControl(method = "cv", number=5, 
            classProbs=T, summaryFunction=mnLogLoss), metric="logLoss")
```

    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2 
    ## 
    ## Convergence warning in plr: 2

``` r
print(modelp1)
```

    ## Penalized Logistic Regression 
    ## 
    ## 180 samples
    ##  13 predictor
    ##   2 classes: 'N', 'Y' 
    ## 
    ## No pre-processing
    ## Resampling: Cross-Validated (5 fold) 
    ## Summary of sample sizes: 144, 144, 144, 144, 144 
    ## Resampling results across tuning parameters:
    ## 
    ##   lambda  logLoss  
    ##   0.0001  0.4694894
    ##   0.1112  0.4538103
    ##   0.2223  0.4448521
    ##   0.3334  0.4386717
    ##   0.4445  0.4341528
    ##   0.5556  0.4307963
    ##   0.6667  0.4281902
    ##   0.7778  0.4261782
    ##   0.8889  0.4246383
    ##   1.0000  0.4234673
    ## 
    ## Tuning parameter 'cp' was held constant at a value of bic
    ## logLoss was used to select the optimal model using  the smallest value.
    ## The final values used for the model were lambda = 1 and cp = bic.

``` r
plot(varImp(modelp1))
```

![](Penalized_files/figure-markdown_github/model1-1.png)

#### Model 2 - Lasso Regression

The difference with the previous model is to use the norm of the coefficients vector in the penalty term:

−*l**o**g* − *l**i**k**e**l**i**h**o**o**d* + *λ* ∗ ∥*β*∥
 Lasso regression is peculiar since it tends to reduce the coefficients of the least important features to exactly 0. It works as an actual features selector. I implement this with the glmnet algorithm by choosing alpha = 1. The data also needs to be normalized with the PreProcess option. The reason for it is that the size of the coefficients affects the choice of the features, due to the penalty term, so the scale of each feature can affect the result, if normalization is not applied. The best model has lambda = 0.0001 and logloss = 0.4631125, worse than the previous model. The higher importance of sex is the main difference with the previous model, the other variables have similar importance.

``` r
set.seed(222)
tunegrid2 <- expand.grid(.alpha=1, .lambda = seq(0.0001, 1, length = 10))
modelLas <- train(heart_disease_present~., data = training[,2:15], method = "glmnet", 
                  tuneGrid=tunegrid2, preProcess = c("center", "scale"), 
                  trControl = trainControl(method = "cv", number=5,
                  classProbs=T, summaryFunction=mnLogLoss), metric="logLoss")
print(modelLas)
```

    ## glmnet 
    ## 
    ## 180 samples
    ##  13 predictor
    ##   2 classes: 'N', 'Y' 
    ## 
    ## Pre-processing: centered (14), scaled (14) 
    ## Resampling: Cross-Validated (5 fold) 
    ## Summary of sample sizes: 144, 144, 144, 144, 144 
    ## Resampling results across tuning parameters:
    ## 
    ##   lambda  logLoss  
    ##   0.0001  0.4631125
    ##   0.1112  0.5221832
    ##   0.2223  0.6540350
    ##   0.3334  0.6869616
    ##   0.4445  0.6869616
    ##   0.5556  0.6869616
    ##   0.6667  0.6869616
    ##   0.7778  0.6869616
    ##   0.8889  0.6869616
    ##   1.0000  0.6869616
    ## 
    ## Tuning parameter 'alpha' was held constant at a value of 1
    ## logLoss was used to select the optimal model using  the smallest value.
    ## The final values used for the model were alpha = 1 and lambda = 1e-04.

``` r
plot(varImp(modelLas))
```

![](Penalized_files/figure-markdown_github/model2-1.png)

#### Model 3 - Ridge Regression

This model differs with the previous one in the form of the penalty term, which here uses the square of the coefficients instead of the absolute value. This makes the coefficients shrink but not go to 0. This method is implemented by choosing alpha = 0. Normalization and Cross-Validation are still applied. The best model has lambda = 0.06673333 and logloss = 0.4117543. The Official Score is 0.3605, very close to Model 1.

``` r
set.seed(222)
tunegrid3 <- expand.grid(.alpha=0, .lambda = seq(0.0001, 0.2, length = 10))
modelRid <- train(heart_disease_present~., data = training[,2:15], method = "glmnet", 
                  tuneGrid=tunegrid3, preProcess = c("center", "scale"), 
                  trControl = trainControl(method = "cv", number=5, 
                  classProbs=T, summaryFunction=mnLogLoss), metric="logLoss")
print(modelRid)
```

    ## glmnet 
    ## 
    ## 180 samples
    ##  13 predictor
    ##   2 classes: 'N', 'Y' 
    ## 
    ## Pre-processing: centered (14), scaled (14) 
    ## Resampling: Cross-Validated (5 fold) 
    ## Summary of sample sizes: 144, 144, 144, 144, 144 
    ## Resampling results across tuning parameters:
    ## 
    ##   lambda      logLoss  
    ##   0.00010000  0.4180043
    ##   0.02231111  0.4180043
    ##   0.04452222  0.4132244
    ##   0.06673333  0.4117543
    ##   0.08894444  0.4127435
    ##   0.11115556  0.4148670
    ##   0.13336667  0.4175754
    ##   0.15557778  0.4205996
    ##   0.17778889  0.4237761
    ##   0.20000000  0.4270435
    ## 
    ## Tuning parameter 'alpha' was held constant at a value of 0
    ## logLoss was used to select the optimal model using  the smallest value.
    ## The final values used for the model were alpha = 0 and lambda = 0.06673333.

#### Model 4 - Elastic Net

Elastic Net is a combination of Ridge and Lasso regression. Lasso fails to do grouped selection since it tends to select one variable from a group and ignore the others. The quadratic part of the penalty removes the limitation on the number of selected variables, encourages grouping effect and stabilizes the regularization path. The penalty term is:

*λ*\[(1 − *α*)/2 \* ∥*β*∥<sup>2</sup> + *α* ∗ ∥*β*∥\]

The tuning parameters are:

-   alpha: mixing parameter (between 0 and 1, alpha = 0 for ridge regression, alpha = 1 for lasso)
-   lambda: regularization parameter

Below a plot that explains the conceptual difference between the 3 types of regularized regressions.

<img src="ElasticNet.png" width="450">

This method is implemented with glmnet algorithm and the best result is with choosing alpha = 0 (Ridge). Normalization and Cross-Validation are still applied. The best model has lambda = 0.0001 and logloss = 0.4180043. The Official Score is 0.40686, worse than previous models.

``` r
set.seed(222)
tunegrid4 <- expand.grid(.alpha=seq(0, 1, length = 5), 
                         .lambda = seq(0.0001, 1, length = 5))
modelEl <- train(heart_disease_present~., data = training[,2:15], method = "glmnet", 
                  tuneGrid=tunegrid4, preProcess = c("center", "scale"), 
                 trControl = trainControl(method = "cv", number=5, 
                        classProbs=T, summaryFunction=mnLogLoss), metric="logLoss")
print(modelEl)
```

    ## glmnet 
    ## 
    ## 180 samples
    ##  13 predictor
    ##   2 classes: 'N', 'Y' 
    ## 
    ## Pre-processing: centered (14), scaled (14) 
    ## Resampling: Cross-Validated (5 fold) 
    ## Summary of sample sizes: 144, 144, 144, 144, 144 
    ## Resampling results across tuning parameters:
    ## 
    ##   alpha  lambda    logLoss  
    ##   0.00   0.000100  0.4180043
    ##   0.00   0.250075  0.4344316
    ##   0.00   0.500050  0.4675605
    ##   0.00   0.750025  0.4935283
    ##   0.00   1.000000  0.5140373
    ##   0.25   0.000100  0.4628779
    ##   0.25   0.250075  0.4835265
    ##   0.25   0.500050  0.5784006
    ##   0.25   0.750025  0.6489669
    ##   0.25   1.000000  0.6823502
    ##   0.50   0.000100  0.4629099
    ##   0.50   0.250075  0.5557323
    ##   0.50   0.500050  0.6795344
    ##   0.50   0.750025  0.6869616
    ##   0.50   1.000000  0.6869616
    ##   0.75   0.000100  0.4629696
    ##   0.75   0.250075  0.6246312
    ##   0.75   0.500050  0.6869616
    ##   0.75   0.750025  0.6869616
    ##   0.75   1.000000  0.6869616
    ##   1.00   0.000100  0.4631125
    ##   1.00   0.250075  0.6764362
    ##   1.00   0.500050  0.6869616
    ##   1.00   0.750025  0.6869616
    ##   1.00   1.000000  0.6869616
    ## 
    ## logLoss was used to select the optimal model using  the smallest value.
    ## The final values used for the model were alpha = 0 and lambda = 1e-04.

#### Model 5 - Regularized Logistic

The caret package also has a method specific for regularized logistic regression. The tuning parameters are:

-   cost
-   loss function
-   tolerance epsilon

The best model has logloss = 0.4252468. The Official Score is 0.3684, close to the best one.

``` r
set.seed(111)
tunegrid5 <- expand.grid(.cost = 1, .loss = c("L1", "L2_dual", "L2_primal"),
                         .epsilon = seq(0.01, 0.1, length.out = 5))
modelReg <- train(heart_disease_present~., data = training[,2:15], method = "regLogistic",                   tuneGrid=tunegrid5, preProcess = c("center", "scale"),
                  trControl = trainControl(method = "cv", number=5, 
                        classProbs=T, summaryFunction=mnLogLoss), metric="logLoss")
print(modelReg)
```

    ## Regularized Logistic Regression 
    ## 
    ## 180 samples
    ##  13 predictor
    ##   2 classes: 'N', 'Y' 
    ## 
    ## Pre-processing: centered (14), scaled (14) 
    ## Resampling: Cross-Validated (5 fold) 
    ## Summary of sample sizes: 144, 144, 144, 144, 144 
    ## Resampling results across tuning parameters:
    ## 
    ##   loss       epsilon  logLoss  
    ##   L1         0.0100   0.4381387
    ##   L1         0.0325   0.4370383
    ##   L1         0.0550   0.4321934
    ##   L1         0.0775   0.4252468
    ##   L1         0.1000   0.4286560
    ##   L2_dual    0.0100   0.4427855
    ##   L2_dual    0.0325   0.4427123
    ##   L2_dual    0.0550   0.4427197
    ##   L2_dual    0.0775   0.4428726
    ##   L2_dual    0.1000   0.4427705
    ##   L2_primal  0.0100   0.4425755
    ##   L2_primal  0.0325   0.4402696
    ##   L2_primal  0.0550   0.4361898
    ##   L2_primal  0.0775   0.4361898
    ##   L2_primal  0.1000   0.4361898
    ## 
    ## Tuning parameter 'cost' was held constant at a value of 1
    ## logLoss was used to select the optimal model using  the smallest value.
    ## The final values used for the model were cost = 1, loss = L1 and epsilon
    ##  = 0.0775.

``` r
# Create Output for Evaluation - Official Score: 0.3576
```
