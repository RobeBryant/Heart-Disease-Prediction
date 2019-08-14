Logistic Regression
================

In this section I'll start building the predictive model, starting from the most simple one, which is based on Logistic Regression. Note: I will build models from a dataset of labeled records that I will divide in training set (to build the model) and testing set (to calculate the logloss). For the models that look more promising, I will use the provided set of unlabeled records to predict the outcome, which will be used to get the "Official Score" used for the competition. The owner of the competition has the actual labels used for the calculation.

The first step is, of course, to split the dataset in training and testing sets, with the training set representing 70% of the total available data.

``` r
# Split dataset
library(caret)
set.seed(666)
indextrain <- createDataPartition(y = training$heart_disease_present, p = 0.7, list = F)
train_set <- training[indextrain,]
test_set <- training[-indextrain,]
```

#### Model 1 - Basic Logistic

The first model, which can be considered as a baseline, is a simple regression model using all features. The log loss estimated on the testing set is 0.5913517, a very poor result. Let's analyze the result to see how it can be improved.

``` r
library(MLmetrics)
# Logistic regression model with all features
# eliminate first column (patient ID)
model1 <- glm(heart_disease_present~.,data = train_set[,2:15],family = binomial)
# Predict outcome and calculate logloss on testing data
pred_outcome <- predict(model1, newdata = test_set, type = "response")
LogLoss(y_pred = pred_outcome, y_true = as.numeric(test_set$heart_disease_present))
```

    ## [1] 0.5913517

One of the fundamental assumptions of logistic regression is that there is a linear relationship between input features and logit of the outcome. The logit is defined as the logarithm of the odds for the probability predicted by the model p:

log(p/(1 - p))

Since most features are categorical, this analysis can be performed only for age, heart rate and cholesterol, which are numeric. The plot shows the value of each of these 3 features in relation to the logit value for each record in the training set. The relationships are clearly not linear which suggests that a transformation of the variables might be necessary.

``` r
# Calculate predicted probability and plot logit vs numeric features
pred_prob <- predict(model1, type = "response")
diag_data <- train_set[,c(9,12,13)] %>% mutate(logit = log(pred_prob/(1-pred_prob))) %>%
  gather(key = c("serum_cholesterol_mg_per_dl", "age", "max_heart_rate_achieved"), 
         value = "predictor.value", -logit)
ggplot(diag_data, aes(logit, predictor.value)) + geom_point(size = 0.5, alpha = 0.5) +
  geom_smooth(method = "loess") + theme_bw() + 
  facet_wrap(~c("serum_cholesterol_mg_per_dl", "age", "max_heart_rate_achieved"), scales = "free_y")
```

![](Logistic_files/figure-markdown_github/diagnostic1-1.png)

By analyzing Cook's distance, observation n.5 is the most influential point. This record has the 3rd highest blood pressure and the 2nd highest ST depression value and it's an outlier for both. It's also the only one to not have the disease among the 13 patients with the highest ST depression. Before using a different formula, I simply eliminate this record and re-run the model.

``` r
# Plot Cook's distance
plot(model1, which = 4)
```

![](Logistic_files/figure-markdown_github/diagnostic2-1.png)

``` r
# Show Observation 5
inf_pt <- which(rownames(train_set)=="5")
train_set[inf_pt,]
```

    ##   patient_id slope_of_peak_exercise_st_segment              thal
    ## 5     oyt4ek                                 3 reversible_defect
    ##   resting_blood_pressure chest_pain_type num_major_vessels
    ## 5                    178               1                 0
    ##   fasting_blood_sugar_gt_120_mg_per_dl resting_ekg_results
    ## 5                                    0                   2
    ##   serum_cholesterol_mg_per_dl oldpeak_eq_st_depression  sex age
    ## 5                         270                      4.2 male  59
    ##   max_heart_rate_achieved exercise_induced_angina heart_disease_present
    ## 5                     145                       F                     0

``` r
# Pressure and ST depression boxplots
boxplot(train_set$resting_blood_pressure, ylab = "Resting Blood Pressure")
```

![](Logistic_files/figure-markdown_github/diagnostic2-2.png)

``` r
boxplot(train_set$oldpeak_eq_st_depression, ylab = "Oldpeak ST Depression")
```

![](Logistic_files/figure-markdown_github/diagnostic2-3.png)

``` r
# Show top records for ST depression
train_set %>% filter(oldpeak_eq_st_depression > 2.4) %>% select(patient_id,
              oldpeak_eq_st_depression, heart_disease_present) %>%
              arrange(desc(oldpeak_eq_st_depression))
```

    ##    patient_id oldpeak_eq_st_depression heart_disease_present
    ## 1      usnkhx                      6.2                     1
    ## 2      noxsnw                      5.6                     1
    ## 3      oyt4ek                      4.2                     0
    ## 4      6r9x2j                      4.2                     1
    ## 5      3nwy2n                      3.4                     1
    ## 6      2s2b1f                      3.4                     1
    ## 7      f1ziva                      3.2                     1
    ## 8      k7ef7h                      3.1                     1
    ## 9      lek9q9                      3.0                     1
    ## 10     v52zcs                      3.0                     1
    ## 11     328lkl                      2.8                     1
    ## 12     2gbyh9                      2.6                     1
    ## 13     syvayq                      2.6                     1
    ## 14     mxabaz                      2.6                     1

#### Model 1b - Basic Logistic Without Outlier

By removing only record \#5 there is a very small improvement but the model still performs poorly. The formula needs to be changed.

``` r
# Build model without influential point
model1b <- glm(heart_disease_present~.,data = train_set[-inf_pt,2:15],family = binomial)
# Calculate logloss
pred_outcome <- predict(model1b, newdata = test_set, type = "response")
LogLoss(y_pred = pred_outcome, y_true = as.numeric(test_set$heart_disease_present))
```

    ## [1] 0.5696305

#### Model 2 - Step function

I'll try first the "lazy" way which is to just apply the built-in function step to model1. This function tries to identify a subset of features that create the model with lowest AIC. The performance of this model is worse than the previous ones so I'll manually define the formula.

``` r
# Build model starting from model1, suppress output
model2 <- step(model1, trace=FALSE)
# Calculate logloss
pred_outcome2 <- predict(model2, newdata = test_set, type = "response")
LogLoss(y_pred = pred_outcome2, y_true = as.numeric(test_set$heart_disease_present))
```

    ## [1] 0.6618384

#### Model 3 - Manually chosen features

I'll start from model1b since it has a lower logloss than model1 and choose the features based on their estimated importance. I'll keep only the features that have Pr(&gt;|z|) &lt; 0.1, which means that the probability that they're not significant is less than 10%. These are: Slope of ST Segment, Chest Pain, Major Vessels, Cholesterol, Blood Sugar, EKG Results, Sex. By combining exploratory plots from section 3 and results of this model, here's some comments on individual features: \* The p-value for the Thallium Test is borderline while the plot shows an effect, maybe combining reversible and fixed defects in the same category or giving them an order (e.g. normal = 1, reversible defect = 2, fixed = 3) could change the result of the logistic regression model. The highest coefficient is actually the one for the reversible defect. \* EKG Results has an unusually high significance compared to the small effect shown in the plot \* the model says that Sex is surprisingly significant compared to what the plots show (especially the ones showing thallium test and chest pain vs sex) \* Blood sugar shows a significant effect with counterintuitively a strong coefficient in the opposite direction of heart disease (I'd assume that with higher blood sugar the risk would go up)

The model is significantly better, the Logloss is 0.4120261.

``` r
# Display model1b summary info
summary(model1b)
```

    ## 
    ## Call:
    ## glm(formula = heart_disease_present ~ ., family = binomial, data = train_set[-inf_pt, 
    ##     2:15])
    ## 
    ## Deviance Residuals: 
    ##      Min        1Q    Median        3Q       Max  
    ## -2.50477  -0.31868  -0.05566   0.20436   2.21840  
    ## 
    ## Coefficients:
    ##                                         Estimate Std. Error z value
    ## (Intercept)                           -16.882547   7.238334  -2.332
    ## slope_of_peak_exercise_st_segment       1.652103   0.827318   1.997
    ## thalnormal                              2.154320   2.897408   0.744
    ## thalreversible_defect                   4.758811   2.945778   1.615
    ## resting_blood_pressure                  0.027409   0.023934   1.145
    ## chest_pain_type                         0.761597   0.375282   2.029
    ## num_major_vessels                       1.567228   0.510748   3.068
    ## fasting_blood_sugar_gt_120_mg_per_dl1  -2.025687   1.184766  -1.710
    ## resting_ekg_results                     0.824296   0.416554   1.979
    ## serum_cholesterol_mg_per_dl             0.015573   0.008630   1.805
    ## oldpeak_eq_st_depression               -0.021738   0.458301  -0.047
    ## sexmale                                 2.608222   1.050732   2.482
    ## age                                     0.008884   0.043030   0.206
    ## max_heart_rate_achieved                -0.018333   0.019375  -0.946
    ## exercise_induced_anginaT                0.413422   0.914713   0.452
    ##                                       Pr(>|z|)   
    ## (Intercept)                            0.01968 * 
    ## slope_of_peak_exercise_st_segment      0.04583 * 
    ## thalnormal                             0.45716   
    ## thalreversible_defect                  0.10621   
    ## resting_blood_pressure                 0.25211   
    ## chest_pain_type                        0.04242 * 
    ## num_major_vessels                      0.00215 **
    ## fasting_blood_sugar_gt_120_mg_per_dl1  0.08731 . 
    ## resting_ekg_results                    0.04783 * 
    ## serum_cholesterol_mg_per_dl            0.07114 . 
    ## oldpeak_eq_st_depression               0.96217   
    ## sexmale                                0.01305 * 
    ## age                                    0.83643   
    ## max_heart_rate_achieved                0.34405   
    ## exercise_induced_anginaT               0.65129   
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for binomial family taken to be 1)
    ## 
    ##     Null deviance: 171.932  on 124  degrees of freedom
    ## Residual deviance:  62.706  on 110  degrees of freedom
    ## AIC: 92.706
    ## 
    ## Number of Fisher Scoring iterations: 7

``` r
# Create model with most important features
model3 <- glm(heart_disease_present~slope_of_peak_exercise_st_segment + 
                chest_pain_type + num_major_vessels + serum_cholesterol_mg_per_dl +
                fasting_blood_sugar_gt_120_mg_per_dl + resting_ekg_results + sex,
              data = train_set[-inf_pt,2:15],family = binomial)
# Calculate logloss
pred_outcome <- predict(model3, newdata = test_set, type = "response")
LogLoss(y_pred = pred_outcome, y_true = as.numeric(test_set$heart_disease_present))
```

    ## [1] 0.4120261

The analysis of influential points shows record \# 124 as a potential outlier, although not as clear as record \#5. I'll eliminate it and see if it improves the performance.

``` r
# Identify and eliminate influential points
plot(model3, which = 4)
```

![](Logistic_files/figure-markdown_github/diagnostic3-1.png)

``` r
inf_pt3 <- c(inf_pt,which(rownames(train_set) == "124"))
```

#### Model 3b - Manually chosen features Without Outliers

After eliminating Observation \# 124 and re-running the model, the logloss is significantly lower: 0.3251456

``` r
# Build model without influential point
model3b <- glm(heart_disease_present~slope_of_peak_exercise_st_segment + 
                chest_pain_type + num_major_vessels + serum_cholesterol_mg_per_dl +
                fasting_blood_sugar_gt_120_mg_per_dl + resting_ekg_results + sex,
              data = train_set[-inf_pt3,2:15],family = binomial)
pred_outcome <- predict(model3b, newdata = test_set, type = "response")
LogLoss(y_pred = pred_outcome, y_true = as.numeric(test_set$heart_disease_present))
```

    ## [1] 0.3251456

Record \# 106 is the most influential one so I'll eliminate it.

``` r
# Identify influential points
plot(model3b, which = 4)
```

![](Logistic_files/figure-markdown_github/diagnostic3b-1.png)

``` r
inf_pt3b <- c(inf_pt3,which(rownames(train_set) == "106"))
```

#### Model 3c - Manually chosen features Without Outliers

After eliminating Observation \# 106 and re-running the model, the logloss is now very low: 0.1726876. I further eliminated additional influential points but this didn't improve the model so I used model3c to generate the prediction for the competition based on the [Testing dataset](/test_values.csv). The official score was 0.4660, much higher than what estimated, which is a typical case of overfitting, in this case probably due to the elimination of observations that were not true outliers.

``` r
# Build model without influential point
model3c <- glm(heart_disease_present~slope_of_peak_exercise_st_segment + 
                 chest_pain_type + num_major_vessels + serum_cholesterol_mg_per_dl +
                 fasting_blood_sugar_gt_120_mg_per_dl + resting_ekg_results + sex,
               data = train_set[-inf_pt3b,2:15],family = binomial)
pred_outcome <- predict(model3c, newdata = test_set, type = "response")
LogLoss(y_pred = pred_outcome, y_true = as.numeric(test_set$heart_disease_present))
```

    ## [1] 0.1726876

#### Model 4 - Interaction Terms

I decided to eliminate only the first influential point identified and keep the other ones to avoid overfitting. The next improvement is to add interaction terms. This is the part where being a subject matter expert really makes a difference and, by not being a doctor, I can only try different combinations and see which one gives the best performance in terms of logloss, without any theory behind it. The best result is by adding an interaction term for Chest Pain with Slope of ST Segment and a term for Major Vessels with ST Segment. The interaction terms end up not being significant, the initial logloss estimate is 0.2535884 while the official score is 0.45279, slightly better than the previous model but still overfitted.

``` r
# Influential point from model1 removed
model4 <- glm(heart_disease_present~slope_of_peak_exercise_st_segment + 
            chest_pain_type + num_major_vessels + serum_cholesterol_mg_per_dl + 
              fasting_blood_sugar_gt_120_mg_per_dl + resting_ekg_results + sex +
                chest_pain_type * slope_of_peak_exercise_st_segment +
                slope_of_peak_exercise_st_segment * num_major_vessels,
              data = train_set[-inf_pt,2:15],family = binomial)
summary(model4)
```

    ## 
    ## Call:
    ## glm(formula = heart_disease_present ~ slope_of_peak_exercise_st_segment + 
    ##     chest_pain_type + num_major_vessels + serum_cholesterol_mg_per_dl + 
    ##     fasting_blood_sugar_gt_120_mg_per_dl + resting_ekg_results + 
    ##     sex + chest_pain_type * slope_of_peak_exercise_st_segment + 
    ##     slope_of_peak_exercise_st_segment * num_major_vessels, family = binomial, 
    ##     data = train_set[-inf_pt, 2:15])
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -2.4800  -0.5042  -0.1331   0.4119   2.2737  
    ## 
    ## Coefficients:
    ##                                                       Estimate Std. Error
    ## (Intercept)                                         -11.108199   3.654772
    ## slope_of_peak_exercise_st_segment                     0.876070   1.562193
    ## chest_pain_type                                       0.778222   0.810744
    ## num_major_vessels                                     0.307021   1.050557
    ## serum_cholesterol_mg_per_dl                           0.012111   0.006463
    ## fasting_blood_sugar_gt_120_mg_per_dl1                -1.484927   0.841250
    ## resting_ekg_results                                   0.531857   0.296502
    ## sexmale                                               2.762190   0.772536
    ## slope_of_peak_exercise_st_segment:chest_pain_type     0.190379   0.463908
    ## slope_of_peak_exercise_st_segment:num_major_vessels   0.866111   0.759078
    ##                                                     z value Pr(>|z|)    
    ## (Intercept)                                          -3.039  0.00237 ** 
    ## slope_of_peak_exercise_st_segment                     0.561  0.57494    
    ## chest_pain_type                                       0.960  0.33711    
    ## num_major_vessels                                     0.292  0.77010    
    ## serum_cholesterol_mg_per_dl                           1.874  0.06094 .  
    ## fasting_blood_sugar_gt_120_mg_per_dl1                -1.765  0.07754 .  
    ## resting_ekg_results                                   1.794  0.07285 .  
    ## sexmale                                               3.575  0.00035 ***
    ## slope_of_peak_exercise_st_segment:chest_pain_type     0.410  0.68153    
    ## slope_of_peak_exercise_st_segment:num_major_vessels   1.141  0.25387    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for binomial family taken to be 1)
    ## 
    ##     Null deviance: 171.932  on 124  degrees of freedom
    ## Residual deviance:  84.524  on 115  degrees of freedom
    ## AIC: 104.52
    ## 
    ## Number of Fisher Scoring iterations: 6

``` r
# Calculate logloss
pred_outcome <- predict(model4, newdata = test_set, type = "response")
LogLoss(y_pred = pred_outcome, y_true = as.numeric(test_set$heart_disease_present))
```

    ## [1] 0.2535884

#### Model 5 - Quadratic terms

The next step is to add quadratic terms to the formula. It makes sense to do it only for numeric features and this makes it easy since cholesterol is the only one. The result shows that the quadratic term is not significant, the initial logloss estimate is 0.2677256 while the official score is 0.4573, essentially no improvement.

``` r
model5 <- glm(heart_disease_present~slope_of_peak_exercise_st_segment + 
            chest_pain_type + num_major_vessels + serum_cholesterol_mg_per_dl + 
              fasting_blood_sugar_gt_120_mg_per_dl + resting_ekg_results + sex +
                chest_pain_type * slope_of_peak_exercise_st_segment +
                slope_of_peak_exercise_st_segment * num_major_vessels +
                I(serum_cholesterol_mg_per_dl^2),
              data = train_set[-inf_pt,2:15],family = binomial)
summary(model5)
```

    ## 
    ## Call:
    ## glm(formula = heart_disease_present ~ slope_of_peak_exercise_st_segment + 
    ##     chest_pain_type + num_major_vessels + serum_cholesterol_mg_per_dl + 
    ##     fasting_blood_sugar_gt_120_mg_per_dl + resting_ekg_results + 
    ##     sex + chest_pain_type * slope_of_peak_exercise_st_segment + 
    ##     slope_of_peak_exercise_st_segment * num_major_vessels + I(serum_cholesterol_mg_per_dl^2), 
    ##     family = binomial, data = train_set[-inf_pt, 2:15])
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -2.4903  -0.5037  -0.1286   0.4353   2.2862  
    ## 
    ## Coefficients:
    ##                                                       Estimate Std. Error
    ## (Intercept)                                         -8.539e+00  5.960e+00
    ## slope_of_peak_exercise_st_segment                    9.107e-01  1.577e+00
    ## chest_pain_type                                      7.779e-01  8.191e-01
    ## num_major_vessels                                    2.580e-01  1.068e+00
    ## serum_cholesterol_mg_per_dl                         -8.870e-03  3.916e-02
    ## fasting_blood_sugar_gt_120_mg_per_dl1               -1.500e+00  8.446e-01
    ## resting_ekg_results                                  5.237e-01  2.976e-01
    ## sexmale                                              2.813e+00  7.866e-01
    ## I(serum_cholesterol_mg_per_dl^2)                     4.029e-05  7.391e-05
    ## slope_of_peak_exercise_st_segment:chest_pain_type    1.824e-01  4.681e-01
    ## slope_of_peak_exercise_st_segment:num_major_vessels  9.184e-01  7.730e-01
    ##                                                     z value Pr(>|z|)    
    ## (Intercept)                                          -1.433 0.151966    
    ## slope_of_peak_exercise_st_segment                     0.577 0.563610    
    ## chest_pain_type                                       0.950 0.342276    
    ## num_major_vessels                                     0.242 0.809082    
    ## serum_cholesterol_mg_per_dl                          -0.227 0.820809    
    ## fasting_blood_sugar_gt_120_mg_per_dl1                -1.776 0.075676 .  
    ## resting_ekg_results                                   1.760 0.078428 .  
    ## sexmale                                               3.577 0.000348 ***
    ## I(serum_cholesterol_mg_per_dl^2)                      0.545 0.585682    
    ## slope_of_peak_exercise_st_segment:chest_pain_type     0.390 0.696749    
    ## slope_of_peak_exercise_st_segment:num_major_vessels   1.188 0.234748    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for binomial family taken to be 1)
    ## 
    ##     Null deviance: 171.93  on 124  degrees of freedom
    ## Residual deviance:  84.24  on 114  degrees of freedom
    ## AIC: 106.24
    ## 
    ## Number of Fisher Scoring iterations: 6

``` r
# Calculate logloss
pred_outcome <- predict(model5, newdata = test_set, type = "response")
LogLoss(y_pred = pred_outcome, y_true = as.numeric(test_set$heart_disease_present))
```

    ## [1] 0.2677256

#### Model 6 - Splines

My next attempt is to use splines, which are polynomial functions where a feature can be raised at a certain power. As we increase the power value, the curve obtained contains high oscillations which will lead to shapes that are over-flexible and therefore overfitting. Once again it can be applied only to cholesterol. By trying different values of the degrees of freedom, the best model has an estimated logloss = 0.1667506 and an official score = 0.4256, so a slight improvement. The spline terms, though, are not significant, which makes me skeptical that this model is truly better. I still keep this as the best logistic regression model. I created a subsequent model after eliminating the most influential point but without improvement.

``` r
# Apply spline to cholesterol (only numeric feature for which it makes sense)
library(splines)
model6 <- glm(heart_disease_present~slope_of_peak_exercise_st_segment + 
        chest_pain_type + num_major_vessels + ns(serum_cholesterol_mg_per_dl,df = 6) + 
          fasting_blood_sugar_gt_120_mg_per_dl + resting_ekg_results + sex +
                chest_pain_type * slope_of_peak_exercise_st_segment +
                slope_of_peak_exercise_st_segment * num_major_vessels,
              data = train_set[-inf_pt,2:15],family = binomial)
summary(model6)
```

    ## 
    ## Call:
    ## glm(formula = heart_disease_present ~ slope_of_peak_exercise_st_segment + 
    ##     chest_pain_type + num_major_vessels + ns(serum_cholesterol_mg_per_dl, 
    ##     df = 6) + fasting_blood_sugar_gt_120_mg_per_dl + resting_ekg_results + 
    ##     sex + chest_pain_type * slope_of_peak_exercise_st_segment + 
    ##     slope_of_peak_exercise_st_segment * num_major_vessels, family = binomial, 
    ##     data = train_set[-inf_pt, 2:15])
    ## 
    ## Deviance Residuals: 
    ##     Min       1Q   Median       3Q      Max  
    ## -2.7173  -0.4951  -0.1085   0.2888   2.2435  
    ## 
    ## Coefficients:
    ##                                                     Estimate Std. Error
    ## (Intercept)                                         -8.56535    3.54508
    ## slope_of_peak_exercise_st_segment                    1.59258    1.65746
    ## chest_pain_type                                      1.09469    0.86739
    ## num_major_vessels                                    0.17886    1.10481
    ## ns(serum_cholesterol_mg_per_dl, df = 6)1            -1.06318    1.93681
    ## ns(serum_cholesterol_mg_per_dl, df = 6)2            -2.61220    2.64507
    ## ns(serum_cholesterol_mg_per_dl, df = 6)3             0.77127    2.16367
    ## ns(serum_cholesterol_mg_per_dl, df = 6)4             0.61467    1.84719
    ## ns(serum_cholesterol_mg_per_dl, df = 6)5            -1.08178    4.59077
    ## ns(serum_cholesterol_mg_per_dl, df = 6)6             2.18912    2.85726
    ## fasting_blood_sugar_gt_120_mg_per_dl1               -1.23640    0.90896
    ## resting_ekg_results                                  0.54949    0.30906
    ## sexmale                                              2.89488    0.81543
    ## slope_of_peak_exercise_st_segment:chest_pain_type   -0.02504    0.49185
    ## slope_of_peak_exercise_st_segment:num_major_vessels  1.03974    0.79923
    ##                                                     z value Pr(>|z|)    
    ## (Intercept)                                          -2.416 0.015687 *  
    ## slope_of_peak_exercise_st_segment                     0.961 0.336623    
    ## chest_pain_type                                       1.262 0.206931    
    ## num_major_vessels                                     0.162 0.871390    
    ## ns(serum_cholesterol_mg_per_dl, df = 6)1             -0.549 0.583051    
    ## ns(serum_cholesterol_mg_per_dl, df = 6)2             -0.988 0.323362    
    ## ns(serum_cholesterol_mg_per_dl, df = 6)3              0.356 0.721494    
    ## ns(serum_cholesterol_mg_per_dl, df = 6)4              0.333 0.739314    
    ## ns(serum_cholesterol_mg_per_dl, df = 6)5             -0.236 0.813710    
    ## ns(serum_cholesterol_mg_per_dl, df = 6)6              0.766 0.443582    
    ## fasting_blood_sugar_gt_120_mg_per_dl1                -1.360 0.173755    
    ## resting_ekg_results                                   1.778 0.075411 .  
    ## sexmale                                               3.550 0.000385 ***
    ## slope_of_peak_exercise_st_segment:chest_pain_type    -0.051 0.959391    
    ## slope_of_peak_exercise_st_segment:num_major_vessels   1.301 0.193283    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for binomial family taken to be 1)
    ## 
    ##     Null deviance: 171.932  on 124  degrees of freedom
    ## Residual deviance:  81.059  on 110  degrees of freedom
    ## AIC: 111.06
    ## 
    ## Number of Fisher Scoring iterations: 7

``` r
# Calculate logloss
pred_outcome <- predict(model6, newdata = test_set, type = "response")
LogLoss(y_pred = pred_outcome, y_true = as.numeric(test_set$heart_disease_present))
```

    ## [1] 0.1667506
