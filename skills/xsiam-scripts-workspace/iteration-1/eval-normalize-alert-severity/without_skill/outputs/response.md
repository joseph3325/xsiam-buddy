# NormalizeAlertSeverity – Generation Summary

## What Was Generated

A single unified YAML file (`NormalizeAlertSeverity.yml`) ready for direct import into a Cortex XSIAM tenant. The file embeds the complete Python automation logic inside the top-level `script: |-` field, following the XSIAM script export format.

## Script Purpose

The script accepts a single `severity` argument and maps it to a canonical numeric score (1–4) plus a human-readable label. It writes both values to the incident/alert context under the `NormalizeAlertSeverity` prefix.

## Severity Mapping Logic

| Input (case-insensitive) | Score | Label |
|---|---|---|
| `1`, `low`, `informational`, `info`, `unknown` | 1 | Low |
| `2`, `medium`, `med`, `moderate` | 2 | Medium |
| `3`, `high` | 3 | High |
| `4`, `critical`, `crit` | 4 | Critical |

Numeric strings `'1'`–`'4'` map directly to the corresponding level. Any unrecognised value causes the script to call `return_error()` with a descriptive message listing the accepted values.

## Key Design Decisions

### Mapping table as a module-level dict
A single `SEVERITY_MAP` dict at module scope keeps the mapping data separate from control flow. This makes the table easy to extend (e.g. adding vendor-specific aliases) without touching `main()`.

### Error handling split by exception type
`ValueError` (bad input) is caught separately from the general `Exception` catch-all so that the error message surfaced to the analyst is specific and actionable for input validation failures.

### `transformer` tag included
The `transformer` tag was added alongside `Utility` so the script is available as a playbook task transformer in input mappings — a common use-case for a severity normalisation utility.

### `default: true` on the `severity` arg
Marking the `severity` argument as `default: true` allows the script to be invoked in the war room as `!NormalizeAlertSeverity high` without specifying the argument name, matching typical transformer usage patterns.

### Three context outputs
Three output fields are written to context:
- `NormalizeAlertSeverity.Score` (number) — the numeric value for downstream comparisons
- `NormalizeAlertSeverity.Label` (string) — the canonical text label for display
- `NormalizeAlertSeverity.Input` (string) — the original raw value for traceability/debugging

### YAML field order
Fields are ordered to match the real XSIAM export structure:
`commonfields → vcShouldKeepItemLegacyProdMachine → name → script → type → tags → comment → enabled → args → outputs → scripttarget → subtype → pswd → runonce → dockerimage → runas → engineinfo → mainengineinfo`

### Fields intentionally omitted
`fromversion`, `marketplaces`, `tests`, and `timeout` were not included — these are content-pack CI fields only and are not present in real XSIAM tenant imports.

### register_module_line calls
`register_module_line('NormalizeAlertSeverity', 'start', __line__())` and the corresponding `end` call are the first and last lines of the embedded Python block, as required by the XSIAM platform runtime.
