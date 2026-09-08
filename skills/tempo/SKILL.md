---
name: tempo-traceql-mcp
license: Apache-2.0
description: >
  Query Grafana Tempo via TraceQL to investigate distributed traces —
  find slow or errored requests, inspect a specific trace, discover
  available attributes, and compute trace-derived metrics (error rate,
  latency percentiles). Use when a task needs trace-level evidence: which
  service in a call chain was slow, whether an error propagated from a
  downstream call, or what a specific trace ID actually did.
metadata:
  author: adapted from grafana/tempo skill (deployment/ops content removed)
  version: 0.1.0-mcp
  universal: true
compatibility: Query-only. Assumes Tempo is already deployed and receiving
  data — this skill has no deployment, ingestion-debugging, or
  multi-tenancy configuration capability. See "Known gaps" below.
---

# Tempo TraceQL (via MCP tools)

Investigate distributed traces by writing and running TraceQL queries.
This skill is scoped to **querying only** — deploying Tempo, configuring
ingestion (OTLP/Jaeger/Zipkin), sizing components, or debugging "no traces
showing" at the ingestion level are all out of scope here; those require
infrastructure access this agent doesn't have.

## Available tools (read-only)

| Tool | Use for |
|---|---|
| `docs-traceql` | Look up TraceQL syntax when unsure — best for retrieval of traces; covers attributes through aggregates, pipelining, structural queries |
| `get-attribute-names` | Discover which attribute names actually exist before writing a query — don't guess |
| `get-attribute-values` | Discover values for a fully-scoped attribute (e.g. all values of `resource.service.name`) |
| `traceql-search` | Search for traces matching a TraceQL filter expression |
| `get-trace` | Retrieve one specific trace by ID, for deep inspection |
| `traceql-metrics-instant` | One metric value at the current instant/end — use for most metric questions |
| `traceql-metrics-range` | A metric series from start to end — use only when a time series (not a single value) is actually needed |

### Known gaps vs. a full Tempo deployment

- **No deployment/ops tools.** Nothing here can install, configure, or
  resize Tempo, nor read component health (`/ready`), ingester pressure
  metrics, or 429 rate-limit errors. If a task asks about any of that,
  say plainly it's outside this agent's capability rather than guessing
  from trace data that doesn't contain that signal.
- **No confirmed multi-tenancy control.** The reference skill's
  `X-Scope-OrgID` header requirement for multi-tenant Tempo isn't
  something these tools expose control over. If queries return
  unexpectedly empty and multi-tenancy is in play, note that as a
  possible explanation, but you cannot verify or fix it from here.
- **No synthetic trace injection.** Can't send a test span to verify
  ingestion end-to-end — this skill only reads what's already there.

## Attribute scopes (for building filters)

```traceql
span.http.status_code        # span-level
resource.service.name        # resource-level (from SDK)
event.name                   # event-level
name                          # intrinsic: span operation name
status                        # intrinsic: ok | error | unset
duration                      # intrinsic: span duration
kind                          # intrinsic: server | client | producer | consumer | internal
traceDuration                 # intrinsic: entire trace duration
rootServiceName                # intrinsic: service of the root span
rootName                        # intrinsic: operation name of the root span
```

## Operators

```
=   !=   >   <   >=   <=      # comparison
=~  !~                          # regex (Go RE2)
&&  ||  !                        # logical
```

## Process

1. **Never guess attribute names or values.** Call `get-attribute-names`
   before writing a filter on an attribute you haven't confirmed exists,
   and `get-attribute-values` when you need to know what values a scoped
   attribute actually takes (e.g. confirm the exact service name before
   filtering on `resource.service.name`). Attribute naming varies by
   instrumentation — don't assume OpenTelemetry semantic convention names
   are all present just because they're common.

2. **Always bound the query by time.** Every `traceql-search` call needs
   an explicit time range appropriate to the incident window — don't
   search unbounded "just in case." An unbounded or overly wide search
   both wastes context on irrelevant traces and increases the chance of
   mixing in unrelated incidents.

3. **Choose the right tool for the question:**
   - "find traces where X happened" → `traceql-search` with a filter
     expression
   - "what exactly did trace ID Y do" → `get-trace`
   - "what's the current error rate / p99 for this service" → a single
     number is enough → `traceql-metrics-instant`
   - "how did error rate/latency change over the incident window" → a
     series is needed → `traceql-metrics-range`
   - Unsure of correct TraceQL syntax for a specific construct → consult
     `docs-traceql` rather than guessing at pipeline/aggregate syntax

4. **Use structural operators for root-cause patterns, not just flat
   filters.** A flat filter like `{ status = error }` only tells you an
   error existed somewhere in a trace — it doesn't tell you where it
   originated or how it propagated. Structural operators answer that:
   - `{ kind = server } >> { status = error }` — a server span that has a
     downstream error (error propagated from something this span called)
   - `{ span.db.system = "redis" } && { span.db.system = "postgresql" }`
     — both conditions present anywhere in the same trace
   Prefer these over separate flat queries plus manual correlation when
   the question is genuinely about *relationship between spans*, not just
   presence of a condition.

5. **Use `with (most_recent=true)` when the question wants "the current/
   latest state"** rather than an arbitrary matching trace — without it,
   which matching trace you get back is not guaranteed to be the most
   recent one, which matters when comparing against a live incident.

6. **Selecting fields:** use `| select(...)` to pull back only the
   attributes actually relevant to the question, rather than the full
   span/trace payload — e.g. `{ status = error } | select(span.http.url,
   duration, resource.service.name)`. Same context-budget logic as any
   other investigator in this system: your output is read by an
   orchestrator that also holds output from other investigators in the
   same turn.

## Essential query patterns

```traceql
# All errors
{ status = error }

# Slow requests from a service
{ resource.service.name = "frontend" && duration > 1s }

# HTTP 5xx
{ span.http.status_code >= 500 }

# Count errors per trace (>=2) — traces with multiple error spans
{ status = error } | count() >= 2

# Select fields only
{ status = error } | select(span.http.url, duration, resource.service.name)

# Structural: server span with downstream error
{ kind = server } >> { status = error }

# Both conditions present anywhere in the same trace
{ span.db.system = "redis" } && { span.db.system = "postgresql" }

# Deterministic most-recent match
{ resource.service.name = "api" } with (most_recent=true)
```

## Metrics from traces

```traceql
# Error rate per service — use traceql-metrics-range for a series over the incident window
{ status = error } | rate() by (resource.service.name)

# P99 latency per service
{ kind = server } | quantile_over_time(duration, .99) by (resource.service.name)
```

Use `traceql-metrics-instant` when the task just needs "what is the error
rate right now/at end of window" and `traceql-metrics-range` when the task
needs to see how it changed over the window (e.g. "did latency spike
before or after the deploy").

## Guidelines

- **Confirm before you filter.** `get-attribute-names` /
  `get-attribute-values` first for any attribute you're not already
  certain exists with that exact name.
- **Bound every search by time.** No exceptions — see Process step 2.
- **Prefer structural queries for causal-looking questions.** A flat
  `status = error` filter shows *that* an error existed; `>>` and `&&`
  show *how spans relate*, which is what root-cause questions actually
  need.
- **`attribute != nil`** for existence checks (e.g. "did this span even
  have a `span.http.url` attribute") rather than assuming absence means
  a zero/empty value.
- **Full reference:** for any TraceQL construct not covered above, use
  `docs-traceql` rather than guessing at syntax — TraceQL has enough
  surface area (pipeline stages, aggregate functions, scoping rules)
  that guessing wrong is easy and produces silent misinterpretation, not
  an obvious error.
