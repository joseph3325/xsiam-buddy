---
name: xsiam-xql
description: >
  Use when the user needs to write XQL queries, hunt for threats, search XSIAM data,
  build a correlation rule, or query any XSIAM dataset. Trigger phrases: "write an
  XQL query", "create an XQL query", "threat hunting query", "XQL", "correlation rule",
  "query dataset", "search XSIAM data".
---

# XQL Query Generation

Generate Cortex Query Language (XQL) queries for XSIAM from natural language descriptions.

## When Not to Use

- **Writing automation logic** — use XSOAR Python scripts or integrations instead
- **Modifying data** — XQL is read-only; use XSOAR actions for data changes
- **Real-time streaming triggers** — use playbook triggers or integration event listeners instead

## Before Starting

Read all four reference files before generating any query:
- `references/xql-syntax-reference.md` — Complete XQL syntax, stages, operators, and functions
- `references/xql-datasets.md` — Available datasets, their fields, and presets
- `references/xql-examples.md` — Real production examples; model syntax and patterns from these
- `references/correlation-template.yml` — YAML wrapper structure for correlation rules

## Workflow

### 1. Understand the Query Goal

Determine what the user wants to find or analyze:
- **Threat hunting** — searching for specific IOCs, suspicious behavior, or attack patterns
- **Investigation** — analyzing a specific incident, timeline, or chain of events
- **Analytics** — aggregating data for dashboards, reports, or trend analysis
- **Correlation rule** — defining detection logic that generates alerts

### 2. Select the Right Dataset

Match the query goal to the appropriate dataset:
- Endpoint activity → `xdr_data` or `preset = xdr_event`
- Network traffic → `panw_ngfw_traffic_raw`
- Network threats → `panw_ngfw_threat_raw`
- Cloud operations → `cloud_audit_log`
- Azure AD activity → `microsoft_azure_ad_raw`
- Authentication → `microsoft_azure_ad_sign_in_raw` or `okta_raw`
- Alerts → `xdr_alerts`
- Incidents → `xdr_incidents`
- Cross-domain → use `preset` (xdr_event, network_story, cloud_story, identity_story)
- XDM-normalized field access → `datamodel dataset = <dataset>` (use `xdm.*` fields; see syntax reference)

If unsure which dataset, ask the user what data source they're working with.

### 3. Build the Query

Construct the XQL query using pipe-separated stages in this order:
1. `dataset = ...`, `preset = ...`, or `datamodel dataset = ...` — data source selection
2. `| filter ...` — narrow results (time range, conditions)
3. `| alter ...` — compute new fields if needed
4. `| comp ...` — aggregate if needed (with `by` for grouping)
5. `| fields ...` — select output columns
6. `| sort ...` — order results
7. `| limit ...` — cap result count
8. `| dedup ...` — remove duplicates if needed

Not all stages are needed for every query. Use only what's required.

**Performance:** For wide datasets or queries with joins/aggregations, add `| fields ...` immediately after `| filter ...` to drop unneeded columns early. It is correct and intentional for `fields` to appear twice — once for early column reduction, once for final output selection.

### 4. Query Construction Rules

**Filtering**:
- Use `_time` for time-based filtering
- `=` string comparisons are case-sensitive; use `lowercase()` in `alter` before comparing with `=` for case-insensitive exact matching
- `contains` is case-insensitive by default — no need for `lowercase()` before `contains`
- Use `incidr()` for IP range matching
- Use `~=` for regex matching
- Chain conditions with `and` / `or`

**Aggregation**:
- `comp count() by field` — count by group
- `comp count_distinct(field) by group_field` — unique counts
- `comp values(field) by group_field` — list unique values
- `comp sum(numeric_field) by group_field` — totals
- Always include a `by` clause unless you want a single global aggregate

**Joins**:
- Use `join` to correlate events across datasets
- Specify join type: `inner`, `left`, `right`, `cross`
- Always alias the subquery: `as alias`
- Join on matching fields: `alias.field = field`

**Time bucketing**:
- `bin _time span = 1h` for hourly buckets
- Combine with `comp count() by _time` for time-series data

### 5. Output Format

**Default — plain query:**

Return the XQL in a single code block. Always include the standard header comment block at the top of the XQL:

```xql
// Title: <meaningful name for this query>
// Description: <one sentence — what this finds>
// Author: xsiam-buddy
// Dataset(s): <dataset or preset used>
// Query last modified: <today's date as YYYY-MM-DD>
// Vendor Reference: <link to vendor docs, or N/A>

dataset = ...
| filter ...
```

No prose description, expected output section, or customization notes. The header comments carry that context.

**Correlation rule:**

When the user explicitly asks for a correlation rule (not just a detection query), output a populated `.yml` file using the structure from `references/correlation-template.yml`. Embed the XQL inside `xql_query` as a YAML literal block scalar (`|`). Do not output a separate plain XQL block.

## Common Query Templates

**Find process execution by hash:**
```xql
// Title: Process Execution by Hash
// Description: Find all endpoints that executed a process matching a given SHA256 hash
// Author: xsiam-buddy
// Dataset(s): xdr_data
// Query last modified: <today's date as YYYY-MM-DD>
// Vendor Reference: N/A

dataset = xdr_data
| filter action_file_sha256 = "HASH_VALUE"
| fields agent_hostname, agent_ip, action_process_image_name, action_file_path, _time
| sort desc _time
```

**Failed login spike detection:**
```xql
// Title: Failed Login Spike Detection
// Description: Identify users with an unusual number of failed login attempts — potential brute force
// Author: xsiam-buddy
// Dataset(s): microsoft_azure_ad_sign_in_raw
// Query last modified: <today's date as YYYY-MM-DD>
// Vendor Reference: N/A

dataset = microsoft_azure_ad_sign_in_raw
| filter status = "Failure"
| comp count() as failed_count by user_principal_name
| filter failed_count > 10
| sort desc failed_count
```

**Network traffic to suspicious destination:**
```xql
// Title: Network Traffic to Suspicious Destination
// Description: Find all connections to a list of known suspicious IPs
// Author: xsiam-buddy
// Dataset(s): panw_ngfw_traffic_raw
// Query last modified: <today's date as YYYY-MM-DD>
// Vendor Reference: N/A

dataset = panw_ngfw_traffic_raw
| filter dst in ("IP1", "IP2", "IP3")
| fields src, dst, dport, app, action, bytes_sent, _time
| sort desc _time
```

**Lateral movement detection:**
```xql
// Title: Lateral Movement Detection
// Description: Detect suspicious remote execution tools used across multiple hosts
// Author: xsiam-buddy
// Dataset(s): xdr_data
// Query last modified: <today's date as YYYY-MM-DD>
// Vendor Reference: N/A

dataset = xdr_data
| filter action_type in ("PROCESS_EXECUTION", "REMOTE_PROCESS_EXECUTION")
| filter action_process_image_name in ("psexec.exe", "wmic.exe", "powershell.exe")
| comp count() as exec_count, count_distinct(agent_hostname) as host_count by actor_process_image_name
| filter host_count > 3
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Quoting ENUM values: `event_type = "ENUM.EVENT_LOG"` | Unquoted: `event_type = ENUM.EVENT_LOG` |
| Wrong dataset name for Azure AD datamodel queries: `microsoft_azure_ad_raw` | Use `msft_azure_ad_raw` with `datamodel dataset =` |
| No early `fields` on wide joins — all columns flow into join/comp | Add `| fields ...` after filter to drop unused columns first |
| Missing pipe before `dataset` with `config timeframe` | Must be: `config timeframe = 7d` then `| dataset = ...` |
| Adding `alter xdm.alert.severity` or `alter mitre_technique` in a correlation rule XQL | Alert metadata goes in the YAML wrapper — only use `alter xdm.alert.severity` when the source has no discrete severity field and the numeric score IS the severity signal |
| Using `lowercase()` before `contains` | `contains` is already case-insensitive — `lowercase()` is only needed with `=` |

## Quality Checklist

Before delivering, verify:
- [ ] Header comments present and fully populated — no placeholder values like `<...>` remaining
- [ ] `// Query last modified:` has today's date
- [ ] Dataset exists and is appropriate for the use case
- [ ] Field names match those in `references/xql-datasets.md` for the chosen dataset
- [ ] Filter conditions use correct operators (`=` exact, `contains` substring, `~=` regex)
- [ ] Aggregations include `by` clause unless a global aggregate is intended
- [ ] Time range specified via `config timeframe` preamble or `_time` filter
- [ ] For wide investigation queries: consider `| view column order = populated` to suppress empty columns
- [ ] For correlation rules: all YAML fields populated, real UUID v4 in `global_rule_id`, no placeholder text
