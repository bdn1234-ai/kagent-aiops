---
name: promql-investigation
description: >
  PromQL syntax reference and diagnostic query patterns for READ-ONLY metric
  investigation during Kubernetes incident RCA. Covers the Prometheus data
  model, vector/range selectors, aggregations, and common diagnostic patterns
  (error rate, saturation, OOM prediction, throttling). Use when the agent
  needs to construct or interpret a PromQL query to gather or verify evidence.
  Does NOT cover alerting rule authoring, recording rule deployment, or any
  write/reload operation on Prometheus — this agent is strictly read-only and
  must never suggest applying, reloading, or modifying Prometheus configuration.
---

# PromQL Reference for Investigation

## Prometheus Data Model — read this before writing any query

- **Counter** (suffix `_total`): monotonically increasing. NEVER read the raw
  value directly — always wrap in `rate()` or `increase()`. A raw counter
  value tells you nothing about current behavior, only cumulative count since
  process start.
- **Gauge**: value that moves up or down (memory, queue length, temperature).
  Safe to read directly, or use `avg_over_time()` / `max_over_time()` to
  smooth over a window.
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

# Container memory trend (for correlating with OOMKilled events)
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
- `promtool`/`amtool` validation or reload commands
- Any operation that mutates Prometheus/Alertmanager configuration

If an investigation seems to call for a config change (e.g. "the alert
threshold is wrong"), report that as a *finding* in the diagnosis — do not
attempt to author or apply the fix. That belongs to a separate remediation
workflow with its own approval gate.
