---
name: ab-testing
description: "Apply when designing, running, or analyzing A/B tests and online controlled experiments: hypothesis formation, metric selection, sample size calculation, statistical analysis, and common pitfalls. Trigger on mentions of A/B testing, experiments, statistical significance, conversion rate, or controlled experiments. Language-specific idioms are in references/."
---

# A/B Testing Skill

A/B testing, or online controlled experimentation, is the gold standard for establishing causal relationships between product changes and user outcomes. As Kohavi, Tang, and Xu emphasize in *Trustworthy Online Controlled Experiments*, randomized controlled experiments are the only reliable method for determining whether a change actually caused an observed effect, as opposed to merely correlating with it. Without experimentation, teams resort to intuition, HiPPO (Highest Paid Person's Opinion), or observational analysis riddled with confounders.

Running a trustworthy experiment requires discipline at every stage: forming a falsifiable hypothesis before writing code, selecting metrics that capture what matters, calculating statistical power to avoid wasting time on underpowered tests, ensuring clean randomization, and analyzing results with appropriate statistical rigor. Each of these stages has well-documented failure modes that invalidate results. The principles below distill the essential practices from the experimentation literature, grounded in decades of work at Microsoft, Google, and other organizations that run thousands of experiments per year.

Getting the statistics right is necessary but not sufficient. Kohavi et al. repeatedly stress that organizational culture determines whether experimentation succeeds or fails. Teams must be willing to accept surprising results, kill features that looked promising, and invest in experimentation infrastructure as a first-class capability.

> **References — language-specific idioms:**
>
> - [references/python.md](references/python.md) — Python idioms (statsmodels, scipy.stats, Bayesian libraries)

---

## 1. Hypothesis Formation

**Every experiment begins with a clear, falsifiable hypothesis stated before the experiment runs.** Kohavi et al. prescribe the format: "Changing X will cause metric Y to change by at least Z." The hypothesis must be specific enough to be wrong. "We think the new checkout flow will be better" is not a hypothesis. "Replacing the three-step checkout with a single-page checkout will increase purchase completion rate by at least 2 percentage points" is one.

**Pre-register the hypothesis, primary metric, and analysis plan before the experiment launches.** Document which statistical test will be used, how outliers will be handled, and what segments (if any) will be examined. Changing any of these after observing data is a form of p-hacking — cherry-picking the analysis that produces a favorable result. Kohavi et al. are blunt: post-hoc rationalization of unexpected results is one of the most common ways teams deceive themselves.

**Use two-sided tests by default.** One-sided tests are appropriate only when there is a strong prior reason to believe the effect can only go in one direction. In practice, this is rare. Product changes intended to help can and do make things worse. A two-sided test protects against the scenario where you declare "no significant effect" when the treatment is actually harmful, simply because you only looked for improvement.

**Specify the mechanism — why you believe the change will have the predicted effect.** This matters because when results come back, the mechanism guides interpretation. If the predicted metric moves but through an unexpected mechanism, the result demands further investigation, not celebration. Kohavi et al. note that understanding the mechanism is what separates learning from mere measurement.

**A complete pre-registration document includes:** the hypothesis in structured format, the primary metric, guardrail metrics, the statistical test, the significance level, the minimum detectable effect, the required sample size, the planned duration, and any pre-specified segments for subgroup analysis. This document should be written, reviewed, and locked before the experiment is turned on.

Resist the temptation to form the hypothesis around what you can easily measure. A hypothesis driven by metric availability rather than product understanding often leads to optimizing the wrong thing. Start with the user problem and the proposed solution, then find a metric that captures whether the solution works.

When an experiment produces an unexpected result — positive or negative — the pre-registered hypothesis and mechanism are your primary tools for interpretation. If the OEC moved but not through the predicted mechanism, treat the result as a new hypothesis to be tested in a follow-up experiment, not as a confirmed win.

---

## 2. Metric Selection

**Define a single Overall Evaluation Criterion (OEC) before the experiment starts.** Kohavi et al. define the OEC as the primary metric the experiment is designed to move — the one metric that, if it improves, justifies shipping the change. Having a single primary metric prevents the team from shopping across dozens of metrics for one that happens to be significant.

**Complement the OEC with guardrail metrics and debug metrics.** Guardrail metrics are things that must not degrade: page load latency, crash rate, revenue per user, error rates. If a guardrail metric degrades significantly, the experiment fails regardless of what the OEC shows. A feature that increases clicks but also increases page load time by 200ms may be a net negative for users even if the OEC improves.

**Debug and diagnostic metrics explain the "why."** They help trace the causal chain from the change to the OEC movement. If the OEC improves, which intermediate steps changed? If the OEC did not move, where in the funnel did the expected effect fail to materialize? These metrics are not used for ship/no-ship decisions but for building understanding that informs future experiments.

**Good metrics are measurable, attributable, sensitive, and timely.** Kohavi et al. identify these four properties as essential. Measurable means you can actually compute the metric from logged data. Attributable means changes can be traced to the experiment. Sensitive means the metric responds to real changes in user experience — a metric with high natural variance requires enormous sample sizes to detect real effects. Timely means the metric is available quickly enough to make decisions.

**Use surrogate metrics when the real outcome takes too long to observe.** Click-through rate is a surrogate for engagement; add-to-cart rate is a surrogate for purchase. The validity of surrogates must be established empirically, not assumed. Kohavi et al. warn that optimizing a surrogate that does not actually predict the long-term outcome leads teams astray. Netflix has published extensively on the challenge of connecting short-term metric movements to long-term member satisfaction and retention.

**Beware of composite metrics.** Combining multiple signals into a single OEC score introduces complexity. The weighting of components must be justified and stable across experiments. If the weights change between experiments, results are not comparable. Kohavi et al. describe the tension between metrics that matter most to the business (long-term revenue, user lifetime value) and metrics that are practical for experimentation (session-level engagement, task completion rate).

Metric sensitivity varies enormously. A metric like "revenue per user" may have such high variance that detecting a real 1% improvement requires millions of users, while "purchase completion rate" may be sensitive enough to detect the same underlying improvement with a fraction of the sample. Choosing the OEC is partly a sensitivity decision — the more sensitive the metric, the faster and cheaper the experiment.

Avoid metrics that can be gamed or that create perverse incentives. If the OEC is "time spent on page," the team may inadvertently optimize for confusion (users spending more time because they cannot find what they need) rather than engagement. Kohavi et al. stress that the OEC should align with long-term user value, not just short-term activity.

---

## 3. Sample Size and Statistical Power

**Calculate the required sample size before launching the experiment, not after.** The inputs to a sample size calculation are: the baseline rate of the primary metric, the minimum detectable effect (MDE), the significance level (typically alpha = 0.05), and the desired statistical power (typically 1 - beta = 0.80, meaning an 80% chance of detecting a real effect of the specified size).

**Teams chronically underestimate required sample sizes.** Kohavi et al. and Georgiev both emphasize this. Detecting a 1% relative change in a metric with a 10% baseline rate requires far more users than most teams expect. Doubling the sensitivity (halving the MDE) quadruples the required sample size. This fundamental relationship means that detecting small but meaningful effects requires either very large traffic volumes or variance reduction techniques.

**Use variance reduction to increase sensitivity without more users.** Experimentation platforms at Microsoft and Google invest heavily in techniques such as CUPED (Controlled-experiment Using Pre-Experiment Data), which uses pre-experiment covariates to reduce noise. CUPED can reduce variance by 50% or more for metrics with strong pre-experiment predictors, effectively doubling the sensitivity of the experiment without requiring additional traffic or longer duration.

**Set the MDE based on practical significance, not statistical convenience.** The MDE should reflect the smallest effect that would be practically meaningful. If a 0.5% improvement in conversion rate would not justify the engineering cost of maintaining a feature, there is no point in powering the experiment to detect effects that small. Conversely, if you power only for large effects, you will miss real but modest improvements that compound over time.

**Run for at least one full business cycle.** Even if you accumulate enough users in two days, Kohavi et al. recommend running for at least one week minimum to capture day-of-week effects. User behavior on weekdays differs systematically from weekends. A test that runs Monday through Wednesday may show effects that vanish when weekend traffic is included. Some experiments may need to run for multiple weeks to capture monthly billing cycles or payroll-driven spending patterns.

**Never stop an experiment early because the results "look significant."** This is the peeking problem, discussed further under pitfalls. The sample size was calculated for a reason: the statistical guarantees only hold at the planned sample size and planned duration.

When traffic is limited, consider trade-offs between the traffic allocation ratio and statistical power. A 50/50 split maximizes power for a two-variant experiment, but if you want to limit risk, you can use a 90/10 or 95/5 split at the cost of requiring more total users. Kohavi et al. note that deviating from 50/50 increases required sample size roughly proportionally to how far you deviate.

For multi-variant experiments (A/B/C/D), power requirements increase because the alpha budget must be split across more comparisons. Consider whether the business question truly requires simultaneous testing of many variants or whether sequential two-variant tests would be more efficient.

---

## 4. Randomization and Assignment

**The randomization unit must match the analysis unit.** If you analyze metrics per user, randomize per user. Randomizing per pageview but analyzing per user introduces correlation between observations within the same user, violating the independence assumption of standard statistical tests. Kohavi et al. identify this mismatch as a common source of invalid results.

**Make assignment deterministic and consistent.** The randomization mechanism must produce the same assignment for a given user across sessions and devices — typically implemented as a hash of the user identifier and experiment ID. Non-deterministic assignment creates a mixed experience that dilutes the measured effect and confuses users. Consistent assignment also ensures that analysis is not contaminated by users switching between variants.

**Interference invalidates the standard analysis.** Interference (also called network effects or spillover) occurs when treated users affect control users. On social networks, a user in the treatment group may share content with a user in the control group, contaminating the control. In marketplaces, changing prices for treatment sellers affects what control buyers see. Cunningham covers this extensively in *Causal Inference: The Mixtape* under the Stable Unit Treatment Value Assumption (SUTVA). When SUTVA is violated, standard A/B test analysis produces biased estimates — typically underestimates of the true treatment effect.

**Use specialized designs when interference is present.** Cluster randomization randomizes groups of connected users together. Geo-based experiments randomize by geographic region. Switchback designs alternate treatment and control over time within the same units. Each strategy trades off statistical power for reduced bias from interference. Geo-based experiments have far fewer independent units (regions) than user-level experiments, requiring larger effect sizes to reach significance.

**Always check for Sample Ratio Mismatch (SRM) before analyzing results.** If you intended a 50/50 split but observe 50.3/49.7 with millions of users, the deviation may be statistically significant and indicates a bug in the randomization, logging, or filtering pipeline. Kohavi et al. dedicate significant attention to SRM because it is surprisingly common and completely invalidates results. Common causes include bots that are unevenly distributed, redirects that lose users in one variant, or crash-induced data loss that differs between variants. If SRM is present, do not interpret the metric results — fix the instrumentation first.

When running multiple experiments simultaneously, ensure that assignment is independent across experiments. Kohavi et al. describe layered experimentation systems (as used at Google) where experiments on different features can run simultaneously without interfering with each other, because each experiment's hash uses a different salt. Overlapping experiments on the same feature are more dangerous and require careful coordination.

Triggering — limiting analysis to users who actually encountered the change — can dramatically increase sensitivity. If only 10% of users assigned to a variant visit the affected page, analyzing all assigned users dilutes the effect by 10x. Kohavi et al. recommend trigger-based analysis but warn that triggering must be symmetric: a user should be counted as triggered only based on actions observable in both treatment and control, to avoid introducing selection bias.

---

## 5. Statistical Analysis

**Report confidence intervals, not just p-values, and understand what both actually mean.** A p-value of 0.04 does not mean there is a 96% probability that the treatment works. It means that if the null hypothesis were true (no real difference), there would be a 4% chance of observing an effect this large or larger. This distinction matters enormously. The p-value says nothing about the probability of the hypothesis being true — that requires Bayesian reasoning and a prior.

**Effect size matters as much as statistical significance.** A 95% confidence interval of [0.1%, 3.2%] tells you the effect is statistically significant and gives you a range for the plausible effect size. A confidence interval of [0.01%, 5.0%] is also significant but far less precise — the true effect could be negligible or substantial. Kohavi et al. stress that a statistically significant improvement of 0.001% is real but may not be worth the engineering cost to maintain. Always ask whether the detected effect is large enough to matter for the business.

**Choose the right test for the metric type.** For continuous metrics, use t-tests or Welch's t-test (which does not assume equal variances). For proportions, use z-tests or chi-squared tests. Georgiev provides detailed treatment of the appropriate test for each metric type. When the metric distribution is heavily skewed (as with revenue or session duration), consider using the delta method or bootstrap methods to construct valid confidence intervals, as the Central Limit Theorem converges slowly for skewed distributions with limited sample sizes.

**Correct for multiple comparisons on secondary metrics.** If you test 20 metrics at alpha = 0.05, you expect one false positive even if the treatment has no real effect on anything. Kohavi et al. recommend distinguishing between the primary metric (which gets the full alpha budget) and secondary metrics (which require correction). The Bonferroni correction divides alpha by the number of comparisons — simple but conservative. Controlling the False Discovery Rate (FDR) using the Benjamini-Hochberg procedure is less conservative and often more appropriate when many metrics are tested simultaneously.

**Treat segment-level results as exploratory unless pre-specified.** Segmented analysis — examining results for subgroups such as new vs returning users, mobile vs desktop, or geographic regions — provides valuable insight but compounds the multiple comparisons problem. A result that appears significant in one segment out of twenty may be a false positive. Only segments that were pre-specified in the analysis plan can be tested with the full alpha budget; segments examined post-hoc require correction or should be treated as hypotheses for follow-up experiments.

Watch for heterogeneous treatment effects — cases where the treatment helps some users and hurts others, producing a null aggregate result. Segmented analysis can reveal these patterns, but only if you look at the right segments. Kohavi et al. recommend a standard set of segments (platform, user tenure, geography) that are examined in every experiment, with appropriate multiple comparison corrections applied.

When the experiment population includes extreme outliers (a few users with abnormally high revenue, for example), these outliers can dominate the mean and inflate variance. Winsorization — capping extreme values at a percentile threshold — is a standard technique described by both Kohavi et al. and Georgiev. The winsorization threshold should be pre-specified, not chosen after examining the data.

---

## 6. Bayesian A/B Testing

**Bayesian methods answer the question stakeholders actually ask: "What is the probability that the treatment is better?"** This is what most people incorrectly believe frequentist p-values tell them. Georgiev covers the statistical foundations of Bayesian A/B testing, including prior selection, posterior computation, and decision rules.

**Choose priors deliberately and document the choice.** Informative priors incorporate reliable historical data about the metric's behavior. Weakly informative priors regularize against implausible effect sizes without strong historical commitment. Flat (uninformative) priors are rarely appropriate — they assign equal probability to a 0.01% effect and a 500% effect, which does not reflect reality. The choice of prior should be part of the pre-registered analysis plan, not a post-hoc decision.

**Credible intervals have the interpretation people want.** A 95% credible interval has a direct probabilistic interpretation: there is a 95% probability that the true parameter lies within this interval, given the data and the prior. This is the interpretation most people incorrectly assign to frequentist confidence intervals. For conversion rate experiments, the Beta-Binomial model provides a natural conjugate framework. For revenue and other continuous metrics, Normal-Normal or more flexible models are appropriate.

**Bayesian methods handle optional stopping more naturally, but not for free.** The posterior probability remains valid regardless of when you compute it. However, repeated peeking with a decision threshold (such as "stop when the probability of treatment being better exceeds 95%") still requires calibration to achieve the desired error rates. Georgiev discusses how to set decision thresholds that account for repeated evaluation. The distinction is that Bayesian methods degrade more gracefully under peeking, not that they are immune to it.

**Use loss functions for principled decisions.** Rather than asking "is the treatment statistically significantly better?" ask "what is the expected loss from choosing the wrong variant?" This reframes the decision in terms of business impact rather than statistical abstraction. A treatment that is probably slightly better has low expected loss from choosing it, even if you are not fully certain. Bayesian methods also enable adaptive experimentation — allocating more traffic to the winning variant over time — which reduces the cost of experimentation when one variant is clearly superior or clearly inferior.

The choice between frequentist and Bayesian methods is partly practical. Frequentist methods are simpler to implement, well-understood by regulators and review boards, and have decades of tooling. Bayesian methods require choosing priors and computing posteriors, which adds complexity but provides more intuitive outputs. Many mature experimentation platforms support both, using frequentist methods as the default and Bayesian methods for specific use cases like early stopping or multi-armed bandit allocation.

---

## 7. Common Pitfalls

**The experimentation literature documents a rich taxonomy of ways experiments go wrong.** Kohavi et al. dedicate entire chapters to pitfalls because they are pervasive, subtle, and often invisible to teams that lack experimentation experience.

**Peeking inflates false positive rates far beyond the nominal level.** Checking results before the planned end date and stopping early if they look good is not a minor sin — with daily peeking over a two-week experiment at alpha = 0.05, the true false positive rate can exceed 25%. If you must monitor results during the experiment, use sequential testing methods (such as alpha spending functions or always-valid p-values) that are specifically designed for continuous monitoring while controlling the overall error rate.

**Survivorship bias hides the treatment's true effect.** Analyzing only users who completed a particular action ignores those who abandoned. If the treatment makes the action harder, fewer users complete it, and the survivors may look better on average — not because the treatment helped, but because only the most motivated users persisted. Always analyze on an intent-to-treat basis: include all users who were assigned to a variant, regardless of whether they completed the target action.

**Novelty and primacy effects make short experiments unreliable.** Novelty effects occur when users react to the change itself rather than the feature — a new button color gets clicks because it is different, not because it is better. The effect decays as users habituate. Primacy effects are the inverse — users resist change initially, and the treatment looks worse until they adapt. Kohavi et al. recommend running experiments long enough for these effects to stabilize and examining the metric trend over time within the experiment to detect decay or ramp-up patterns.

**Simpson's paradox reverses conclusions when groups are combined.** A trend that appears in every subgroup can reverse when the groups are combined, typically because the treatment changes the composition of the groups. Always segment results by key dimensions and compare segment-level and aggregate-level conclusions to check for this paradox.

**Instrumentation effects masquerade as real changes.** If the treatment variant loads a new JavaScript library that affects event logging, apparent metric changes may reflect logging differences rather than user behavior changes. Kohavi et al. recommend A/A tests (experiments with no actual treatment) to validate the instrumentation pipeline before running real experiments. Run A/A tests periodically and after any changes to the logging or assignment infrastructure.

**Carryover effects contaminate sequential experiments.** When a user participates in one experiment and then another on the same surface, the first experiment may affect their behavior in the second. This is particularly problematic in overlapping experiments and in ramp-up/ramp-down scenarios. Kohavi et al. recommend isolation between experiments that might interact and wash-out periods between sequential experiments on the same feature.

**Twyman's law: any statistic that appears interesting or different is usually wrong.** Kohavi et al. cite this adage frequently. When an experiment shows a surprisingly large effect — positive or negative — the most likely explanation is a bug in the instrumentation, a data pipeline error, or an SRM, not a genuine breakthrough. Investigate surprising results thoroughly before acting on them.

**Not every decision should be made by A/B test.** As Kohavi et al. put it: "The difference between a data-driven and data-informed organization is that the latter knows when to override the data." Some decisions are strategic, ethical, or too small to measure. Experimentation is a tool, not a religion.

---

## 8. Experimentation Culture

**Most experiments do not produce statistically significant positive results, and that is expected.** At Microsoft, Kohavi et al. report that only about one-third of experiments show a statistically significant improvement. This is not a failure of the experimentation program — it is a success. It means the organization is testing ideas rigorously rather than shipping everything and hoping for the best. The corollary is that the value of experimentation lies not in confirming ideas but in preventing bad ideas from reaching users.

**Review experiment designs before launch, not just results after.** Experiment review boards, as advocated by Kohavi et al., catch flawed hypotheses, inadequate sample sizes, missing guardrail metrics, and instrumentation gaps at the design stage. This is analogous to code review — it is cheaper to catch problems in the design than in the analysis. A review board also enforces consistency in methodology across teams.

**Document every experiment regardless of outcome.** Record the hypothesis, design parameters, results, learnings, and the decision that followed. This documentation builds institutional memory. Without it, teams repeat experiments that have already been run, rediscover known effects, and lose the cumulative knowledge that makes an experimentation program valuable over time. A searchable experiment repository — where anyone can look up what has been tested on a particular feature or metric — accelerates learning across the organization.

**Tolerate negative results or experimentation will die.** Kohavi et al. emphasize that building an experimentation culture requires executive support and tolerance for experiments that fail to show improvement. Teams that are punished for negative results will stop running honest experiments. The value of experimentation comes precisely from its ability to reveal that confident intuitions are wrong. As Kohavi states: "It is difficult to assess the value of an idea without running a controlled experiment."

**Invest in infrastructure to lower the marginal cost of experimentation.** Organizations that run experiments at scale invest in automated sample size calculators, SRM detection, metric pipelines, and experiment dashboards. The marginal cost of running an experiment should be low enough that teams default to testing rather than guessing. Democratizing experimentation — making it accessible to product managers and designers, not just data scientists — multiplies the number of ideas tested and accelerates organizational learning.

**Build an experimentation ramp-up process.** Start new features at low traffic percentages (1-5%) to catch severe regressions before they affect many users, then ramp up to the full experiment allocation. This reduces risk and builds confidence. Kohavi et al. describe this as standard practice at Microsoft and Google — no experiment goes from 0% to 50% in a single step.

**Share results broadly, including failures.** Experiment results that stay within one team are a missed opportunity. A feature that failed in one context may succeed in another, or may save another team from repeating the same test. Regular experiment review meetings — where teams present results, discuss surprises, and debate interpretation — accelerate organizational learning and raise the overall quality of experimental design.

**Maintain a healthy skepticism toward "best practices" that have not been tested in your context.** Kohavi et al. caution that what works at one company may not transfer to another. Traffic patterns, user populations, product maturity, and metric sensitivity all differ. The only way to know whether a practice works in your environment is to test it. This meta-principle — applying experimentation to experimentation itself — is the hallmark of a mature experimentation program.

---

## Applying This Skill

When designing an A/B test, start with the hypothesis. Write it down in the prescribed format before touching any code. Select the OEC and guardrail metrics, then run the power analysis to determine sample size and duration. Verify that the randomization unit matches the analysis unit and check for potential interference effects.

When analyzing results, check for SRM first — if the sample ratio does not match the intended split, stop and fix the instrumentation before looking at any metrics. Then examine confidence intervals and effect sizes alongside p-values. Correct for multiple comparisons on secondary metrics. Look for heterogeneous treatment effects across pre-specified segments.

When interpreting results, distinguish between statistical significance and practical significance. A real but tiny effect may not be worth shipping. A null result does not mean the treatment has no effect — it may mean the experiment was underpowered to detect the true effect size.

Document everything — the hypothesis, design, results, and decision — regardless of whether the experiment succeeded or failed. Treat negative and null results as valuable information, not as failures to be hidden. The most valuable experiments are often those that prevent a bad idea from reaching users.
