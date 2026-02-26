---
name: xsiam-xql
description: >
  This skill should be used when the user asks to "write an XQL query",
  "create an XQL query", "hunt for threats", "search XSIAM data",
  "build a correlation rule", "query dataset", "XQL", "threat hunting query",
  or needs to generate Cortex Query Language queries for XSIAM from
  natural language descriptions.
version: 0.1.0
---

# XQL Query Generation

Generate Cortex Query Language (XQL) queries for XSIAM from natural language descriptions.

## Before Starting

Read the reference files to understand XQL syntax and available datasets:
- `references/xql-syntax-reference.md` — Complete XQL syntax, stages, operators, and functions
- `references/xql-datasets.md` — Available datasets, their fields, and presets

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

If unsure which dataset, ask the user what data source they're working with.

### 3. Build the Query

Construct the XQL query using pipe-separated stages in this order:
1. `dataset = ...` or `preset = ...` — data source selection
2. `| filter ...` — narrow results (time range, conditions)
3. `| alter ...` — compute new fields if needed
4. `| comp ...` — aggregate if needed (with `by` for grouping)
5. `| fields ...` — select output columns
6. `| sort ...` — order results
7. `| limit ...` — cap result count
8. `| dedup ...` — remove duplicates if needed

Not all stages are needed for every query. Use only what's required.

### 4. Query Construction Rules

**Filtering**:
- Use `_time` for time-based filtering
- String comparisons are case-sensitive by default; use `lowercase()` in alter for case-insensitive
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

Present each query with:
1. **Description** — what the query finds, in plain language
2. **Query** — the XQL query in a code block
3. **Expected output** — what columns/data the user will see
4. **Customization notes** — fields the user may want to adjust (time range, specific values)

### 6. For Correlation Rules

When generating correlation rules, additionally include:
- Alert name and severity mapping
- MITRE ATT&CK technique mapping if applicable
- Recommended threshold/frequency settings
- False positive considerations

## Common Query Templates

**Find process execution by hash**:
```xql
dataset = xdr_data
| filter action_file_sha256 = "HASH_VALUE"
| fields agent_hostname, agent_ip, action_process_image_name, action_file_path, _time
| sort desc _time
```

**Failed login spike detection**:
```xql
dataset = microsoft_azure_ad_sign_in_raw
| filter status = "Failure"
| comp count() as failed_count by user_principal_name
| filter failed_count > 10
| sort desc failed_count
```

**Network traffic to suspicious destination**:
```xql
dataset = panw_ngfw_traffic_raw
| filter dst in ("IP1", "IP2", "IP3")
| fields src, dst, dport, app, action, bytes_sent, _time
| sort desc _time
```

**Lateral movement detection**:
```xql
dataset = xdr_data
| filter action_type in ("PROCESS_EXECUTION", "REMOTE_PROCESS_EXECUTION")
| filter action_process_image_name in ("psexec.exe", "wmic.exe", "powershell.exe")
| comp count() as exec_count, count_distinct(agent_hostname) as host_count by actor_process_image_name
| filter host_count > 3
```

## Quality Checklist

Before delivering a query, verify:
- [ ] Dataset exists and is appropriate for the use case
- [ ] Field names are valid for the chosen dataset
- [ ] Filter conditions use correct operators (= for exact, contains for substring, ~= for regex)
- [ ] Aggregations include appropriate `by` clauses
- [ ] Time ranges are reasonable for the data volume
- [ ] Query is readable with clear stage separation
