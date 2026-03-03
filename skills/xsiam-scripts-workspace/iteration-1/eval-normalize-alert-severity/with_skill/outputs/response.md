# NormalizeAlertSeverity — Generation Summary

## What Was Generated

A single unified YAML file `NormalizeAlertSeverity.yml` ready for direct import into Cortex XSIAM.

## Script Purpose

The script accepts a `severity` string argument and maps it to a normalised numeric score (1–4) plus a human-readable label. It writes three values to context:

| Context Path | Type | Description |
|---|---|---|
| `NormalizeAlertSeverity.Input` | string | The original input value |
| `NormalizeAlertSeverity.Score` | number | Numeric severity (1–4) |
| `NormalizeAlertSeverity.Label` | string | Human-readable label (Low / Medium / High / Critical) |

## Mapping Logic

| Score | Label | Accepted Inputs |
|---|---|---|
| 1 | Low | `low`, `1`, `info`, `informational` |
| 2 | Medium | `medium`, `med`, `2`, `warn`, `warning` |
| 3 | High | `high`, `3`, `severe` |
| 4 | Critical | `critical`, `crit`, `4`, `urgent`, `emergency` |

All matching is case-insensitive. Unrecognised values raise a `ValueError` that is surfaced via `return_error()`.

## Key Decisions

1. **Tags** — tagged `transformer` (so the script appears as a transformer option in playbook input mappings) and `Utility` (informational). This is appropriate because the script transforms a severity string into a structured value.

2. **`default: true` on the `severity` arg** — marks it as the script's default argument, allowing it to be used inline in war room without specifying the argument name.

3. **No `demisto.alert()` call** — the severity is passed as an explicit argument rather than read from alert context. This makes the script reusable across both alert-context and non-alert-context invocations (e.g., from a playbook transformer, war room, or another script via `execute_command()`).

4. **Alias coverage** — included common real-world aliases (`info`, `warn`, `severe`, `urgent`, `emergency`) to handle the variety of severity strings seen in SIEM-sourced alerts.

5. **`outputs_key_field='Input'`** — deduplicates context entries by the original input string.

6. **Field ordering** — strictly follows the spec: `commonfields → vcShouldKeepItemLegacyProdMachine → name → script → type → tags → comment → enabled → args → outputs → scripttarget → subtype → pswd → runonce → dockerimage → runas → engineinfo → mainengineinfo`.

## Validation Checklist

- [x] Python embedded via top-level `script: |-`
- [x] Consistent 2-space indentation inside the YAML block
- [x] Standard imports present (`demistomock`, `CommonServerPython`, `CommonServerUserPython`)
- [x] `main()` has `try/except` with `return_error()`
- [x] Field ordering matches spec
- [x] `vcShouldKeepItemLegacyProdMachine: false` present
- [x] `enabled: true`, `scripttarget: 0`, `pswd: ""`, `runas: DBotWeakRole`, `engineinfo: {}`, `mainengineinfo: {}` all present
- [x] Every `args` entry has `supportedModules: []` as its first key
- [x] Arg uses `defaultValue` key (not applicable here — no default value; arg is required)
- [x] All args have descriptions
- [x] All outputs have `contextPath`, `description`, and `type`
- [x] Docker image is pinned `3.12.x` (`demisto/python3:3.12.12.6947692`)
- [x] No `fromversion`, `marketplaces`, `tests`, or `timeout` fields
- [x] First line of embedded Python is `register_module_line('NormalizeAlertSeverity', 'start', __line__())`
- [x] Last line of embedded Python is `register_module_line('NormalizeAlertSeverity', 'end', __line__())`
- [x] No tab characters; consistent YAML indentation
- [x] No `demisto.alert()` call (not needed — input comes via args)
- [x] Arg parsing uses `args.get()` pattern (no raw casting; input is a string that is looked up in a dict)
