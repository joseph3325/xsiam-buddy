---
name: xsiam-xql
description: >
  This skill should be used when the user asks to "write an XQL query",
  "create an XQL query", "threat hunting query", "XQL", "query dataset",
  "search XSIAM data", "hunt for", "investigate in XSIAM", or needs to
  build standalone XQL queries for threat hunting, investigation, analytics,
  or data enrichment.
version: 1.0.0
---

# XQL Query Generation

## Scope

Generate standalone XQL queries for Cortex XSIAM.

**This skill handles:**
- Threat hunting queries
- Investigation queries
- Analytics and reporting queries
- Data enrichment queries (joins, lookups)

**This skill does NOT handle:**
- Correlation rules with YAML wrappers → use `xsiam-correlations`
- Translating Splunk SPL to XQL → use `xsiam-splunk-to-xql`
- Writing automation scripts → use `xsiam-scripts`
- Building integrations → use `xsiam-integrations`

## Before Starting

Read these reference files before generating any query:

**Always read (core):**
1. `../xsiam-shared/references/xql-core-reference.md` — All 23 stages, 60 common functions, operators, arrow notation, best practices
2. `../xsiam-shared/references/xql-datasets-core.md` — Key datasets, presets, field conventions, join patterns

**Read on-demand (when the query needs it):**
3. `../xsiam-shared/references/xql-advanced-functions.md` — Read when the query involves:
   - Array manipulation (arraymap, arrayfilter, arrayexpand + join chains)
   - Complex JSON parsing (json_extract_array, json_path_extract, object_create)
   - Window functions (ranking, row_number, lag, running aggregates)
   - URL parsing or advanced IP operations
4. `../xsiam-shared/references/xql-datasets-extended.md` — Read when querying:
   - Third-party alert sources (Abnormal Security, Canary, Darktrace, Google Workspace)
   - Email datasets (Office 365, email gateway)
   - Cloud Identity Engine (pan_dss_raw) for OU/group enrichment
   - Custom HTTP collector data (xdr_http_collector)
   - Cold storage (cold_dataset)
5. `../xsiam-shared/references/xql-federated-search.md` — Read when querying:
   - External data in S3, GCS, or Azure Blob Storage
6. `references/xql-examples.md` — Production query cookbook organized by use case

## Workflow

### Step 1 — Classify Intent

Determine the query type:
- **Threat hunting**: Proactive search for indicators or suspicious patterns
- **Investigation**: Drill into a specific alert, incident, or entity
- **Analytics**: Aggregate data for reporting, dashboards, or trend analysis
- **Enrichment**: Join or correlate data across multiple sources

### Step 2 — Select Dataset

Use the dataset selection guide in xql-datasets-core.md to pick the right data source.
Consider: preset (broad analysis), dataset (specific source), or datamodel (XDM-normalized).

### Step 3 — Build the Query

Construct multi-stage XQL following the recommended stage order:

`dataset/preset/datamodel` → `filter` → `alter` → `comp` → `fields` → `sort` → `limit` → `dedup`

Key principles:
- Filter early to reduce data volume
- Use `fields` early when joining wide datasets
- Place `comp` after `filter` and `alter` for clean aggregation
- `fields` can appear twice: once early for reduction, once late for final output

### Step 4 — Apply Construction Rules

**Filtering:**
- Use `_time` for time-based filtering
- `=` string comparisons are case-sensitive; use `lowercase()` in `alter` before comparing with `=` for case-insensitive exact matching
- `contains` is case-insensitive by default — no need for `lowercase()` before `contains`
- Use `incidr()` for IP range matching
- Use `~=` for regex matching
- Chain conditions with `and` / `or`

**Aggregation:**
- `comp count() by field` — count by group
- `comp count_distinct(field) by group_field` — unique counts
- `comp values(field) by group_field` — list unique values
- `comp sum(numeric_field) by group_field` — totals
- Always include a `by` clause unless you want a single global aggregate

**Joins:**
- Use `join` to correlate events across datasets
- Specify join type: `inner`, `left`, `right`, `cross`
- Always alias the subquery: `as alias`
- Join on matching fields: `alias.field = field`

**Time bucketing:**
- `bin _time span = 1h` for hourly buckets
- Combine with `comp count() by _time` for time-series data

**Non-default time windows:**
- Use `config timeframe = Nd` preamble before the dataset stage
- Must be: `config timeframe = 7d` then `| dataset = ...`

### Step 5 — Format Output

Return XQL in a fenced code block with mandatory header comments:

```xql
// Title: <descriptive title>
// Description: <what the query does and why>
// Author: <author>
// Datasets: <dataset(s) used>
// Modified: <YYYY-MM-DD>
```

No prose description, expected output section, or customization notes. The header comments carry that context.

## Quality Checklist

Before delivering the query, verify:

1. Header comments are complete (all 5 fields)
2. Dataset is valid and appropriate for the stated intent
3. Field names match the dataset schema (check field lists in dataset reference)
4. Correct operators for field types (ENUM unquoted, strings quoted, regex with ~=)
5. Every `comp` aggregation has a `by` clause (unless intentionally producing a single-row result)
6. Time range specified (via `config timeframe`, `filter _time`, or query context)
7. No placeholder text (no TODO, TBD, <placeholder>)
8. Stage order follows recommendation (filter early, fields early for joins)
9. `fields` appears early when joining wide datasets (reduce before join)
10. ENUM values are unquoted (ENUM.EVENT_LOG, not "ENUM.EVENT_LOG")
11. Functions applied to correct types (no string functions on arrays, etc.)
12. `dedup` only used on numbers/strings (not arrays or objects)

## Common Mistakes

| Mistake | Correct |
|---------|---------|
| `"ENUM.EVENT_LOG"` (quoted) | `ENUM.EVENT_LOG` (unquoted) |
| `dataset = msft_azure_ad_raw` (wrong name for direct query) | `dataset = microsoft_azure_ad_raw` |
| `lowercase(field) contains "value"` (unnecessary) | `field contains "value"` (case-insensitive already) |
| Missing `fields` on wide joins | Add `fields` before join to reduce columns |
| `search` stage for data > 90 days old | Use `filter` stage instead (no time limit) |
| `comp count(field) ...` without `by` clause | Add `by group_field` or confirm single-row result is intended |
| `dedup` on array or object fields | `dedup` only works on numbers and strings |
| No `config timeframe` when non-default window needed | Add `config timeframe = Nd` preamble |
