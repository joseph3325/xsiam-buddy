# Correlation Rule JSON Specification

Cortex XSIAM exports and imports correlation rules as **JSON arrays**. Each rule is a
JSON object inside a single-element array. This specification defines the complete field
schema, canonical field order, and value constraints for correlation rule JSON files.

When generating a correlation rule, always produce a valid JSON array containing one rule
object. When the user provides multiple rules, each rule is a separate object in the array.

---

## Output Structure

Every correlation rule file is a JSON array wrapper containing one or more rule objects:

```json
[
  {
    "rule_id": 123,
    "name": "Detection Name",
    ...
  }
]
```

- The outer container is always a JSON array `[...]`
- Each rule is a JSON object `{...}` inside the array
- Most exports contain a single rule; multi-rule arrays are valid
- All string values use double quotes (standard JSON)
- Newlines inside string values (e.g., `xql_query`) are encoded as `\n`

---

## Canonical Field Order

Fields must appear in this exact order to match XSIAM exports. Maintaining this order
ensures clean diffs and reliable round-trip import/export.

```
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
```

---

## Field Reference Table

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `rule_id` | integer | **Yes** | — | Numeric identifier for the rule. Prompt the user if not provided. Must be unique within the tenant. |
| `name` | string | **Yes** | — | Display name shown in the correlation rules list. |
| `severity` | string | **Yes** | — | Severity enum value (e.g., `"SEV_030_MEDIUM"`) or `"User Defined"` for dynamic severity. See Severity Guide. |
| `xql_query` | string | **Yes** | — | The detection XQL query. Newlines are encoded as `\n` in JSON. See XQL Query as JSON String. |
| `is_enabled` | boolean | **Yes** | `true` | Whether the rule is active. Set to `false` to disable without deleting. |
| `description` | string \| null | Optional | `null` | Human-readable description of what the rule detects and why it matters. |
| `alert_name` | string | **Yes** | — | Name displayed on generated alerts. Supports `$field` variable references. |
| `alert_category` | string | **Yes** | `"OTHER"` | Alert category. Use a fixed value like `"OTHER"` or `"User Defined"` for dynamic category. |
| `alert_type` | null | Optional | `null` | Reserved field. Always set to `null`. |
| `alert_description` | string \| null | Optional | `null` | Description displayed on generated alerts. Supports `$field` variable references. |
| `alert_domain` | string | **Yes** | — | Domain classification for the alert. See alert_domain Values. |
| `alert_fields` | object \| null | Optional | `{}` | Custom alert field mappings as key-value pairs: `{"display_name": "xdm.field.path"}`. |
| `execution_mode` | string | **Yes** | — | `"REAL_TIME"` or `"SCHEDULED"`. See Execution Mode Guide. |
| `search_window` | string \| null | Conditional | — | Lookback window for SCHEDULED rules (e.g., `"1 hours"`, `"15 minutes"`). Required for SCHEDULED, `null` for REAL_TIME. |
| `simple_schedule` | string \| null | Conditional | — | Human-readable schedule mirroring the crontab. Required for SCHEDULED, `null` for REAL_TIME. |
| `timezone` | string \| null | Conditional | — | Timezone for the schedule (e.g., `"UTC"`). Required for SCHEDULED, `null` for REAL_TIME. |
| `crontab` | string \| null | Conditional | — | Standard 5-field cron expression. Required for SCHEDULED, `null` for REAL_TIME. |
| `suppression_enabled` | boolean | **Yes** | — | Whether duplicate alert suppression is active. |
| `suppression_duration` | string \| null | Conditional | — | How long to suppress duplicates (e.g., `"1 hours"`). Required when suppression is enabled, `null` when disabled. |
| `suppression_fields` | array \| null | Conditional | — | Fields that define a "duplicate" alert. Required when suppression is enabled, `null` when disabled. |
| `dataset` | string | **Yes** | `"alerts"` | Always `"alerts"` for correlation rules. |
| `user_defined_severity` | string \| null | Conditional | `null` | XDM field reference for dynamic severity. Required when severity is `"User Defined"`, `null` otherwise. |
| `user_defined_category` | string \| null | Conditional | `null` | XDM field reference for dynamic category. Required when alert_category is `"User Defined"`, `null` otherwise. |
| `mitre_defs` | object | **Yes** | `{}` | MITRE ATT&CK mapping. Keys are tactic labels, values are arrays of technique labels. See MITRE ATT&CK Mapping. |
| `investigation_query_link` | string \| null | Optional | `""` | Link to an investigation query for deeper analysis. |
| `drilldown_query_timeframe` | string | Optional | `"ALERT"` | Time window for the drilldown query link. |
| `mapping_strategy` | string | Optional | `"AUTO"` | How alert fields are populated. Usually `"AUTO"`. |
| `action` | string | **Yes** | `"ALERTS"` | Always `"ALERTS"` — correlation rules produce alerts. |
| `lookup_mapping` | array \| null | Optional | `null` | Lookup table mapping configuration. Usually `null`. |

---

## Severity Guide

### Enum Values

XSIAM uses a fixed set of severity enum values. There is **no SEV_050** — this is
intentional and not a gap.

| Enum Value | Meaning | Use When |
|------------|---------|----------|
| `SEV_010_CRITICAL` | Critical | Confirmed active compromise, data breach, or ransomware execution |
| `SEV_020_HIGH` | High | Strong indicator of compromise requiring immediate investigation |
| `SEV_030_MEDIUM` | Medium | Suspicious activity warranting investigation within SLA |
| `SEV_040_LOW` | Low | Anomalous but potentially benign activity for triage |
| `SEV_060_INFORMATIONAL` | Informational | Audit events, policy violations, or telemetry for enrichment |

### Static Severity

Use a fixed severity enum when the detection always warrants the same response priority.

```json
{
  "severity": "SEV_020_HIGH",
  "user_defined_severity": null
}
```

**When to use:** Honeypot/canary alerts (any trigger is suspicious), known-bad IOC
matches, critical policy violations.

### User-Defined (Dynamic) Severity

Use `"User Defined"` when severity should be derived from a field in the XQL output.
The XQL query must compute an `xdm.alert.severity` field.

```json
{
  "severity": "User Defined",
  "user_defined_severity": "xdm.alert.severity"
}
```

**When to use:** Source data has a numeric score (e.g., Darktrace `percentscore`) that
maps to different severity levels via an `if()` chain in the XQL query.

**Decision criteria:**
- If every alert from this rule deserves the same priority → static severity
- If severity depends on a value in the matched event → `"User Defined"` with an XQL-computed field

---

## Execution Mode Guide

### REAL_TIME

Triggered on each matching event as it is ingested. Best for single-event detections
where every match independently warrants an alert.

**XQL constraints for REAL_TIME:** Only these stages are permitted: `dataset`, `datamodel`,
`filter`, `alter`, `fields`, `config`. Aggregation stages (`comp`, `sort`, `dedup`, `join`)
are **not allowed** in real-time rules.

```json
{
  "execution_mode": "REAL_TIME",
  "search_window": null,
  "simple_schedule": null,
  "timezone": null,
  "crontab": null
}
```

**Best for:** Canary/honeypot triggers, known-bad IOC matches, critical policy violations,
firewall deny events on sensitive ports.

### SCHEDULED

Runs on a cron schedule. Best for aggregation and threshold detections that require
counting or grouping events over a time window.

```json
{
  "execution_mode": "SCHEDULED",
  "search_window": "1 hours",
  "simple_schedule": "Every 5 minutes",
  "timezone": "UTC",
  "crontab": "*/5 * * * *"
}
```

**Best for:** Brute force / password spray (count failures), data exfiltration volume
thresholds, lateral movement (distinct host count), anomaly detection with baselines.

### Duration Format

The `search_window` and `suppression_duration` fields use human-readable duration strings:

| Value | Meaning |
|-------|---------|
| `"5 minutes"` | 5 minutes |
| `"15 minutes"` | 15 minutes |
| `"30 minutes"` | 30 minutes |
| `"1 hours"` | 1 hour |
| `"2 hours"` | 2 hours |
| `"4 hours"` | 4 hours |
| `"12 hours"` | 12 hours |
| `"24 hours"` | 24 hours |

Note: The format uses the plural unit name even for singular values (e.g., `"1 hours"`,
not `"1 hour"`). This matches XSIAM export behavior.

### simple_schedule and crontab Correspondence

The `simple_schedule` field is a human-readable label that corresponds to the `crontab`
value. Both must be present and consistent for SCHEDULED rules.

| simple_schedule | crontab |
|-----------------|---------|
| `"Every 5 minutes"` | `"*/5 * * * *"` |
| `"Every 10 minutes"` | `"*/10 * * * *"` |
| `"Every 15 minutes"` | `"*/15 * * * *"` |
| `"Every 30 minutes"` | `"*/30 * * * *"` |
| `"Every 1 hours"` | `"0 */1 * * *"` |
| `"Every 4 hours"` | `"0 */4 * * *"` |
| `"Every 12 hours"` | `"0 */12 * * *"` |
| `"Every 24 hours"` | `"0 0 * * *"` |

---

## Suppression Patterns

Suppression prevents duplicate alerts from flooding the SOC. Choose suppression fields
that define what makes two alerts "the same incident."

| Detection Type | Suppress By | Recommended Duration | Rationale |
|----------------|-------------|----------------------|-----------|
| Brute force / password spray | `source_ip` | `"1 hours"` | Same source attacking = same incident |
| Lateral movement | `hostname` | `"2 hours"` | Same compromised host = same incident |
| Data exfiltration | `user_name` | `"4 hours"` | Same user exfiltrating = same incident |
| Malware detection | `file_hash` | `"24 hours"` | Same malware = same incident |
| Repeated failed auth | `user_name`, `source_ip` | `"30 minutes"` | Same user from same source = same incident |
| Canary / honeypot trigger | `alert_name` | `"1 hours"` | Same canary = same incident |
| Firewall block (repeated) | `source_ip`, `destination_port` | `"1 hours"` | Same source hitting same port = same incident |
| Privilege escalation | `user_name` | `"2 hours"` | Same user escalating = same incident |
| Suspicious process execution | `hostname`, `process_name` | `"1 hours"` | Same process on same host = same incident |

**When suppression is disabled**, set both `suppression_duration` and `suppression_fields`
to `null`:

```json
{
  "suppression_enabled": false,
  "suppression_duration": null,
  "suppression_fields": null
}
```

**When suppression is enabled:**

```json
{
  "suppression_enabled": true,
  "suppression_duration": "1 hours",
  "suppression_fields": ["source_ip"]
}
```

**Guidelines:**
- Always enable suppression unless the detection is extremely rare (< 1 alert/day expected)
- Shorter durations for time-sensitive detections (auth failures)
- Longer durations for persistent threats (malware, exfiltration)
- Use multiple suppression fields when a single field is too broad

---

## MITRE ATT&CK Mapping

### Format

The `mitre_defs` field maps tactic labels to arrays of technique labels. Both tactics
and techniques use their full label format: `"TAXXXX - Tactic Name"` and
`"TXXXX - Technique Name"`.

```json
{
  "mitre_defs": {
    "TA0006 - Credential Access": [
      "T1110 - Brute Force"
    ]
  }
}
```

### Format Rules

- Tactic keys use the format `"TAXXXX - Tactic Name"` (e.g., `"TA0006 - Credential Access"`)
- Technique values use the format `"TXXXX - Technique Name"` (e.g., `"T1110 - Brute Force"`)
- Sub-techniques use `"TXXXX.XXX - Sub-technique Name"` (e.g., `"T1110.003 - Password Spraying"`)
- A single tactic can contain multiple techniques
- A rule can map to multiple tactics
- Use an empty object `{}` when no MITRE mapping applies

### Multi-Tactic Example

```json
{
  "mitre_defs": {
    "TA0006 - Credential Access": [
      "T1110 - Brute Force",
      "T1110.003 - Password Spraying"
    ],
    "TA0001 - Initial Access": [
      "T1078 - Valid Accounts"
    ]
  }
}
```

### Common Mappings Reference

| Detection Category | Tactic | Technique |
|--------------------|--------|-----------|
| Brute force | `TA0006 - Credential Access` | `T1110 - Brute Force` |
| Password spray | `TA0006 - Credential Access` | `T1110.003 - Password Spraying` |
| Valid accounts | `TA0001 - Initial Access` | `T1078 - Valid Accounts` |
| Valid accounts (persistence) | `TA0003 - Persistence` | `T1078 - Valid Accounts` |
| MFA bypass | `TA0006 - Credential Access` | `T1556.006 - Multi-Factor Authentication` |
| MFA bypass (defense evasion) | `TA0005 - Defense Evasion` | `T1556.006 - Multi-Factor Authentication` |
| Lateral movement (RDP) | `TA0008 - Lateral Movement` | `T1021.001 - Remote Desktop Protocol` |
| Lateral movement (SMB) | `TA0008 - Lateral Movement` | `T1021.002 - SMB/Windows Admin Shares` |
| Data exfiltration | `TA0010 - Exfiltration` | `T1048 - Exfiltration Over Alternative Protocol` |
| Privilege escalation | `TA0004 - Privilege Escalation` | `T1068 - Exploitation for Privilege Escalation` |
| Account manipulation | `TA0003 - Persistence` | `T1098 - Account Manipulation` |
| Suspicious process | `TA0002 - Execution` | `T1059 - Command and Scripting Interpreter` |
| Masquerading | `TA0005 - Defense Evasion` | `T1036 - Masquerading` |
| Network scanning | `TA0043 - Reconnaissance` | `T1046 - Network Service Discovery` |
| Honeypot / canary | `TA0043 - Reconnaissance` | `T1595 - Active Scanning` |
| C2 communication | `TA0011 - Command and Control` | `T1071 - Application Layer Protocol` |
| Traffic signaling | `TA0011 - Command and Control` | `T1205 - Traffic Signaling` |

---

## XQL Query as JSON String

In JSON, the `xql_query` value is a single string with newlines encoded as `\n`.
XQL comments (`//`) are preserved inside the string.

```json
{
  "xql_query": "dataset = xdr_data\n| filter event_type = ENUM.EVENT_LOG\n    and action_evtlog_event_id = 4625\n| comp count() as failed_count by agent_hostname\n| filter failed_count >= 10"
}
```

This represents the XQL query:

```xql
dataset = xdr_data
| filter event_type = ENUM.EVENT_LOG
    and action_evtlog_event_id = 4625
| comp count() as failed_count by agent_hostname
| filter failed_count >= 10
```

### What Goes in XQL vs JSON Fields

| Concern | Where It Lives | Example |
|---------|----------------|---------|
| Event filtering | XQL `xql_query` | `filter event_type = ENUM.EVENT_LOG` |
| Field extraction | XQL `xql_query` | `alter user = event_data -> TargetUserName` |
| Aggregation / threshold | XQL `xql_query` | `comp count() ... \| filter count >= 5` |
| Dynamic severity computation | XQL `xql_query` | `alter xdm.alert.severity = if(score >= 90, "CRITICAL", ...)` |
| Alert name | JSON `alert_name` | `"Failed Auth Spike - $source_ip"` |
| Severity (static) | JSON `severity` | `"SEV_030_MEDIUM"` |
| Severity (dynamic flag) | JSON `severity` | `"User Defined"` |
| Alert category | JSON `alert_category` | `"OTHER"` |
| MITRE mapping | JSON `mitre_defs` | `{"TA0006 - Credential Access": [...]}` |
| Suppression config | JSON `suppression_*` | `"suppression_fields": ["source_ip"]` |

---

## alert_domain Values

The `alert_domain` field classifies the security domain for generated alerts.

| Value | Use When |
|-------|----------|
| `"DOMAIN_SECURITY"` | General security detections — brute force, malware, lateral movement, privilege escalation, most SOC rules |
| `"DOMAIN_MSIAM_IMPACT_REPORTS"` | Identity and access management detections — account manipulation, MFA changes, permission modifications |
| `"DOMAIN_EXTERNAL"` | Detections based on external/third-party data sources or threat intelligence feeds |

When in doubt, use `"DOMAIN_SECURITY"` — it is the most common domain for correlation rules.

---

## Field Variable References

The `alert_name` and `alert_description` fields support `$field` variable references
that are resolved at alert time using values from the XQL query output.

### Syntax

- Reference a field from the XQL output using `$field_name`
- The field name must match an output column from the XQL query
- Multiple references can be used in a single string
- Literal text and variable references can be combined

### Examples

```json
{
  "alert_name": "Brute Force Detected - $source_ip",
  "alert_description": "User $user_name experienced $failed_count failed login attempts from $source_ip"
}
```

For these references to resolve, the XQL query must output columns named `source_ip`,
`user_name`, and `failed_count` (via `fields`, `alter`, or `comp ... as` stages).

### Rules

- Only fields present in the XQL query output can be referenced
- If a referenced field is not in the output, the variable renders as the literal string (e.g., `$missing_field`)
- Field references work in `alert_name` and `alert_description` only — other string fields do not support variable substitution
