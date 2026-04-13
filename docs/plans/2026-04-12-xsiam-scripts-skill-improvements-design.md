# XSIAM Scripts Skill Improvements

**Date:** 2026-04-12
**Scope:** Improve xsiam-scripts skill accuracy and coverage based on XSIAM knowledge vault review

## Problem

The xsiam-scripts skill has gaps that reduce its effectiveness for specialized script types and common patterns. A review against the XSIAM knowledge vault (`/Users/joeplunkett/Documents/RagVault/`) identified 8 gaps spanning script type guidance, examples, API coverage, and anti-patterns.

## Approach

Approach B: Add one new reference file for specialized script types. Extend existing reference files for everything else. Minimal changes to SKILL.md to route the model to the right patterns.

## Changes

### 1. SKILL.md

**1a. New "Script Types" section** — inserted between "What is a Script?" and "Workflow".

Adds a table mapping script types to their tags and when to use each:

| Type | Tag | When to use |
|------|-----|-------------|
| Standard | (none) | General-purpose: data processing, executeCommand chaining, utilities |
| Transformer | `transformer` | Transform a value inline in playbook task input mappings |
| Filter | `filter` | Evaluate a condition in playbook task conditions |
| Dynamic Section | `dynamic-section` | Render read-only display content in case/issue layouts |
| Field-Change Triggered | `field-change-triggered` | React to a field value change on a layout |
| Widget | `widget` | Power a dashboard or report widget |

**1b. Update "Before Starting"** — add the new reference file to the list:
```
- `references/script-types-patterns.md` — Specialized script types: transformers, filters, dynamic sections, field-change triggers
```

### 2. New file: `references/script-types-patterns.md`

Covers each specialized (non-standard) script type with:
- When to use
- Tag required
- Argument/output conventions
- Minimal but complete example
- Notes on behavior

**Sections:**

- **Transformer Scripts**: Receive `value` arg, return transformed value. Example: timestamp normalizer.
- **Filter Scripts**: Receive `value` arg, return boolean-like. Example: RFC1918 IP check.
- **Dynamic Section Scripts**: No args, no outputs, return `readable_output` only. Pull from `demisto.context()` or `demisto.alert()`. Re-execute on tab open. Example: DBot score display table (based on `jp-gpds-display-dbotscores` vault pattern).
- **Field-Change-Triggered Scripts**: Receive field change event via `demisto.args()`. Key fields: `new`, `old`, `name`, `cliName`, `type`, `selectValues`. Cross-references payload docs in script-yaml-spec.md. Example: resolution reason validator.
- **Widget Scripts**: Must return dashboard-renderer-compatible structure. Brief coverage — least common type.

### 3. `script-yaml-spec.md` edits

**3a. `scripttarget` clarification:**
- `0` = server-side execution (default, most scripts)
- `1` = endpoint-side execution (runs on XDR agent)
- Note: use `1` only for scripts that execute directly on endpoints (forensic collection, OS-level commands)

**3b. `runas` clarification:**
- `DBotWeakRole` = restricted permissions (default)
- `DBotRole` = elevated permissions (create incidents, modify integrations)
- Note: prefer `DBotWeakRole` unless privileged operations are required

### 4. `common-patterns.md` edits

**4a. Enhanced `tableToMarkdown`** — expand existing example to include:
- `removeNull=True` — omit columns where all values are None
- `url_keys` — auto-format as clickable markdown links
- `date_fields` — auto-format date strings
- `is_auto_json_transform=True` — pretty-print nested dicts/lists

**4b. Polling / ScheduledCommand pattern** — new section after "Error Handling":
- Anti-pattern: never use `time.sleep()`
- Show `ScheduledCommand` + `CommandResults` pattern
- Note: polling is platform-managed; the script exits and gets re-invoked

**4c. Unit Testing pattern** — new section at end:
- Minimal pytest pattern: mock `demisto.args()`, call `main()`, assert on `return_results()` args
- Note: `demistomock` is the test layer; tests run outside Docker with `pytest`

## Files Changed

| File | Action |
|------|--------|
| `skills/xsiam-scripts/SKILL.md` | Edit |
| `skills/xsiam-scripts/references/script-types-patterns.md` | Create |
| `skills/xsiam-scripts/references/script-yaml-spec.md` | Edit |
| `skills/xsiam-shared/references/common-patterns.md` | Edit |

## Out of Scope

- No changes to other skills (integrations, XQL, correlations, etc.)
- No changes to plugin.json or marketplace.json
- No new skill directories
