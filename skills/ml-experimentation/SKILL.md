---
name: ml-experimentation
description: "Apply when building, training, evaluating, or deploying machine learning models: experiment tracking, data validation, feature engineering, model selection, hyperparameter tuning, and MLOps. Trigger on mentions of training, model evaluation, feature engineering, experiment tracking, ML pipeline, or model deployment. Language-specific idioms are in references/."
---

# ML Experimentation Skill

Machine learning development is fundamentally different from traditional software engineering. The source of truth is not a specification — it is data, and data changes. Models are not deterministic artifacts — they are statistical approximations that degrade over time. Every decision, from data collection to deployment, involves tradeoffs that compound silently. Treating ML development with the same rigor as software engineering — versioning, testing, monitoring, documentation — is not optional; it is the difference between a working system and an expensive random number generator.

The principles below address the full lifecycle: from early experimentation through production deployment. They are grounded in the reality that most ML projects fail not because of model architecture, but because of data quality issues, poor experiment discipline, misaligned metrics, or the gap between a notebook prototype and a reliable production system. Applied well, they prevent the most common failure modes and build a foundation for systematic, measurable improvement.

**Language-specific frameworks and examples are in the `references/` directory.** Read the relevant file when generating ML code:
- `references/python.md` — scikit-learn, PyTorch, experiment tracking tools

---

## 1. Experiment Tracking and Reproducibility

Every ML experiment must be reproducible. An experiment you cannot reproduce is an experiment you cannot trust, debug, or build upon. Huyen identifies reproducibility as a foundational requirement of any ML system — without it, you cannot systematically improve.

**Track every input to the experiment.** This means: code version (git commit hash), data version (snapshot or hash of the dataset used), all hyperparameters, the software environment (library versions, hardware configuration), random seeds, and the complete results including intermediate metrics.

If any of these are missing, you cannot reliably reproduce the outcome or diagnose why a later run produced different results. A common failure mode is tracking the model and metrics but not the data version — making it impossible to determine whether a performance change came from a code change or a data change.

**Version your data as rigorously as your code.** Burkov emphasizes that data is a first-class artifact in ML engineering. Code changes are visible in diffs; data changes are invisible unless you actively track them. A model that performed well last week and poorly today may be reacting to a silent data change, not a code regression. Use dataset versioning tools or immutable dataset snapshots tied to each experiment run.

**Get the infrastructure right before optimizing the model.** Google's Rules of ML, Rule 4: "Keep the first model simple and get the infrastructure right." The experiment tracking infrastructure — logging, metric collection, artifact storage, comparison dashboards — matters more than the sophistication of your first model. A simple model with excellent tracking will be systematically improved over time. A complex model with no tracking is a dead end.

**Automate experiment logging.** Manual logging is error-prone and incomplete. Configure your training pipeline to automatically record every run with its full context. When you review results weeks later, the logs should tell you exactly what happened without relying on memory or notebook state.

**Set random seeds everywhere, but don't trust them blindly.** Random seeds control initialization, data shuffling, dropout, and sampling. Setting them is necessary for reproducibility but not sufficient — GPU nondeterminism, multithreading, and library version differences can still produce variation. Document the expected level of reproducibility (exact match vs. within tolerance) and validate it by running the same experiment twice.

**Compare experiments systematically.** An experiment log is only useful if you can query and compare across runs. Structure your tracking so you can answer questions like: which hyperparameter configuration produced the best validation metric? How did performance change when the dataset version changed? What was the effect of adding feature X? Without structured comparison, experiment logs become write-only archives.

---

## 2. Data Quality Over Model Complexity

Most ML failures are data failures, not model failures. Huyen is unambiguous: data quality issues — mislabeled examples, distribution shifts, missing values, train/test contamination — cause far more production incidents than poor model architecture. Burkov distills this to the foundational ML principle: garbage in, garbage out.

**Start without ML if the data isn't ready.** Google's Rules of ML, Rule 1: "Don't be afraid to launch a product without machine learning." If your data is noisy, incomplete, or poorly labeled, a heuristic or rule-based system will often outperform a model trained on bad data — and will be far easier to debug. Rule 16 reinforces this: "Plan to launch and iterate." Ship a simple solution, collect better data in production, then revisit ML.

**Validate data at every stage.** Implement schema checks (expected columns, types, value ranges), distribution monitoring (detect shifts between training data and incoming data), and completeness validation (flag missing values, unexpected nulls, truncated records). Data validation is not a one-time task — it is a continuous process that runs on every data pipeline execution and before every training run.

**Prevent train/test contamination.** Data leakage between training and evaluation sets is one of the most common and devastating mistakes. It produces artificially high evaluation metrics that collapse in production.

Contamination can be subtle: temporal leakage (using future data to predict the past), group leakage (same entity appearing in both train and test), or preprocessing leakage (fitting scalers or encoders on the full dataset before splitting). Always split the data first, then preprocess each split independently.

**Invest in labeling quality.** For supervised learning, the labels are the ground truth your model learns from. Noisy labels set a ceiling on model performance that no architecture improvement can break through. Establish clear labeling guidelines, measure inter-annotator agreement, and treat labeling disagreements as data to investigate rather than noise to average over.

**Version datasets and track lineage.** Every model should be traceable to the exact dataset it was trained on, including the transformations applied. Burkov stresses that data pipelines introduce silent changes — a new data source, a changed join condition, a filtering bug — that propagate into model behavior. Immutable dataset snapshots with clear lineage from raw sources through each transformation step make these changes visible and debuggable.

---

## 3. Feature Engineering

Feature engineering is the highest-leverage activity in most ML projects. Huyen argues that the difference between a mediocre model and a strong one is more often the features than the algorithm. A well-engineered feature encodes domain knowledge that the model would otherwise need orders of magnitude more data to discover on its own. Before reaching for a more complex model, exhaust the possibilities of better features with the model you have.

**Combine and modify features in human-understandable ways.** Google's Rules of ML, Rule 20: create new features by combining and transforming existing ones in ways that are interpretable. Ratios, differences, aggregations over time windows, and interaction terms derived from domain knowledge are powerful because they encode known relationships directly. A feature like "ratio of failed logins to total logins in the past 24 hours" carries more signal for fraud detection than raw counts alone, and its meaning is immediately clear to anyone debugging the model.

**Start with domain expertise, validate with data.** Thakur emphasizes that the best features come from understanding the problem domain. Talk to domain experts, study existing heuristics, and examine what features manual decision-makers use. Then validate these intuitions empirically: compute feature importance, check correlation with the target, and verify that the feature adds predictive power beyond what existing features provide.

**Guard against feature leakage.** Feature leakage occurs when training-time features contain information that would not be available at prediction time. This is the feature-level equivalent of train/test contamination and is equally devastating.

Common sources: using the target variable (or a proxy for it) as a feature, using data that arrives after the prediction point in temporal problems, or applying global statistics computed across the entire dataset rather than only on the training fold. When a model achieves suspiciously high performance, feature leakage should be the first hypothesis to investigate.

**Build feature stores for reuse.** When features are computed ad hoc in notebooks, they diverge between training and serving, creating training-serving skew. A feature store provides a single source of truth for feature definitions and values, ensuring that the features used during training match those available at inference time. This is particularly critical for real-time serving where feature computation must be fast and consistent.

**Analyze feature importance systematically.** After training, examine which features contribute most to predictions and which contribute noise. Thakur advocates systematic feature engineering: combine domain knowledge with data exploration, then validate through feature importance analysis. Remove features that add complexity without improving performance — they increase overfitting risk and inference latency.

**Handle missing values and categorical variables deliberately.** The strategy for missing values (imputation, indicator variables, or model-native handling) and categorical encoding (one-hot, target encoding, embeddings) should be a conscious decision documented alongside the experiment, not an afterthought buried in preprocessing code. Different strategies can produce meaningfully different model behavior, and the choice should be informed by the nature of the missingness (random, systematic, or informative) and the characteristics of the categorical variable (cardinality, ordering, frequency distribution).

**Beware of high-cardinality categorical features.** One-hot encoding a feature with thousands of categories creates a sparse, high-dimensional representation that inflates memory usage and can degrade model performance. Target encoding, hashing, or learned embeddings are more appropriate for high-cardinality features, but each introduces its own tradeoffs — target encoding risks leakage if not done within cross-validation folds, and embeddings require sufficient training data to learn meaningful representations.

---

## 4. Model Selection and Evaluation

Choosing the right model starts with choosing the right metric. Google's Rules of ML, Rule 38: "Don't waste time on new features if unaligned objectives is the issue." If your evaluation metric does not reflect what matters to the business or the user, optimizing it is wasted effort regardless of how sophisticated the model is. Align on the metric with stakeholders before training a single model.

**Select metrics appropriate to the problem type.** For classification: precision, recall, F1-score, and the tradeoff between them depends on the cost of false positives versus false negatives. For regression: RMSE penalizes large errors disproportionately; MAE treats all errors equally. For ranking: AUC-ROC measures discrimination ability across thresholds. Choose the metric that aligns with the real-world cost of errors, not the one that produces the most flattering number.

**Beware of misleading metrics on imbalanced datasets.** Accuracy is nearly meaningless when one class dominates — a classifier that always predicts the majority class achieves high accuracy while being completely useless. For imbalanced problems, precision-recall curves and the area under them (PR-AUC) are more informative than ROC curves. The choice of threshold is itself a decision that should reflect the relative costs of false positives and false negatives in the specific application.

**Understand the bias-variance tradeoff.** Bishop formalizes this fundamental tension: complex models have low bias but high variance (they fit training data well but generalize poorly), while simple models have high bias but low variance (they underfit but are stable). The goal is to find the complexity sweet spot for your data size and noise level. More data generally allows more complexity; noisy data penalizes it.

**Split data with discipline.** Huyen outlines the standard approach: train/validation/test splits where the test set is touched only for final evaluation — never for model selection or hyperparameter tuning. For small datasets, use cross-validation to get more reliable performance estimates. For temporal data, use time-based splits that respect the arrow of time. Stratified splitting ensures class proportions are preserved across splits.

**Evaluate on held-out data and report honestly.** The test set performance is the number that matters. If there is a large gap between validation and test performance, you have overfit to the validation set through repeated model selection. Report confidence intervals or variance across folds, not just point estimates.

**Establish baselines before building complex models.** A baseline — even a trivial one like predicting the majority class or the mean — establishes a floor. If your carefully tuned neural network barely beats a logistic regression, the complexity is not justified. Huyen recommends starting with the simplest model that could work and adding complexity only when the data and metrics justify it. Every increment of complexity must earn its place through measurable improvement on the chosen metric.

**Distinguish between offline and online evaluation.** Offline metrics (computed on held-out data) measure statistical performance. Online metrics (computed in production) measure real-world impact.

A model can have excellent offline metrics but poor online performance due to distribution shift, latency issues, or user behavior that the offline data does not capture. Conversely, a model with modest offline metrics may perform well in production because it handles the common cases correctly. Both evaluation stages are necessary; neither alone is sufficient.

---

## 5. Hyperparameter Tuning

Hyperparameters control model capacity and training dynamics. Tuning them systematically, rather than by intuition, produces better models and — equally important — a documented record of what was tried. Burkov dedicates significant attention to this topic in ML Engineering, emphasizing that undisciplined tuning wastes compute and produces results that cannot be reproduced or explained.

**Choose the search strategy based on the problem.** Burkov outlines the progression: grid search exhaustively covers a predefined set of combinations and works for small parameter spaces (two or three hyperparameters with a handful of values each).

Random search, as shown by Bergstra and Bengio (2012), is more efficient than grid search because it samples more distinct values of each hyperparameter — important when some hyperparameters matter much more than others. Use random search for moderate spaces.

Bayesian optimization builds a surrogate model of the objective function and intelligently selects the next configuration to evaluate. It is appropriate when each evaluation is expensive (large datasets, complex models, long training times) and the number of trials must be kept small.

**Track every combination and result.** Huyen stresses that hyperparameter tuning is experimentation, and all experiments must be logged. Record not just the final metric but training curves, resource usage, and wall-clock time. A configuration that achieves slightly lower accuracy but trains in one-tenth the time may be the better production choice.

**Use early stopping to prevent overfitting.** Monitor validation loss during training and stop when it begins to increase, even if training loss continues to decrease. Early stopping is both a regularization technique and a compute-saving measure. The patience parameter (how many epochs to wait before stopping) is itself a hyperparameter worth tuning. Save model checkpoints at each improvement so that the best model is preserved even if training continues past the optimum.

**Use learning rate schedules to improve convergence.** A fixed learning rate is rarely optimal throughout training. Huyen discusses how learning rate warmup (starting small and increasing), decay schedules (reducing over time), and cyclical learning rates can improve both convergence speed and final performance. The schedule interacts with batch size and optimizer choice, so treat the combination as a unit when tuning.

**Prioritize the most impactful hyperparameters.** For deep learning, the learning rate is typically the single most important hyperparameter. Tuning it well often matters more than tuning all other hyperparameters combined. Batch size, regularization strength, and architecture choices (layer sizes, depth) come next. Don't waste compute tuning parameters with marginal impact.

**Define a computational budget before tuning begins.** Hyperparameter search can consume unbounded compute if left unconstrained. Set a maximum number of trials, a wall-clock time limit, or a compute cost ceiling. Random search and Bayesian optimization both work within budgets naturally — grid search does not, because it requires completing the full grid to be useful. Budget discipline forces you to focus on the hyperparameters that matter most.

---

## 6. From Notebook to Production

The gap between a working notebook and a production ML system is the hardest part of applied ML. Huyen identifies this as the stage where most ML projects stall or fail. A model that works on your laptop with clean data and manual preprocessing is not a production system — it is a proof of concept. The engineering effort required to make it reliable, maintainable, and observable in production often exceeds the effort to build the model in the first place.

**Test the infrastructure independently from the ML.** Google's Rules of ML, Rule 5: verify that data flows correctly, features are computed as expected, and the serving pipeline returns predictions — all before evaluating model quality. Infrastructure bugs masquerade as model performance issues and are far harder to diagnose when entangled with model behavior. Write unit tests for feature computation, integration tests for data pipelines, and end-to-end tests that verify a model can be loaded and produce predictions in the serving environment.

**Treat models as perishable artifacts.** Chen et al. in Reliable Machine Learning argue that ML models degrade as the world changes. User behavior shifts, data distributions drift, upstream data sources change schemas.

A model deployed today may be stale in weeks or months. Plan for this from the start: build monitoring for data drift and model performance degradation, and define retraining triggers before they are needed. Unlike traditional software, which works until someone changes the code, ML models can break without anyone touching them.

**Design the inference pipeline deliberately.** The choice between batch inference (precompute predictions on a schedule) and real-time inference (compute predictions on demand) has profound implications for architecture, latency, cost, and complexity. Batch is simpler and cheaper; real-time is necessary when predictions depend on fresh data. Many systems use both: batch for the common case, real-time for time-sensitive requests. Huyen covers this decision in depth, noting that the infrastructure requirements for real-time serving — low-latency feature retrieval, model optimization for inference speed, autoscaling — are substantially more complex than batch processing and should only be adopted when the use case genuinely requires it.

**Version and serialize models carefully.** A deployed model is an artifact that must be reproducible. Store the trained model alongside its training code version, data version, hyperparameters, and evaluation metrics. When a production model misbehaves, you need to trace back to exactly what was trained, on what data, with what configuration.

**Monitor continuously after deployment.** Track prediction distributions, feature distributions, latency, error rates, and downstream business metrics. A model can silently degrade without any code changes — the world changed, not the model. Automated alerts on distribution shifts and performance drops are essential.

**Define retraining triggers before deployment.** Chen et al. stress that retraining should not be ad hoc. Establish clear criteria: retrain when validation performance drops below a threshold, when data drift exceeds a limit, on a fixed schedule, or when a threshold amount of new labeled data becomes available. Automated retraining pipelines that can be triggered by these criteria reduce the operational burden and prevent models from silently degrading for months.

**Bridge the training-serving skew.** One of the most common production ML bugs is a mismatch between how features are computed during training and during serving. If training uses batch-computed features from a data warehouse but serving computes them in real time from a different data source, subtle differences in joins, aggregation windows, or null handling will cause the model to see different inputs than it was trained on. Huyen identifies training-serving skew as a primary cause of production ML failures.

---

## 7. Responsible AI and Model Documentation

Every model encodes assumptions about the world, and those assumptions have consequences for the people affected by the model's decisions. Documentation and fairness evaluation are not bureaucratic overhead — they are engineering requirements. A model deployed without documentation or fairness analysis is a risk to users, the organization, and the credibility of the ML team.

**Ship a model card with every model.** Mitchell et al. propose model cards as a standard documentation format. Each model card should specify:

- The model's intended use cases and explicitly out-of-scope uses
- A description of the training data (sources, size, demographics)
- Evaluation metrics broken down across relevant subgroups (age, gender, geography)
- Known limitations and ethical considerations

A model without documentation is a liability. The model card provides the information that downstream consumers need to decide whether the model is appropriate for their use case.

**Evaluate fairness across subgroups.** Huyen covers key fairness metrics: demographic parity (equal prediction rates across groups), equalized odds (equal true positive and false positive rates across groups), and calibration (predicted probabilities match observed frequencies within each group). No single metric captures all dimensions of fairness — the choice depends on the specific application and its potential for harm. Critically, some fairness metrics are mathematically incompatible with each other — you cannot simultaneously achieve demographic parity, equalized odds, and calibration when base rates differ across groups. This means fairness is a deliberate design choice, not a box to check.

**Invest in interpretability.** When a model's decisions affect people, the ability to explain why a particular prediction was made is not a luxury. Feature importance scores, SHAP values, and partial dependence plots provide different lenses into model behavior. Interpretability also helps debug the model: unexpected feature importances often reveal data leakage or labeling errors. Huyen distinguishes between global interpretability (understanding the model's overall behavior patterns) and local interpretability (explaining a specific prediction). Both are valuable, but local explanations are typically what users and auditors require.

**Be careful about transfer assumptions.** Google's Rules of ML, Rule 43: "Your friends tend to be the same across different products. Your interests tend not to be." Models trained in one context do not automatically transfer to another. Populations differ, behaviors differ, and the cost of errors differs. Validate explicitly before assuming that a model's performance generalizes to a new domain, population, or use case.

**Document failure modes and known limitations.** Every model has boundaries where its predictions become unreliable — rare subgroups underrepresented in training data, out-of-distribution inputs, adversarial examples. The model card should explicitly state these limitations so that downstream consumers know when to trust the model and when to fall back to human judgment or alternative systems. Mitchell et al. argue that documenting limitations is as important as documenting capabilities.

---

## Applying This Skill

When starting a new ML project:
1. Set up experiment tracking infrastructure before writing any model code
2. Validate data quality — schema, distributions, completeness, labeling consistency
3. Start with a simple baseline model and a well-chosen evaluation metric
4. Engineer features based on domain knowledge before increasing model complexity
5. Track every experiment with full reproducibility metadata

When training and evaluating models:
1. Split data with discipline — respect temporal ordering, stratify classes, isolate the test set
2. Tune hyperparameters systematically — random search or Bayesian optimization, not manual guessing
3. Use early stopping to prevent overfitting
4. Evaluate on held-out data and report metrics with variance, not just point estimates
5. Document the model with a model card before declaring it ready for deployment

When moving to production:
1. Test infrastructure independently from model quality
2. Version the model artifact alongside its code, data, and configuration
3. Monitor data drift, prediction distributions, and downstream business metrics
4. Define retraining triggers and automate the retraining pipeline
5. Evaluate fairness across subgroups and document limitations explicitly

When the target language is known, read the corresponding file in `references/` for language-specific frameworks and idioms.
