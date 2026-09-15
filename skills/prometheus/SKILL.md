---
name: promql-investigation
description: >
  PromQL syntax reference and diagnostic query patterns for READ-ONLY metric
  investigation during Kubernetes incident RCA. Covers the Prometheus data
  model, vector types, vector/range selectors, vector matching across
  differing label sets, aggregations, and common diagnostic patterns (error
  rate, saturation, OOM prediction, throttling). Use when the agent needs to
  construct or interpret a PromQL query to gather or verify evidence.
  Does NOT cover alerting rule authoring, recording rule deployment, SLO/
  error-budget design, or any write/reload operation on Prometheus — this
  agent is strictly read-only and must never suggest applying, reloading, or
  modifying Prometheus configuration.
---

# PromQL Reference for Investigation

## Prometheus Data Model — read this before writing any query

- **Counter** (suffix `_total`): monotonically increasing. NEVER read the raw
  value directly — always wrap in `rate()` or `increase()`. A raw counter
  value tells you nothing about current behavior, only cumulative count since
  process start.
- **Gauge**: value that moves up or down (memory, queue length, temperature).
  Safe to read directly, or use `avg_over_time()` / `max_over_time()` to
  smooth over a window, or `delta()` / `deriv()` to measure change (see
  below).
- **Histogram** (suffixes `_bucket`, `_sum`, `_count`): pre-bucketed
  observations. Use `histogram_quantile()` — never try to average or compare
  raw bucket values directly.
- **Summary**: pre-computed quantiles, already exposed per-quantile. No
  `histogram_quantile()` needed, but quantiles can't be re-aggregated across
  instances (unlike histograms).

**Staleness caution**: Prometheus serves the last known sample for up to 5
minutes if no new sample has arrived. A metric that "looks normal" may be a
stale value, not a live one — if a query result seems inconsistent with other
evidence (e.g. logs showing an active crash), check `up{job="..."}` and the
sample timestamp before trusting the value.

## Vector Types — know which one you're producing before writing the query

- **Instant Vector**: one most-recent sample per time series, at a single
  point in time. Most label-filtered queries without `[range]` return this.
- **Range Vector**: multiple samples per series over a time window, produced
  by the `[duration]` syntax (e.g. `metric[5m]`). Range vectors are not
  directly readable as a value — they must be passed into a function like
  `rate()`, `increase()`, or `avg_over_time()` to reduce back to an instant
  vector or scalar.
- **Scalar**: a single unitless number, with no labels (e.g. the result of
  `count()` with no `by`, or a literal like `100`).
- **String**: rarely used in practice; not needed for investigation queries.

Match your query shape to what you actually need to report: a single current
value → instant vector; a trend over the incident window → range vector
reduced via an appropriate function.

## Instant Vector Selectors

```promql
# By metric name
http_requests_total

# Label filter
http_requests_total{job="api-server"}

# Multiple labels (AND)
http_requests_total{job="api-server", method="GET"}

# Regex
http_requests_total{job=~"api.*", status=~"5.."}

# Negative
http_requests_total{status!="200"}
```

## Range Vectors & Rates

```promql
# Per-second rate over 5 minutes (counters only)
rate(http_requests_total[5m])

# Increase over interval
increase(http_requests_total[1h])

# Instant rate — last two samples only, noisier, use for fast-moving debugging
irate(http_requests_total[5m])

# Offset — compare against a past window
rate(http_requests_total[5m] offset 5m)

# Subquery — aggregate a range-vector function over an even longer window
max_over_time(rate(http_requests_total[5m])[1h:1m])
```

## Additional functions for gauges and stability checks

```promql
# Absolute change of a gauge over a window (NOT for counters — use rate() there)
delta(node_memory_MemAvailable_bytes[1h])

# Number of times a value changed — useful for spotting flapping/restart
# loops on a value that should be stable
changes(kube_pod_status_ready[10m])

# Per-second derivative of a gauge — instantaneous rate of change,
# complements predict_linear() when you need "how fast is this moving
# right now" rather than "when will it cross a threshold"
deriv(node_memory_MemAvailable_bytes[10m])

# Number of counter resets (e.g. process restarted, counter reset to 0)
resets(http_requests_total[1h])
```

Use `delta()`/`deriv()` on gauges, never on counters — a counter's raw value
only increases, so delta/deriv on it reflects process uptime, not meaningful
signal. Use `rate()`/`increase()` on counters instead.

## Combining metrics with different label sets

Comparisons and arithmetic between two metrics only work automatically when
their label sets match exactly. When they don't (a common case: one metric
has a `pod` label, another only has `container` or a different label name
for the same resource), use vector matching:

```promql
# Compare current usage against its configured limit — labels differ
# between the two metrics, so this needs an explicit join
container_memory_working_set_bytes{namespace="prod", pod="checkout-abc"}
  / on(pod, namespace)
  kube_pod_container_resource_limits{resource="memory", namespace="prod", pod="checkout-abc"}

# ignoring() drops a label from the match instead of listing what to keep
rate(http_requests_total[5m])
  / ignoring(status) group_left
  sum without(status) (rate(http_requests_total[5m]))
```

- `on(labels)`: match series using only the listed labels.
- `ignoring(labels)`: match on all labels except the ones listed.
- `group_left` / `group_right`: use when one side has extra labels the other
  doesn't (many-to-one match) — put it on the side with fewer series. Needed
  for the resource-usage-vs-limit pattern above, since limits are often
  reported with fewer/different labels than usage.

Comparison operators (`==`, `!=`, `>`, `<`, `>=`, `<=`) between two instant
vectors follow the same matching rules — a bare `metric_a > metric_b` will
silently drop series that don't have a matching partner on the other side,
which can look like "no problem found" when it's actually a label mismatch.
If a comparison query returns unexpectedly empty, check for a label mismatch
before concluding the condition is false.

## Aggregations

```promql
# Sum by label
sum by (job) (rate(http_requests_total[5m]))

# Average
avg by (instance) (node_cpu_seconds_total)

# Top-K — useful for "which pod is consuming the most X"
topk(5, rate(http_requests_total[5m]))

# Histogram quantiles (p95, p99 latency)
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Count distinct series
count(up{job="api"})
```

## Query Construction — practical guidance

- **Time window sizing**: too short → too few samples, noisy `rate()`; too
  long → real spikes get averaged away. Start at 5m for `rate()`, widen only
  if the scrape interval is coarse.
- **Cardinality awareness**: grouping by a high-cardinality label (e.g. `pod`
  in a cluster with thousands of pods) can produce a very large result set or
  a slow/failing query. Prefer grouping by `job`/`namespace`/`deployment`
  first, then narrow to a specific `pod` once a candidate is identified.
- **Don't over-fetch**: when verifying one specific evidence claim, scope the
  query as tightly as possible (exact label match, narrow time range) rather
  than pulling a broad range and filtering client-side.

## Common Diagnostic Patterns

```promql
# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) * 100

# CPU saturation (%)
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage (bytes)
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# Container memory trend (for correlating with OOMKilled events — use
# working_set, not container_memory_usage_bytes, since working_set is what
# the kernel actually uses to decide an OOM kill; usage_bytes includes
# reclaimable cache and can overstate real pressure)
container_memory_working_set_bytes{pod="..."}

# CPU throttling (periods throttled vs total periods)
rate(container_cpu_cfs_throttled_periods_total[5m])
  / rate(container_cpu_cfs_periods_total[5m])

# Predict resource exhaustion (linear extrapolation, e.g. disk full in 24h)
predict_linear(node_filesystem_free_bytes[6h], 24*3600) < 0

# Service availability ratio
sum(up{job="..."}) / count(up{job="..."})
```

## What this skill does NOT cover (out of scope by design)

- Writing or deploying Prometheus alerting rules or recording rules
- SLO/error-budget design, burn-rate calculations, or multi-window alerting
  — these define future thresholds, not present evidence
- `promtool`/`amtool` validation or reload commands
- Any operation that mutates Prometheus/Alertmanager configuration

If an investigation seems to call for a config change (e.g. "the alert
threshold is wrong"), report that as a *finding* in the diagnosis — do not
attempt to author or apply the fix. That belongs to a separate remediation
workflow with its own approval gate.
