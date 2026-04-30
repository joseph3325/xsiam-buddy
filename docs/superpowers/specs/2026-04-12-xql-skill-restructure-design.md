# XQL Skill Restructure Design

**Date:** 2026-04-12
**Status:** Approved
**Scope:** Restructure xsiam-xql into three focused skills with shared references, incorporating knowledge from the Obsidian RagVault.

## Problem

The current xsiam-xql skill is a single 2,700-line skill that covers standalone XQL queries, correlation rules, and all reference material in a flat structure. Key issues:

- **Missing coverage:** 10 of 23 XQL stages undocumented, ~40 of 96 functions missing, no Federated Search, no Splunk translation, thin `windowcomp`/`transaction`/`union` docs
- **Monolithic loading:** All ~25K tokens load on every invocation regardless of query complexity
- **Mixed concerns:** Correlation rule YAML spec is interleaved with query writing guidance
- **No cookbook:** Examples organized by complexity rather than use case

## Solution

Split into three focused skills with a tiered shared reference layer:

```
skills/
  xsiam-xql/                         # Standalone XQL queries
  xsiam-correlations/                 # Correlation rule .yml files
  xsiam-splunk-to-xql/               # SPL -> XQL translation
  xsiam-shared/
    references/                       # Shared XQL language references (tiered)
```

## Architecture

### Shared Reference Layer (`xsiam-shared/references/`)

#### Always-Loaded Core

**`xql-core-reference.md`** (~1,200 lines, ~11K tokens)
- Data selection: `dataset`, `preset`, `datamodel`, `config timeframe`
- All 23 stages with full syntax and 1-2 examples each
  - Currently missing stages to add: `call`, `getrole`, `iploc`, `replacenull`, `search`, `tag`, `target`, `top`, `windowcomp`, `view`
- ~60 most-used functions: string, math, type conversion, time/date, conditional, core aggregates
- Operators: comparison, boolean, string matching, `IN`, `INCIDR`/`INCIDR6`, null checks
- ENUM types: syntax, common values, unquoted rule
- Arrow notation: JSON navigation with `->`, nested paths, array handling
- Best practices: performance, readability, null handling, UTC, early `fields`

**`xql-datasets-core.md`** (~800 lines, ~7K tokens)
- Top datasets by domain: `xdr_data`, `endpoints`, `xdr_alerts`, `xdr_incidents`, `panw_ngfw_traffic_raw`, `panw_ngfw_threat_raw`, `cloud_audit_log`, `okta_raw`, `microsoft_azure_ad_raw`/`sign_in_raw`, `email_gateway_raw`
- All 5 presets with constituent dataset breakdowns (currently missing)
- Field naming conventions: `_time`, IP, hash, user, process, file patterns
- Dataset selection guide: use-case to dataset mapping table
- Cross-dataset join patterns: 3 core join examples

#### On-Demand Specialized

**`xql-advanced-functions.md`** (~400 lines, ~3.5K tokens)
- Advanced array: `arraymap`, `arrayfilter`, `arraydistinct`, `arrayindex`, `arraymerge`, `array_any`, `array_all` with `@element` iterator
- Advanced JSON: `json_extract_array`, `json_extract_scalar_array`, `json_path_extract`, `object_create`, `object_merge`
- Window functions: `rank`, `row_number`, `first_value`, `last_value`, `lag` with partition/order
- Approximate aggregates: `approx_count`, `approx_quantiles`, `approx_top`
- URL functions: `extract_url_host`, `extract_url_pub_suffix`, `extract_url_registered_domain`
- IP functions: `incidr6`, `ip_to_int`, `is_ipv4`, `is_ipv6`, `is_known_private_ipv4`/`ipv6`
- Regex: `regexcapture` (named groups), `regextract` + `arrayindex` pattern, flavor/escaping notes

**`xql-datasets-extended.md`** (~500 lines, ~4.5K tokens)
- Third-party alert datasets: Abnormal Security, Canary, Darktrace, Google Workspace (with field lists and arrow notation patterns)
- Email datasets: `microsoft_office_365_raw`, `email_gateway_raw` full field catalog
- Identity deep-dive: `pan_dss_raw` (CIE users/computers), OU/security group join patterns
- `xdr_http_collector`: custom-ingested data querying
- Cold storage: `cold_dataset` syntax and constraints

**`xql-federated-search.md`** (~200 lines, ~1.8K tokens)
- External datasets: S3/GCS/Azure Blob setup
- Naming: must start with `external_`
- Formats: CSV, Parquet, JSONL (Parquet recommended)
- Hive partitioning: `ds=yyyy-mm-dd`
- Limitations: no correlations, widgets, dashboards; no `search`/`target`/`view` stages
- Regional constraints

#### Unchanged

**`common-patterns.md`** — existing shared Python patterns for scripts/integrations (not XQL-relevant)

---

### Skill 1: xsiam-xql

**Purpose:** Generate standalone XQL queries for threat hunting, investigation, and analytics.

**SKILL.md** (~200 lines)

Trigger phrases: "write an XQL query", "XQL", "threat hunting query", "query dataset", "search XSIAM data", "hunt for", "investigate"

Scope boundaries:
- NOT correlation rules -> use `xsiam-correlations`
- NOT translating SPL -> use `xsiam-splunk-to-xql`
- NOT writing automation logic -> use `xsiam-scripts`

Reference loading (tiered):
1. Always: `xsiam-shared/references/xql-core-reference.md`, `xsiam-shared/references/xql-datasets-core.md`
2. On-demand based on query needs:
   - Array/JSON manipulation or window functions -> `xql-advanced-functions.md`
   - Uncommon dataset or third-party alerts -> `xql-datasets-extended.md`
   - External data source -> `xql-federated-search.md`

Workflow:
1. Classify intent: hunting, investigation, analytics, or enrichment
2. Select dataset using the dataset selection guide
3. Build query following stage order: `dataset/preset/datamodel` -> `filter` -> `alter` -> `comp` -> `fields` -> `sort` -> `limit` -> `dedup`
4. Apply query construction rules: filtering operators, aggregation with `by`, joins, time bucketing
5. Format output: XQL in code block with header comments (Title, Description, Author, Datasets, Modified Date)

Quality checklist (expanded from current 10 items):
- Header comments complete
- Dataset valid and appropriate for intent
- Field names match dataset schema
- Correct operators for field types
- `by` clause present in all `comp` aggregations
- Time range specified
- No placeholder text
- Stage order follows recommendation
- `fields` appears early when joining wide datasets
- ENUM values unquoted
- No functions applied to wrong types
- `dedup` only on numbers/strings

Common mistakes table: carried forward and expanded with `search` 90-day limit, missing `by` clause, `dedup` on arrays, missing `config timeframe`.

**`references/xql-examples.md`** (~500 lines)

Reorganized as cookbook by use case:
- **Threat Hunting:** IOC search, hash reputation, lateral movement detection
- **Authentication:** Failed login spikes, password spray, multi-location auth
- **Cloud/Identity:** Azure AD role assignment, Google Workspace admin actions, Okta MFA bypass
- **Network:** Data exfiltration volume, firewall block analysis
- **Enrichment:** Endpoint-to-firewall join, alert-to-incident join, CIE user/group enrichment
- **Advanced Patterns:** Window functions, `arrayexpand` + join chains, `transaction`, `call` for saved queries

Each example: header comment block, query, brief annotation of pattern demonstrated.

---

### Skill 2: xsiam-correlations

**Purpose:** Generate correlation rule `.yml` files with embedded XQL detection logic.

**SKILL.md** (~180 lines)

Trigger phrases: "correlation rule", "detection rule", "create alert rule", "XSIAM alert", "build a detection", "correlation YAML"

Scope boundaries:
- NOT standalone queries -> use `xsiam-xql`
- NOT translating SPL -> use `xsiam-splunk-to-xql`

Reference loading:
1. Always: `xsiam-shared/references/xql-core-reference.md`, `xsiam-shared/references/xql-datasets-core.md`, `references/correlation-rule-spec.md`
2. On-demand: same shared specialized files as xsiam-xql

Workflow:
1. Understand detection goal: behavior to detect, data source, severity
2. Select execution mode: `REAL_TIME` vs `SCHEDULED` (with crontab). Real-time for single-event, scheduled for aggregation/threshold.
3. Write XQL detection query following xql-core-reference stage order; query contains detection logic only
4. Build YAML wrapper with all correlation rule fields
5. Map MITRE ATT&CK technique IDs and tactics
6. Configure suppression: duration and fields to prevent alert flooding
7. Format output: complete `.yml` file with XQL as literal block scalar (`|`)

Quality checklist:
- XQL contains detection logic only (no alert metadata in query)
- Severity uses correct enum (`SEV_010_CRITICAL` through `SEV_060_INFORMATIONAL`, no `SEV_050`)
- `alert_name` and `name` match
- `global_rule_id` is valid UUID v4
- `dataset: alerts` always set
- `action: ALERTS` always set
- MITRE technique IDs valid (T#### or T####.###)
- `search_window` appropriate for execution mode
- Suppression fields make sense for the detection type
- Dynamic severity only when source lacks discrete severity field

**`references/correlation-rule-spec.md`** (~400 lines)

Expanded from current 75-line template:
- Complete YAML field reference: every field with type, required/optional, valid values, description
- Canonical field order matching XSIAM export format
- Execution modes: REAL_TIME vs SCHEDULED with crontab syntax and `search_window` interaction
- Severity mapping: static vs dynamic, with examples
- Suppression patterns: common field combinations by detection type
- MITRE mapping: `mitre_defs` structure, common technique/tactic pairs
- `alert_fields`, `alert_domain`, `alert_category`: when to use non-defaults

**`references/correlation-examples.md`** (~300 lines)

4 complete `.yml` examples:
1. **Real-time single event:** Critical canary alert -> immediate high-severity alert
2. **Scheduled threshold:** Failed auth spike (>= N in window) -> medium severity
3. **Scheduled aggregation with dynamic severity:** Third-party numeric score -> computed severity
4. **Real-time with suppression:** Repeated firewall blocks -> suppress by IP for 1h

Each example: complete `.yml` with inline annotations.

---

### Skill 3: xsiam-splunk-to-xql

**Purpose:** Translate Splunk SPL queries into equivalent XQL queries.

**SKILL.md** (~120 lines)

Trigger phrases: "translate SPL", "convert Splunk", "Splunk to XQL", "SPL to XQL", "migrate Splunk query", "translate this Splunk"

Scope boundaries:
- NOT writing XQL from scratch -> use `xsiam-xql`
- NOT building correlation rules -> use `xsiam-correlations`

Reference loading:
1. Always: `references/spl-to-xql-mapping.md`, `xsiam-shared/references/xql-core-reference.md`
2. On-demand: shared specialized files when translated query uses advanced features

Workflow:
1. Parse SPL: identify commands, functions, lookups, macros, field references
2. Map commands to XQL equivalents using the mapping table
3. Flag untranslatable constructs with manual workaround notes
4. Map `index=`/`sourcetype=` to appropriate XSIAM dataset or preset
5. Assemble XQL in correct stage order
6. Validate against xsiam-xql quality checklist

Quality checklist:
- Every SPL command accounted for (translated or flagged)
- No SPL syntax remnants in output
- Dataset mapping correct (not blind index name carry-over)
- XQL stage order follows recommendation
- Header comment includes "Translated from SPL" note

**`references/spl-to-xql-mapping.md`** (~350 lines)

- Command mapping table: `stats`->`comp`, `eval`->`alter`, `table`->`fields`, `where`->`filter`, `sort`->`sort`, `top`->`top`, `rex`->`arrayindex(regextract(...))`, `mvexpand`->`arrayexpand`, `mvjoin`->`arraystring`, `mvdedup`->`arraydistinct`, `inputlookup`->`union`, `iplocation`->`iploc`, `fillnull`->`replacenull`, `dedup`->`dedup`, `rename`->`alter...as`, `head`->`limit`, `eventstats`->`windowcomp`
- Index/sourcetype to dataset mapping: common Splunk source types -> XSIAM datasets
- Function mapping: SPL eval functions -> XQL alter functions
- Known untranslatable constructs with workarounds
- 3-4 side-by-side translation examples with annotations

---

## Token Budget

| Invocation | Always Loaded | Max On-Demand | Worst Case |
|---|---|---|---|
| xsiam-xql | ~22.5K | +9.8K | ~32K |
| xsiam-correlations | ~22K | +9.8K | ~32K |
| xsiam-splunk-to-xql | ~15K | +9.8K | ~25K |

Typical invocation for a standard query: ~22K tokens (down from current ~25K with significantly more content available on-demand).

## Plugin Integration

- Add `xsiam-correlations` and `xsiam-splunk-to-xql` to `marketplace.json`
- Update `plugin.json` version
- Existing skills (`xsiam-scripts`, `xsiam-integrations`, `xsiam-playbooks`, `xsiam-docs`) unchanged

## Migration

| Current File | Disposition |
|---|---|
| `skills/xsiam-xql/SKILL.md` | Replaced with new workflow, tiered loading, updated scope |
| `references/xql-syntax-reference.md` | Split into `xql-core-reference.md` + `xql-advanced-functions.md` |
| `references/xql-datasets.md` | Split into `xql-datasets-core.md` + `xql-datasets-extended.md` |
| `references/xql-examples.md` | Reorganized into cookbook format, stays skill-specific |
| `correlation-template.yml` | Absorbed into `correlation-rule-spec.md` |

No breaking changes. Old `xsiam-xql` trigger phrases still work. Correlation rule requests now match the dedicated `xsiam-correlations` skill.

## Build Order

1. Shared references (`xsiam-shared/references/`)
2. `xsiam-xql` (skill + examples)
3. `xsiam-correlations` (skill + spec + examples)
4. `xsiam-splunk-to-xql` (skill + mapping)

## Knowledge Source

All new content sourced from the Obsidian XSIAM knowledge vault, cross-referenced with existing skill content. Key vault directories:
- `xsiam-xql-language/` — 23 stages, 96 functions, operator reference
- `xsiam-build-xql-queries/` — Query Builder UI, XDM syntax, Splunk translation, federated search
- `xsiam-xql-query/` — XQL Query API (not included in skills per user decision)
- `xsiam-query-library/` — Query Library API (`call` stage context)
- `xsiam-xql-user-datasets/` — User datasets (BigQuery-backed)
- `xsiam-dataset-management/` — Dataset types, lookup datasets, cold storage
- `xsiam-content-examples/` — HTTP collector integration pattern
