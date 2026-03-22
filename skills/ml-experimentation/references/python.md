# Python ML Experimentation Reference

Idiomatic Python patterns for ML experimentation using scikit-learn, MLflow, SHAP, and pandas. Each section maps to a SKILL.md principle with code examples covering data validation through production deployment.

---

## Experiment Tracking and Reproducibility

```python
import mlflow
import numpy as np

SEED = 42
np.random.seed(SEED)  # set seeds everywhere for reproducibility

mlflow.set_experiment("churn-prediction")

with mlflow.start_run(run_name="baseline-logistic"):
    mlflow.log_param("model_type", "LogisticRegression")
    mlflow.log_param("C", 1.0)
    mlflow.log_param("random_seed", SEED)
    mlflow.log_param("dataset_version", "v2.3")  # track data version explicitly
    mlflow.log_metric("roc_auc", 0.91)
    mlflow.log_metric("f1_score", 0.78)
    mlflow.log_artifact("requirements.txt")  # pin software environment
```

## Data Quality Over Model Complexity

```python
import pandas as pd
from sklearn.model_selection import train_test_split

def validate_dataframe(df: pd.DataFrame) -> None:
    """Schema and distribution checks — run before every training job."""
    required = {"age", "tenure", "monthly_charges", "churn"}
    assert not (required - set(df.columns)), f"Missing: {required - set(df.columns)}"
    assert df[list(required)].notnull().all().all(), "Unexpected nulls found"
    assert df["age"].between(0, 120).all(), "Age out of range"
    assert df["churn"].isin([0, 1]).all(), "Churn must be binary"
    if df["churn"].mean() < 0.05 or df["churn"].mean() > 0.95:
        print("WARNING: severe class imbalance detected")

# Always split BEFORE preprocessing to prevent train/test contamination
X_train, X_test, y_train, y_test = train_test_split(
    df.drop("churn", axis=1), df["churn"],
    test_size=0.2, random_state=SEED,
    stratify=df["churn"],  # preserve class proportions
)
```

## Feature Engineering

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import StandardScaler, OneHotEncoder
from sklearn.impute import SimpleImputer
from sklearn.linear_model import LogisticRegression

numeric_features = ["age", "tenure", "monthly_charges"]
categorical_features = ["contract_type", "payment_method"]

# ColumnTransformer routes each feature group to its own pipeline
preprocessor = ColumnTransformer(transformers=[
    ("num", Pipeline([("imputer", SimpleImputer(strategy="median")),
                       ("scaler", StandardScaler())]), numeric_features),
    ("cat", Pipeline([("imputer", SimpleImputer(strategy="most_frequent")),
                       ("encoder", OneHotEncoder(handle_unknown="ignore",
                                                 sparse_output=False))]), categorical_features),
])

full_pipeline = Pipeline([  # full pipeline prevents training-serving skew
    ("preprocessor", preprocessor),
    ("classifier", LogisticRegression(random_state=SEED)),
])
full_pipeline.fit(X_train, y_train)  # fit on training data only
```

## Model Selection and Evaluation

```python
from sklearn.model_selection import cross_val_score
from sklearn.metrics import classification_report, confusion_matrix, roc_auc_score

# Cross-validation for reliable performance estimates
cv_scores = cross_val_score(full_pipeline, X_train, y_train, cv=5, scoring="roc_auc")
print(f"CV ROC-AUC: {cv_scores.mean():.3f} +/- {cv_scores.std():.3f}")

# Final evaluation on held-out test set — run only once
y_pred = full_pipeline.predict(X_test)
y_proba = full_pipeline.predict_proba(X_test)[:, 1]

print(classification_report(y_test, y_pred))       # precision, recall, F1 per class
print(confusion_matrix(y_test, y_pred))             # understand error types
print(f"Test ROC-AUC: {roc_auc_score(y_test, y_proba):.3f}")
```

## Hyperparameter Tuning

```python
from sklearn.model_selection import RandomizedSearchCV
from sklearn.ensemble import RandomForestClassifier
from scipy.stats import randint

# Random search: more efficient than grid search for moderate parameter spaces
rf_pipeline = Pipeline([
    ("preprocessor", preprocessor),
    ("classifier", RandomForestClassifier(random_state=SEED)),
])

search = RandomizedSearchCV(
    rf_pipeline,
    {"classifier__n_estimators": randint(50, 300),
     "classifier__max_depth": randint(3, 20),
     "classifier__min_samples_split": randint(2, 20)},
    n_iter=50, cv=5, scoring="roc_auc",       # budget: 50 trials
    random_state=SEED, n_jobs=-1,
)
search.fit(X_train, y_train)
print(f"Best ROC-AUC: {search.best_score_:.3f}, params: {search.best_params_}")

# Log tuning results to MLflow
with mlflow.start_run(run_name="rf-tuned"):
    for param, value in search.best_params_.items():
        mlflow.log_param(param, value)
    mlflow.log_metric("cv_roc_auc", search.best_score_)
```

## From Notebook to Production

```python
import joblib

# Serialize full pipeline — preprocessor + model stay together
joblib.dump(search.best_estimator_, "model_pipeline.joblib")

def predict_churn(input_data: dict) -> dict:
    """Production inference: load pipeline, predict, return result."""
    pipeline = joblib.load("model_pipeline.joblib")
    df = pd.DataFrame([input_data])  # match training schema
    prob = pipeline.predict_proba(df)[:, 1][0]  # pipeline handles all transforms
    return {"prediction": int(prob >= 0.5), "probability": round(float(prob), 4)}

result = predict_churn({"age": 35, "tenure": 12, "monthly_charges": 65.0,
                        "contract_type": "month-to-month", "payment_method": "credit_card"})
```

## Responsible AI and Model Documentation

```python
import shap

# SHAP: model interpretability — global and local explanations
X_test_transformed = search.best_estimator_.named_steps["preprocessor"].transform(X_test)
explainer = shap.TreeExplainer(search.best_estimator_.named_steps["classifier"])
shap_values = explainer.shap_values(X_test_transformed)

shap.summary_plot(shap_values[1], X_test_transformed)  # global feature importance
shap.force_plot(                                         # single prediction explanation
    explainer.expected_value[1],
    shap_values[1][0], X_test_transformed[0],
)

# Fairness: check for performance disparities across subgroups
def evaluate_subgroup_metrics(y_true, y_pred, groups, group_name):
    for val in groups.unique():
        mask = groups == val
        r = classification_report(y_true[mask], y_pred[mask], output_dict=True)
        print(f"{group_name}={val}: prec={r['1']['precision']:.3f}, rec={r['1']['recall']:.3f}")
```
