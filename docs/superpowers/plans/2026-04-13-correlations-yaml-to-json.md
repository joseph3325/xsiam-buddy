# Correlation Skill YAML→JSON Migration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite the `xsiam-correlations` skill to output import-ready JSON files matching the real XSIAM correlation rule export format.

**Architecture:** Three reference/skill files get full rewrites (spec, examples, SKILL.md) to target JSON output with full export fidelity. CLAUDE.md and plugin manifests get minor updates. No runtime code — this is a content-only plugin.

**Tech Stack:** Markdown, JSON (output format), XQL (detection query language)

**Design Spec:** `docs/superpowers/specs/2026-04-13-correlations-yaml-to-json-design.md`

---

### Task 1: Rewrite `correlation-rule-spec.md`

**Files:**
- Rewrite: `skills/xsiam-correlations/references/correlation-rule-spec.md`

This is the JSON schema reference that the skill reads before generating output. It replaces the current YAML-based spec entirely.

- [ ] **Step 1: Replace the entire file with the JSON-based spec**

Write the complete new `correlation-rule-spec.md` with these sections:

```markdown
# Correlation Rule JSON Specification

This document defines the complete JSON schema for Cortex XSIAM correlation rules.
Correlation rules are exported and imported as JSON arrays containing one or more
rule objects. Each rule wraps XQL detection logic with alert metadata, severity,
MITRE ATT&CK mapping, and suppression configuration.

---

## Output Structure

Correlation rules are delivered as a JSON array containing a single rule object:

\`\`\`json
[
  {
    "rule_id": 100,
    "name": "Rule Name",
    ...
  }
]
\`\`\`

The array wrapper is required — XSIAM's import function expects it even for single rules.

---

## Canonical Field Order

Fields must appear in this order to match XSIAM export format:

\`\`\`
rule_id
name
severity
xql_query
is_enabled
description
alert_name
alert_category
alert_type
alert_description
alert_domain
alert_fields
execution_mode
search_window
simple_schedule
timezone
crontab
suppression_enabled
suppression_duration
suppression_fields
dataset
user_defined_severity
user_defined_category
mitre_defs
investigation_query_link
drilldown_query_timeframe
mapping_strategy
action
lookup_mapping
\`\`\`

---

## Field Reference Table

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `rule_id` | integer | **Yes** | — | Numeric rule identifier. Prompt user for this value if not provided. |
| `name` | string | **Yes** | — | Rule display name shown in the XSIAM rules list. |
| `severity` | string | **Yes** | — | Severity enum (e.g., `SEV_030_MEDIUM`) or `"User Defined"` for dynamic severity. |
| `xql_query` | string | **Yes** | — | The XQL detection logic as a string. Newlines are `\n` in JSON. |
| `is_enabled` | boolean | **Yes** | `true` | Whether the rule is active on import. |
| `description` | string \| null | No | `null` | Detailed description of what the rule detects. |
| `alert_name` | string | **Yes** | — | Alert name displayed in the alerts table. Can use `$field` variable references. |
| `alert_category` | string | **Yes** | `"OTHER"` | Category classification. Use `"User Defined"` with `user_defined_category` for dynamic category. |
| `alert_type` | null | No | `null` | Reserved field. Always `null` in current exports. |
| `alert_description` | string \| null | No | `null` | Description shown on the alert. Can use `$field` variable references. |
| `alert_domain` | string | **Yes** | — | Domain classification enum: `DOMAIN_SECURITY`, `DOMAIN_MSIAM_IMPACT_REPORTS`, `DOMAIN_EXTERNAL`, etc. |
| `alert_fields` | object \| null | No | `{}` | Custom field mappings as key-value pairs: `{"display_name": "xdm.field.path"}`. Empty object `{}` if no custom mappings. |
| `execution_mode` | string | **Yes** | — | `"REAL_TIME"` or `"SCHEDULED"`. See Execution Mode Guide. |
| `search_window` | string \| null | Conditional | `null` | Lookback window. Required for `SCHEDULED`. Format: `"1 hours"`, `"15 minutes"`, `"30 minutes"`. `null` for `REAL_TIME`. |
| `simple_schedule` | string \| null | Conditional | `null` | Human-readable schedule. Required for `SCHEDULED`. Mirrors `crontab`. `null` for `REAL_TIME`. |
| `timezone` | string \| null | Conditional | `null` | Timezone for scheduled rules (e.g., `"UTC"`, `"America/New_York"`). `null` for `REAL_TIME`. |
| `crontab` | string \| null | Conditional | `null` | Standard 5-field cron expression. Required for `SCHEDULED`. `null` for `REAL_TIME`. |
| `suppression_enabled` | boolean | **Yes** | — | Whether duplicate alert suppression is active. |
| `suppression_duration` | string \| null | Conditional | `null` | How long to suppress duplicates. Required when `suppression_enabled: true`. Format: `"1 hours"`, `"30 minutes"`. |
| `suppression_fields` | array \| null | Conditional | `null` | Field names that define a "duplicate" alert. Required when `suppression_enabled: true`. |
| `dataset` | string | **Yes** | `"alerts"` | Always `"alerts"` for correlation rules. |
| `user_defined_severity` | string \| null | Conditional | `null` | Field reference string (e.g., `"xdm.alert.severity"`) when `severity` is `"User Defined"`. `null` for static severity. |
| `user_defined_category` | string \| null | Conditional | `null` | Field reference string (e.g., `"xdm.alert.subcategory"`) when `alert_category` is `"User Defined"`. `null` for static category. |
| `mitre_defs` | object | **Yes** | `{}` | MITRE ATT&CK mapping. Tactic label → technique label array. See MITRE ATT&CK Mapping section. |
| `investigation_query_link` | string \| null | No | `""` | XQL query or link for investigation drilldown. |
| `drilldown_query_timeframe` | string | No | `"ALERT"` | Time window for the drilldown query. `"ALERT"` uses the alert's time range. |
| `mapping_strategy` | string | No | `"AUTO"` | How alert fields are populated. `"AUTO"` for automatic XDM mapping. |
| `action` | string | **Yes** | `"ALERTS"` | Always `"ALERTS"` — correlation rules produce alerts. |
| `lookup_mapping` | array \| null | No | `null` | Lookup dataset mapping configuration. `null` when not used. |

---

## Severity Guide

### Enum Values

XSIAM uses a fixed set of severity enum values. There is **no SEV_050** — this is
intentional and not a gap.

| Enum Value | Meaning | Use When |
|-----------|---------|----------|
| `SEV_010_CRITICAL` | Critical | Confirmed active compromise, data breach, or ransomware execution |
| `SEV_020_HIGH` | High | Strong indicator of compromise requiring immediate investigation |
| `SEV_030_MEDIUM` | Medium | Suspicious activity warranting investigation within SLA |
| `SEV_040_LOW` | Low | Anomalous but potentially benign activity for triage |
| `SEV_060_INFORMATIONAL` | Informational | Audit events, policy violations, or telemetry for enrichment |

### Static vs User-Defined Severity

**Static severity** — set a severity enum directly:
\`\`\`json
{
  "severity": "SEV_030_MEDIUM",
  "user_defined_severity": null
}
\`\`\`
Use when the detection always warrants the same response priority.

**User-defined severity** — computed dynamically from a field:
\`\`\`json
{
  "severity": "User Defined",
  "user_defined_severity": "xdm.alert.severity"
}
\`\`\`
Use when the source data has a severity or score field that should drive alert severity.
The `user_defined_severity` value is a field reference string pointing to the XDM or
computed field that contains the severity value. Severity computation happens in the
XQL query (via `alter`) or through XDM datamodel mapping.

**Decision criteria:**
- Source data has a discrete severity field (e.g., vendor alert severity) → User Defined with field reference
- Source data has a numeric score (e.g., Darktrace percentscore) → User Defined; compute severity via `if()` in XQL, map to `xdm.alert.severity`
- Detection is custom logic with no source severity → Static severity enum

---

## Execution Mode Guide

### REAL_TIME

Triggered on each matching event as it is ingested.

**JSON field values for REAL_TIME:**
\`\`\`json
{
  "execution_mode": "REAL_TIME",
  "search_window": null,
  "simple_schedule": null,
  "timezone": null,
  "crontab": null
}
\`\`\`

**XQL constraints for REAL_TIME:**
- Only supports: `dataset`, `datamodel`, `filter`, `alter`, `fields`, and `config case_sensitive`
- Must include at least one `filter` stage
- `comp`, `sort`, `dedup`, `join`, `union` are NOT supported
- `json_extract_scalar_array`, `parse_epoch`, `time_frame_end` are NOT supported

**Best for:** canary/honeypot triggers, known-bad IOC matches, critical policy violations, vendor alert forwarding

### SCHEDULED

Runs on a cron schedule with a lookback window.

**JSON field values for SCHEDULED:**
\`\`\`json
{
  "execution_mode": "SCHEDULED",
  "search_window": "15 minutes",
  "simple_schedule": "5 minutes",
  "timezone": "UTC",
  "crontab": "*/5 * * * *"
}
\`\`\`

**Guidelines:**
- `search_window` should be at least 2x the cron interval to avoid gaps
- Use `time_frame_end()` instead of `current_time()` for time calculations

**Best for:** brute force / password spray, data exfiltration volume thresholds, anomaly detection

### Duration Format

Durations use human-readable strings — NOT shorthand:

| Format | Meaning |
|--------|---------|
| `"5 minutes"` | 5 minutes |
| `"10 minutes"` | 10 minutes |
| `"15 minutes"` | 15 minutes |
| `"30 minutes"` | 30 minutes |
| `"1 hours"` | 1 hour |
| `"4 hours"` | 4 hours |
| `"24 hours"` | 24 hours |

### simple_schedule ↔ crontab Correspondence

| simple_schedule | crontab |
|-----------------|---------|
| `"5 minutes"` | `*/5 * * * *` |
| `"10 minutes"` | `*/10 * * * *` |
| `"15 minutes"` | `*/15 * * * *` |
| `"20 minutes"` | `*/20 * * * *` |
| `"30 minutes"` | `*/30 * * * *` |
| `"1 hour"` | `*/60 * * * *` |
| `"4 hours"` | `0 */4 * * *` |
| `"24 hours"` | `0 0 * * *` |

---

## Suppression Patterns

| Detection Type | Suppress By | Recommended Duration | Rationale |
|---------------|-------------|---------------------|-----------|
| Brute force / password spray | `source_ip` | `"1 hours"` | Same source attacking = same incident |
| Lateral movement | `hostname` | `"2 hours"` | Same compromised host = same incident |
| Data exfiltration | `user_name` | `"4 hours"` | Same user exfiltrating = same incident |
| Malware detection | `file_hash` | `"24 hours"` | Same malware = same incident |
| Repeated failed auth | `user_name`, `source_ip` | `"30 minutes"` | Same user from same source = same incident |
| Canary / honeypot trigger | `alert_name` | `"1 hours"` | Same canary = same incident |
| Firewall block (repeated) | `source_ip`, `destination_port` | `"1 hours"` | Same source hitting same port = same incident |
| Privilege escalation | `user_name` | `"2 hours"` | Same user escalating = same incident |
| Suspicious process execution | `hostname`, `process_name` | `"1 hours"` | Same process on same host = same incident |

When `suppression_enabled` is `false`, set both `suppression_duration` and `suppression_fields` to `null`.

---

## MITRE ATT&CK Mapping

### Structure

The `mitre_defs` field maps tactic labels to arrays of technique labels. Both use the
format `"ID - Human Readable Name"`:

\`\`\`json
{
  "mitre_defs": {
    "TA0006 - Credential Access": [
      "T1110 - Brute Force",
      "T1110.003 - Password Spraying"
    ]
  }
}
\`\`\`

### Format Rules

- Tactic keys: `"TA#### - Tactic Name"` (e.g., `"TA0006 - Credential Access"`)
- Technique values: `"T#### - Technique Name"` or `"T####.### - Sub-technique Name"`
- A single tactic can map to multiple techniques (array)
- A rule can map to multiple tactics (multiple keys)
- Empty object `{}` when no MITRE mapping applies

### Common Mappings Reference

| Detection Category | Tactic Label | Technique Label(s) |
|-------------------|-------------|-------------------|
| Brute force | `TA0006 - Credential Access` | `T1110 - Brute Force` |
| Password spray | `TA0006 - Credential Access` | `T1110.003 - Password Spraying` |
| Valid accounts | `TA0001 - Initial Access` | `T1078 - Valid Accounts` |
| Valid accounts (persistence) | `TA0003 - Persistence` | `T1078 - Valid Accounts` |
| MFA bypass | `TA0006 - Credential Access` | `T1556.006 - Multi-Factor Authentication` |
| MFA bypass (evasion) | `TA0005 - Defense Evasion` | `T1556.006 - Multi-Factor Authentication` |
| Lateral movement (RDP) | `TA0008 - Lateral Movement` | `T1021.001 - Remote Desktop Protocol` |
| Lateral movement (SMB) | `TA0008 - Lateral Movement` | `T1021.002 - SMB/Windows Admin Shares` |
| Data exfiltration (network) | `TA0010 - Exfiltration` | `T1048 - Exfiltration Over Alternative Protocol` |
| Privilege escalation | `TA0004 - Privilege Escalation` | `T1068 - Exploitation for Privilege Escalation` |
| Account manipulation | `TA0003 - Persistence` | `T1098 - Account Manipulation` |
| Suspicious process | `TA0002 - Execution` | `T1059 - Command and Scripting Interpreter` |
| Masquerading | `TA0005 - Defense Evasion` | `T1036.005 - Match Legitimate Resource Name or Location` |
| Network scanning | `TA0043 - Reconnaissance` | `T1046 - Network Service Scanning` |
| Honeypot / canary | `TA0043 - Reconnaissance` | `T1595 - Active Scanning` |
| C2 communication | `TA0011 - Command and Control` | `T1071 - Application Layer Protocol` |
| Traffic signaling | `TA0005 - Defense Evasion` | `T1205 - Traffic Signaling` |

---

## XQL Query as JSON String

In JSON, the XQL query is a regular string value. Newlines within the query become `\n`
characters in the JSON string:

\`\`\`json
{
  "xql_query": "dataset = xdr_data\n| filter event_type = ENUM.EVENT_LOG\n  and action_evtlog_event_id = 4625\n| comp count() as failed_count by agent_hostname\n| filter failed_count >= 10"
}
\`\`\`

When writing the query, compose it as multi-line XQL first, then serialize it into the
JSON string with `\n` for line breaks. XQL comments use `//` and are preserved in the string.

### What Goes in XQL vs JSON Fields

| Concern | Where It Lives | Example |
|---------|---------------|---------|
| Event filtering | XQL | `filter event_type = ENUM.EVENT_LOG` |
| Field extraction | XQL | `alter user = event_data -> TargetUserName` |
| Aggregation / threshold | XQL | `comp count() ... \| filter count >= 5` |
| Dynamic severity computation | XQL | `alter xdm.alert.severity = if(score >= 90, "CRITICAL", ...)` |
| Alert name | JSON `alert_name` | `"Failed Auth Spike"` or `"$xdm.event.description"` |
| Severity (static) | JSON `severity` | `"SEV_030_MEDIUM"` |
| Severity (dynamic) | JSON `severity` + `user_defined_severity` | `"User Defined"` + `"xdm.alert.severity"` |
| MITRE mapping | JSON `mitre_defs` | `{"TA0006 - Credential Access": [...]}` |
| Suppression | JSON `suppression_*` | `"suppression_fields": ["source_ip"]` |

---

## alert_domain Values

| Value | Use When |
|-------|----------|
| `DOMAIN_SECURITY` | General security alerts, vendor alert forwarding |
| `DOMAIN_MSIAM_IMPACT_REPORTS` | Threat intelligence impact reports (e.g., Unit42) |
| `DOMAIN_EXTERNAL` | External/third-party sourced alerts |

---

## Field Variable References

Several JSON fields support `$field` variable substitution at alert creation time:

- `alert_name` — e.g., `"$xdm.event.description"` dynamically sets the alert name from a field
- `alert_description` — e.g., `"$xdm.alert.description"` dynamically sets the alert description

**Syntax rules:**
- Standard fields: `$fieldName` (alphanumeric + underscore)
- XDM fields: `$xdm.email.recipient` (dots allowed)
- Special characters: `$\`field name\`` (backtick-wrapped)
- Quoted text `"$field"` inside the XQL query is treated as literal (no replacement)
- Cannot be purely numeric: `$123_data` works, `$456` does not
```

- [ ] **Step 2: Commit**

```bash
git add skills/xsiam-correlations/references/correlation-rule-spec.md
git commit -m "feat(xsiam-correlations): rewrite correlation spec for JSON export format"
```

---

### Task 2: Rewrite `correlation-examples.md`

**Files:**
- Rewrite: `skills/xsiam-correlations/references/correlation-examples.md`

Convert all 4 existing examples to JSON format and add companion markdown summaries. Preserve the same detection logic (XQL queries) but wrap them in the correct JSON structure.

- [ ] **Step 1: Replace the entire file with JSON-based examples**

Write the complete new `correlation-examples.md` with these 4 examples. Each example has:
1. A use case description
2. Key patterns highlighted
3. The complete JSON (wrapped in `[{...}]` array)
4. A "Design Decisions" companion summary

The 4 examples to convert:

**Example 1: Real-Time Single Event (Canary Alert)**
- `execution_mode`: `"REAL_TIME"`
- All schedule fields: `null`
- Static severity: `"SEV_020_HIGH"`
- Suppression: enabled, by `alert_name`, `"1 hours"`
- MITRE: `"TA0043 - Reconnaissance": ["T1595 - Active Scanning"]`
- `alert_fields`: `{}`
- XQL: the Thinkst Canary query from current examples (filter + alter + fields only — valid for REAL_TIME)

**Example 2: Scheduled Threshold (Failed Auth Spike)**
- `execution_mode`: `"SCHEDULED"`
- `search_window`: `"15 minutes"`, `simple_schedule`: `"5 minutes"`, `crontab`: `"*/5 * * * *"`, `timezone`: `"UTC"`
- Static severity: `"SEV_030_MEDIUM"`
- Suppression: enabled, by `source_ip`, `"30 minutes"`
- MITRE: `"TA0006 - Credential Access": ["T1110 - Brute Force", "T1110.003 - Password Spraying"]`
- XQL: the failed auth aggregation query from current examples

**Example 3: Real-Time with User-Defined Severity (Vendor Alert Forwarding)**
- Based on the Graph API Alerts vault export
- `execution_mode`: `"REAL_TIME"`
- `severity`: `"User Defined"`, `user_defined_severity`: `"xdm.alert.severity"`
- `alert_name`: `"$xdm.event.description"` (dynamic)
- `alert_category`: `"User Defined"`, `user_defined_category`: `"xdm.alert.subcategory"`
- `alert_description`: `"$xdm.alert.description"` (dynamic)
- Suppression: disabled (all suppression fields `null`)
- `alert_fields`: `{"email": "xdm.source.user.upn", "accountid": "xdm.source.user.upn", "fw_email_subject": "xdm.email.subject", "eventdescriptions": "xdm.event.description", "fw_email_recipient": "xdm.email.recipients"}`

**Example 4: Scheduled with Dynamic Severity (Third-Party Score)**
- Based on the Darktrace example from current examples
- `execution_mode`: `"SCHEDULED"`
- `search_window`: `"1 hours"`, `simple_schedule`: `"1 hour"`, `crontab`: `"*/60 * * * *"`, `timezone`: `"UTC"`
- `severity`: `"User Defined"`, `user_defined_severity`: `"xdm.alert.severity"`
- XQL computes `xdm.alert.severity` via `if()` chain on `percentscore`
- Suppression: enabled, by `xdm.alert.original_threat_name`, `"2 hours"`
- MITRE: `"TA0011 - Command and Control": ["T1071 - Application Layer Protocol"]`

Each JSON example must:
- Be wrapped in `[{...}]` array
- Include ALL fields (including nulls) in canonical order
- Use `\n` for newlines in XQL strings
- Use the human-readable duration format (`"1 hours"`, `"15 minutes"`)

Each companion summary must explain:
- Why this execution mode was chosen
- Why this severity pattern was chosen
- Why these suppression fields/duration were chosen
- Why these MITRE mappings were chosen

- [ ] **Step 2: Commit**

```bash
git add skills/xsiam-correlations/references/correlation-examples.md
git commit -m "feat(xsiam-correlations): convert all examples to JSON export format"
```

---

### Task 3: Rewrite `SKILL.md`

**Files:**
- Rewrite: `skills/xsiam-correlations/SKILL.md`

Update the frontmatter, scope, workflow, output format, and quality checklist to target JSON output.

- [ ] **Step 1: Replace the entire SKILL.md with the JSON-targeting version**

Key changes to make:

**Frontmatter:**
```yaml
---
name: xsiam-correlations
description: >
  This skill should be used when the user asks to "create a correlation rule",
  "build a detection rule", "write a detection", "XSIAM alert rule",
  "correlation JSON", "detection engineering", "build a detection for",
  or needs to generate correlation rule JSON files with embedded XQL detection logic.
version: 2.0.0
---
```

**Scope section:**
- Change `.yml` → `.json` throughout
- Change "YAML wrapper" → "JSON object"
- Change "literal block scalar" → "JSON string"

**Before Starting section:** unchanged (same reference files)

**Workflow updates:**

- Step 1 (Understand the Detection Goal): unchanged
- Step 2 (Select Execution Mode): update the table to show `null` fields for REAL_TIME, human-readable durations for SCHEDULED, add `simple_schedule` and `timezone`
- Step 3 (Write the XQL Detection Query): unchanged (same XQL stage order)
- Step 4: rename from "Build the YAML Wrapper" to "Build the JSON Object"
  - Prompt user for `rule_id` (integer) if not provided
  - Set `is_enabled: true` by default
  - Include all fields in canonical order, including null fields
  - Document the two severity patterns (static vs User Defined)
  - Document `alert_fields` as an object/dict
- Step 5 (Map MITRE ATT&CK): update format to tactic→technique with full labels
- Step 6 (Configure Suppression): update duration format to human-readable, set null fields when disabled
- Step 7: rename from "Format Output" to "Deliver Output"
  - JSON array `[{...}]` with all fields in canonical order
  - XQL as JSON string with `\n` for newlines
  - Companion markdown summary block with design decisions
  - No inline comments (JSON doesn't support them)

**Output Format section:**
- Show a JSON template instead of YAML template
- Describe the companion markdown summary

**Quality Checklist:**
Remove:
- YAML literal block scalar checks
- YAML field order conventions
- `global_rule_id` UUID validation
- `alert_name` must match `name`

Add:
1. Output is valid JSON (no trailing commas, proper quoting)
2. Output wrapped in array `[{...}]`
3. All fields present in canonical order (including null fields)
4. `rule_id` is an integer (prompted from user if not provided)
5. `mitre_defs` uses tactic→technique with full human-readable labels
6. Duration strings use human-readable format (`"1 hours"`, `"15 minutes"`)
7. `search_window`, `simple_schedule`, `timezone`, `crontab` are all `null` for REAL_TIME
8. XQL for REAL_TIME rules uses only supported stages (no `comp`, `sort`, `dedup`, `join`)
9. Companion markdown summary included with design decisions

Keep:
1. Severity enum validation (no SEV_050)
2. `dataset` always `"alerts"`
3. `action` always `"ALERTS"`
4. XQL contains detection logic only (exception: dynamic severity via `if()` or `alter`)
5. Suppression fields make sense for the detection type

- [ ] **Step 2: Commit**

```bash
git add skills/xsiam-correlations/SKILL.md
git commit -m "feat(xsiam-correlations): rewrite skill workflow for JSON output v2.0.0"
```

---

### Task 4: Update `CLAUDE.md`

**Files:**
- Modify: `CLAUDE.md:29` (Plugin Architecture tree)
- Modify: `CLAUDE.md:82-84` (XQL Skill Family table)
- Modify: `CLAUDE.md:61` (How Skills Work paragraph)

- [ ] **Step 1: Update the Plugin Architecture tree comment**

Change line 29:
```
  xsiam-correlations/  # Correlation rule .yml files with embedded XQL
```
to:
```
  xsiam-correlations/  # Correlation rule JSON files with embedded XQL
```

- [ ] **Step 2: Update the XQL Skill Family table**

Change the xsiam-correlations row output column from:
```
| xsiam-correlations | Detection rules (correlation rule engineering) | Complete `.yml` with embedded XQL |
```
to:
```
| xsiam-correlations | Detection rules (correlation rule engineering) | Complete `.json` with embedded XQL |
```

- [ ] **Step 3: Update the How Skills Work paragraph**

Change line 61:
```
Skills never generate code outside of YAML output files. All XSIAM content is delivered as unified `.yml` files (Python embedded inside YAML via `script: |-` for scripts, `script.script: |-` for integrations).
```
to:
```
Skills never generate code outside of YAML/JSON output files. All XSIAM content is delivered as unified `.yml` files (Python embedded inside YAML via `script: |-` for scripts, `script.script: |-` for integrations), except correlation rules which are delivered as `.json` files matching the XSIAM export/import format.
```

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for correlation skill JSON output"
```

---

### Task 5: Version Bump

**Files:**
- Modify: `.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Bump version in plugin.json**

Change `"version": "0.3.0"` to `"version": "0.4.0"` in `.claude-plugin/plugin.json`.

(Plugin version bumps to 0.4.0; the skill's internal version goes to 2.0.0 in its frontmatter, which was handled in Task 3.)

- [ ] **Step 2: Bump version in marketplace.json**

Change `"version": "0.3.0"` to `"version": "0.4.0"` in `.claude-plugin/marketplace.json` under `metadata`.

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "chore: bump plugin version to 0.4.0"
```

---

### Task 6: Final Review

- [ ] **Step 1: Verify all files are consistent**

Read through each modified file and check:
- `correlation-rule-spec.md` field order matches the examples in `correlation-examples.md`
- `SKILL.md` workflow references match the spec field names and types
- `SKILL.md` quality checklist covers all the patterns shown in examples
- `CLAUDE.md` references are consistent with the new format
- No stale YAML references remain in any correlation skill file

- [ ] **Step 2: Verify the skill description triggers in SKILL.md frontmatter**

Confirm the `description` field includes `"correlation JSON"` and no longer mentions `"correlation YAML"`.

- [ ] **Step 3: Commit any fixes found during review**

If any inconsistencies were found, fix them and commit:
```bash
git add -A
git commit -m "fix(xsiam-correlations): address review findings from final consistency check"
```
