---
name: xsiam-correlations
description: >
  This skill should be used when the user asks to "create a correlation rule",
  "build a detection rule", "write a detection", "XSIAM alert rule",
  "correlation JSON", "detection engineering", "build a detection for",
  or needs to generate correlation rule JSON files with embedded XQL detection logic.
version: 2.0.0
---

# Correlation Rule Generation

## Scope

Generate correlation rule `.json` files for Cortex XSIAM. Each output file contains a
JSON array with a single rule object wrapping XQL detection logic, alert metadata,
severity, MITRE ATT&CK mapping, and suppression configuration.

**This skill handles:**
- Real-time single-event correlation rules
- Scheduled threshold / aggregation correlation rules
- Dynamic severity correlation rules (numeric score to severity mapping)
- MITRE ATT&CK mapping for detection rules
- Suppression configuration for alert deduplication

**This skill does NOT handle:**
- Standalone XQL queries without a JSON wrapper -- use `xsiam-xql`
- SPL-to-XQL translation -- use `xsiam-splunk-to-xql`
- Automation scripts triggered by alerts -- use `xsiam-scripts`
- Playbook workflows for alert response -- use `xsiam-playbooks`

---

## Before Starting

### Always Read

Load these reference files before generating any correlation rule:

1. `../xsiam-shared/references/xql-core-reference.md` -- XQL syntax, stage order, operators
2. `../xsiam-shared/references/xql-datasets-core.md` -- Dataset names and field schemas
3. `references/correlation-rule-spec.md` -- Correlation JSON field specification and templates

### Read On-Demand (when the detection query needs these)

- `../xsiam-shared/references/xql-advanced-functions.md` -- Complex functions (arraymap, arrayfilter, join)
- `../xsiam-shared/references/xql-datasets-extended.md` -- Vendor-specific raw datasets
- `../xsiam-shared/references/xql-federated-search.md` -- Cross-tenant federated queries
- `references/correlation-examples.md` -- Complete worked examples for each execution mode

---

## Workflow

### Step 1: Understand the Detection Goal

Gather from the user:
- **Behavior to detect** -- What malicious or suspicious activity triggers the alert?
- **Data source** -- Which dataset(s) contain the relevant events?
- **Severity** -- How critical is this detection? Is severity static or dynamic?
- **Context** -- Any suppression requirements, MITRE mappings, or specific thresholds?

If the user provides only a high-level description (e.g., "detect brute force attacks"),
ask clarifying questions before proceeding.

### Step 2: Select Execution Mode

Choose between:

| Mode | When to Use | Schedule Fields |
|------|-------------|-----------------|
| `REAL_TIME` | Single-event detections -- each matching event triggers an alert | `search_window`: null, `simple_schedule`: null, `timezone`: null, `crontab`: null |
| `SCHEDULED` | Aggregation / threshold detections -- runs on a schedule | All four required: `search_window` (lookback), `simple_schedule` (human-readable), `timezone`, `crontab` (cron expression) |

**Guidelines:**
- Use `REAL_TIME` when every matching event is independently alertable
- Use `SCHEDULED` when the detection requires counting, grouping, or comparing across events
- `search_window` for `SCHEDULED` should cover at least 2x the cron interval to avoid gaps
- REAL_TIME XQL is limited to: `dataset`, `datamodel`, `filter`, `alter`, `fields`, `config case_sensitive`

### Step 3: Write the XQL Detection Query

Build the XQL query following the standard stage order from `xql-core-reference.md`:

1. `config` (if needed for extended timeframe)
2. `dataset` / `datamodel dataset` / `preset`
3. `filter` -- narrow to relevant events
4. `alter` -- extract, transform, compute fields
5. `comp` -- aggregate (for threshold detections)
6. `filter` -- post-aggregation threshold
7. `fields` -- select output columns

**REAL_TIME restriction:** For REAL_TIME rules, only `dataset`/`datamodel`, `filter`,
`alter`, `fields`, and `config` stages are supported. Do not use `comp`, `sort`, `dedup`,
or `join` in REAL_TIME rules.

**Critical rule:** The XQL query contains detection logic ONLY. Do not set alert metadata
(name, severity, MITRE, suppression) inside the query. Those belong in the JSON fields.

**Exception:** Dynamic severity via `if()` is allowed in XQL when the source data has a
numeric score but no discrete severity field. In that case, compute `xdm.alert.severity`
in the query and set a static fallback severity in JSON.

### Step 4: Build the JSON Object

Populate all correlation rule fields following the canonical field order from
`correlation-rule-spec.md`. Key instructions:

- Prompt the user for `rule_id` (integer) if not provided
- Set `is_enabled: true` by default
- Include ALL fields in canonical order, including null fields for full export fidelity
- Two severity patterns:
  - **Static:** `"severity": "SEV_030_MEDIUM"`, `"user_defined_severity": null`
  - **User Defined:** `"severity": "User Defined"`, `"user_defined_severity": "field.reference"`
- `alert_fields` is an object mapping display names to XDM paths: `{"email": "xdm.source.user.upn"}`
- `alert_name` and `alert_description` can use `$field` variable references
- When `suppression_enabled` is false, set `suppression_duration` and `suppression_fields` to null

### Step 5: Map MITRE ATT&CK

Assign relevant MITRE ATT&CK technique(s) and tactic(s) using labeled format:

```json
"mitre_defs": {
  "TA0006 - Credential Access": [
    "T1110 - Brute Force"
  ]
}
```

- Tactic keys: `"TA#### - Tactic Name"`
- Technique values: `"T#### - Technique Name"` or `"T####.### - Sub-technique Name"`
- Empty object `{}` when no MITRE mapping applies
- Map to the most specific technique available; prefer sub-techniques when applicable

### Step 6: Configure Suppression

Set suppression to prevent alert fatigue:

- `suppression_enabled` -- `true` for most detections
- `suppression_duration` -- Human-readable format: `"1 hours"`, `"30 minutes"`, `"15 minutes"`
- `suppression_fields` -- Which fields define a "duplicate" (e.g., `["source_ip"]`, `["user_name"]`)
- When suppression is disabled, set `suppression_duration` and `suppression_fields` to `null`

Refer to the suppression patterns table in `correlation-rule-spec.md` for recommended
values by detection type.

### Step 7: Deliver Output

Deliver:

1. A `.json` file containing a JSON array with one rule object: `[{...}]`
2. All fields in canonical order from `correlation-rule-spec.md`, including null fields
3. XQL query serialized as a JSON string with `\n` for newlines
4. A companion markdown summary block explaining key design decisions (execution mode
   choice, severity pattern, suppression rationale, MITRE mapping rationale)

---

## Output Format

The final output is a `.json` file containing a single-element JSON array:

```json
[
  {
    "rule_id": 100,
    "name": "Rule Name",
    "severity": "SEV_030_MEDIUM",
    "xql_query": "dataset = xdr_data\n| filter event_type = ENUM.EVENT_LOG\n| filter action_evtlog_event_id = 4625\n| fields agent_hostname, action_evtlog_event_id",
    "is_enabled": true,
    "description": "Detects ...",
    "alert_name": "Rule Name",
    ...
  }
]
```

After the JSON file, include a companion markdown summary:

```markdown
### Design Decisions

- **Execution mode:** [rationale]
- **Severity:** [rationale]
- **Suppression:** [rationale]
- **MITRE mapping:** [rationale]
```

---

## Quality Checklist

Before delivering the correlation rule, verify ALL of the following:

1. **Output is valid JSON** -- no trailing commas, proper quoting, correct nesting
2. **Output wrapped in array** -- `[{...}]` even for single rules
3. **All fields present** -- all 29 fields in canonical order, including null fields
4. **`rule_id` is an integer** -- prompted from user if not provided
5. **Severity enum valid** -- one of: `SEV_010_CRITICAL`, `SEV_020_HIGH`,
   `SEV_030_MEDIUM`, `SEV_040_LOW`, `SEV_060_INFORMATIONAL` (no `SEV_050`),
   or `"User Defined"`
6. **`dataset` always `"alerts"`**
7. **`action` always `"ALERTS"`**
8. **`mitre_defs` uses labeled format** -- tactic-to-technique with
   `"TA#### - Name": ["T#### - Name"]`
9. **Duration strings use human-readable format** -- `"1 hours"`, `"15 minutes"`,
   `"30 minutes"` (not shorthand)
10. **REAL_TIME schedule fields are null** -- `search_window`, `simple_schedule`,
    `timezone`, `crontab` all `null`
11. **REAL_TIME XQL uses only supported stages** -- `dataset`/`datamodel`, `filter`,
    `alter`, `fields`, `config` (no `comp`, `sort`, `dedup`, `join`)
12. **XQL contains detection logic only** -- alert metadata in JSON fields
    (exception: dynamic severity via `alter`/`if()`)
13. **Suppression fields match detection type** -- fields that define "duplicate"
    for this specific detection
14. **Companion markdown summary included** -- explains execution mode, severity,
    suppression, and MITRE choices
