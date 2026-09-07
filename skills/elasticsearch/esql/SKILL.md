---
name: elasticsearch-esql-mcp
description: >
  Execute ES|QL (Elasticsearch Query Language) queries via the Elasticsearch MCP
  server tools (list_indices, get_mappings, esql). Use when the log-investigator
  agent needs to query Elasticsearch data, analyze logs, aggregate metrics, or
  explore data to support root cause analysis.
metadata:
  author: adapted from elastic/agent-skills (elasticsearch-esql, v0.7.0)
  version: 0.1.0-mcp
  universal: true
compatibility: Elasticsearch 8.14+ for ES|QL GA. Runs through the official
  Docker MCP server `mcp/elasticsearch` (https://github.com/elastic/mcp-server-elasticsearch),
  which exposes exactly 5 read-only tools — this skill uses 3 of them
  (list_indices, get_mappings, esql).
---

# Elasticsearch ES|QL (via MCP tools)

Execute ES|QL queries against Elasticsearch through the MCP tools available in
this environment: discover the schema, choose the right ES|QL feature for the
task, generate the simplest correct query, and run it.

## Available tools (read-only)

This skill runs entirely through MCP tool calls, not a CLI. Only these tools
exist — do not attempt to call an HTTP API directly or invent tools that
aren't listed:

| Tool | Use for |
|---|---|
| `list_indices(index_pattern)` | Discover which indices exist for a given pattern |
| `get_mappings(index)` | Get field names and types for one index |
| `esql(query)` | Run a complete ES|QL query string and get results back |

All three are read-only and cannot modify the cluster.

### Known gaps vs. a full Elasticsearch deployment

This MCP server does **not** expose:
- Cluster/version info (no equivalent of `GET /`). You cannot auto-detect
  `build_flavor` (Serverless vs. Stack) or the stack version. **If the
  cluster version is known ahead of time, state it directly in this
  agent's configuration** so version-gated features (`LOOKUP JOIN` 8.18+,
  `MATCH` 8.17+, `INLINE STATS` 9.2+) can be used confidently. If it is
  not known, default to the conservative feature set (avoid the features
  above; use `ENRICH` instead of `LOOKUP JOIN`) and only try newer syntax
  if the simpler form doesn't answer the question.
- Index settings (no equivalent of `GET /{index}/_settings/index.mode`).
  You cannot auto-detect whether an index is a time-series (TSDS) index.
  If the relevant indices are known to be TSDS, say so in configuration;
  otherwise, infer from `get_mappings` output (presence of
  `time_series_metric` on fields is a strong signal) and fall back to
  plain `FROM` + `STATS` if unsure — a wrong `TS` guess fails loudly, a
  wrong `FROM` guess on TSDS data just gives less-optimal aggregation.
- License tier (no equivalent of `GET /_license`). `CATEGORIZE` and
  `CHANGE_POINT` require a Platinum license. Try them if they fit the
  question; if the query errors on license grounds, fall back to
  `STATS ... BY field` (for CATEGORIZE-style grouping) or manual
  time-bucketed inspection (for CHANGE_POINT-style trend questions)
  without asking the user first.

## Process

1. **Discover the schema (required — never guess index or field names).**
   Call `list_indices` with a pattern to narrow candidates, then call
   `get_mappings` on the chosen index.

   Index names and field names vary across deployments and cannot be
   reliably guessed. Even common-sounding data (e.g., "logs") may live in
   indices named `logs-test`, `logs-app-*`, or `application_logs`. Field
   names may use ECS dotted notation (`source.ip`, `service.name`) or flat
   custom names — the only way to know is to check via `get_mappings`.

   **Prefer simplicity:** query a single index unless the task explicitly
   needs data across multiple sources. Don't combine indices with
   different schemas using `COALESCE` unless specifically needed — pick
   the single most relevant index.

2. **Choose the right ES|QL feature for the task.** Match the task's
   intent to the most appropriate ES|QL feature. Prefer one advanced query
   over several basic ones.
   - "find patterns," "categorize," "group similar messages" →
     `CATEGORIZE(field)` (Platinum — see fallback above)
   - "spike," "dip," "anomaly," "when did X change" → `CHANGE_POINT value
     ON key` (Platinum — see fallback above)
   - "trend over time" → `STATS ... BY BUCKET(@timestamp, interval)`, or
     `TS ... BY TBUCKET(interval)` if the index is confirmed/likely TSDS
   - "search," "find documents matching" → `MATCH` (default full-text
     search), `QSTR` (advanced boolean), `KQL` only if confirmed 8.18+
   - "count," "average," "breakdown" → `STATS` with aggregation functions
   - Do **not** reach for the `PROMQL` source command here — if this
     deployment also has a dedicated Prometheus investigator agent, metric
     queries belong there; keep this skill scoped to log/document data in
     Elasticsearch to avoid two agents answering the same metric question
     differently.

3. **Generate the query.** Prefer the simplest query that answers the
   question — don't add extra indices, fields, or transformations unless
   asked. Only include fields in `KEEP` that directly answer the question.
   Don't add filter conditions beyond what was asked (e.g. don't add
   `OR level == "ERROR"` when the task just said "errors").
   - Start with `FROM index-pattern` (or `TS index-pattern` only if TSDS
     is confirmed or strongly indicated — see gap above)
   - Add `WHERE` for filtering
   - Use `EVAL` for computed fields
   - Use `STATS ... BY` for aggregations
   - Add `SORT` and `LIMIT` as needed

   **Context budget — always apply, this is not optional:** this agent's
   output feeds into an orchestrator that also collects results from other
   investigators in the same turn. Never return full `_source` documents.
   Always narrow to a small `KEEP` list of only the fields the question
   needs. A reasonable default `KEEP` for log-shaped data: `message,
   error.message, service.name, container.name, host.name,
   kubernetes.pod.name, kubernetes.namespace, @timestamp`. Cap sample rows
   at 10–100 with `LIMIT` unless the task explicitly needs more.

   `log.level` filtering can be unreliable (missing or mis-set values) —
   prefer matching on `message`/`error.message` content. Plain keyword
   search for "error"/"fail" is often flawed (matches "no error", "error
   code 0") — scope by entity (`service.name`, `kubernetes.pod.name`, …)
   first, then narrow by real message patterns.

4. **Execute the query** by calling `esql(query)`. Read the returned rows
   directly — there is no separate "format" option to request, so keep
   the query itself narrow (per the `KEEP`/`LIMIT` guidance above) rather
   than relying on output formatting to control size.

## ES|QL Quick Reference

### Basic structure

```esql
FROM index-pattern
| WHERE condition
| EVAL new_field = expression
| STATS aggregation BY grouping
| SORT field DESC
| LIMIT n
```

### Common patterns

**Filter and limit:**

```esql
FROM logs-*
| WHERE @timestamp > NOW() - 24 hours AND level == "error"
| KEEP @timestamp, message, service.name, kubernetes.pod.name
| SORT @timestamp DESC
| LIMIT 50
```

**Top N with count:**

```esql
FROM web-logs
| STATS count = COUNT(*) BY response.status_code
| SORT count DESC
| LIMIT 10
```

**Text search:** `MATCH` is the default for full-text search — faster than
`LIKE`/`RLIKE` and supports relevance scoring. Don't add redundant keyword
equality filters alongside `MATCH` unless explicitly needed. The first
argument to `MATCH` must be one real field name (not a comma-list of
fields) — combine fields with `MATCH(a, "q") OR MATCH(b, "q")`.

```esql
FROM documents METADATA _score
| WHERE MATCH(content, "connection refused")
| KEEP @timestamp, message, service.name, _score
| SORT _score DESC
| LIMIT 20
```

**String extraction:** `DISSECT` for structured delimiter patterns
(preferred — named fields), `GROK` for regex-based extraction. `SPLIT`
returns a multivalue — use `MV_FIRST`/`MV_LAST`/`MV_SLICE`. There is no
`REGEXP_EXTRACT` — use `GROK`. There is no `INSTR`/`STRPOS` — use `LOCATE`.

```esql
FROM logs-*
| DISSECT message "%{method} %{path} %{status_text}"
| KEEP @timestamp, method, path, status_text
| LIMIT 50
```

**Log categorization (Platinum — see fallback in Known gaps):**

```esql
FROM logs-*
| WHERE @timestamp > NOW() - 24 hours
| STATS count = COUNT(*) BY category = CATEGORIZE(message)
| SORT count DESC
| LIMIT 20
```

**Change point / spike detection (Platinum — see fallback):**

```esql
FROM logs-*
| STATS c = COUNT(*) BY t = BUCKET(@timestamp, 30 seconds)
| SORT t
| CHANGE_POINT c ON t
| WHERE type IS NOT NULL
```

**Data enrichment with LOOKUP JOIN (8.18+ only — confirm version first):**
match by field name (`LOOKUP JOIN idx ON field_name`); `RENAME` first if
the join key has a different name in the source. Sort/limit *before*
`LOOKUP JOIN` for top-N questions to reduce enrichment cost.

```esql
FROM orders
| STATS total_spent = SUM(total) BY customer_id
| SORT total_spent DESC
| LIMIT 3
| LOOKUP JOIN customers_lookup ON customer_id
| KEEP name, customer_id, total_spent
```

**Multivalue field filtering:**

```esql
FROM employees
| WHERE MV_CONTAINS(languages, "Python")
```

## Error handling

Read the error from the `esql` tool result and correct the query — don't
guess blindly:

- **Field doesn't exist** → re-check via `get_mappings`; never guess field
  or index names, they vary across deployments.
- **Type mismatch** → use `TO_STRING`, `TO_INTEGER`, etc.
- **Syntax error** → double quotes for strings, never single quotes. Use
  underscored function names (`STD_DEV()` not `STDDEV()`). Use `CONCAT()`
  for strings, not `+`. Use `CASE(cond, val, ...)`, not
  `CASE WHEN...THEN...END`.
- **No results** → check the time range and filter conditions before
  assuming there's nothing to find. Report `no_logs_available` rather
  than concluding "no errors occurred" — an empty result is not evidence
  of absence.
- **License error on `CATEGORIZE`/`CHANGE_POINT`** → fall back per "Known
  gaps" above, don't retry the same query.
- **Version-gated syntax error** (`LOOKUP JOIN`, `INLINE STATS`, …) → fall
  back to the pre-8.18/9.2 equivalent (e.g. `ENRICH` instead of
  `LOOKUP JOIN`) rather than repeating the same failing query.

## Guidelines

- **Inspect before querying.** Always call `get_mappings` before writing a
  query against an index you haven't already checked this session.
- **Filter early.** Put `WHERE` before `STATS` so aggregation runs over
  the smallest row set.
- **Always bound results.** Every query ends with `LIMIT`, and every query
  returning documents (not just aggregates) uses `KEEP` to narrow fields —
  see the Context budget note in step 3.
- **Quote correctly.** Double quotes for string literals, never single.
- **Don't guess cluster capabilities.** Version and license aren't
  queryable through these tools — rely on configuration if provided, and
  fall back gracefully on error otherwise (see Known gaps and Error
  handling above).
- **Stay in scope.** This skill answers questions from log/document data
  in Elasticsearch. Metric/Prometheus-shaped questions belong to the
  Prometheus investigator, not here.
