### Performance Metric

The performance of the model is measured by log loss. The formula is:

![](LogLoss.png){width=350px}

where:

y is the label (1 for heart disease presence)
p(y) is the probability predicted by the model
N is the number of records

Mathematically this formula penalizes large errors (probabilities far from the true label), which makes sense for an healthcare related problem where big mistakes can have catastrophic consequences. This plot shows the log loss as a function of the predicted probability for a case in which the true value is 1. You can see how large it becomes as the prediction falls under 0.5.

![](LogLossPlot.png){width=350px}

For error rate between 0 and 1, the Log Loss is equal to Cross-Entropy, which is another term often used in Machine Learning.

Because the goal of the competition is minimizing the log loss, simplifying the model by reducing the number of features will not be pursued in this project.

The benchmark used by the host of the competition is Logistic Regression with a log loss = 0.5381.