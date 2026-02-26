---
name: xsiam-automations
description: >
  This skill should be used when the user asks to "create an automation",
  "write an XSIAM script", "build an integration", "generate automation YAML",
  "XSOAR script", "demisto automation", "content pack script", or needs to
  develop Python automation scripts with their corresponding YAML metadata
  files for Cortex XSIAM or XSOAR.
version: 0.2.0
---

# XSIAM Automation Development

Generate importable unified YAML files for Cortex XSIAM/XSOAR automations. The Python code MUST be embedded directly inside the YAML file — this is the only format XSIAM accepts for import.

## Before Starting

Read the reference files to understand the required formats:
- `references/automation-yaml-spec.md` — Unified YAML structure with embedded script field
- `references/script-patterns.md` — Python code patterns and conventions

## Critical Rule: Unified YAML Format

XSIAM imports require **unified YAML** — a single `.yml` file that contains both the metadata AND the Python code embedded in the `script` field. Never generate separate `.py` and `.yml` files for import.

**For scripts**, the Python code goes in a top-level `script` field:
```yaml
script: |-
  import demistomock as demisto
  from CommonServerPython import *

  def main():
      ...

  if __name__ in ('__main__', '__builtin__', 'builtins'):
      main()
```

**For integrations**, the Python code goes inside `script.script`:
```yaml
script:
  script: |-
    import demistomock as demisto
    from CommonServerPython import *

    class MyClient(BaseClient):
        ...

    def main():
        ...

    if __name__ in ('__main__', '__builtin__', 'builtins'):
        main()
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.10.14.100715
  commands:
    - name: my-command
      ...
```

The `|-` YAML literal block scalar is required — it preserves newlines and strips the trailing newline. Indent all Python code consistently (2 spaces from the YAML key level).

## Workflow

### 1. Determine Automation Type

Ask the user which type they need:
- **Integration** — connects to an external product/API with multiple commands
- **Script** — standalone automation that processes data or orchestrates actions

### 2. Gather Requirements

For an **integration**, determine:
- Target product/vendor and API documentation
- Authentication method (API key, OAuth, basic auth)
- Commands needed (what operations to support)
- For each command: required/optional arguments and expected outputs

For a **script**, determine:
- What the script does (input → processing → output)
- Arguments it accepts
- Context output it produces
- Whether it operates on incident context, indicators, or standalone data

### 3. Generate the Unified YAML

Build a single `.yml` file containing:

1. **Metadata fields** — commonfields, name, display, category, description
2. **Configuration** (integrations only) — parameters for authentication, URL, proxy, etc.
3. **Script section with embedded Python code** — the actual code inside `script: |-` or `script.script: |-`
4. **Commands/arguments/outputs** — command definitions with their args and context outputs
5. **Version and marketplace** — fromversion, marketplaces

The Python code inside the YAML follows XSOAR conventions:
- Standard imports: `demistomock`, `CommonServerPython`, `CommonServerUserPython`
- Main function pattern with try/except and `return_error()`
- For integrations: `BaseClient` subclass with `_http_request` calls
- `CommandResults` for output with `outputs_prefix`, `outputs_key_field`, `readable_output`
- `tableToMarkdown()` for human-readable tables
- Command routing in main: `if command == 'test-module'` → `elif command == 'command-name'`

### 4. File Output

Generate a single file:
- `AutomationName.yml` — the unified YAML ready for import into XSIAM

Optionally also generate:
- `AutomationName_test.py` — pytest test file (separate, not for import)
- `README.md` — command reference documentation (separate)

### 5. Validation Checklist

Before delivering, verify:
- [ ] Python code is embedded in the `script` field using `|-` block scalar
- [ ] Python code indentation is consistent within the YAML block
- [ ] Python imports are correct (demistomock, CommonServerPython)
- [ ] main() has proper try/except with return_error()
- [ ] YAML has all required fields (commonfields, name, type/subtype)
- [ ] All command arguments have descriptions
- [ ] All outputs have contextPath, description, and type
- [ ] test-module command is implemented (integrations)
- [ ] dockerimage is specified with a real version tag
- [ ] fromversion is set (minimum 6.5.0 for XSIAM compatibility)
- [ ] marketplaces includes "marketplacev2" for XSIAM
- [ ] The YAML is valid — no tab characters, consistent indentation

## Key Conventions

- Use `snake_case` for Python function and variable names
- Use `PascalCase` for class names
- Command names: `vendor-action-object` (e.g., `xdr-get-endpoints`)
- Context prefix: `Vendor.Object` (e.g., `PaloAltoNetworksXDR.Endpoint`)
- Always include a `test-module` command for integrations
- Use `@logger` decorator on command functions for debug logging
- Handle pagination in API calls — never assume single-page results
- Docker image: use a pinned version like `demisto/python3:3.10.14.100715`, not `:latest`
