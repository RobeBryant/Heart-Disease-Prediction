# Heart Disease Prediction

The scope of this project is to predict the probability of heart disease in patients by using the dataset available via the UCI Machine Learning repository at this link: http://archive.ics.uci.edu/ml/datasets/statlog+(heart)

### Features

The study collects various measurements on patient health and cardiovascular statistics. The 13 features are:

* __slope_of_peak_exercise_st_segment__ (<span style="color:blue">integer</span>): the slope of the peak exercise ST segment, an electrocardiography read out indicating quality of blood flow to the heart
* __thal__ (<span style="color:blue">categorical</span>): results of thallium stress test measuring blood flow to the heart, with 3 possible values equal to normal, fixed_defect, reversible_defect
* __resting_blood_pressure__ (<span style="color:blue">integer</span>): resting blood pressure
* __chest_pain_type__ (<span style="color:blue">integer</span>): it takes 4 values from 1 to 4
* __num_major_vessels__ (<span style="color:blue">integer</span>): number of major vessels colored by flouroscopy (from 0 to 3)
* __fasting_blood_sugar_gt_120_mg_per_dl__ (<span style="color:blue">binary</span>): 1 if fasting blood sugar > 120 mg/dl, 0 otherwise
* __resting_ekg_results__ (<span style="color:blue">integer</span>): resting electrocardiographic results (values 0, 1, 2)
* __serum_cholesterol_mg_per_dl__ (<span style="color:blue">integer</span>): serum cholesterol in mg/dl
* __oldpeak_eq_st_depression__ (<span style="color:blue">decimal</span>): oldpeak = ST depression induced by exercise relative to rest, a measure of abnormality in electrocardiograms
* __sex__ (<span style="color:blue">binary</span>): 0 for female, 1 for male
* __age__ (<span style="color:blue">integer</span>): age in years
* __max_heart_rate_achieved__ (<span style="color:blue">integer</span>): maximum heart rate achieved (beats per minute)
* __exercise_induced_angina__ (<span style="color:blue">binary</span>): exercise-induced chest pain (0: False, 1: True)

### Performance Metric

The performance of the model is measured by binary log loss.