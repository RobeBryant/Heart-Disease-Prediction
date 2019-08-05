# Heart Disease Prediction

The scope of this project is to predict the probability of heart disease in patients by using the dataset available via the UCI Machine Learning repository at this link: http://archive.ics.uci.edu/ml/datasets/statlog+(heart)

The project is divided in the following sections:

* [1. Features Description](/Features.md)
* 2. Performance Metrics
* 3. Exploratory Analysis
* 4. Correlation Analysis
* 5. Logistic Regression Models
* 6. Random Forest Models
* 7. Penalized  Regression Models
* 8. Boosting Models
* 9. Neural Network Models
* 10. Ensemble Models
* 11. Conclusions

### Performance Metric

The performance of the model is measured by log loss (also called Binary Cross-Entropy). The formula is:

![](LogLoss.png){width=350px}

where:

y is the label (1 for heart disease presence)
p(y) is the probability predicted by the model
N is the number of records

Mathematically this formula penalizes large errors (probabilities far from the true label), which makes sense for an healthcare related problem where big mistakes can have catastrophic consequences. This plot shows the log loss as a function of the predicted probability for a case in which the true value is 1. You can see how large it becomes as the prediction falls under 0.5.

![](LogLossPlot.png){width=350px}

Because the goal of the competition is minimizing the log loss, simplifying the model by reducing the number of features will not be pursued in this project.