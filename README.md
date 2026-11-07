# Instructor Effectiveness Analysis

## Project Overview

This project analyzes instructor-level teaching and learner-outcome
indicators and develops an **Instructor Effectiveness Score (IES)**.
Instructors are grouped into three descriptive tiers --- **Low, Medium,
and High** --- using their entropy-based IES. The notebook also compares
weighting approaches, explores instructor workload, and trains
classification models to reproduce the defined effectiveness tiers.

> **Important interpretation:** The effectiveness tiers are constructed
> from the dataset's own indicators and the entropy-weighted IES.
> Therefore, the classification model predicts these rule-derived
> labels; it does not independently establish an instructor's true
> teaching quality or prove that an instructor caused a particular
> student outcome.

## Objectives

-   Explore the batch-level dataset through descriptive statistics and
    visualizations.
-   Examine instructor workload, course coverage, learner outcomes,
    engagement, and feedback.
-   Aggregate batch-level observations into instructor-level indicators.
-   Construct an IES using data-driven weighting approaches.
-   Compare equal, entropy, and PCA weighting methods.
-   Assign instructors to Low, Medium, and High effectiveness tiers.
-   Compare classification models and tune a Random Forest model with
    Optuna.
-   Evaluate model performance and feature importance.
-   Examine the relationship between instructor workload and IES.
-   Discuss limitations and responsible use of the results.

## Dataset

The notebook uses the provided instructor effectiveness CSV dataset.

  Dataset characteristic     Value
  ------------------------ -------
  Batch-level records        2,000
  Unique instructors           120
  Unique courses                25
  Columns                       12
  Missing values found           0
  Duplicate rows found           0

Each row represents a batch. An instructor can teach multiple batches
and courses.

### Dataset columns

-   `batch_id`, `instructor_id`, `course_id`
-   `completion_rate`
-   `avg_score_improvement`
-   `avg_quiz_score`
-   `dropout_rate`
-   `avg_watch_time`
-   `assignment_submission_rate`
-   `forum_activity_rate`
-   `avg_feedback_score`
-   `feedback_response_rate`

The notebook expects the CSV file to be available in the same working
directory. Update the CSV path in the data-loading cell if your filename
or folder location is different.

## Methodology

### 1. Exploratory Data Analysis

The notebook examines data types, descriptive statistics, missing
values, duplicate records, feature distributions, boxplots,
correlations, and potential outliers. It also explores instructor-level
course and batch counts and relationships between workload and learner
outcomes.

Potential outliers are identified using the IQR rule and inspected; they
are not automatically removed.

### 2. Instructor-level aggregation

Batch-level records are grouped by `instructor_id`. The nine outcome,
engagement, and feedback indicators are aggregated using their mean. The
notebook also calculates:

-   `total_batches`: number of distinct batches taught by the
    instructor.
-   `total_courses`: number of distinct courses taught by the
    instructor.

These workload counts are used for contextual analysis, not as
components of the IES.

### 3. Normalization and indicators

The indicators are scaled to a common 0--1 range using Min-Max
normalization. Dropout rate is reversed so that a higher normalized
value represents a more favorable retention direction.

Completion and reversed dropout are combined into one `retention_score`
by taking their mean. This avoids treating the two strongly inversely
correlated measures as separate IES indicators.

The final eight indicators are:

1.  `avg_score_improvement`
2.  `avg_quiz_score`
3.  `avg_watch_time`
4.  `assignment_submission_rate`
5.  `forum_activity_rate`
6.  `avg_feedback_score`
7.  `feedback_response_rate`
8.  `retention_score`

### 4. Weighting methods and IES

Three weighting approaches are compared:

-   **Equal weighting:** assigns the same weight to each of the eight
    indicators.
-   **Entropy weighting:** derives weights from the
    variation/information distribution of the indicators in the
    instructor-level data.
-   **PCA weighting:** uses the absolute loadings of the first principal
    component, normalized so the weights sum to one.

The notebook compares the weights and their rankings. The primary score
used for tier assignment is the **Entropy-based IES**:

`IES_Entropy = sum(normalized_indicator × entropy_weight) × 100`

Entropy weights measure data variation; they should not be interpreted
as proven educational or causal importance.

### 5. Effectiveness tiers

The 33rd and 67th percentiles of `IES_Entropy` are used as cut-offs:

-   **Low:** at or below the lower threshold.
-   **Medium:** above the lower threshold and at or below the upper
    threshold.
-   **High:** above the upper threshold.

The notebook's resulting distribution is 40 instructors in each tier
(33.33% each). These are relative, percentile-based categories, not
externally validated performance standards.

The notebook also creates three separate instructor tables --- High,
Medium, and Low --- with each table sorted by `IES_Entropy` in
descending order.

### 6. Classification and hyperparameter tuning

The notebook compares:

-   Logistic Regression
-   Random Forest
-   XGBoost

The initial model comparison uses weighted F1-score to select a model
for tuning. Random Forest is then tuned using **Optuna** with 30 trials
and 5-fold stratified cross-validation, optimizing weighted F1-score.

### 7. Evaluation and interpretation

Evaluation includes accuracy, weighted precision, weighted recall,
weighted F1-score, a confusion matrix, a classification report, and a
comparison of training, cross-validation, and test performance.

Feature interpretation includes Random Forest impurity-based feature
importance and permutation importance. Instructor workload is examined
using Spearman correlation and plots against the entropy-based IES.

## Results recorded in the notebook

### Initial model comparison

  ------------------------------------------------------------------------
  Model              Accuracy       Weighted       Weighted    Weighted F1
                                   Precision         Recall 
  ------------ -------------- -------------- -------------- --------------
  Random               0.8333         0.8558         0.8333         0.8359
  Forest                                                    

  Logistic             0.8333         0.8889         0.8333         0.8325
  Regression                                                

  XGBoost              0.8000         0.8139         0.8000         0.8017
  ------------------------------------------------------------------------

### Baseline vs. Optuna-optimized Random Forest

  ------------------------------------------------------------------------------
  Model                    Accuracy       Weighted       Weighted    Weighted F1
                                         Precision         Recall 
  ------------------ -------------- -------------- -------------- --------------
  Baseline Random            0.8333         0.8558         0.8333         0.8359
  Forest                                                          

  Optuna-optimized           0.8000         0.8139         0.8000         0.8017
  Random Forest                                                   
  ------------------------------------------------------------------------------

The optimized model's recorded 5-fold cross-validation weighted F1-score
is **0.9106**. Its recorded training weighted F1-score is **0.9667**,
and test weighted F1-score is **0.8017** (a train--test difference of
**0.1650**).

**Interpretation:** Although Optuna found a strong cross-validation
score, the optimized model did not outperform the baseline Random Forest
on the held-out test set. The training--test gap also suggests possible
overfitting. The held-out test result should not be hidden or replaced
by the higher cross-validation score.

### Feature importance

In the notebook's optimized Random Forest, the largest impurity-based
importance values are recorded for:

-   `retention_score`: 0.5445
-   `feedback_response_rate`: 0.3599

Permutation importance also places these two indicators highest in the
recorded test-set analysis. These values describe how the model
reproduces the constructed tiers; they do not establish causal teaching
importance.

### Workload relationship

The recorded Spearman correlations with `IES_Entropy` are:

-   `total_batches`: 0.0814
-   `total_courses`: 0.0532

These values indicate very weak positive rank associations in this
dataset. They do not establish that teaching more batches or courses
improves effectiveness.

## Key Limitations and Responsible Use

1.  **Constructed target:** Low, Medium, and High labels are created
    from `IES_Entropy`, which itself is calculated from the same
    indicators used as model inputs. This creates a circular prediction
    task. The classifier learns to reproduce the scoring rule rather
    than predict an independent, verified measure of teaching
    effectiveness.
2.  **Entropy is not educational importance:** Entropy weights reflect
    variation in the observed data, not the educational value or causal
    influence of each indicator.
3.  **Limited instructor sample:** The instructor-level dataset contains
    120 instructors. Results may not generalize to other institutions,
    courses, or time periods.
4.  **Confounding:** Course difficulty, student background, prior
    knowledge, batch size, resources, and other factors may influence
    learner outcomes.
5.  **Proxy limitations:** Watch time, assignment submission, feedback,
    completion, and quiz scores are imperfect proxies for teaching
    quality and learning.
6.  **Overfitting and test performance:** The optimized Random Forest
    has a lower held-out test weighted F1 than the baseline model and a
    noticeable training--test gap.
7.  **No causal conclusion:** Correlations and model feature importances
    do not prove that an instructor caused the observed outcomes.

**The model should not be used as the sole basis for instructor
appraisal, hiring, promotion, or disciplinary decisions.** It may be
used as an exploratory decision-support tool alongside classroom
observations, student feedback, course context, independent learning
assessments, and human review.

## Tools and Libraries

-   Python
-   pandas
-   NumPy
-   Matplotlib
-   Seaborn
-   scikit-learn
-   XGBoost
-   Optuna
-   Jupyter Notebook

## How to Run

1.  Clone or download this repository.
2.  Place the provided CSV dataset in the notebook's working directory,
    or update the CSV path in the first data-loading section.
3.  Install the required packages in your Python environment.
4.  Open `instructor_effectiveness.ipynb` in Jupyter Notebook,
    JupyterLab, or Google Colab.
5.  Run the notebook cells from top to bottom.

Example installation command:

``` bash
pip install pandas numpy matplotlib seaborn scikit-learn xgboost optuna jupyter
```

## Repository Contents

``` text
instructor-effectiveness-analysis/
├── instructor_effectiveness.ipynb
├── README.md
└── instructor_effectiveness_dataset_2000_rows - instructor_effectiveness_dataset_2000_rows.csv.csv
```

Include the dataset in the repository only if you have permission to
redistribute it. Otherwise, provide it separately through an approved
access method and explain where the evaluator can obtain it.

## Project Status

The notebook contains the EDA, instructor-level aggregation, three
weighting comparisons, entropy-based IES and tiers, separate tier
tables, model comparison, Optuna tuning, evaluation, feature-importance
analysis, workload analysis, and written limitations.
