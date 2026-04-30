---
name: xsiam-splunk-to-xql
description: >
  This skill should be used when the user asks to "translate SPL",
  "convert Splunk to XQL", "Splunk to XQL", "SPL to XQL",
  "migrate Splunk query", "translate this Splunk query",
  or needs to convert existing Splunk SPL queries into equivalent XQL.
---

# Splunk SPL to XQL Translation

## Scope

Translate existing Splunk SPL queries into equivalent Cortex XSIAM XQL.

**This skill handles:**
- Converting SPL commands and functions to XQL equivalents
- Mapping Splunk index/sourcetype references to XSIAM datasets
- Flagging untranslatable SPL constructs with workaround notes
- Preserving query intent through structural translation

**This skill does NOT handle:**
- Writing XQL from scratch (no SPL input) -> use `xsiam-xql`
- Building correlation rules -> use `xsiam-correlations`
- Writing automation scripts -> use `xsiam-scripts`
- Building integrations -> use `xsiam-integrations`

## Before Starting

Read these reference files before translating any query:

**Always read (core):**
1. `references/spl-to-xql-mapping.md` -- Command mapping, function mapping, dataset mapping, untranslatable constructs, translation examples
2. `../xsiam-shared/references/xql-core-reference.md` -- All 23 XQL stages, 60 common functions, operators, arrow notation, best practices

**Read on-demand (when the translated query uses these features):**
3. `../xsiam-shared/references/xql-advanced-functions.md` -- Read when the SPL query involves array manipulation, complex JSON parsing, window functions, or advanced IP operations
4. `../xsiam-shared/references/xql-datasets-extended.md` -- Read when the SPL query references third-party data sources, email datasets, cloud identity, or custom HTTP collector data
5. `../xsiam-shared/references/xql-federated-search.md` -- Read when the SPL query references external data stores (S3, GCS, Azure Blob)

## Workflow

### Step 1 -- Parse SPL

Break down the input SPL query into its component parts:
- Identify each SPL command in the pipeline (stats, eval, where, rex, etc.)
- List all functions used within commands (lower, substr, mvcount, etc.)
- Note any lookups, macros, or saved-search references
- Extract field names and their usage context
- Identify index/sourcetype references

If the SPL contains macros, ask the user to expand them first or provide the macro definitions. Macros cannot be translated without knowing their contents.

### Step 2 -- Map Commands

Translate each SPL command to its XQL equivalent using the command mapping table in `references/spl-to-xql-mapping.md`:
- Replace each SPL command with the corresponding XQL stage
- Pay attention to syntax differences (e.g., `sort` requires explicit `asc`/`desc` in XQL)
- Handle renamed fields: SPL `rename X as Y` becomes XQL `alter Y = X`

### Step 3 -- Flag Untranslatable Constructs

Check for SPL features that have no direct XQL equivalent:
- Consult the "Known Untranslatable Constructs" table in the mapping reference
- For each untranslatable feature, add a comment in the output noting the limitation
- Provide a workaround where one exists (e.g., manual array construction for mvzip)
- If the untranslatable construct is central to the query logic, warn the user that the translation is partial

### Step 4 -- Map Datasets

Translate Splunk data source references to XSIAM datasets:
- Map `index=` and `sourcetype=` to the appropriate XSIAM dataset using the mapping table
- Do NOT blindly carry over Splunk index names as dataset names
- When no exact mapping exists, suggest the most likely XSIAM dataset and note the assumption
- Add appropriate `filter` stages for event type or vendor discrimination within broad datasets

### Step 5 -- Assemble XQL

Build the final XQL query following the recommended stage order:

`dataset/preset/datamodel` -> `filter` -> `alter` -> `comp` -> `fields` -> `sort` -> `limit` -> `dedup`

Key assembly rules:
- Place filters early to reduce data volume
- Combine multiple SPL `eval` commands into a single `alter` stage where possible
- Ensure `comp` aggregations include `by` clauses (unless a single-row result is intended)
- Use `fields` to select only the columns needed in the final output

### Step 6 -- Validate

Run through the quality checklist below before delivering the translated query.

## Output Format

Return the translated XQL in a fenced code block with header comments:

```xql
// Title: <descriptive title>
// Translated from SPL
// Description: <what the query does>
// Datasets: <dataset(s) used>
// Modified: <YYYY-MM-DD>
// Translation notes: <any assumptions, partial translations, or workarounds>
```

If any SPL constructs could not be translated, add inline `// UNTRANSLATED:` comments at the relevant position in the query, plus a summary after the code block listing each gap.

## Quality Checklist

Before delivering the translated query, verify:

1. **Every SPL command accounted for** -- each command is either translated to XQL or explicitly flagged as unsupported with a workaround note
2. **No SPL syntax remnants** -- no `stats`, `eval`, `table`, `where`, `rex`, or other SPL keywords appear in the XQL output
3. **Dataset mapping correct** -- Splunk index/sourcetype references are translated to valid XSIAM datasets, not blindly carried over
4. **XQL stage order follows recommendation** -- filter early, alter before comp, fields for final output
5. **Header comment includes "Translated from SPL" note** -- so the query origin is always clear

## Known Limitations

- **Not all SPL is translatable.** Some SPL commands (cluster, kmeans, predict, anomalydetection) have no XQL equivalent. The skill will flag these and suggest alternatives where possible.
- **Macros must be expanded first.** SPL macros are opaque references; ask the user for definitions before translating.
- **Custom lookups need import.** Splunk CSV lookups must be imported as XSIAM lookup datasets before the translated `join` will work.
- **SPL subsearches translate differently.** SPL subsearches become XQL subqueries within `join` or `union` stages; complex nesting may require restructuring.
- **Format string differences.** SPL `strftime`/`strptime` format strings differ from XQL `format_timestamp`/`parse_timestamp`; manual adjustment is required.
