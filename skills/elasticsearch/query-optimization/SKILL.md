---
name: elasticsearch-query-optimization-mcp
description: >
  Diagnose slow Elasticsearch Query DSL searches and propose measured fixes,
  using the Elasticsearch MCP server tools (list_indices, get_mappings,
  search). Use when a search is slow, profile output shows an expensive
  clause, exact-match filters sit in scoring context, or leading wildcards
  dominate latency. Ground every recommendation in search profiling — move
  non-scoring clauses to filter context, eliminate leading wildcards, and
  re-profile to confirm improvement.
metadata:
  author: adapted from elastic/agent-skills (elasticsearch-query-optimization, v0.1.0)
  version: 0.1.0-mcp
  universal: true
compatibility: Elasticsearch 8.x or 9.x. Runs through the official Docker MCP
  server `mcp/elasticsearch`, which exposes 5 read-only tools — this skill
  uses `list_indices`, `get_mappings`, and `search`. Relies on the `search`
  tool forwarding a `profile` field inside `query_body` — see "Known gaps"
  before assuming this works on your server build.
---

# Elasticsearch Query DSL Optimization (via MCP tools)

Diagnose why a Query DSL search is slow, identify the dominant cost from the
profile (not guesswork), rewrite the query to remove that cost while
preserving match semantics, and re-measure with profiling enabled.

> **Scope:** Query DSL searches via the `search` tool. This skill does not
> migrate queries to ES|QL — see the separate `elasticsearch-esql-mcp` skill
> for that. It optimizes the existing bool/match/term/wildcard structure
> already in use.
>
> **Ground rule:** Never recommend "add shards" or "scale hardware" as the
> primary fix when the profile names a specific clause (for example
> `WildcardQuery` at ~3.8s). Fix the query first; infrastructure changes
> require evidence the query is already optimal.

## Available tools (read-only)

| Tool | Use for |
|---|---|
| `list_indices(index_pattern)` | Confirm the target index/pattern exists |
| `get_mappings(index)` | Confirm field types before rewriting a clause |
| `search(index, query_body, fields?)` | Run the Query DSL body, with or without profiling |

### Known gaps vs. a full Elasticsearch deployment

This MCP server does **not** expose:
- **Cluster info** (no equivalent of `GET /`). You cannot confirm
  connectivity or deployment type up front the way the original skill does.
  Skip that check and go straight to `list_indices` — if it fails, report
  the tool error rather than guessing at cluster state.
- **`_validate/query` (no equivalent tool).** Step 4c below (validating a
  rewrite's semantics before profiling) has no direct tool call available.
  When semantics are uncertain, skip straight to re-profiling the rewrite
  (step 5) and compare **hit counts**, not just latency — a rewrite that
  returns a different number of hits than the original is a semantic
  regression even if it's faster.
- **A guaranteed `profile: true` passthrough.** The `search` tool's
  `query_body` is documented as accepting "query, size, from, sort, etc." —
  it is not confirmed whether arbitrary top-level keys like `profile` are
  forwarded to Elasticsearch untouched. **Try it first**: call `search` with
  `query_body: { "query": {...}, "profile": true }` and check whether the
  result contains a `profile` object.
  - If yes → proceed exactly as described below.
  - If no (result has no `profile` key, or the tool errors on the extra
    field) → you have no profiling signal at all. Fall back to a coarser
    diagnosis: compare only `took` before/after a rewrite, and reason about
    likely cost from the query structure itself using the "Common profile
    signatures" table in the reference doc (leading wildcard, unscored
    `term` inside `must`, etc.) rather than measured `time_in_nanos`. Say
    explicitly in the report that this is a structural inference, not a
    profiled measurement, since that's a materially weaker claim.

## Process

1. **Confirm the target index.** Call `list_indices` with the pattern named
   in the request (or inferred from context) and pick the index the slow
   query actually targets.

   **Decision:** proceed only when the index is known. **Data needed:**
   index name or pattern, and the slow Query DSL body.

2. **Profile the slow query to find the dominant cost.** Call `search` with
   `query_body` containing the user's query unchanged plus `"profile":
   true` (see the passthrough caveat above). Read `took`, then inspect
   `profile.shards[].searches[].query` if present — sort child collectors
   by `time_in_nanos` and identify the top contributor.

   **Decision:** classify the bottleneck from profile evidence (or, if
   profiling isn't available, from structure alone per the fallback above):
   - **`TermQuery` / `PointRangeQuery` / `MatchNoDocsQuery` inside `must`
     alongside a scoring clause** — exact-match or range filters are being
     scored unnecessarily. Fix: move to `filter` context (step 4a).
   - **`WildcardQuery` with a leading `*`** (for example
     `message:*timeout*`) — cannot use the inverted index; scans terms per
     document. Fix: remove the leading wildcard (step 4b).
   - **`MatchQuery` on a `text` field** — expected scoring cost; optimize
     only if it dominates *after* filter-context fixes.
   - **High `aggregation` time** — separate from query tuning; out of scope
     unless the task is specifically about aggregations.

   **Data needed:** profile tree with `type`, `description`,
   `time_in_nanos`, `breakdown` (especially `next_doc` for wildcards) — or,
   in the no-profiling fallback, just the query structure itself. Quote the
   top contributor verbatim when explaining the diagnosis.

3. **Inspect field mappings before rewriting.** Call `get_mappings` on the
   target index. For every clause you will move or rewrite, confirm the
   field type:
   - **`term` / `terms` / `filter` on exact values** — field must be
     `keyword` (or another non-analyzed type). A `term` on a `text` field is
     a common bug; if types are wrong, say so and suggest the correct
     sub-field (e.g. `service.keyword`) or a mapping change — do not
     silently rewrite around it.
   - **`match` / `match_phrase`** — target a `text` field (analyzed).
   - **`wildcard`** — works on `keyword` or `wildcard` types; a leading `*`
     still forces a scan regardless of type.

   **Decision:** only propose rewrites that match confirmed types. **Data
   needed:** mapping for each field referenced in the query.

4. **Rewrite the query to remove the profiled (or inferred) bottleneck.**

   ### 4a. Move non-scoring clauses from `must` to `filter`

   When exact-match `term`/`terms`/`range` clauses sit in `must` alongside a
   full-text `match` that should drive relevance:
   - Move exact-match clauses into `bool.filter`.
   - Keep only clauses that must affect `_score` in `bool.must` (typically
     the full-text `match`).

   **Why:** filter context skips scoring and participates in the
   filter/bitset cache on repeated queries. **Semantics:** the same
   documents match; only scoring and performance change — state this
   explicitly.

   ```json
   {
     "query": {
       "bool": {
         "filter": [{ "term": { "status": "active" } }, { "term": { "tenant_id": "acme" } }],
         "must": [{ "match": { "description": "wireless keyboard" } }]
       }
     }
   }
   ```

   ### 4b. Eliminate leading wildcards

   When the profile (or structural inspection) shows a leading-wildcard
   clause like `message:*timeout*`, the leading `*` prevents index lookup.
   Choose a fix based on mapping and intent:

   | Intent | Preferred rewrite |
   |---|---|
   | Full-text substring in logs | `match` or `match_phrase` on the analyzed `message` `text` field |
   | Literal substring on keyword | `wildcard`-typed field, or reindex with ngram analyzer |
   | Prefix only (`timeout*`) | `prefix` query on `keyword`, or edge ngram at index time |

   Also move any non-scoring exact match into `filter` — use `term` on the
   keyword field once the mapping confirms it.

   ```json
   {
     "query": {
       "bool": {
         "filter": [{ "term": { "service.keyword": "checkout" } }],
         "must": [{ "match": { "message": "timeout" } }]
       }
     }
   }
   ```

   Adjust field names (`service` vs `service.keyword`) to match the mapping
   from step 3.

   ### 4c. Validate the rewrite (no dedicated tool — see Known gaps)

   There is no `_validate/query` equivalent. When semantics are uncertain
   (e.g. `wildcard` → `match` may include or exclude different tokens),
   don't try to reason it out abstractly — run the rewritten query with
   `search` and compare **hit counts** against the original in step 5,
   rather than assuming equivalence.

   **Decision:** pick the smallest rewrite that addresses the (profiled or
   inferred) cost. **Data needed:** rewritten Query DSL body.

5. **Re-run the rewritten query and compare.** Call `search` again with
   `query_body` containing the rewritten query (plus `"profile": true` if
   the passthrough worked in step 2). Compare `took`, hit count, and — if
   available — the top profile collector against the baseline from step 2.

   **Decision:** report success only when latency dropped materially *and*
   the hit count matches the original (or any difference is explained and
   intentional). If profiling is available and still shows a leading
   wildcard or scored filters, iterate — don't declare victory from `took`
   alone.

   **Data needed:** before/after `took` and hit counts; before/after
   profile summaries if profiling is available.

6. **Report findings in this order.**
   1. **Root cause** — quote the profile if available (e.g. "`WildcardQuery`
      `message:*timeout*` ≈ 3.8s, mostly `next_doc`"), or state clearly that
      this is a structural inference if profiling wasn't available.
   2. **Rewrite** — show the optimized bool structure with filter vs must
      separation.
   3. **Mapping notes** — keyword vs text confirmations from `get_mappings`.
   4. **Measured improvement** — before/after `took`/hit-count, or profile
      if available.
   5. **Semantic caveat** — only if the rewrite could change which
      documents match (e.g. `match` vs substring `wildcard`).

## Guidelines

- **Profile first if you can** — confirm the passthrough works (see Known
  gaps) before relying on `time_in_nanos` numbers in your report.
- **Filter is for equality, must is for relevance.** Status, tenant ID,
  service name, and time ranges rarely belong in `must` when a text query
  drives ranking.
- **Leading wildcards are almost never the right fix for log search.**
  Prefer analyzed `match`/`match_phrase`; reserve `wildcard` for suffix
  patterns (`timeout*`) on keyword or `wildcard`-typed fields.
- **Do not conflate slow with wrong.** A slow query can return correct
  results; optimization preserves the result set unless you explicitly warn
  about a semantic trade-off — and without `_validate/query`, hit-count
  comparison is your only real check for that.
- **Deep reference:** profile collector types, filter-cache behavior, and
  wildcard alternatives — see the reference doc.

## Examples

### Unscored terms in `must`

**Input:** `bool.must` contains `term` on `status`, `term` on `tenant_id`,
and `match` on `description`.

**Diagnosis:** profile (or structure) shows scored `TermQuery` alongside
`MatchQuery`; exact filters don't need scoring.

**Fix:** move both `term` clauses to `filter`; keep `match` in `must`.
Confirm `status` and `tenant_id` are `keyword` via `get_mappings`.

### Leading wildcard dominates latency

**Input:** `wildcard` `message:*timeout*` plus `match` on `service` in
`must`. Profile (if available): `WildcardQuery` ~3.8s.

**Diagnosis:** leading `*` forces term enumeration; not an index/shard
problem.

**Fix:** `match` on analyzed `message`; move `service` to `filter` as `term`
on keyword. Re-run — expect lower `took` and matching (or explained) hit
count.
