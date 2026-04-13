# Correlation Skill: YAML to JSON Migration

## Problem

The `xsiam-correlations` skill generates `.yml` files, but XSIAM correlation rules are imported/exported as `.json` files. The current spec, examples, and workflow are all built around a YAML schema that was inferred rather than derived from real exports. This causes the skill output to be incompatible with XSIAM's import function.

## Decision

Full rewrite of the correlation rule spec (`correlation-rule-spec.md`), examples (`correlation-examples.md`), and skill workflow (`SKILL.md`) to target JSON output matching the real XSIAM export format. Version bump to `2.0.0`.

## JSON Schema

The canonical field order and types, derived from three real XSIAM correlation rule exports:

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `rule_id` | integer | Yes | — | Prompt user if not provided |
| `name` | string | Yes | — | Rule display name |
| `severity` | string | Yes | — | Enum (e.g., `SEV_030_MEDIUM`) or `"User Defined"` |
| `xql_query` | string | Yes | — | Detection logic as a string |
| `is_enabled` | boolean | Yes | `true` | Whether the rule is active |
| `description` | string \| null | No | `null` | Rule description |
| `alert_name` | string | Yes | — | Can use `$field` references |
| `alert_category` | string | Yes | `"OTHER"` | Category or `"User Defined"` |
| `alert_type` | null | No | `null` | Always null in current exports |
| `alert_description` | string \| null | No | `null` | Can use `$field` references |
| `alert_domain` | string | Yes | — | `DOMAIN_SECURITY`, `DOMAIN_MSIAM_IMPACT_REPORTS`, etc. |
| `alert_fields` | object \| null | No | `{}` | Field name → XDM path mapping |
| `execution_mode` | string | Yes | — | `"REAL_TIME"` or `"SCHEDULED"` |
| `search_window` | string \| null | Conditional | `null` | Human-readable (e.g., `"1 hours"`) |
| `simple_schedule` | string \| null | Conditional | `null` | Mirrors crontab (e.g., `"1 hour"`) |
| `timezone` | string \| null | Conditional | `null` | For scheduled rules (e.g., `"UTC"`) |
| `crontab` | string \| null | Conditional | `null` | Standard cron expression |
| `suppression_enabled` | boolean | Yes | — | Whether dedup is active |
| `suppression_duration` | string \| null | Conditional | `null` | Human-readable (e.g., `"1 hours"`) |
| `suppression_fields` | array \| null | Conditional | `null` | Field names defining duplicates |
| `dataset` | string | Yes | `"alerts"` | Always `"alerts"` |
| `user_defined_severity` | string \| null | Conditional | `null` | Field reference when severity is `"User Defined"` |
| `user_defined_category` | string \| null | Conditional | `null` | Field reference when category is `"User Defined"` |
| `mitre_defs` | object | Yes | `{}` | Tactic label → technique label array |
| `investigation_query_link` | string \| null | No | `""` | Drilldown query or link |
| `drilldown_query_timeframe` | string | No | `"ALERT"` | Time window for drilldown |
| `mapping_strategy` | string | No | `"AUTO"` | `"AUTO"` default |
| `action` | string | Yes | `"ALERTS"` | Always `"ALERTS"` |
| `lookup_mapping` | array \| null | No | `null` | Lookup mapping config |

### Key Differences from Current YAML Spec

- `rule_id`: numeric integer (prompted from user), not snake_case string
- `global_rule_id`: removed entirely (not in export schema)
- `rule_name`, `rule_labels`, `rule_source`: removed (not in export schema)
- `mitre_defs`: inverted direction — tactic→technique with human-readable labels (e.g., `"TA0005 - Defense Evasion": ["T1036.005 - Match Legitimate Resource Name or Location"]`)
- `alert_fields`: object/dict, not list of mapping objects
- Duration strings: `"1 hours"`, `"15 minutes"` format, not `"1h"`, `"15m"`
- New fields: `alert_type`, `alert_description`, `simple_schedule`, `timezone`, `lookup_mapping`, `is_enabled`
- All null fields included for full export fidelity
- Output wrapped in JSON array: `[{...}]`

## Severity Patterns

Two mutually exclusive patterns:

### Static Severity
```json
{
  "severity": "SEV_030_MEDIUM",
  "user_defined_severity": null
}
```
Use when the detection always warrants the same response priority.

### User-Defined Severity
```json
{
  "severity": "User Defined",
  "user_defined_severity": "xdm.alert.severity"
}
```
Use when the source data has a severity/score field that should drive alert severity dynamically. The `user_defined_severity` value is a field reference string, not a mapping object. Severity computation happens in the XQL query (via `alter`) or through XDM field mapping.

## MITRE ATT&CK Format

Tactic→technique with full human-readable labels:

```json
{
  "mitre_defs": {
    "TA0005 - Defense Evasion": [
      "T1036.005 - Match Legitimate Resource Name or Location"
    ]
  }
}
```

The spec will include an expanded reference table with correct label strings for common mappings.

## Output Format

The skill delivers:

1. A `.json` file containing a JSON array with one object: `[{...}]`
2. A companion markdown summary block (outside the JSON) explaining key design decisions — replaces the inline `# ANNOTATION:` comments that YAML supported

## Workflow Changes

| Step | Current | Updated |
|------|---------|---------|
| Step 4 | "Build the YAML Wrapper" | "Build the JSON Object" |
| Step 5 | MITRE with bare IDs, technique→tactic | MITRE with labels, tactic→technique |
| Step 6 | Duration shorthand (`"1h"`) | Duration longhand (`"1 hours"`) |
| Step 7 | Deliver `.yml` with inline annotations | Deliver `.json` + companion markdown summary |

Steps 1-3 (detection goal, execution mode, XQL query) are unchanged.

## Quality Checklist Updates

Remove:
- YAML literal block scalar checks
- YAML field order conventions
- `global_rule_id` UUID validation
- `alert_name` must match `name` (JSON export shows these can differ — `alert_name` can use `$field` refs)

Add:
- Output is valid JSON (no trailing commas, proper quoting)
- Output wrapped in array `[{...}]`
- All null fields present for export fidelity
- `rule_id` is an integer (prompted from user)
- `mitre_defs` uses tactic→technique with full labels
- Duration strings use `"X hours/minutes"` format

Keep:
- Severity enum validation (no SEV_050)
- `dataset: "alerts"` always set
- `action: "ALERTS"` always set
- XQL contains detection logic only (exception: dynamic severity)

## Files Changed

| File | Change |
|------|--------|
| `skills/xsiam-correlations/SKILL.md` | Rewrite: JSON output, updated workflow, updated checklist, version `2.0.0` |
| `skills/xsiam-correlations/references/correlation-rule-spec.md` | Full rewrite: JSON schema, field reference, MITRE labels, templates |
| `skills/xsiam-correlations/references/correlation-examples.md` | Convert all 4 examples to JSON + companion summaries |
| `CLAUDE.md` | Update XQL Skill Family table: correlation output column |
| `.claude-plugin/plugin.json` | Version bump |
| `.claude-plugin/marketplace.json` | Version bump |

## No Changes Required

- `xsiam-shared/references/` — XQL references are format-agnostic
- `xsiam-xql` and `xsiam-splunk-to-xql` — separate output formats, no cross-dependency
- `xsiam-scripts` and `xsiam-integrations` — unrelated
