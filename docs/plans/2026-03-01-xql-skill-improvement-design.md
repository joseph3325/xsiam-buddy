# XQL Skill Improvement — Design

**Date:** 2026-03-01
**Status:** Approved

## Problem

The `xsiam-xql` skill is untested and has two classes of issues:

1. **Inaccurate guidance** — the correlation rule section incorrectly implies that severity, MITRE, and alert name are set as `alter` fields inside the XQL query. In reality, correlation rules are separate `.yml` config files that wrap the XQL.

2. **Missing syntax** — the syntax reference omits several real-world XQL patterns that appear in production queries: `config timeframe` preamble, `datamodel dataset =` combined form, arrow notation for JSON traversal (`->`), advanced array functions (`regextract`, `arraymap`, `arrayfilter`, `arraydistinct`, `arrayindex`, `arraymerge`), `ENUM.*` types, inline dataset lookups, and `view column order = populated`.

3. **Verbose output format** — the skill currently asks the model to produce description + query + expected output + customization notes. The desired default is bare XQL with standard header comments.

4. **No real examples** — the skill has no grounded examples to model queries after.

## Solution

Option B: rewrite the syntax reference for accuracy, add examples and correlation template as new reference files, update the skill workflow and output format.

## Files Changed

| File | Action |
|---|---|
| `skills/xsiam-xql/SKILL.md` | Rewrite workflow and output sections |
| `skills/xsiam-xql/references/xql-syntax-reference.md` | Rewrite — fix inaccuracies, add missing syntax |
| `skills/xsiam-xql/references/xql-examples.md` | New — 6 real production examples |
| `skills/xsiam-xql/references/correlation-template.yml` | New — YAML template for correlation rules |

`xql-datasets.md` is unchanged.

## SKILL.md Changes

### References

Read 4 references before starting (currently reads 2):

- `references/xql-syntax-reference.md`
- `references/xql-datasets.md`
- `references/xql-examples.md`
- `references/correlation-template.yml`

### Output Format

**Default (plain query):** Bare XQL in a code block with standard header comments:

```
// Title: <meaningful name>
// Description: <what this query finds>
// Author: xsiam-buddy
// Dataset(s): <dataset or preset used>
// Query last modified: <today's date>
// Vendor Reference: <link or N/A>

dataset = ...
| filter ...
```

No prose description, expected output section, or customization notes.

**Correlation rule:** When the user explicitly requests a correlation rule, output a populated `.yml` file using the `correlation-template.yml` structure, with the XQL embedded in the `xql_query` field. No plain XQL block.

### Checklist Updates

- Header comments always present with no placeholder values left in
- If correlation rule: all required YAML fields populated, UUID generated for `global_rule_id`

## Syntax Reference Additions

Sections added to `xql-syntax-reference.md`:

- **`config timeframe` preamble** — `config timeframe = 7D` used as the first line, followed by `| dataset = ...`
- **`datamodel dataset =`** — combined form for XDM-mapped sources (e.g., `datamodel dataset = microsoft_windows_raw`)
- **Arrow notation `->` for JSON traversal** — `field -> key`, `field -> []` (array), `field -> key{}` (nested array of objects)
- **Advanced array functions** — `regextract(str, pattern)` returns array; `arraymap(arr, expr)`, `arrayfilter(arr, condition)`, `arraydistinct(arr)`, `arrayindex(arr, n)`, `arraymerge(arr1, arr2)`
- **`ENUM.*` types** — `ENUM.EVENT_LOG`, `ENUM.STORY` for filtering on `event_type`
- **Inline dataset lookup** — `filter field in (dataset = lookup_table | fields col)`
- **`view column order = populated`** — show only non-null columns; useful for investigation queries
- **Correlation rule section rewritten** — XQL is detection logic only; alert name, severity, MITRE, suppression, execution mode live in the surrounding `.yml` wrapper

## Examples File

`xql-examples.md` contains 6 real queries sourced from `joseph3325/xsiam-dev/examples/xql`:

- Google Workspace Administrator Role Assigned
- Multiple Users Failing to Login from the Same Location
- Top Issue Sources Last 7 Days
- Windows On-Prem AD Group Monitoring
- Process Executions of Hash in Last Month
- Top 10 Users Failing to Authenticate from Multiple Locations

These demonstrate arrow notation, `regextract`/`arraymap`/`arrayfilter`, `ENUM.*`, XDM field naming, inline dataset lookups, and `config timeframe` in real-world context.

## Correlation Template

`correlation-template.yml` is the YAML structure exported from XSIAM for a correlation rule. Key fields:

- `name` / `alert_name` — display name
- `xql_query` — the detection XQL (multiline string)
- `severity` — `SEV_010_CRITICAL`, `SEV_020_HIGH`, `SEV_030_MEDIUM`, `SEV_040_LOW`, `SEV_060_INFORMATIONAL`
- `execution_mode` — `REAL_TIME` or `SCHEDULED`
- `search_window` / `crontab` — used for scheduled rules
- `suppression_enabled` / `suppression_duration` / `suppression_fields`
- `mitre_defs` — MITRE ATT&CK mapping
- `global_rule_id` — UUID v4, generated fresh for each new rule
- `alert_domain` — `DOMAIN_SECURITY` (default)
- `alert_category` — `OTHER` or more specific category
