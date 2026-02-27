---
name: xsiam-scripts
description: >
  This skill should be used when the user asks to "create a script",
  "write an XSIAM script", "build a script", "write an automation",
  "XSOAR script", "demisto script", "generate script YAML", or needs to
  develop a standalone Python script with its YAML metadata file for
  Cortex XSIAM or XSOAR. For connecting to external APIs with multiple
  commands, use the xsiam-integrations skill instead.
version: 1.0.0
---

# XSIAM Script Development

Generate importable unified YAML files for Cortex XSIAM/XSOAR scripts. The Python code MUST be embedded directly inside the YAML file — this is the only format XSIAM accepts for import.

## Before Starting

Read the reference files:
- `references/script-yaml-spec.md` — Script YAML structure, field ordering, and complete example
- `../xsiam-shared/references/common-patterns.md` — Python patterns: outputs, error handling, demisto API, arg parsing, indicators

## What is a Script?

Scripts are **standalone automations** that process data, orchestrate actions, or transform context — without connecting directly to an external product. The Python code is embedded in a top-level `script: |-` field.

Use a script when:
- Processing or transforming incident/alert context data
- Chaining multiple integration commands together via `execute_command()`
- Implementing utility logic (formatting, enrichment, normalization)
- Running calculations or lookups on existing context data

Use the **xsiam-integrations** skill instead when you need to connect to an external API with authentication and multiple commands.

## Workflow

### 1. Gather Requirements

Determine:
- What the script does (input → processing → output)
- Arguments it accepts (names, types, required/optional, defaults)
- Context output it produces (`contextPath` prefix and fields)
- Whether it operates on incident/alert context, indicators, or standalone data
- Whether it chains to other integrations via `execute_command()`

### 2. Generate the Unified YAML

Build a single `.yml` file. Field order must match real XSIAM export structure:

1. `commonfields` — `id` and `version: -1`
2. `vcShouldKeepItemLegacyProdMachine: false` — required, always present
3. `name` — the script name
4. `script: |-` — embedded Python code
5. `type: python`
6. `tags` — list of tags or `[]`
7. `comment` — description block
8. `enabled: true` — required
9. `args` — each entry must start with `supportedModules: []`
10. `outputs` — `contextPath`, `description`, `type` for each
11. `scripttarget: 0` — required
12. `subtype: python3`
13. `pswd: ""` — required
14. `runonce: false`
15. `dockerimage` — pinned `3.12.x` version
16. `runas: DBotWeakRole` — required
17. `engineinfo: {}` — required
18. `mainengineinfo: {}` — required

### 3. Python Code Conventions

The embedded Python follows XSOAR/XSIAM conventions:
- Standard imports: `demistomock`, `CommonServerPython`, `CommonServerUserPython`
- Simple `main()` with `try/except` and `return_error()`
- Parse args with `argToList()`, `argToBoolean()`, `arg_to_number()`, `arg_to_datetime()` — never raw casting
- Return results with `CommandResults` and `return_results()`
- Chain to integrations with `execute_command('command-name', args)` — not raw `demisto.executeCommand()`
- In XSIAM alert-context scripts: use `demisto.alert()` (not `demisto.incident()`)

### 4. File Output

Generate a single file:
- `ScriptName.yml` — the unified YAML ready for import into XSIAM

Optionally also generate:
- `ScriptName_test.py` — pytest test file (separate, not for import)

### 5. Validation Checklist

Before delivering, verify:
- [ ] Python code is embedded using `script: |-` (top-level string, not nested mapping)
- [ ] Python indentation is consistent within the YAML block (2 spaces from key level)
- [ ] Standard imports present (`demistomock`, `CommonServerPython`, `CommonServerUserPython`)
- [ ] `main()` has `try/except` with `return_error()`
- [ ] Field ordering matches spec: `commonfields → vcShouldKeepItemLegacyProdMachine → name → script → type → tags → comment → enabled → args → outputs → scripttarget → subtype → pswd → runonce → dockerimage → runas → engineinfo → mainengineinfo`
- [ ] `vcShouldKeepItemLegacyProdMachine: false` is present at top level
- [ ] `enabled: true`, `scripttarget: 0`, `pswd: ""`, `runas: DBotWeakRole`, `engineinfo: {}`, `mainengineinfo: {}` are all present
- [ ] Every `args` entry has `supportedModules: []` as its first key
- [ ] Arg defaults use `defaultValue:` key (not `default:`); list args include `isArray: true`
- [ ] All args have descriptions
- [ ] All outputs have `contextPath`, `description`, and `type`
- [ ] Docker image is a pinned `3.12.x` version (not `:latest`)
- [ ] **Do not include** `fromversion`, `marketplaces`, `tests`, `timeout` — content-pack CI fields only
- [ ] **Do not include** `register_module_line()` calls — platform-injected on export
- [ ] No tab characters; consistent YAML indentation throughout
- [ ] XSIAM alert-context scripts use `demisto.alert()` not `demisto.incident()`
- [ ] Arg parsing uses helpers (`argToList`, `argToBoolean`, `arg_to_number`, `arg_to_datetime`), not raw casting

## Key Conventions

- Script names: `kebab-case` (e.g., `jp-get-case-url`, `normalize-timestamp`)
- Python functions and variables: `snake_case`
- Context prefix format: `Vendor.Object` (e.g., `FormattedData.Result`)
- Docker image: pinned `3.12.x` (e.g., `demisto/python3:3.12.12.6947692`) — check your tenant for the latest available build
- `demisto.alert()` for XSIAM alert context; `demisto.incident()` is the XSOAR equivalent
- Use `execute_command()` (CommonServerPython alias) not `demisto.executeCommand()` directly
