# Python A/B Testing Reference

Idiomatic Python patterns for designing, powering, and analyzing A/B tests. Uses scipy, statsmodels, and pandas — the standard stack for experimentation analysis in Python.

---

## Sample Size and Statistical Power

```python
import math
from statsmodels.stats.power import NormalIndPower, TTestIndPower
from statsmodels.stats.proportion import proportion_effectsize

# --- Proportion metric (e.g., conversion rate) ---
power_analysis = NormalIndPower()
baseline, mde = 0.10, 0.02  # 10% baseline, 2pp absolute MDE
effect_size = proportion_effectsize(baseline, baseline + mde)  # Cohen's h

n_per_group = math.ceil(power_analysis.solve_power(
    effect_size=effect_size, alpha=0.05, power=0.80, alternative="two-sided",
))

# --- Continuous metric (e.g., revenue per user) ---
tt_power = TTestIndPower()
# effect_size = Cohen's d = expected_diff / pooled_std
n_continuous = math.ceil(tt_power.solve_power(
    effect_size=0.05, alpha=0.05, power=0.80, alternative="two-sided",
))
```

## Metric Selection

```python
import pandas as pd
from scipy.stats import mstats

# df has columns: user_id, variant, converted (0/1), revenue, page_load_ms
# Conversion rate per variant
metrics = df.groupby("variant").agg(
    users=("user_id", "nunique"),
    conversions=("converted", "sum"),
).assign(conversion_rate=lambda x: x["conversions"] / x["users"])

# Continuous metric summary per variant
revenue_metrics = df.groupby("variant")["revenue"].agg(["mean", "std", "count"])

# Winsorize extreme values before analysis (pre-specified threshold)
df["revenue_winsorized"] = mstats.winsorize(df["revenue"], limits=[0, 0.01])
```

## Statistical Analysis

```python
import numpy as np
from scipy import stats
from statsmodels.stats.proportion import proportions_ztest

control = df[df["variant"] == "control"]
treatment = df[df["variant"] == "treatment"]

# --- Proportion z-test (conversion rates) ---
count = [treatment["converted"].sum(), control["converted"].sum()]
nobs = [len(treatment), len(control)]
z_stat, p_value = proportions_ztest(count, nobs, alternative="two-sided")

# Chi-squared alternative (equivalent for 2x2 table)
table = pd.crosstab(df["variant"], df["converted"])
chi2, p_val, dof, expected = stats.chi2_contingency(table)

# --- Welch's t-test (continuous metrics, unequal variances) ---
t_stat, p_value = stats.ttest_ind(
    treatment["revenue"], control["revenue"], equal_var=False,
)

# --- Confidence intervals ---
# Difference in means
diff = treatment["revenue"].mean() - control["revenue"].mean()
se = np.sqrt(
    treatment["revenue"].var() / len(treatment)
    + control["revenue"].var() / len(control)
)
ci_means = (diff - 1.96 * se, diff + 1.96 * se)

# Difference in proportions
p_t, p_c = treatment["converted"].mean(), control["converted"].mean()
se_prop = np.sqrt(p_t * (1 - p_t) / len(treatment) + p_c * (1 - p_c) / len(control))
ci_prop = (p_t - p_c - 1.96 * se_prop, p_t - p_c + 1.96 * se_prop)
```

## Randomization and Assignment

```python
import hashlib
from scipy.stats import chisquare

def assign_variant(user_id: str, experiment_id: str, num_variants: int = 2) -> int:
    """Deterministic assignment via hashing. Same user+experiment -> same variant."""
    key = f"{experiment_id}:{user_id}"
    hash_val = int(hashlib.sha256(key.encode()).hexdigest(), 16)
    return hash_val % num_variants  # 0 = control, 1+ = treatment

# Sample Ratio Mismatch (SRM) check — always run before analyzing metrics
observed = [len(control), len(treatment)]
expected_counts = [sum(observed) * 0.5, sum(observed) * 0.5]
chi2_srm, p_srm = chisquare(observed, f_exp=expected_counts)
if p_srm < 0.001:
    raise ValueError(f"SRM detected (p={p_srm:.6f}). Fix instrumentation before analysis.")
```

## Common Pitfalls

```python
from statsmodels.stats.multitest import multipletests

# Multiple comparisons correction on secondary metrics
p_values = [0.03, 0.12, 0.045, 0.001, 0.78]
metric_names = ["conversion", "revenue", "latency", "clicks", "bounce_rate"]

# Benjamini-Hochberg FDR correction (less conservative than Bonferroni)
rejected, corrected_p, _, _ = multipletests(p_values, method="fdr_bh", alpha=0.05)

# Bonferroni correction (simple, conservative)
rejected_bonf, corrected_bonf, _, _ = multipletests(p_values, method="bonferroni", alpha=0.05)

for name, raw_p, adj_p, sig in zip(metric_names, p_values, corrected_p, rejected):
    status = "SIGNIFICANT" if sig else "not significant"
    print(f"{name}: raw p={raw_p:.4f}, adjusted p={adj_p:.4f} — {status}")
```

## Bayesian A/B Testing

```python
from scipy.stats import beta as beta_dist
import numpy as np

# Beta-Binomial model for conversion rates
# Prior: Beta(1, 1) = uniform (weakly informative)
prior_a, prior_b = 1, 1
control_successes, control_trials = 500, 5000
treatment_successes, treatment_trials = 540, 5000

# Posterior = Beta(prior_a + successes, prior_b + failures)
post_ctrl = beta_dist(prior_a + control_successes, prior_b + control_trials - control_successes)
post_treat = beta_dist(prior_a + treatment_successes, prior_b + treatment_trials - treatment_successes)

# Monte Carlo: P(treatment > control) and expected loss
n_samples = 100_000
s_ctrl = post_ctrl.rvs(n_samples)
s_treat = post_treat.rvs(n_samples)
prob_treat_better = (s_treat > s_ctrl).mean()

# Expected loss from choosing each variant
loss_if_treatment = np.maximum(s_ctrl - s_treat, 0).mean()
loss_if_control = np.maximum(s_treat - s_ctrl, 0).mean()

# 95% credible interval for lift — direct probabilistic interpretation
lift_samples = s_treat - s_ctrl
ci_95 = np.percentile(lift_samples, [2.5, 97.5])
```
