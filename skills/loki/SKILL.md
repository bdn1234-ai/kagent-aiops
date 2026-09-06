---
name: loki-investigator
description: >
  LogQL syntax reference and tool-sequencing strategy for READ-ONLY log
  investigation during Kubernetes incident RCA. Covers stream selectors,
  line filters, parsers (json/logfmt/pattern/regexp), label filters, metric
  queries, diagnostic patterns, and the order in which available Loki tools
  should be used together. Use when the agent needs to construct a LogQL
  query or decide which Loki tool to call next. Does NOT cover ingestion/
  push pipelines (Alloy, Promtail), Loki server configuration, or plugin
  management — this agent only queries existing logs, never writes to Loki.
---

# LogQL Reference for Investigation

Loki indexes only metadata (labels), not full log content — a tight label
selector is always cheap; a broad line filter over a huge stream is not.
Always narrow by label first, then filter lines.

## Required parameters that are easy to get wrong

- **`datasourceUid` is never `"default"` or guessed** — it must be the real
  UID of the Loki datasource on this cluster. Call `list_datasources` (or
  read the `datasource.uid` field from a `get_dashboard_panel_queries`
  result) to obtain it before calling any tool that requires it. A wrong
  UID fails loudly ("datasource not found"); do not retry with another
  guessed value — look it up properly instead.
- **Time parameters must be absolute RFC3339, never relative strings.**
  `"now"` or `"now-1h"` will fail with a parse error — these tools do not
  accept Grafana-style relative time syntax. Compute an actual timestamp,
  e.g. `2026-09-06T09:00:00Z` for now and `2026-09-06T08:00:00Z` for one
  hour ago. If the current time is not directly known, use a tool or
  context value that provides it rather than guessing a plausible-looking
  timestamp.

### Worked example — correct vs. incorrect tool call

Question: *"Did app=grafana log any errors in the last hour?"*

**Wrong** (guessed UID, relative time — will fail):
```json
{"datasourceUid": "default", "startRfc3339": "now-1h", "endRfc3339": "now", "labelName": "app"}
```

**Correct** (look up the UID first, compute absolute timestamps):
```
Step 1 → call list_datasources
       → result includes {"uid": "loki-monitoring", "type": "loki", ...}
Step 2 → compute current time as an absolute RFC3339 timestamp, e.g.
         end = "2026-09-06T09:00:00Z", start = "2026-09-06T08:00:00Z"
Step 3 → call list_loki_label_values with:
{"datasourceUid": "loki-monitoring", "labelName": "app",
 "startRfc3339": "2026-09-06T08:00:00Z", "endRfc3339": "2026-09-06T09:00:00Z"}
```
Never skip Step 1 — an unresolved `datasourceUid` or a relative time string
is the most common cause of tool call failure in this environment.

## Tool sequencing strategy (not covered by any single tool's own description)

The available Loki tools are complementary, not interchangeable — use them
in this order rather than jumping straight to a narrow query:

1. **`query_loki_patterns`** — start here on a noisy or unfamiliar stream.
   Give it a PURE stream selector only (e.g. `{app="nginx"}`) — it does not
   accept line filters or aggregations. It returns the dominant log
   structures and their counts, giving a structural overview before you
   commit to a specific filter. Not supported on VictoriaLogs datasources —
   if the target datasource is VictoriaLogs, skip this and use a `| stats`
   pipeline within `query_loki_logs` instead to get a similar overview.
2. **`find_error_pattern_logs`** — use when checking whether current error
   volume is anomalous, not just present. It compares against the last
   day's average automatically, so prefer it over manually computing a
   `rate()` comparison by hand. Note it is a longer-running, asynchronous
   call — do not apply the same short timeout you use for `query_loki_logs`.
3. **`get_dashboard_panel_queries`** — check this early if the affected
   service likely has an existing dashboard. A pre-built panel query is
   often better-scoped (correct labels, sensible thresholds) than a query
   constructed from scratch, and reduces the chance of an agent-authored
   query using a wrong or stale label.
4. **`query_loki_logs`** — use once a specific pattern or hypothesis from
   the steps above needs to be confirmed with actual, verbatim log lines.
5. **`list_loki_label_names` / `list_loki_label_values`** — use whenever a
   label name or value is uncertain, rather than guessing. An incorrect
   guessed label silently returns zero results, which is easy to
   misinterpret as "no evidence" when it actually means "wrong query."
6. **`query_loki_stats`** — use before running a broad, unscoped query to
   check volume/cardinality first, avoiding an expensive scan.

Do not use `analyze_loki_labels`, `search_plugin_information`, or
`suggest_loki_alloy_label_config` — these are cost/administration tools
(label cardinality auditing, plugin discovery, Alloy config generation) and
are out of scope for an investigation agent; the last one in particular
generates a pipeline config change, which this agent must never propose.

## Log Stream Selector (required in every query)

```logql
{app="nginx"}                        # exact match
{app!="nginx"}                       # not equal
{app=~"nginx|apache"}               # regex match
{app!~"debug.*"}                     # regex not match
{app="nginx", env="prod"}           # AND (multiple labels)
```

## Line Filters (put first — cheapest way to narrow down)

```logql
{app="nginx"} |= "error"            # contains string
{app="nginx"} != "info"             # does not contain
{app="nginx"} |~ "error|warn"       # regex match
{app="nginx"} !~ "health.*check"    # regex not match
{app="nginx"} |= `"status":5`       # backtick avoids escaping
```

## Parsers

```logql
# JSON
{app="api"} | json
{app="api"} | json status="http_status", path="request.path"

# Logfmt
{app="api"} | logfmt

# Pattern (positional, _ discards)
{app="nginx"} | pattern `<ip> - - <_> "<method> <uri> <_>" <status> <bytes>`

# Regexp (named capture groups)
{app="nginx"} | regexp `(?P<method>\w+) (?P<path>\S+) HTTP/(?P<version>\S+)`
```

## Label Filters (after parsers)

```logql
{app="api"} | json | status >= 500
{app="api"} | json | status == 200 and method != "OPTIONS"
{app="api"} | logfmt | duration > 1s
{app="api"} | json | level =~ "error|warn"
```

## Line / Label Format (reshape output for readability)

```logql
{app="api"} | json | line_format "{{.method}} {{.path}} -> {{.status}} ({{.duration}})"
{app="api"} | logfmt | label_format severity=level, svc=app
```

## Metric Queries (turn logs into numbers — for correlation with Prometheus evidence)

```logql
# Requests per second
rate({app="nginx"}[5m])

# Total log lines in window
count_over_time({app="nginx"}[1h])

# Returns 1 if no logs in range — useful to detect "went silent"
absent_over_time({app="nginx"}[5m])

# Error rate by service
sum(rate({env="prod"} |= "error" [5m])) by (app)

# Top 5 most active services
topk(5, sum(rate({env="prod"}[5m])) by (app))
```

## Unwrapped Range Aggregations (numeric values extracted from log fields)

```logql
# Average request duration from logfmt
avg_over_time({app="api"} | logfmt | unwrap duration [5m])

# 95th percentile latency
quantile_over_time(0.95, {app="api"} | logfmt | unwrap duration [5m]) by (app)
```

## Diagnostic Patterns

```logql
# Error rate alert query
sum(rate({env="prod"} |= "error" [5m])) by (service)
  / sum(rate({env="prod"}[5m])) by (service)

# Slow requests
{app="api"} | logfmt | duration > 1s | line_format "SLOW: {{.method}} {{.path}} {{.duration}}"

# HTTP 5xx errors with details
{app="nginx"} | pattern `<ip> - - <_> "<method> <uri> <_>" <status> <bytes>` | status >= 500

# Credential leak detection (useful when investigating a config_conflict
# or permission_issue where secrets may have leaked into logs)
{namespace="prod"} |~ `https?://\w+:\w+@`
```

## What this skill does NOT cover (out of scope by design)

- Sending/pushing logs into Loki (Alloy, Promtail, HTTP push API) — this
  agent only reads existing logs, never writes.
- Loki server configuration, retention, or component architecture — not
  needed to construct or interpret a query.
- Label cardinality auditing, plugin catalog search, or generating Alloy
  label-enforcement config — administrative/authoring concerns, not
  investigation (see tool sequencing note above).

If gathering evidence seems to require changing what gets logged (e.g. "we
need more detail here"), report that as a *finding* in the diagnosis — do
not attempt to modify logging pipelines. That is a separate concern outside
this agent's read-only scope.
