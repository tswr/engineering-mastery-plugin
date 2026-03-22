---
name: observability
description: "Apply when adding logging, metrics, tracing, alerting, or debugging production issues. Trigger on mentions of structured logging, distributed tracing, SLIs, SLOs, dashboards, monitoring, or systematic debugging. Not for security logging — use the security skill for that. Language-specific idioms are in references/."
---

# Observability Skill

Observability is the ability to understand the internal state of a system by examining its external outputs. In a distributed system, you cannot step through code with a debugger. You depend entirely on the signals your system emits — logs, metrics, and traces — to understand what happened, why it happened, and what to do about it.

This skill covers the three pillars of observability (logging, metrics, tracing), the operational practices that make them useful (alerting, dashboards, runbooks), and the disciplined process of debugging when things go wrong. The goal is not to instrument everything — it is to instrument the right things so that any question about system behavior can be answered quickly from production data.

**Language-specific idioms and examples are in the `references/` directory.** Read the relevant file when generating code:
- `references/cpp.md` — C++ idioms (spdlog, Prometheus client, OpenTelemetry C++ SDK)
- `references/python.md` — Python idioms (structlog, prometheus_client, OpenTelemetry Python SDK)
- `references/rust.md` — Rust idioms (tracing crate, metrics crate, OpenTelemetry Rust SDK)
- `references/java.md` — Java idioms (SLF4J/Logback, Micrometer, OpenTelemetry Java SDK)

---

## 1. Structured Logging

Logs are the most fundamental pillar of observability. As Sridharan emphasizes, logging is where observability begins — it is the most accessible and most universal form of telemetry. Every request, every significant state change, and every failure should produce a structured log event — a set of key-value pairs, not a free-text string. Free-text logs require regular expressions to parse and break the moment someone changes the message format. Structured events are queryable, indexable, and machine-readable from the start.

**Structured fields.** Every log event should carry a consistent set of baseline fields:

- Timestamp (ISO 8601 with timezone, or Unix epoch with sufficient precision)
- Severity level
- Service name and version
- A human-readable message

Beyond these baseline fields, attach the contextual data that makes the event useful for diagnosis — request ID, user ID, operation name, duration, error details. The key names should follow a consistent naming convention across all services. Inconsistent naming (one service uses `user_id`, another uses `userId`, a third uses `uid`) fractures your ability to query across services.

**Correlation IDs.** Assign a unique request ID at the entry point of every operation and propagate it through every service call, background job, and log statement that request touches. When a user reports a problem, the correlation ID is the thread you pull to unravel the full sequence of events across services. In systems that use distributed tracing, the trace ID serves as the correlation ID — use it in both logs and spans to unify the two signals.

**Log levels.** Use severity levels with precise, agreed-upon semantics:

- **ERROR** — something is broken and a human needs to look at it now. An ERROR log should be rare enough that each one gets investigated.
- **WARN** — the system is degraded but still functioning. A retry succeeded, a fallback was used, a deadline is approaching.
- **INFO** — business-significant events. A request was served, a job completed, a configuration was loaded.
- **DEBUG** — development and troubleshooting detail. Verbose output disabled in production by default.

If every log is INFO, no log is INFO. If ERRORs fire routinely without anyone investigating, the level is wrong.

**Wide events.** Rather than emitting many small log lines per request, prefer a single wide event per unit of work that captures all relevant context in one record. This approach, advocated by Majors et al., makes it possible to query for any combination of fields without joining across multiple log lines. A wide event for an HTTP request might carry the method, path, status code, duration, user ID, region, upstream service calls, cache hit/miss, and error details — all in a single structured record.

**Context propagation in logs.** In concurrent systems, log statements from different requests interleave. Without per-request context, a wall of log output is nearly useless during an incident. Use your language's context mechanism — thread-local storage, context objects, or tracing spans — to automatically attach request-scoped fields to every log statement without passing them explicitly through every function signature.

---

## 2. Metrics and Instrumentation

Metrics are pre-aggregated, time-series numerical measurements of system behavior. Unlike logs, which describe individual events, metrics describe the aggregate — how many, how fast, how full. They are cheap to collect, cheap to store, and the foundation of alerting.

**The four golden signals.** The Google SRE book identifies four signals that every service should measure:

- **Latency** — how long requests take. Distinguish between the latency of successful requests and the latency of failed requests; a fast error is not the same as a fast success.
- **Traffic** — how many requests the system is handling. For a web service, this is typically HTTP requests per second.
- **Errors** — the rate of failed requests, whether explicitly (HTTP 5xx) or implicitly (HTTP 200 with wrong content, or a response that violates an SLO).
- **Saturation** — how full the system's resources are. This is the most predictive signal: rising saturation warns of imminent failure before users are affected.

If you instrument nothing else, instrument these four.

**RED and USE methods.** Two complementary frameworks help you decide what to measure beyond the golden signals. For request-driven services, the RED method (Rate, Errors, Duration) captures the user-facing behavior. For infrastructure resources (CPU, memory, disk, network), the USE method (Utilization, Saturation, Errors), developed by Brendan Gregg, identifies bottlenecks systematically. RED tells you whether users are happy. USE tells you whether the infrastructure can sustain the load. Apply RED to your services and USE to their underlying resources.

**Metric types.** Choose the right type for what you are measuring:

- **Counters** are monotonically increasing values — total requests, total errors, total bytes processed. They only go up (or reset to zero on restart). Rate is computed by taking the difference over time.
- **Gauges** are point-in-time measurements — current queue depth, current memory usage, active connections. They go up and down.
- **Histograms** capture the distribution of values — request latency broken into buckets — and let you compute percentiles (p50, p95, p99) without storing every individual measurement. Prefer histograms over averages for latency: an average can hide a bimodal distribution where most requests are fast but some are catastrophically slow.

**SLIs and SLOs.** A Service Level Indicator (SLI) is a quantitative measure of service behavior, defined as the ratio of good events to total events — for example, the proportion of requests that complete in under 300ms. A Service Level Objective (SLO) is the target threshold for that SLI — for example, 99.9% of requests should be fast. The error budget is 1 minus the SLO: the amount of unreliability you can tolerate. Error budgets turn reliability into a resource that can be spent on feature velocity and reclaimed by investing in stability.

**Metric naming and labels.** Use a consistent naming convention for metrics across all services. Names should describe what is being measured and include the unit — `http_request_duration_seconds`, not `request_time`. Labels (also called tags or dimensions) add queryable metadata — `method`, `status_code`, `endpoint`. Be cautious with label cardinality: a label with unbounded values (like user ID) creates a time series for every distinct value and can overwhelm your metrics backend.

---

## 3. Distributed Tracing

In a monolithic application, a stack trace tells you the call chain. In a distributed system, a request crosses process boundaries and the stack trace ends at each network hop. Distributed tracing reconstructs the full causal chain by linking spans across services.

**Spans and traces.** A trace represents the entire journey of a request through the system. It is composed of spans, each representing a single unit of work — an HTTP handler, a database query, a message queue publish. Each span carries an operation name, start and end timestamps, a set of attributes (key-value metadata), and a status code indicating success or failure.

**Context propagation.** For tracing to work across service boundaries, trace context must be propagated with every outgoing request. The W3C Trace Context standard defines two HTTP headers:

- `traceparent` — carries the trace ID, parent span ID, and trace flags
- `tracestate` — carries vendor-specific trace data

OpenTelemetry provides propagators that inject and extract this context automatically for HTTP, gRPC, and messaging protocols. If you use a non-standard transport, you must propagate context manually — without it, traces break at the boundary and the distributed picture is lost.

**Sampling.** In high-throughput systems, tracing every request is prohibitively expensive. Two strategies exist:

- **Head-based sampling** decides at the entry point whether to trace a request, typically by keeping a fixed percentage. It is simple and predictable but blindly discards interesting traces.
- **Tail-based sampling** waits until the request completes and then decides based on the outcome — keeping traces that errored, exceeded a latency threshold, or matched other criteria. It captures the interesting traces but requires buffering spans in memory until the decision is made.

Always trace 100% of errors regardless of sampling strategy. An error you cannot trace is an error you cannot diagnose.

**Span attributes and events.** Enrich spans with attributes that aid debugging — the SQL query (parameterized, never with literal values), the queue name, the cache key pattern, the response size. Spans can also carry events (sometimes called logs within a span) that mark significant moments during the span's lifetime — a retry attempt, a fallback activation, a cache miss. These events bind diagnostic detail to the causal structure of the trace.

**When tracing adds the most value.** Tracing is most valuable in systems with many services, where latency or errors emerge from the interaction between components rather than from any single component. For a monolithic application, structured logging with correlation IDs often provides sufficient visibility. Invest in tracing when you find yourself unable to answer "where did this request spend its time?" from logs alone.

---

## 4. Alerting

An alert is a demand for human attention. Every alert that fires takes an engineer away from other work, and every false alert erodes trust in the alerting system. Treat alerts as a scarce, high-value resource.

**Alert on symptoms, not causes.** Users care about symptoms: the page is slow, the API returns errors, the job didn't finish. Alert on the user-visible impact — elevated error rate, increased latency, SLO burn rate — not on internal causes like high CPU or a single node failure. Internal causes may or may not affect users. Symptoms always do.

**Target the SLO.** The SLO defines what "good enough" means. An alert should fire when the service is consuming its error budget faster than expected — when the burn rate threatens to exhaust the budget before the end of the window. The Google SRE Workbook describes multi-window, multi-burn-rate alerting: a fast burn rate over a short window catches acute outages, while a slow burn rate over a long window catches chronic degradation. This approach produces alerts that are both sensitive enough to catch real problems and tolerant enough to ignore transient blips.

**Every alert must be actionable.** The Google SRE book states that every alert must require intelligence to handle — a human should need to take a concrete action that mitigates the problem. Apply three tests to every alert:

- If the correct response is "wait and see if it resolves itself," the alert should not exist — it is noise.
- If the correct response is fully automatable, automate it and remove the alert — it is toil.
- If no one can explain what action to take when the alert fires, it is not ready for production.

**Alert fatigue.** If an alert fires regularly and is routinely ignored or suppressed, it is worse than useless — it trains the team to ignore alerts. Delete it, fix the underlying condition, or raise the threshold. The number of alerts that fire in a week should be small enough that each one receives genuine investigation.

**High-cardinality data for debugging.** Alerts tell you *that* something is wrong. Diagnosing *why* requires high-cardinality data — the ability to break down by user ID, endpoint, region, build version, or any combination of attributes. Pre-aggregated metrics power alerts; high-cardinality event data powers the investigation that follows.

---

## 5. Systematic Debugging

Debugging is not an art practiced by wizards. It is a systematic process of forming hypotheses and testing them against evidence. Agans codified this into nine rules that apply to any system, from embedded firmware to distributed services.

**Understand the system.** Before debugging, understand what the system is supposed to do. Read the documentation, the architecture diagrams, the code. You cannot find a deviation from correct behavior if you do not know what correct behavior looks like.

**Make it fail.** Find a reliable reproduction. A bug you cannot reproduce is a bug you cannot verify you have fixed. Narrow the reproduction to the smallest possible input, configuration, or sequence of actions. Agans stresses that intermittent bugs are still reproducible — you may need to control more variables (timing, concurrency, input ordering) to make the failure deterministic.

**Quit thinking and look.** Engineers love forming theories. Resist the temptation to theorize before you have data. Instrument the system, read the logs, examine the traces, check the metrics. Observe what the system is actually doing, not what you think it should be doing.

**Divide and conquer.** When the problem could be anywhere in a long chain, bisect. Check the midpoint. If the data is correct at the midpoint, the bug is downstream; if not, it is upstream. Binary search is as effective for debugging as it is for algorithms. In distributed systems, this means checking data at service boundaries — is the request correct when it leaves service A? Is it correct when it arrives at service B?

**Change one thing at a time.** When testing a hypothesis, change exactly one variable. If you change two things and the problem disappears, you do not know which change fixed it — and you may have introduced a new bug with the other change.

**Keep an audit trail.** Write down what you tried, what you observed, and what you concluded. Debugging sessions can last hours or span multiple engineers. Without a trail, work is repeated and context is lost. The audit trail also feeds into the postmortem — it becomes the timeline of investigation that helps the organization learn from the incident.

**Check the plug.** Before investigating complex hypotheses, verify the simple assumptions. Is the service running? Is it connected to the right database? Is the configuration deployed? Is the DNS resolving correctly? A surprising number of incidents have trivial root causes hidden by the assumption that "the basics are obviously fine."

**Get a fresh view.** When you are stuck, explain the problem to someone else — or to a rubber duck. The act of articulating the problem forces you to restate your assumptions, and stale assumptions are where bugs hide.

**Debugging as hypothesis and experiment.** The Google SRE book frames debugging as a scientific process: observe the symptoms, form a hypothesis about the cause, design a test that would disprove the hypothesis, run the test, and update your understanding based on the result. This disciplined cycle prevents the common failure mode of shotgun debugging — making random changes and hoping something works.

---

## 6. Dashboards and Operational Readiness

Observability data is only useful if it is accessible when you need it. Dashboards, runbooks, and postmortems are the operational scaffolding that turns raw telemetry into reliable incident response.

**Service dashboards.** Every service should have a dashboard displaying the four golden signals:

- Latency distribution (p50, p95, p99)
- Traffic volume (requests per second)
- Error rate (percentage of failed requests)
- Saturation (CPU, memory, queue depth, open file descriptors)

This dashboard is the first thing an on-call engineer opens during an incident. It should load fast and answer the question "is this service healthy?" within seconds. Keep service dashboards focused — a dashboard that requires scrolling through twenty panels to find the relevant one is not an effective incident response tool.

**Dashboards for known-unknowns, exploration for unknown-unknowns.** Dashboards are excellent for monitoring conditions you have anticipated — the known-unknowns. But novel failures produce signals you did not predict. For these unknown-unknowns, you need the ability to query raw, high-cardinality event data interactively — slicing and grouping by arbitrary fields to discover patterns you did not design a dashboard for. Majors et al. emphasize that traditional monitoring asks "is the system broken?" while observability asks "why is the system broken for this specific user, in this specific region, on this specific build?" The latter requires exploratory querying, not static dashboards.

**Runbooks tied to alerts.** Every alert should link to a runbook that describes: what the alert means, how to verify the problem, likely causes, and step-by-step mitigation actions. Runbooks encode institutional knowledge so that any on-call engineer — not just the service author — can respond effectively. A runbook is a living document — update it after every incident where the runbook was insufficient or inaccurate.

**Blameless postmortems.** After every significant incident, conduct a blameless postmortem. The Google SRE book is explicit: focus on systemic causes, not individual blame. A postmortem that blames a person discourages honesty and ensures the systemic cause will produce another incident. The postmortem document should include a timeline of events, user and business impact, root cause analysis, and concrete action items with owners and deadlines. Action items without owners and deadlines are wishes, not commitments — they do not get done.

**Operational readiness reviews.** Before a new service enters production, verify that it meets a baseline of observability: structured logging is in place, the four golden signals are instrumented as metrics, dashboards exist, alerts are configured with SLO targets, runbooks are written for each alert, and an on-call rotation is established. A service that launches without these is a service that will be debugged blind during its first incident.

---

## Applying This Skill

When generating code, apply these defaults automatically:
1. All log statements emit structured key-value events, not free-text strings
2. Every request carries a correlation ID propagated through all downstream calls
3. Services expose the four golden signals as metrics (latency, traffic, errors, saturation)
4. SLIs are defined as ratios of good events to total events
5. Traces propagate context using W3C Trace Context headers via OpenTelemetry
6. Alerts target symptoms and SLO burn rates, not internal causes
7. Every alert links to a runbook with verification and mitigation steps
8. Debugging follows a systematic process: reproduce, observe, bisect, change one thing
9. Dashboards display the four golden signals for every service
10. Postmortems focus on systemic causes and produce concrete action items

When the target language is known, read the corresponding file in `references/` for language-specific idioms and examples.
