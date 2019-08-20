Random Forest
================

In this section I'll build models based on Random Forest. Since the provided training dataset is small, I'll use cross-validation to estimate the logloss and build the model on the entire set instead of splitting it as I did for Logistic Regression models. I'll start relying heavily on the caret package, which I will use for most of models in the following sections too.

Due to how Random Forest works, some additional features need to be renamed, binary variables can't be expressed as 0 and 1.

``` r
training$heart_disease_present <- revalue(training$heart_disease_present, 
                                         c("0" = "N", "1" = "Y"))
training$fasting_blood_sugar_gt_120_mg_per_dl <- revalue(training$fasting_blood_sugar_gt_120_mg_per_dl, c("0" = "N", "1" = "Y"))
training$exercise_indiced_angina <- revalue(training$exercise_indiced_angina, 
                                            c("0" = "F", "1" = "T"))
```

#### Model 1 - Initial Cross Validation Model

The first model is simple, it uses 5-fold Cross Validation with method "rf" which is Random Forest. The tuning parameter is mtry, which represents the number of predictors that are randomly selected to build each individual tree. The best model has logloss 0.4340239 with mtry = 2.

``` r
library(caret)
set.seed(333)
modelrf1 <- train(heart_disease_present~., data = training[,2:15], method = "rf", 
                  trControl = trainControl(method = "cv", number=5, classProbs=T, 
                                         summaryFunction=mnLogLoss), metric="logLoss")
print(modelrf1)
```

    ## Random Forest 
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
    ##   mtry  logLoss  
    ##    2    0.4340239
    ##    8    0.4513206
    ##   14    0.4826158
    ## 
    ## logLoss was used to select the optimal model using  the smallest value.
    ## The final value used for the model was mtry = 2.

#### Model 2 - Grid Search

The first obvious improvement is to try more models by expanding the choices for mtry. This is achieved by using grid search. By using all values from 1 to 14 the best model has logloss 0.4282123 with mtry = 1.

``` r
tunegrid <- expand.grid(.mtry=c(1:14))
set.seed(333)
modelrf2 <- train(heart_disease_present~., data = training[,2:15], method = "rf", 
            tuneGrid=tunegrid, trControl = trainControl(method = "cv", number=5, 
            classProbs=T, search = "grid", summaryFunction=mnLogLoss), metric="logLoss")
print(modelrf2)
```

    ## Random Forest 
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
    ##   mtry  logLoss  
    ##    1    0.4282123
    ##    2    0.4315341
    ##    3    0.4383019
    ##    4    0.4422467
    ##    5    0.4495833
    ##    6    0.4554429
    ##    7    0.4634540
    ##    8    0.4651895
    ##    9    0.4661079
    ##   10    0.4710141
    ##   11    0.4782962
    ##   12    0.4764271
    ##   13    0.4784960
    ##   14    0.4859837
    ## 
    ## logLoss was used to select the optimal model using  the smallest value.
    ## The final value used for the model was mtry = 1.

#### Model 3 - Change ntree

Another parameter of random forest is the number of trees ntree. The default value is 500 so I tried different values between 100 and 400 finding the best as 250. This model had logloss 0.4259374 with mtry = 2. I used this model to create the final prediction for the competition and got an official score of 0.3756, even better than the estimated error on the training data, which show how random forest and cross-validation reduce over-fitting. I also looked at the importance of the features for this model. There are significant differences with the logistic regression model (I'll use model 6, the best one):

-   chest pain and major vessels much more important in RF
-   ST depression, heart rate, age and thallium test very important in RF, not even in the LR model
-   EKG results, sex important for LR, not for RF

By looking at the exploratory plots of features vs outcome, the RF model seems to be right for these features: thallium test, chest pain, major vessels EKG results seems influential but not as other features. Sex looked very important in the plots but not if we consider the physiology. Overall it seems that random forest does a better job at extrapolating the importance that each feature actually has.

``` r
set.seed(333)
modelrf3 <- train(heart_disease_present~., data = training[,2:15], method = "rf", 
            tuneGrid=tunegrid, trControl = trainControl(method = "cv", number=5, 
            classProbs=T, search = "grid", summaryFunction=mnLogLoss), metric="logLoss", 
            ntree = 250)
print(modelrf3)
```

    ## Random Forest 
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
    ##   mtry  logLoss  
    ##    1    0.4292279
    ##    2    0.4259374
    ##    3    0.4358934
    ##    4    0.4440550
    ##    5    0.4519908
    ##    6    0.4548599
    ##    7    0.4632057
    ##    8    0.4654709
    ##    9    0.4701025
    ##   10    0.4742590
    ##   11    0.4742797
    ##   12    0.4802043
    ##   13    0.4772014
    ##   14    0.4860149
    ## 
    ## logLoss was used to select the optimal model using  the smallest value.
    ## The final value used for the model was mtry = 2.

``` r
varImp(modelrf3)
```

    ## rf variable importance
    ## 
    ##                                       Overall
    ## chest_pain_type                        100.00
    ## oldpeak_eq_st_depression                97.43
    ## max_heart_rate_achieved                 85.71
    ## num_major_vessels                       83.61
    ## age                                     78.00
    ## thalreversible_defect                   74.75
    ## thalnormal                              73.17
    ## serum_cholesterol_mg_per_dl             69.02
    ## resting_blood_pressure                  60.54
    ## exercise_induced_anginaT                48.94
    ## slope_of_peak_exercise_st_segment       37.25
    ## sexmale                                 28.89
    ## resting_ekg_results                     14.93
    ## fasting_blood_sugar_gt_120_mg_per_dlY    0.00

Model 4 - Oblique Random Forest
-------------------------------

Caret has an algorithm called Oblique Random Forest. The only information I found was from academic papers, which are not the most enjoyable and easy reads. The fundamental concept of Oblique Random Forest is that each split is not determined with thresholds (i.e. x &gt; k) on individual features in every split ("orthogonal" trees) but with multiple features separated by oriented hyperplanes i.e.
*a*<sub>1</sub> \* *x*<sub>1</sub> + ... + *a*<sub>*n*</sub> \* *x*<sub>*n*</sub> ≥ *k*

Conceptually it makes sense how they can outperform regular Random Forest so I'll apply the algorithm to the next model. The model has logloss 0.4314805 with Official Score = 0.3988, slightly worse than the previous one.

``` r
set.seed(555)
modelrf4 <- train(heart_disease_present~., data = training[,2:15], method = "ORFlog", 
                  trControl = trainControl(method = "cv", number=5, classProbs=T, 
                                           summaryFunction=mnLogLoss), metric="logLoss")
```

    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.
    ## obliqueRF: using logistic regression as node model.
    ## obliqueRF: no test set defined. Will use training data.

``` r
print(modelrf4)
```

    ## Oblique Random Forest 
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
    ##   mtry  logLoss  
    ##    2    0.4337641
    ##    8    0.4341088
    ##   14    0.4314805
    ## 
    ## logLoss was used to select the optimal model using  the smallest value.
    ## The final value used for the model was mtry = 14.
