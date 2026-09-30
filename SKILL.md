---
name: reusable-tabular-classification
description: Run a reusable, human-reviewed, leakage-safe end-to-end classification workflow for CSV or TSV tabular datasets; ask the AI to select and justify suitable preprocessing and two classification models; tune, evaluate, and compare them; and automatically generate a verified two-page IN6227 Assignment 1 Variant 2 PDF report with the required identity, LLM, and GitHub metadata. Use when a user supplies tabular classification data and needs a generalizable workflow and assignment-ready report. Reflection writing and final PDF merging are outside this skill.
---

# Reusable Tabular Classification

## 1. Objective and boundaries

Execute a complete classification workflow that generalizes to tabular datasets beyond the assignment dataset. Prioritize justified decisions, leakage prevention, reproducibility, and clear reasoning rather than maximum accuracy alone.

Automatically generate the main assignment report. Do not write, review, rewrite, approve, or append the student's Reflection. Do not merge the report with a Reflection. Reflection writing and final submission assembly are separate human tasks after this skill finishes.

## 2. Inputs

### Data inputs

Require:

- `dataset_path`: a CSV/TSV training file or a directory containing an obvious training file.

Accept optionally:

- `test_path`: a separate test file;
- `target_column`;
- `positive_class` for binary classification;
- `random_seed`, default `42`;
- `holdout_size`, default `0.20` when no labelled test set exists;
- `split_strategy`: `auto`, `random`, `group`, or `temporal`, default `auto`;
- `group_column` when repeated entities, subjects, customers, devices, or other groups must not cross splits;
- `time_column` when prediction must respect chronological order;
- `review_mode`, default `true`;
- `report_mode`: `draft` or `final`, default `draft`.

### Report metadata

Require before generating a final report:

- `full_name`;
- `matric_number`;
- `llm_model_name_version`;
- `llm_interface`;
- `repository_url`: the real GitHub repository URL containing the final `SKILL.md`.

Treat all metadata as human-supplied. Never invent identity details, model versions, interfaces, or repository links.

### Draft and final modes

- In `draft` mode, allow missing report metadata, but label the PDF `DRAFT - NOT FOR SUBMISSION` and identify every missing item.
- In `final` mode, stop unless all report metadata is present.
- In `final` mode, require `repository_url` to begin with `https://github.com/`, identify a repository rather than a file page or `.git` clone URL, and contain none of: `pending`, `placeholder`, `not provided`, `example.com`, `<...>`, or `TBD`.
- Ask the human to confirm that the repository is accessible to the marker and contains the final `SKILL.md`.
- Reflection availability is never a condition for generating the two-page report.

## 3. Outputs

Create:

- `classification_report.pdf`: the automatically generated main report, maximum two pages;
- `classification_results.json`: the numerical source of truth used to build and verify the report;
- `classification_run_log.md`: data decisions, model settings, human approvals, overrides, and verification results;
- any small figure required by the report.

If analysis code is created during execution, preserve it with the supporting outputs for reproducibility. The JSON, run log, code, and figures are supporting artifacts, not separate NTULearn submission files. Do not create a combined submission PDF.

## 4. Resolve the task and lock the evaluation set

1. Load the CSV/TSV data with pandas.
2. Resolve the training file from `dataset_path`. If it is a directory, identify an obvious file such as `train.csv` and look for an optional file such as `test.csv`.
3. Resolve `target_column` from common names such as `label`, `target`, `class`, `outcome`, `response`, or `y` only when exactly one candidate is plausible. Otherwise stop and ask the human.
4. Remove rows with missing targets only after reporting the exact number removed.
5. Confirm that the target is categorical. Stop if it appears continuous or regression-like.
6. For binary classification, obtain explicit human confirmation of `positive_class` before calculating positive-class metrics.
7. Before choosing a split, inspect the columns and ask whether rows have group dependence or temporal order. Resolve `split_strategy` as follows:
   - use `group` when the same entity can appear in multiple rows and groups must not cross development/evaluation or cross-validation folds;
   - use `temporal` when evaluation must simulate prediction of later observations from earlier observations;
   - use `random` only when rows are reasonably independent and identically distributed;
   - in `auto`, stop and ask when group or temporal structure is plausible but ambiguous.
8. If a separate test file exists, determine whether it is labelled:
   - if labelled, verify compatible predictors and target classes, lock it immediately, and use it only once for final evaluation;
   - if unlabelled, do not use it for evaluation; create a locked holdout from the training data using the resolved split strategy instead.
9. If no labelled external test set exists, create the locked evaluation set before decision-oriented exploration:
   - for `random`, use a stratified `train_test_split(..., test_size=holdout_size, stratify=y, random_state=random_seed)`;
   - for `group`, use a group-aware holdout that keeps each group entirely on one side of the split;
   - for `temporal`, sort by `time_column` and use earlier observations for development and later observations for evaluation without shuffling.
10. Never use the locked evaluation set for cleaning decisions, feature selection, preprocessing choices, model selection, hyperparameter tuning, or threshold selection.
11. Record development/evaluation origins, row counts, class counts, split strategy, group/time fields when applicable, the seed, and split parameters.

## 5. Explore and clean the development data

Use only the development/training partition for analysis-driven decisions.

Record:

- row count, column count, predictor count, column names, and data types;
- target counts, percentages, and imbalance ratio;
- missing counts and percentages by predictor;
- exact duplicate-row count;
- numeric summaries: count, mean, median, standard deviation, minimum, quartiles, maximum, missingness, and IQR flags;
- categorical summaries: unique counts, common levels, missingness, rare levels, and high cardinality.

Treat IQR outlier flags as diagnostics rather than automatic deletion rules. Remove, cap, or transform observations only with a clear data-quality or modelling reason. Remove duplicate rows only when evidence suggests accidental duplication rather than legitimate repeated observations.

Identify one or two clear predictor-target associations from development data. Use suitable summaries such as class-specific medians, standardized differences, category rates, or an association measure. Describe them as associations, never causal effects.

## 6. Select and justify preprocessing and features

Ask the AI to choose preprocessing strategies from the observed dataset rather than applying a fixed recipe blindly. Justify every cleaning, transformation, feature-selection, or feature-engineering decision.

Place all learned preprocessing and the classifier inside a scikit-learn `Pipeline`, using a `ColumnTransformer` when predictors require different treatments. Suitable choices may include:

- numeric imputation with the median;
- categorical imputation with the most frequent level or an explicit `Missing` level;
- one-hot encoding with unknown categories ignored;
- scaling for distance-based or regularized linear models;
- rare-level grouping when justified;
- dropping an identifier-like or leakage-prone field only with a stated reason.

Fit every learned preprocessing step inside each cross-validation fold. Do not engineer features merely to make the workflow appear more complex. If no feature selection or engineering is necessary, record that conclusion and explain why.

## 7. Human checkpoint 1: approve the analysis plan

When `review_mode=true`, present a compact plan containing:

- target, task type, and confirmed positive class when binary;
- development and locked evaluation sets, including the split strategy and its rationale;
- missingness, imbalance, duplicates, and outlier diagnostics;
- proposed preprocessing and feature decisions with reasons;
- two proposed classifiers and dataset-specific reasons;
- proposed tuning metric;
- whether false-positive and false-negative costs are known.

Require explicit approval or an override before tuning. Record the human's response in the run log. Update the recorded plan if the human changes any decision.

## 8. Select and tune exactly two classifiers

Ask the AI to select exactly two classifiers that suit the dataset. Prefer complementary models whose comparison is informative. Reasonable candidates include:

- Logistic Regression for an interpretable linear baseline;
- Decision Tree for interpretable nonlinear rules;
- Random Forest for a robust ensemble comparator;
- K-Nearest Neighbours when sample size and dimensionality are practical and scaling is used;
- another established classifier when its assumptions and complexity are justified.

Do not select a model solely because it is complex or likely to maximize accuracy. State why each selected model fits the dataset.

Tune using development data only. Match cross-validation to the data structure:

- ordinarily use `StratifiedKFold(n_splits=5, shuffle=True, random_state=random_seed)` for independent rows;
- use group-aware folds such as `StratifiedGroupKFold` or `GroupKFold` when groups must remain separate;
- use chronological or time-series folds without shuffling for temporal prediction.

Use a restrained `GridSearchCV` or `RandomizedSearchCV`. Record:

- the complete search space;
- the scoring metric;
- best parameters;
- mean cross-validation score;
- relevant stopping or complexity controls.

Choose evaluation metrics according to the task:

- meaningfully imbalanced binary data: positive-class F1, average precision, balanced accuracy, or another justified minority-sensitive metric;
- reasonably balanced binary data: accuracy together with positive-class F1;
- multiclass data: macro-F1 or balanced accuracy.

Ask whether false-positive and false-negative costs are known. When costs are unknown, keep the default decision threshold and avoid claiming application-optimal performance. If a threshold is tuned, use development validation predictions only.

For models without iterative early stopping, state that early stopping is not applicable. Describe controls such as `max_depth`, `min_samples_leaf`, regularization, or `n_estimators` accurately.

## 9. Final evaluation and comparison

After tuning and plan approval:

1. Refit each complete pipeline on all development data.
2. Evaluate each model exactly once on the locked evaluation set.
3. Do not change preprocessing, features, models, hyperparameters, metrics, or thresholds after inspecting evaluation results unless the human explicitly requests a new analysis run.

For binary classification, calculate at minimum:

- Accuracy, Precision, Recall, and F1 for the confirmed positive class;
- confusion matrix with explicit class order;
- false-positive and false-negative counts;
- ROC-AUC and Average Precision/PR-AUC when valid scores are available;
- majority-class accuracy baseline.

For multiclass classification, calculate at minimum:

- Accuracy;
- macro Precision, macro Recall, and macro F1;
- confusion matrix;
- a suitable baseline;
- ROC-AUC OVR only when valid and useful.

Compare the models using the full metric pattern rather than a single number. Discuss imbalance, precision-recall trade-offs, interpretability, complexity, and uncertainty. Do not declare a universal winner from a trivial difference.

## 10. Human checkpoint 2: approve the results

When `review_mode=true`, show:

- selected models and best hyperparameters;
- cross-validation scores;
- complete final metrics;
- labelled confusion matrices;
- baseline performance;
- concise comparison, limitations, and any suspicious result.

Ask the human to confirm that the results are plausible before generating the report. Record approval, corrections, overrides, or a requested rerun. Never hide a failed or suspicious verification.

## 11. Generate the two-page PDF report

Generate `classification_report.pdf` automatically from `classification_results.json`. Do not manually retype stored numerical values.

At the top of page 1, include all four items required by the assignment:

- matric number;
- full name;
- `IN6227-Assignment-1`;
- `Variant-2`.

Also include:

- LLM model name and version;
- LLM interface used;
- the real GitHub repository URL in final mode.

Cover every required reporting criterion within no more than two pages:

1. **Data exploration and cleaning:** describe the dataset; report missing values, outliers, class imbalance, duplicates, and other relevant issues; explain every cleaning and preprocessing step and why it was chosen.
2. **Feature selection or engineering:** explain what was done and the rationale, or state clearly that none was necessary and why.
3. **Model training:** explain why the two models were selected, how each was configured, the cross-validation and hyperparameter-tuning method, best settings, and stopping or complexity controls.
4. **Evaluation and comparison:** compare both models using suitable metrics, labelled confusion-matrix information, and an appropriate baseline.
5. **Findings and discussion:** summarize important associations and results, explain the preferred model for the stated objective or metric, and discuss limitations, interpretability, imbalance, error-cost uncertainty, and generalization.

Use compact tables and at most one small useful figure when it materially improves comprehension. Keep text readable; do not meet the page limit by shrinking text excessively. Separate observed facts, modelling decisions, and interpretation. Never imply causality from predictive associations.

The PDF must contain only the generated two-page report. Do not include the assignment brief or Reflection.

## 12. Verify before delivery

### Data and leakage checks

Verify that:

- all reported row, predictor, and class counts match the data actually used;
- the evaluation set was locked before decision-oriented exploration and tuning;
- the holdout and cross-validation strategies respect any group or temporal structure;
- imputation, encoding, scaling, feature selection, and the classifier were fitted inside training folds;
- tuning used development data only;
- final evaluation occurred only after tuning.

### Metric checks

Verify that:

- the positive class and confusion-matrix order are explicit;
- Precision, Recall, F1, FP, and FN use the same positive class;
- confusion-matrix cells sum to the evaluation row count;
- recomputed metrics from the confusion matrix match stored results within rounding tolerance;
- ROC-AUC and Average Precision use scores rather than hard labels;
- every number in the report matches `classification_results.json`.

### Assignment and layout checks

Verify that:

- the generated report has no more than two pages;
- page 1 contains matric number, full name, `IN6227-Assignment-1`, and `Variant-2`;
- model name/version, LLM interface, and the real GitHub repository URL are present;
- all data, preprocessing, feature, model-training, tuning, stopping-control, evaluation, comparison, findings, and discussion requirements are covered;
- decisions are justified rather than presented as unexplained choices;
- conclusions are not based on accuracy alone;
- the rendered PDF has no clipping, overlap, unreadable text, broken tables, or placeholder text.

Render the PDF to images and inspect every page before delivery.

## 13. Stop conditions

Stop and ask the human when:

- the target is ambiguous or regression-like;
- the positive class for a binary task is unconfirmed;
- train/test schemas are incompatible;
- no valid labelled evaluation set can be created;
- leakage or a target-encoding predictor is suspected;
- checkpoint approval is required but missing;
- reported values cannot be reproduced;
- final mode lacks identity details, LLM metadata, or a real GitHub repository URL;
- the report exceeds two pages or fails visual verification.

Do not stop because a Reflection has not been supplied.

## 14. Final response

Return:

1. `classification_report.pdf`;
2. `classification_results.json`;
3. `classification_run_log.md`;
4. selected models and primary metric;
5. unresolved human decisions, if any;
6. verification status.

State `FINAL REPORT READY` only when every final-mode report requirement passes. Otherwise state `DRAFT` and list the blockers. Never state that the complete assignment submission is ready, because Reflection writing and final PDF assembly remain the human's responsibility.

Remind the human to write their own short Reflection covering Human oversight, Critical evaluation, and Trustworthiness; manually place it after the unchanged two-page report; export one combined PDF; and submit only that PDF to NTULearn -> Assignments by Wednesday, 2026-10-07, 23:59:59. Remind them to confirm that the final GitHub repository contains the submitted `SKILL.md`.
