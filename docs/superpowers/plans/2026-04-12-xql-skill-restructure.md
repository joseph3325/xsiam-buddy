# XQL Skill Restructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure the monolithic xsiam-xql skill into three focused skills (xsiam-xql, xsiam-correlations, xsiam-splunk-to-xql) with a tiered shared reference layer, incorporating all XQL knowledge from the Obsidian RagVault.

**Architecture:** Three independent skills share a common XQL reference layer under `xsiam-shared/references/`. The shared layer is tiered: core files always load (~18K tokens), specialized files load on-demand when queries need advanced features (~10K tokens). Each skill has its own SKILL.md workflow and skill-specific reference files.

**Tech Stack:** Markdown (skills + references), YAML (correlation templates), JSON (plugin manifests). No runtime code.

**Knowledge Source:** Obsidian XSIAM knowledge vault (referenced below as `vault://`). Key directories:
- `xsiam-xql-language/xql-stages-reference.md` — 23 stages catalog
- `xsiam-xql-language/xql-functions-reference.md` — 96 functions catalog
- `xsiam-xql-language/xql-getting-started.md` — Operators, data selection, string handling
- `xsiam-build-xql-queries/federated-search.md` — Federated Search config
- `xsiam-build-xql-queries/xql-query-builder.md` — Splunk-to-XQL mappings, best practices
- `xsiam-build-xql-queries/query-templates-and-management.md` — Query Library, `call` stage context
- `xsiam-dataset-management/` — Dataset types, lookup datasets, cold storage

**Existing files to reference:**
- `skills/xsiam-xql/SKILL.md` — Current skill workflow (208 lines)
- `skills/xsiam-xql/references/xql-syntax-reference.md` — Current syntax reference (971 lines)
- `skills/xsiam-xql/references/xql-datasets.md` — Current dataset catalog (1015 lines)
- `skills/xsiam-xql/references/xql-examples.md` — Current examples (409 lines)
- `skills/xsiam-xql/references/correlation-template.yml` — Current correlation template (75 lines)

**Design spec:** `docs/superpowers/specs/2026-04-12-xql-skill-restructure-design.md`

---

## Task 1: Create `xql-core-reference.md` (Shared)

**Files:**
- Create: `skills/xsiam-shared/references/xql-core-reference.md`

This is the foundational file — all three skills depend on it. It combines content from the existing `xql-syntax-reference.md` with the 10 missing stages and ~35 missing functions from the RagVault.

- [ ] **Step 1: Read source material**

Read these files to gather all content:
- Existing: `skills/xsiam-xql/references/xql-syntax-reference.md` (current syntax — 971 lines covering 13 stages, ~55 functions, operators, patterns)
- RagVault: `vault://xsiam-xql-language/xql-stages-reference.md` (23 stages with syntax/examples)
- RagVault: `vault://xsiam-xql-language/xql-functions-reference.md` (96 functions in 9 categories)
- RagVault: `vault://xsiam-xql-language/xql-getting-started.md` (operators, data selection, string handling, ENUM types)

- [ ] **Step 2: Write `xql-core-reference.md`**

Create `skills/xsiam-shared/references/xql-core-reference.md` (~1,200 lines). Structure:

```markdown
# XQL Core Reference

## Data Selection

### dataset
### preset
### datamodel
### config timeframe

## Operators

### Comparison Operators
### Boolean Operators
### String/Range Operators (IN, CONTAINS, ~=, INCIDR)
### ENUM Types
### Null Checks

## Pipeline Stages

(All 23 stages, each with: Purpose, Syntax, Key notes, 1-2 examples)

### alter
### arrayexpand
### bin
### call
### comp
### config
### dedup
### fields
### filter
### getrole
### iploc
### join
### limit
### replacenull
### search
### sort
### tag
### target
### top
### transaction
### union
### view
### windowcomp

## Common Functions

(~60 most-used functions — those that appear in everyday queries)

### String Functions
(concat, len, lowercase, uppercase, trim, ltrim, rtrim, replace, split, regextract, format_string, sha256)

### Math Functions
(add, subtract, multiply, divide, round, floor, pow)

### Type Conversion Functions
(to_string, to_number, to_integer, to_float, to_boolean, coalesce, if, convert_from_base_64)

### Time/Date Functions
(current_time, format_timestamp, parse_timestamp, timestamp_diff, to_timestamp, to_epoch, parse_epoch, extract_time, date_floor, timestamp_seconds, time_frame_end)

### IP Functions
(incidr, incidr6, is_ipv4, is_ipv6, is_known_private_ipv4)

### Core Aggregate Functions
(count, count_distinct, sum, avg, min, max, values, first, last, earliest, latest, median, list)

### JSON Access
(Arrow notation: field -> key, field -> [], field -> key{}; json_extract, json_extract_scalar)

## Arrow Notation (JSON Navigation)

(Detailed section on ->, nested paths, array handling)

## Best Practices

(Performance tips, readability, null handling, UTC, early fields)
```

Content rules:
- Merge existing `xql-syntax-reference.md` content (stages, functions, operators, arrow notation, patterns, best practices) with RagVault content
- Add the 10 missing stages from RagVault `xql-stages-reference.md`: `call`, `getrole`, `iploc`, `replacenull`, `search`, `tag`, `target`, `top`, `windowcomp`, `view` (full `view` docs replacing the current 2-line entry)
- Add missing core functions from RagVault `xql-functions-reference.md`: `format_string`, `len`, `replex`, `string_count`, `wildcard_match`, `sha256`, `pow`, `floor`, `to_float`, `to_integer`, `to_boolean`, `convert_from_base_64`, `date_floor`, `extract_time`, `parse_epoch`, `time_frame_end`, `timestamp_seconds`, `to_epoch`, `is_known_private_ipv4`, `is_ipv6`, `is_known_private_ipv6`, `ip_to_int`, `ip_in_int`
- Add operator details from RagVault `xql-getting-started.md`: string quoting (single quotes for wildcards, triple quotes for regex/escapes), empty value distinction (null vs "")
- Preserve the existing common query patterns section (IOC hunting, timeline analysis, lateral movement, etc.) — these are proven patterns
- Do NOT include: advanced array functions (`arraymap`, `arrayfilter`, `array_any`, etc.), advanced JSON functions (`json_extract_array`, `json_path_extract`, `object_create`, etc.), window functions (`rank`, `row_number`, `lag`, etc.), approximate aggregates, URL functions — these go in `xql-advanced-functions.md`
- Do NOT include: correlation rule content — that moves to `xsiam-correlations`
- Do NOT include: dataset/field catalogs — those stay in `xql-datasets-core.md`

- [ ] **Step 3: Verify file structure**

Run: `wc -l skills/xsiam-shared/references/xql-core-reference.md`
Expected: ~1,000-1,400 lines

Run: `head -5 skills/xsiam-shared/references/xql-core-reference.md`
Expected: Shows `# XQL Core Reference` heading

Verify all 23 stages are present:
Run: `grep -c "^### " skills/xsiam-shared/references/xql-core-reference.md`
Expected: 30+ (23 stages + function category headings + other subsections)

- [ ] **Step 4: Commit**

```bash
git add skills/xsiam-shared/references/xql-core-reference.md
git commit -m "feat(xsiam-shared): create xql-core-reference with all 23 stages and 60 functions

Merges existing xql-syntax-reference.md content with 10 new stages and ~25 new
functions from the RagVault knowledge base. This is the always-loaded core
reference shared by xsiam-xql, xsiam-correlations, and xsiam-splunk-to-xql."
```

---

## Task 2: Create `xql-datasets-core.md` (Shared)

**Files:**
- Create: `skills/xsiam-shared/references/xql-datasets-core.md`

- [ ] **Step 1: Read source material**

Read these files:
- Existing: `skills/xsiam-xql/references/xql-datasets.md` (current dataset catalog — 1015 lines)
- RagVault: `vault://xsiam-xql-language/xql-getting-started.md` (preset definitions, default dataset info)
- RagVault: `vault://xsiam-build-xql-queries/xql-query-builder.md` (XDM dataset mappings, hot/cold storage, default limits)

- [ ] **Step 2: Write `xql-datasets-core.md`**

Create `skills/xsiam-shared/references/xql-datasets-core.md` (~800 lines). Structure:

```markdown
# XQL Datasets — Core Reference

## Dataset Selection Guide

(Use-case to dataset mapping table — carried from existing skill)

## Presets

(All 5 presets with which datasets they combine — NEW content from RagVault)
### xdr_event
### network_story
### cloud_story
### identity_story
### ad_users

## Endpoint Datasets
### xdr_data
### endpoints

## Network Datasets
### panw_ngfw_traffic_raw
### panw_ngfw_threat_raw

## Cloud Datasets
### cloud_audit_log
### microsoft_azure_ad_raw
### microsoft_azure_ad_sign_in_raw
### msft_azure_ad_raw (datamodel variant)

## Identity Datasets
### okta_raw

## Alert and Incident Datasets
### xdr_alerts
### xdr_incidents

## Field Naming Conventions

## Cross-Dataset Join Patterns
(3 core join examples — carried from existing)
```

Content rules:
- Carry forward the top ~10 datasets with full field lists from existing `xql-datasets.md`
- Add preset definitions from RagVault (which datasets each preset combines) — this is currently missing
- Add hot/cold storage syntax from RagVault `xql-query-builder.md`: `dataset` vs `cold_dataset`
- Add default result limits by context from RagVault: 1,000 interactive, 1,000,000 API/widgets/scheduled, 10,000 legacy templates
- Add XDM dataset mappings info from RagVault: automatic (xdr_data, NGFW), marketplace, custom
- Add user name format note: `<company domain>\<username>` normalized format
- Do NOT include: third-party alert datasets, email datasets, `pan_dss_raw` deep-dive, `xdr_http_collector`, cold storage details — those go in `xql-datasets-extended.md`
- Do NOT include: `email_gateway_raw` full field catalog — goes in extended

- [ ] **Step 3: Verify file structure**

Run: `wc -l skills/xsiam-shared/references/xql-datasets-core.md`
Expected: ~700-900 lines

Verify presets section exists (new content):
Run: `grep -c "^### " skills/xsiam-shared/references/xql-datasets-core.md`
Expected: 15+ (dataset headings + preset headings)

- [ ] **Step 4: Commit**

```bash
git add skills/xsiam-shared/references/xql-datasets-core.md
git commit -m "feat(xsiam-shared): create xql-datasets-core with presets and top datasets

Carries forward top 10 datasets with field lists from existing xql-datasets.md.
Adds preset breakdowns, hot/cold storage syntax, result limits, XDM mappings,
and user name format from RagVault."
```

---

## Task 3: Create `xql-advanced-functions.md` (Shared, On-Demand)

**Files:**
- Create: `skills/xsiam-shared/references/xql-advanced-functions.md`

- [ ] **Step 1: Read source material**

Read these files:
- Existing: `skills/xsiam-xql/references/xql-syntax-reference.md` (advanced array functions section, arrow notation edge cases)
- RagVault: `vault://xsiam-xql-language/xql-functions-reference.md` (array, JSON, window, IP, URL function tables)

- [ ] **Step 2: Write `xql-advanced-functions.md`**

Create `skills/xsiam-shared/references/xql-advanced-functions.md` (~400 lines). Structure:

```markdown
# XQL Advanced Functions Reference

This file is loaded on-demand when a query requires advanced array/JSON manipulation,
window functions, approximate aggregates, URL parsing, or advanced IP operations.

## Advanced Array Functions

(arraymap, arrayfilter, arraydistinct, arrayindex, arraymerge, arrayconcat,
arrayrange, arraystring, arrayindexof, array_any, array_all, arraycreate,
array_length — with @element iterator pattern, examples)

## Advanced JSON Functions

(json_extract_array, json_extract_scalar_array, json_path_extract, object_create,
object_merge, to_json_string — with $.path notation, examples)

## Window Functions (windowcomp)

(rank, row_number, first_value, last_value, lag — with partition/order syntax,
note that standard aggregates also work as running calculations in windowcomp)

## Approximate Aggregate Functions

(approx_count, approx_quantiles, approx_top — with performance trade-off notes)

## URL Functions

(extract_url_host, extract_url_pub_suffix, extract_url_registered_domain)

## Advanced IP Functions

(incidr6, ip_to_int, ip_in_int, is_known_private_ipv6 — extending core IP functions)

## Regex Deep-Dive

(regexcapture named groups, regextract + arrayindex scalar pattern,
replex for regex replacement, string quoting for regex patterns,
wildcard_match as a simpler alternative)
```

Content rules:
- Functions listed here are NOT duplicated in `xql-core-reference.md`
- Each function: syntax, description, 1 example
- `@element` iterator pattern for `arraymap`/`arrayfilter`/`array_any`/`array_all` must be clearly documented with examples
- Window functions: show `partition by` + `order by` syntax, note that `lag(field, offset, default)` supports offset and default value
- Regex section: document that `~=` is case-sensitive, `contains` is case-insensitive, triple-quote strings for regex patterns with escape sequences

- [ ] **Step 3: Verify file structure**

Run: `wc -l skills/xsiam-shared/references/xql-advanced-functions.md`
Expected: ~350-450 lines

- [ ] **Step 4: Commit**

```bash
git add skills/xsiam-shared/references/xql-advanced-functions.md
git commit -m "feat(xsiam-shared): add on-demand xql-advanced-functions reference

Covers advanced array/JSON manipulation, window functions, approximate aggregates,
URL parsing, advanced IP operations, and regex patterns. Loaded only when queries
need these features."
```

---

## Task 4: Create `xql-datasets-extended.md` (Shared, On-Demand)

**Files:**
- Create: `skills/xsiam-shared/references/xql-datasets-extended.md`

- [ ] **Step 1: Read source material**

Read these files:
- Existing: `skills/xsiam-xql/references/xql-datasets.md` (third-party alert datasets, email datasets, pan_dss_raw sections)
- RagVault: `vault://xsiam-build-xql-queries/xql-query-builder.md` (cold storage details)
- RagVault: `vault://xsiam-content-examples/` (xdr_http_collector patterns)

- [ ] **Step 2: Write `xql-datasets-extended.md`**

Create `skills/xsiam-shared/references/xql-datasets-extended.md` (~500 lines). Structure:

```markdown
# XQL Datasets — Extended Reference

This file is loaded on-demand when a query targets uncommon datasets,
third-party alert sources, email data, identity deep-dives, or cold storage.

## Third-Party Alert Datasets
### abnormal_security_generic_alert_raw
### thinkst_canary_generic_alert_raw
### darktrace_darktrace_raw
### google_workspace_alerts_raw

## Email Datasets
### microsoft_office_365_raw
### email_gateway_raw

## Identity Deep-Dive
### pan_dss_raw (Cloud Identity Engine)
(User records, computer records, OU/security group join patterns)

## Custom-Ingested Data
### xdr_http_collector

## Cold Storage
(cold_dataset syntax, constraints, querying both hot and cold)
```

Content rules:
- Carry forward third-party, email, and identity dataset content from existing `xql-datasets.md`
- Add `xdr_http_collector` from RagVault content examples
- Add cold storage section from RagVault `xql-query-builder.md`
- Each dataset: field list, key notes, example query snippet where helpful

- [ ] **Step 3: Verify and commit**

Run: `wc -l skills/xsiam-shared/references/xql-datasets-extended.md`
Expected: ~400-600 lines

```bash
git add skills/xsiam-shared/references/xql-datasets-extended.md
git commit -m "feat(xsiam-shared): add on-demand xql-datasets-extended reference

Covers third-party alert datasets, email datasets, Cloud Identity Engine deep-dive,
xdr_http_collector, and cold storage querying. Loaded on-demand for uncommon datasets."
```

---

## Task 5: Create `xql-federated-search.md` (Shared, On-Demand)

**Files:**
- Create: `skills/xsiam-shared/references/xql-federated-search.md`

- [ ] **Step 1: Read source material**

Read: RagVault `vault://xsiam-build-xql-queries/federated-search.md` (110 lines — complete federated search guide)

- [ ] **Step 2: Write `xql-federated-search.md`**

Create `skills/xsiam-shared/references/xql-federated-search.md` (~200 lines). Structure:

```markdown
# XQL Federated Search Reference

This file is loaded on-demand when a query targets external data sources
(S3, GCS, Azure Blob Storage).

## Overview
(Query distributed data without ingestion)

## Supported Configurations
(Storage providers, formats, partitioning)

## Limitations
(No correlations/widgets/dashboards/APIs, no search/target/view stages,
regional constraints, query costs by timeframe/complexity/egress)

## Dataset Naming
(Must start with external_, schema auto-detected from first 500 records,
ds field cannot be deleted)

## Setup Summary
### Amazon S3
### Google Cloud Storage
### Azure Blob Storage

## Querying External Datasets
(XQL syntax, joining with ingested data, ad-hoc queries only)
```

Content rules:
- Adapt directly from RagVault `federated-search.md` (110 lines)
- Condense setup flows to key steps (not full click-by-click UI instructions — the skill generates XQL, not configures the platform)
- Emphasize the XQL-relevant parts: dataset naming, query limitations, stage restrictions, joining external with ingested data

- [ ] **Step 3: Verify and commit**

Run: `wc -l skills/xsiam-shared/references/xql-federated-search.md`
Expected: ~150-250 lines

```bash
git add skills/xsiam-shared/references/xql-federated-search.md
git commit -m "feat(xsiam-shared): add on-demand xql-federated-search reference

Covers querying external S3/GCS/Azure Blob data via XQL, dataset naming
conventions, format requirements, and stage limitations."
```

---

## Task 6: Rebuild `xsiam-xql` SKILL.md

**Files:**
- Replace: `skills/xsiam-xql/SKILL.md`

- [ ] **Step 1: Read source material**

Read:
- Existing: `skills/xsiam-xql/SKILL.md` (current workflow — 208 lines)
- Other skill SKILL.md files for frontmatter pattern: `skills/xsiam-scripts/SKILL.md` (first 30 lines), `skills/xsiam-integrations/SKILL.md` (first 30 lines)

- [ ] **Step 2: Write new `SKILL.md`**

Replace `skills/xsiam-xql/SKILL.md` (~200 lines). Structure:

```yaml
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
```

Body sections:

```markdown
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
(Hunting, investigation, analytics, or enrichment)

### Step 2 — Select Dataset
(Use the dataset selection guide in xql-datasets-core.md)

### Step 3 — Build the Query
(Construct multi-stage XQL following recommended stage order)
Recommended stage order:
dataset/preset/datamodel → filter → alter → comp → fields → sort → limit → dedup

### Step 4 — Apply Construction Rules
(Filtering, aggregation with by, joins, time bucketing)

### Step 5 — Format Output
(XQL in fenced code block with mandatory header comments)

## Output Format

(Header comment template + code block format)

## Quality Checklist

(12-item expanded checklist)

## Common Mistakes

(Table — carried forward and expanded)
```

Content rules:
- Carry forward the proven workflow from existing SKILL.md
- Update scope boundaries to redirect to new sibling skills
- Update "Before Starting" to use tiered shared references
- Expand quality checklist with 2 new items (stage order, dedup on arrays)
- Expand common mistakes with 4 new entries (search 90-day limit, missing by clause, dedup on arrays, missing config timeframe)
- Remove all correlation rule content (workflow steps, output format, checklist items)

- [ ] **Step 3: Verify no correlation content remains**

Run: `grep -i "correlation\|corr_rule\|\.yml\|SEV_0" skills/xsiam-xql/SKILL.md`
Expected: No matches (or only the redirect line "use xsiam-correlations")

- [ ] **Step 4: Commit**

```bash
git add skills/xsiam-xql/SKILL.md
git commit -m "feat(xsiam-xql): rebuild SKILL.md with tiered loading and updated scope

Removes correlation rule content (now in xsiam-correlations), adds tiered
reference loading, expands quality checklist and common mistakes table,
redirects to sibling skills for out-of-scope requests."
```

---

## Task 7: Rebuild `xql-examples.md` as Cookbook

**Files:**
- Replace: `skills/xsiam-xql/references/xql-examples.md`

- [ ] **Step 1: Read source material**

Read:
- Existing: `skills/xsiam-xql/references/xql-examples.md` (current 9 examples — 409 lines)
- Existing examples in `skills/xsiam-xql/references/xql-syntax-reference.md` (common query patterns section)

- [ ] **Step 2: Write new `xql-examples.md`**

Replace `skills/xsiam-xql/references/xql-examples.md` (~500 lines). Structure:

```markdown
# XQL Query Cookbook

Production-ready query examples organized by use case. Each example includes
a header comment block, the query, and an annotation of the pattern demonstrated.

## Threat Hunting

### IOC Search (IP/Domain/Hash)
### File Hash Reputation Lookup
### Lateral Movement Detection

## Authentication

### Failed Login Spike Detection
### Password Spray Detection
### Multi-Location Authentication

## Cloud and Identity

### Azure AD Administrator Role Assigned
### Google Workspace Admin Actions
### Okta MFA Bypass Detection

## Network

### Data Exfiltration Volume Analysis
### Firewall Block Analysis

## Enrichment

### Endpoint-to-Firewall Join
### Alert-to-Incident Join
### CIE User/Group Enrichment (pan_dss_raw)

## Advanced Patterns

### Window Functions (Session Ranking)
### arrayexpand + Multi-Join Chain (Canary → Endpoints → AD)
### Transaction (User Session Grouping)
### Saved Query Reuse (call stage)
### Top Issues with Datamodel (config timeframe)
### Process Execution Hash Lookup (config timeframe + view)
```

Content rules:
- Carry forward all 9 existing examples (reformatted into category sections)
- Add 3-4 new examples to fill gaps: window function, `call` stage, `transaction`, network analysis
- Each example format:

```markdown
### Example Title

**Pattern:** Brief description of the XQL technique demonstrated
**Datasets:** dataset_name

​```xql
// Title: Example Title
// Description: What this query does
// Author: xsiam-buddy
// Datasets: dataset_name
// Modified: 2026-04-12

dataset = ...
| filter ...
| ...
​```

**Annotation:** Explain the key technique (e.g., "Uses `arrayexpand` to flatten IP array before joining to firewall data, enabling per-IP traffic analysis")
```

- Remove any correlation rule examples (those move to `xsiam-correlations/references/correlation-examples.md`)

- [ ] **Step 3: Verify categories present**

Run: `grep "^## " skills/xsiam-xql/references/xql-examples.md`
Expected: Threat Hunting, Authentication, Cloud and Identity, Network, Enrichment, Advanced Patterns

- [ ] **Step 4: Commit**

```bash
git add skills/xsiam-xql/references/xql-examples.md
git commit -m "feat(xsiam-xql): reorganize examples as use-case cookbook

Restructures 9 existing examples into categorized sections (hunting, auth,
cloud, network, enrichment, advanced). Adds window function, call stage,
transaction, and network analysis examples."
```

---

## Task 8: Clean Up Old xsiam-xql Reference Files

**Files:**
- Delete: `skills/xsiam-xql/references/xql-syntax-reference.md`
- Delete: `skills/xsiam-xql/references/xql-datasets.md`
- Delete: `skills/xsiam-xql/references/correlation-template.yml`

- [ ] **Step 1: Verify replacement files exist**

Run: `ls -la skills/xsiam-shared/references/xql-core-reference.md skills/xsiam-shared/references/xql-datasets-core.md`
Expected: Both files exist with non-zero size

- [ ] **Step 2: Delete old files**

```bash
git rm skills/xsiam-xql/references/xql-syntax-reference.md
git rm skills/xsiam-xql/references/xql-datasets.md
git rm skills/xsiam-xql/references/correlation-template.yml
```

- [ ] **Step 3: Verify xsiam-xql/references/ contains only examples**

Run: `ls skills/xsiam-xql/references/`
Expected: Only `xql-examples.md`

- [ ] **Step 4: Commit**

```bash
git commit -m "refactor(xsiam-xql): remove old references replaced by shared layer

xql-syntax-reference.md → xsiam-shared/references/xql-core-reference.md + xql-advanced-functions.md
xql-datasets.md → xsiam-shared/references/xql-datasets-core.md + xql-datasets-extended.md
correlation-template.yml → xsiam-correlations/references/correlation-rule-spec.md"
```

---

## Task 9: Create `xsiam-correlations` Skill

**Files:**
- Create: `skills/xsiam-correlations/SKILL.md`
- Create: `skills/xsiam-correlations/references/correlation-rule-spec.md`
- Create: `skills/xsiam-correlations/references/correlation-examples.md`

- [ ] **Step 1: Read source material**

Read:
- Existing: `skills/xsiam-xql/references/correlation-template.yml` (75 lines — YAML template with annotations)
- Existing: `skills/xsiam-xql/SKILL.md` (correlation-related workflow steps and checklist items)
- Existing: `skills/xsiam-xql/references/xql-syntax-reference.md` (correlation rules section)
- Existing: `skills/xsiam-xql/references/xql-examples.md` (correlation rule examples — Abnormal Security, Canary, Darktrace)

- [ ] **Step 2: Create directory structure**

```bash
mkdir -p skills/xsiam-correlations/references
```

- [ ] **Step 3: Write `SKILL.md`**

Create `skills/xsiam-correlations/SKILL.md` (~180 lines):

```yaml
---
name: xsiam-correlations
description: >
  This skill should be used when the user asks to "create a correlation rule",
  "build a detection rule", "write a detection", "XSIAM alert rule",
  "correlation YAML", "detection engineering", "build a detection for",
  or needs to generate correlation rule .yml files with embedded XQL detection logic.
version: 1.0.0
---
```

Body sections:
- Scope (correlation rules only; redirect standalone queries and SPL translation)
- Before Starting (always load: xql-core-reference, xql-datasets-core, correlation-rule-spec; on-demand: same shared files)
- Workflow (7 steps: understand detection → select execution mode → write XQL → build YAML → map MITRE → configure suppression → format output)
- Output Format (complete .yml with XQL as literal block scalar)
- Quality Checklist (10 items from design spec)
- Common Mistakes (embed alert metadata in XQL, wrong severity enum, mismatched name/alert_name, missing UUID)

- [ ] **Step 4: Write `correlation-rule-spec.md`**

Create `skills/xsiam-correlations/references/correlation-rule-spec.md` (~400 lines):

Expand the existing 75-line `correlation-template.yml` into a full spec. Structure:

```markdown
# Correlation Rule YAML Specification

## Canonical Field Order
(Exact order matching XSIAM export)

## Field Reference
(Every field: name, type, required/optional, valid values, description)

### Core Fields
- global_rule_id (UUID v4, required)
- name / alert_name (must match)
- dataset (always "alerts")
- action (always "ALERTS")
- alert_domain (DOMAIN_SECURITY default)
- alert_category

### Severity
- severity enum values: SEV_010_CRITICAL, SEV_020_HIGH, SEV_030_MEDIUM, SEV_040_LOW, SEV_060_INFORMATIONAL
- Static vs dynamic severity (when to use each, with examples)
- Note: no SEV_050

### Execution
- execution_mode: REAL_TIME vs SCHEDULED
- crontab syntax (for SCHEDULED only)
- search_window interaction with execution_mode

### Suppression
- suppression_enabled / suppression_duration / suppression_fields
- Common suppression patterns by detection type:
  - Brute force → suppress by source IP
  - Lateral movement → suppress by hostname
  - Data exfiltration → suppress by user
  - Malware → suppress by file hash

### MITRE ATT&CK
- mitre_defs structure: { "technique_id": ["tactic", ...] }
- Common technique/tactic pairs for detection categories

### Other Fields
- mapping_strategy (CUSTOM)
- alert_fields ({} usually)
- user_defined_category / user_defined_severity
- drilldown_query_timeframe
- investigation_query_link

## XQL Embedding
- Use YAML literal block scalar (|) for xql_query field
- XQL contains detection logic only
- Alert metadata lives in YAML wrapper, not XQL
- Exception: dynamic severity when source has numeric score but no discrete severity field

## Template
(Complete annotated template — evolved from existing correlation-template.yml)
```

Content rules:
- Absorb all content from existing `correlation-template.yml`
- Expand each field with type info, valid values, and when to use
- Add suppression patterns table (new)
- Add MITRE mapping guidance (new)
- Add execution mode decision guide (new)

- [ ] **Step 5: Write `correlation-examples.md`**

Create `skills/xsiam-correlations/references/correlation-examples.md` (~300 lines):

4 complete .yml examples, each with inline annotations:

1. **Real-time single event** — Canary alert (carry from existing Canary example, add YAML wrapper)
2. **Scheduled threshold** — Failed auth spike (new — `comp count_distinct(user) >= 5` pattern with `search_window: "15m"`, crontab)
3. **Scheduled with dynamic severity** — Third-party numeric score (carry from existing Abnormal/Darktrace examples, cleaned up)
4. **Real-time with suppression** — Firewall block repeated source (new — `suppression_fields: ["source_ip"]`, `suppression_duration: "1h"`)

Each example: complete `.yml` file with `# ANNOTATION:` comments explaining design choices inline.

- [ ] **Step 6: Verify directory structure**

Run: `find skills/xsiam-correlations -type f`
Expected:
```
skills/xsiam-correlations/SKILL.md
skills/xsiam-correlations/references/correlation-rule-spec.md
skills/xsiam-correlations/references/correlation-examples.md
```

- [ ] **Step 7: Commit**

```bash
git add skills/xsiam-correlations/
git commit -m "feat(xsiam-correlations): create dedicated correlation rule skill

New skill for generating correlation rule .yml files with embedded XQL.
Includes full YAML field spec (expanded from 75-line template to 400-line
reference), 4 complete annotated examples, and 7-step workflow."
```

---

## Task 10: Create `xsiam-splunk-to-xql` Skill

**Files:**
- Create: `skills/xsiam-splunk-to-xql/SKILL.md`
- Create: `skills/xsiam-splunk-to-xql/references/spl-to-xql-mapping.md`

- [ ] **Step 1: Read source material**

Read:
- RagVault: `vault://xsiam-build-xql-queries/xql-query-builder.md` (Splunk-to-XQL mapping table, lines ~126-151)

- [ ] **Step 2: Create directory structure**

```bash
mkdir -p skills/xsiam-splunk-to-xql/references
```

- [ ] **Step 3: Write `SKILL.md`**

Create `skills/xsiam-splunk-to-xql/SKILL.md` (~120 lines):

```yaml
---
name: xsiam-splunk-to-xql
description: >
  This skill should be used when the user asks to "translate SPL",
  "convert Splunk to XQL", "Splunk to XQL", "SPL to XQL",
  "migrate Splunk query", "translate this Splunk query",
  or needs to convert existing Splunk SPL queries into equivalent XQL.
version: 1.0.0
---
```

Body sections:
- Scope (SPL translation only; redirect XQL from scratch and correlation rules)
- Before Starting (always load: spl-to-xql-mapping, xql-core-reference; on-demand: shared files when translated query uses advanced features)
- Workflow (6 steps: parse SPL → map commands → flag untranslatable → map datasets → assemble XQL → validate)
- Output Format (XQL code block with "Translated from SPL" header note)
- Quality Checklist (5 items)
- Known Limitations (not all SPL translatable; macros, custom lookups, some eval functions have no equivalent)

- [ ] **Step 4: Write `spl-to-xql-mapping.md`**

Create `skills/xsiam-splunk-to-xql/references/spl-to-xql-mapping.md` (~350 lines):

```markdown
# SPL to XQL Mapping Reference

## Command Mapping

| SPL Command | XQL Equivalent | Notes |
|-------------|---------------|-------|
| `stats` | `comp` | |
| `eval` | `alter` | |
| `table` | `fields` | |
| `where` | `filter` | |
| `sort` | `sort asc/desc` | XQL requires explicit asc/desc |
| `top` | `top N` | |
| `rex` | `arrayindex(regextract(...), 0)` | Wrap with arrayindex for scalar |
| `mvexpand` | `arrayexpand` | |
| `mvjoin` | `arraystring` | |
| `mvdedup` | `arraydistinct` | |
| `inputlookup` | `union (dataset = lookup_name)` | |
| `iplocation` | `iploc` | IPv4 only |
| `fillnull` | `replacenull` | Null only, not empty strings |
| `dedup` | `dedup` | Numbers/strings only in XQL |
| `rename X as Y` | `alter Y = X` | |
| `head N` | `limit N` | |
| `eventstats` | `windowcomp` | Closest equivalent |
| `transaction` | `transaction` | Max 50 fields in XQL |
| `lookup` | `join (dataset = lookup_name)` | |
| `append` | `union` | |

## Function Mapping

| SPL Function | XQL Function | Notes |
|-------------|-------------|-------|
| `lower()` | `lowercase()` | |
| `upper()` | `uppercase()` | |
| `len()` | `len()` | Same |
| `substr()` | `substring()` | |
| `replace()` | `replace()` | Same for literal; `replex()` for regex |
| `split()` | `split()` | Same |
| `mvcount()` | `array_length()` | |
| `mvindex()` | `arrayindex()` | |
| `mvfilter()` | `arrayfilter()` | Uses @element in XQL |
| `mvjoin()` | `arraystring()` | |
| `now()` | `current_time()` | |
| `strftime()` | `format_timestamp()` | Different format strings |
| `strptime()` | `parse_timestamp()` | Different format strings |
| `tonumber()` | `to_number()` | |
| `tostring()` | `to_string()` | |
| `cidrmatch()` | `incidr()` | |
| `if()` | `if()` | Same syntax |
| `coalesce()` | `coalesce()` | Same syntax |
| `json_extract()` | `json_extract_scalar()` | Or arrow notation |
| `spath` | `json_extract()` / `->` | Arrow notation preferred |
| `md5()` / `sha256()` | `sha256()` | Only SHA-256 available |

## Index/Sourcetype to Dataset Mapping

| Splunk Source | XSIAM Dataset | Notes |
|--------------|--------------|-------|
| `index=main sourcetype=WinEventLog` | `dataset = xdr_data` | Filter by event_type/event_sub_type |
| `index=main sourcetype=syslog` | `dataset = xdr_data` | |
| `index=firewall` | `dataset = panw_ngfw_traffic_raw` | Or panw_ngfw_threat_raw for threats |
| `index=o365` | `dataset = microsoft_office_365_raw` | |
| `index=azure` | `dataset = microsoft_azure_ad_raw` | Or sign_in_raw for auth |
| `index=okta` | `dataset = okta_raw` | |
| `index=cloudtrail` | `dataset = cloud_audit_log` | Filter by cloud_provider |

## Known Untranslatable Constructs

| SPL Feature | Workaround |
|------------|-----------|
| `eval mvzip()` | Manual array construction with `alter` + `arraycreate` |
| Custom macros | Expand macro inline before translating |
| `lookup` with CSV | Import CSV as lookup dataset first, then `join` |
| `outputlookup` | `target lookup = "name"` (50 MB limit) |
| `map` / `foreach` | No equivalent; rewrite as explicit stages |
| `cluster` / `kmeans` | No equivalent in XQL |
| `predict` | No equivalent in XQL |
| `anomalydetection` | No equivalent; use threshold-based `comp` + `filter` |

## Translation Examples

### Example 1: Simple Stats Query
### Example 2: Multi-Value Expansion with Rex
### Example 3: Lookup Join with Aggregation
### Example 4: Time-Bucketed Trend Analysis
```

Each translation example: side-by-side SPL (in splunk code block) and XQL (in xql code block) with annotations explaining each mapping decision.

- [ ] **Step 5: Verify directory structure**

Run: `find skills/xsiam-splunk-to-xql -type f`
Expected:
```
skills/xsiam-splunk-to-xql/SKILL.md
skills/xsiam-splunk-to-xql/references/spl-to-xql-mapping.md
```

- [ ] **Step 6: Commit**

```bash
git add skills/xsiam-splunk-to-xql/
git commit -m "feat(xsiam-splunk-to-xql): create dedicated SPL translation skill

New skill for translating Splunk SPL queries to XQL. Includes command mapping
(20 commands), function mapping (22 functions), index-to-dataset mapping,
untranslatable construct workarounds, and 4 side-by-side translation examples."
```

---

## Task 11: Update Plugin Manifests

**Files:**
- Modify: `.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Read current manifests**

Read: `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`

- [ ] **Step 2: Update `plugin.json`**

Bump version from `0.1.6` to `0.2.0` (minor version bump for new skills):

```json
{
  "name": "xsiam-buddy",
  "version": "0.2.0",
  "description": "Develop Cortex XSIAM content: automation scripts, XQL queries, correlation rules, playbooks, and documentation",
  "author": {
    "name": "Joe"
  },
  "keywords": ["xsiam", "xsoar", "cortex", "xql", "correlation", "splunk", "playbook", "automation", "security"]
}
```

Changes:
- Version: `0.1.6` → `0.2.0`
- Description: add "correlation rules" (new skill)
- Keywords: add `"correlation"`, `"splunk"` (new skills)

- [ ] **Step 3: Update `marketplace.json`**

```json
{
  "name": "xsiam-buddy",
  "owner": {
    "name": "joseph3325"
  },
  "metadata": {
    "description": "Cortex XSIAM content development tools for Claude Code",
    "version": "0.2.0"
  },
  "plugins": [
    {
      "name": "xsiam-buddy",
      "description": "Develop Cortex XSIAM content: automation scripts, XQL queries, correlation rules, playbooks, and documentation",
      "source": "./",
      "strict": true,
      "skills": [
        "./skills/xsiam-scripts",
        "./skills/xsiam-xql",
        "./skills/xsiam-correlations",
        "./skills/xsiam-splunk-to-xql",
        "./skills/xsiam-playbooks",
        "./skills/xsiam-docs",
        "./skills/xsiam-shared"
      ]
    }
  ]
}
```

Changes:
- Version: `0.1.6` → `0.2.0`
- Description: add "correlation rules"
- Skills array: add `"./skills/xsiam-correlations"` and `"./skills/xsiam-splunk-to-xql"`
- Note: `xsiam-integrations` is not in the current marketplace.json skills list (confirm this is intentional before adding)

- [ ] **Step 4: Commit**

```bash
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "feat(plugin): bump to v0.2.0, add correlations and splunk-to-xql skills

Adds xsiam-correlations and xsiam-splunk-to-xql to the plugin manifest.
Bumps version to 0.2.0 for the XQL skill restructure."
```

---

## Task 12: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Read current CLAUDE.md**

Read: `CLAUDE.md`

- [ ] **Step 2: Update plugin architecture section**

Update the directory tree in the "Plugin Architecture" section to reflect the new structure:

```markdown
```
.claude-plugin/
  plugin.json
  marketplace.json
skills/
  xsiam-scripts/
    SKILL.md
    references/script-yaml-spec.md
  xsiam-integrations/
    SKILL.md
    references/integration-yaml-spec.md
    references/integration-patterns.md
  xsiam-xql/
    SKILL.md
    references/xql-examples.md
  xsiam-correlations/
    SKILL.md
    references/correlation-rule-spec.md
    references/correlation-examples.md
  xsiam-splunk-to-xql/
    SKILL.md
    references/spl-to-xql-mapping.md
  xsiam-playbooks/
    SKILL.md
    references/playbook-format.md
  xsiam-docs/
    SKILL.md
    references/doc-templates.md
  xsiam-shared/
    references/
      common-patterns.md
      xql-core-reference.md
      xql-datasets-core.md
      xql-advanced-functions.md
      xql-datasets-extended.md
      xql-federated-search.md
docs/plans/
```
```

- [ ] **Step 3: Update "How Skills Work" section**

Add a note about tiered reference loading:

```markdown
The xsiam-xql, xsiam-correlations, and xsiam-splunk-to-xql skills use tiered reference loading.
Core XQL references always load; specialized references (advanced functions, extended datasets,
federated search) load on-demand based on query requirements. This reduces token cost for
simple queries while keeping advanced knowledge available.
```

- [ ] **Step 4: Update shared references note**

Update the bullet about `xsiam-shared`:

```markdown
- The `xsiam-shared/references/` directory contains both shared Python patterns (`common-patterns.md`, used by scripts and integrations) and shared XQL references (`xql-core-reference.md`, `xql-datasets-core.md`, etc., used by xsiam-xql, xsiam-correlations, and xsiam-splunk-to-xql) — changes to shared files affect all consuming skills
```

- [ ] **Step 5: Add XQL skill family section**

Add after the "Scripts vs Integrations" table:

```markdown
## XQL Skill Family

Three skills share a common XQL reference layer:

| Skill | Purpose | Output |
|---|---|---|
| xsiam-xql | Standalone queries | XQL in code block with header comments |
| xsiam-correlations | Detection rules | Complete `.yml` with embedded XQL |
| xsiam-splunk-to-xql | SPL translation | XQL in code block with "Translated from SPL" note |
```

- [ ] **Step 6: Commit**

```bash
git add CLAUDE.md
git commit -m "docs(claude): update CLAUDE.md for XQL skill restructure

Reflects new three-skill XQL architecture, tiered shared references,
and updated directory tree."
```

---

## Task 13: Final Verification

- [ ] **Step 1: Verify complete file tree**

```bash
find skills/ -name "*.md" -o -name "*.yml" -o -name "*.json" | sort
```

Expected output should include:
```
skills/xsiam-correlations/SKILL.md
skills/xsiam-correlations/references/correlation-examples.md
skills/xsiam-correlations/references/correlation-rule-spec.md
skills/xsiam-docs/SKILL.md
skills/xsiam-docs/references/doc-templates.md
skills/xsiam-integrations/SKILL.md
skills/xsiam-integrations/references/integration-patterns.md
skills/xsiam-integrations/references/integration-yaml-spec.md
skills/xsiam-playbooks/SKILL.md
skills/xsiam-playbooks/references/playbook-format.md
skills/xsiam-scripts/SKILL.md
skills/xsiam-scripts/references/script-yaml-spec.md
skills/xsiam-shared/references/common-patterns.md
skills/xsiam-shared/references/xql-advanced-functions.md
skills/xsiam-shared/references/xql-core-reference.md
skills/xsiam-shared/references/xql-datasets-core.md
skills/xsiam-shared/references/xql-datasets-extended.md
skills/xsiam-shared/references/xql-federated-search.md
skills/xsiam-splunk-to-xql/SKILL.md
skills/xsiam-splunk-to-xql/references/spl-to-xql-mapping.md
skills/xsiam-xql/SKILL.md
skills/xsiam-xql/references/xql-examples.md
```

- [ ] **Step 2: Verify old files are gone**

```bash
ls skills/xsiam-xql/references/xql-syntax-reference.md 2>&1
ls skills/xsiam-xql/references/xql-datasets.md 2>&1
ls skills/xsiam-xql/references/correlation-template.yml 2>&1
```

Expected: All three should show "No such file or directory"

- [ ] **Step 3: Verify manifests**

```bash
python3 -c "import json; d=json.load(open('.claude-plugin/plugin.json')); print(d['version'])"
python3 -c "import json; d=json.load(open('.claude-plugin/marketplace.json')); print(d['metadata']['version'])"
```

Expected: Both print `0.2.0`

- [ ] **Step 4: Verify skill count in marketplace**

```bash
python3 -c "import json; d=json.load(open('.claude-plugin/marketplace.json')); print(len(d['plugins'][0]['skills']), 'skills')"
```

Expected: `7 skills` (scripts, xql, correlations, splunk-to-xql, playbooks, docs, shared)

- [ ] **Step 5: Verify cross-references**

Check that SKILL.md files reference correct shared paths:
```bash
grep -r "xsiam-shared/references" skills/xsiam-xql/SKILL.md skills/xsiam-correlations/SKILL.md skills/xsiam-splunk-to-xql/SKILL.md
```

Expected: Each skill references `../xsiam-shared/references/xql-core-reference.md` at minimum

- [ ] **Step 6: Final commit (if any fixes needed)**

Only if verification steps revealed issues that required fixes.
