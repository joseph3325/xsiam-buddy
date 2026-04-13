---
name: xsiam-correlations
description: >
  This skill should be used when the user asks to "create a correlation rule",
  "build a detection rule", "write a detection", "XSIAM alert rule",
  "correlation YAML", "detection engineering", "build a detection for",
  or needs to generate correlation rule .yml files with embedded XQL detection logic.
version: 1.0.0
---

# Correlation Rule Generation

## Scope

Generate correlation rule `.yml` files for Cortex XSIAM. Each output file contains
a complete YAML wrapper (metadata, severity, MITRE mapping, suppression) with embedded
XQL detection logic as a literal block scalar.

**This skill handles:**
- Real-time single-event correlation rules
- Scheduled threshold / aggregation correlation rules
- Dynamic severity correlation rules (numeric score to severity mapping)
- MITRE ATT&CK mapping for detection rules
- Suppression configuration for alert deduplication

**This skill does NOT handle:**
- Standalone XQL queries without a YAML wrapper -- use `xsiam-xql`
- SPL-to-XQL translation -- use `xsiam-splunk-to-xql`
- Automation scripts triggered by alerts -- use `xsiam-scripts`
- Playbook workflows for alert response -- use `xsiam-playbooks`

---

## Before Starting

### Always Read

Load these reference files before generating any correlation rule:

1. `../xsiam-shared/references/xql-core-reference.md` -- XQL syntax, stage order, operators
2. `../xsiam-shared/references/xql-datasets-core.md` -- Dataset names and field schemas
3. `references/correlation-rule-spec.md` -- Correlation YAML field specification and templates

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

| Mode | When to Use | crontab Required? | search_window |
|------|-------------|-------------------|---------------|
| `REAL_TIME` | Single-event detections -- each matching event triggers an alert | No | Short or omitted (e.g., `1m`) |
| `SCHEDULED` | Aggregation / threshold detections -- runs on a schedule | Yes (cron syntax) | Defines lookback period (e.g., `15m`, `1h`, `24h`) |

**Guidelines:**
- Use `REAL_TIME` when every matching event is independently alertable
- Use `SCHEDULED` when the detection requires counting, grouping, or comparing across events
- `search_window` for `SCHEDULED` should cover at least 2x the cron interval to avoid gaps

### Step 3: Write the XQL Detection Query

Build the XQL query following the standard stage order from `xql-core-reference.md`:

1. `config` (if needed for extended timeframe)
2. `dataset` / `datamodel dataset` / `preset`
3. `filter` -- narrow to relevant events
4. `alter` -- extract, transform, compute fields
5. `comp` -- aggregate (for threshold detections)
6. `filter` -- post-aggregation threshold
7. `fields` -- select output columns

**Critical rule:** The XQL query contains detection logic ONLY. Do not set alert metadata
(name, severity, MITRE, suppression) inside the query. Those belong in the YAML wrapper.

**Exception:** Dynamic severity via `if()` is allowed in XQL when the source data has a
numeric score but no discrete severity field. In that case, compute `xdm.alert.severity`
in the query and set a static fallback severity in YAML.

### Step 4: Build the YAML Wrapper

Populate all correlation rule fields following the canonical field order from
`correlation-rule-spec.md`. Required fields:

- `global_rule_id` -- Generate a valid UUID v4
- `rule_id`, `rule_name`, `name`, `alert_name` -- Rule identifiers (`alert_name` = `name`)
- `description` -- Clear description of what the detection does
- `dataset` -- Always `alerts`
- `action` -- Always `ALERTS`
- `execution_mode` -- `REAL_TIME` or `SCHEDULED`
- `xql_query` -- The XQL from Step 3 as a YAML literal block scalar (`|`)
- `severity` -- Static severity enum value

### Step 5: Map MITRE ATT&CK

Assign relevant MITRE ATT&CK technique(s) and tactic(s):

- Use the `mitre_defs` field with technique IDs mapping to tactic lists
- Technique IDs follow the format `T####` or `T####.###` (sub-technique)
- Tactic IDs follow the format `TA####`
- Map to the most specific technique available; prefer sub-techniques when applicable

### Step 6: Configure Suppression

Set suppression to prevent alert fatigue:

- `suppression_enabled` -- `true` for most detections
- `suppression_duration` -- How long to suppress duplicate alerts (e.g., `1h`, `24h`)
- `suppression_fields` -- Which fields define a "duplicate" (e.g., `source_ip`, `user_name`)

Refer to the suppression patterns table in `correlation-rule-spec.md` for recommended
values by detection type.

### Step 7: Format Output

Deliver the complete `.yml` file:

1. All fields in canonical order (matching XSIAM export format)
2. XQL query as a YAML literal block scalar (`|`) -- preserving indentation
3. Inline `# ANNOTATION:` comments explaining key design choices
4. No trailing whitespace or extra blank lines inside the YAML

---

## Output Format

The final output is a single `.yml` file:

```yaml
global_rule_id: "<uuid-v4>"
rule_id: "<snake_case_rule_id>"
rule_name: "<Human Readable Rule Name>"
# ... all fields in canonical order ...
xql_query: |
  dataset = <dataset_name>
  | filter <conditions>
  | <detection logic>
severity: SEV_030_MEDIUM
# ... remaining fields ...
```

---

## Quality Checklist

Before delivering the correlation rule, verify ALL of the following:

1. **XQL contains detection logic only** -- no alert metadata in the query
   (exception: dynamic severity via `if()` when source lacks discrete severity field)
2. **Severity uses correct enum** -- one of: `SEV_010_CRITICAL`, `SEV_020_HIGH`,
   `SEV_030_MEDIUM`, `SEV_040_LOW`, `SEV_060_INFORMATIONAL` (there is NO `SEV_050`)
3. **`alert_name` and `name` match** -- these two fields must have identical values
4. **`global_rule_id` is a valid UUID v4** -- lowercase hex with hyphens
5. **`dataset: alerts` always set** -- correlation rules always target the alerts dataset
6. **`action: ALERTS` always set** -- correlation rules always produce alerts
7. **MITRE technique IDs are valid** -- format `T####` or `T####.###`
8. **`search_window` appropriate for execution mode** -- short/omitted for REAL_TIME,
   adequate lookback for SCHEDULED (at least 2x cron interval)
9. **Suppression fields make sense** -- fields that define a "duplicate" for this
   specific detection type (e.g., `source_ip` for brute force, not `user_name`)
10. **Dynamic severity only when appropriate** -- only use `if()` in XQL for severity
    when the source has a numeric score but no discrete severity field
