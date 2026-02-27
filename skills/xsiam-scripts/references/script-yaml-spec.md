# XSIAM Script YAML Specification

Scripts use **unified YAML** for import: a single `.yml` file containing metadata AND embedded Python code. This is the only format XSIAM accepts for direct import.

---

## Script Structure

The correct field order matches real XSIAM exported files:

```yaml
commonfields:
  id: ScriptName           # UUID for exported scripts; use name or UUID for new imports
  version: -1              # Always -1 for new/authored scripts

vcShouldKeepItemLegacyProdMachine: false   # Required — always false

name: script-name          # Script name (matches commonfields.id for new scripts)
script: |-
  import demistomock as demisto
  from CommonServerPython import *
  from CommonServerUserPython import *

  def main():
      try:
          args = demisto.args()
          result = process(args.get('input'))
          return_results(CommandResults(
              readable_output=tableToMarkdown('Results', [result]),
              outputs_prefix='ScriptName',
              outputs=result
          ))
      except Exception as e:
          return_error(f'Error: {str(e)}')

  if __name__ in ('__main__', '__builtin__', 'builtins'):
      main()

type: python
tags:
  - Utility          # or tags: []
comment: |
  Description of what this script does.
enabled: true        # Required — always true

args:
  - supportedModules: []    # Required as first key on every arg entry
    name: input
    required: true
    description: The input data
  - supportedModules: []
    name: limit
    description: Maximum number of results
    defaultValue: '50'      # String default; key is defaultValue, not default
    isArray: false          # Set true if this arg accepts a comma-separated list

outputs:
  - contextPath: ScriptName.Result
    description: The result
    type: string

scripttarget: 0      # Required — always 0
subtype: python3
pswd: ""             # Required — always empty string
runonce: false
dockerimage: demisto/python3:3.12.12.6947692   # Use latest 3.12 available in your tenant
runas: DBotWeakRole  # Required — always DBotWeakRole
engineinfo: {}       # Required — always empty map
mainengineinfo: {}   # Required — always empty map
```

### Field Notes

- `script` is a **string** (top-level `|-` literal block), not a mapping
- `type` and `subtype` are **top-level** fields, not nested under `script`
- `dockerimage` is **top-level**, not nested
- No `configuration` section — scripts do not connect to external APIs
- No `commands` array — the script itself is a single invocable unit
- `comment` (not `description`) is the human-readable description field
- `args` and `outputs` are top-level (not inside a command)

### Arg Field Reference

| Field | Required | Notes |
|-------|----------|-------|
| `supportedModules: []` | Yes | Always first key; always empty list |
| `name` | Yes | Argument name in `kebab-case` or `snake_case` |
| `required` | No | Omit if false; set `required: true` if mandatory |
| `description` | Yes | Human-readable description |
| `defaultValue` | No | String default value (key is `defaultValue`, not `default`) |
| `isArray` | No | `true` if the arg accepts a comma-separated list |
| `default` | No | Boolean `true` marks this as the script's "default argument" (no name needed in war room) — separate from `defaultValue` |
| `predefined` | No | List of allowed values for dropdown selection |
| `secret` | No | `true` if the value should be masked |

### Platform-Generated Fields

These appear in XSIAM **exports** — do not write them by hand:
- `restrictioncenter: {}` and `signature: ""` — conditionally present; always empty
- `register_module_line('script-name', 'start', __line__())` / `register_module_line('script-name', 'end', __line__())` — injected by the platform at export time; writing them manually causes double-wrapping

### Content-Pack-Only Fields (Never Include for Direct XSIAM Import)

These are CI/content-pack conventions only — real XSIAM tenant imports do not include them:
- `fromversion` — content pack minimum version
- `marketplaces` — marketplace targeting
- `tests` — CI test configuration
- `timeout` — not present in real XSIAM exports

---

## Script Main Function Pattern

Scripts do not use a `BaseClient`. The `main()` function reads args directly:

```python
def main():
    try:
        args = demisto.args()

        # Parse arguments with helpers — never raw casting
        input_data = args.get('input_data', '')
        items = argToList(args.get('ids'))           # "a,b,c" → ['a','b','c']
        limit = arg_to_number(args.get('limit')) or 50
        enabled = argToBoolean(args.get('enabled', 'true'))

        # Process
        result = process(input_data, items, limit)

        # Return
        return_results(CommandResults(
            readable_output=tableToMarkdown('Results', result),
            outputs_prefix='ScriptName',
            outputs_key_field='id',
            outputs=result
        ))

    except Exception as e:
        return_error(f'Failed to execute script. Error: {str(e)}')


if __name__ in ('__main__', '__builtin__', 'builtins'):
    main()
```

---

## Execute Command Pattern (Scripts Only)

Scripts chain to integrations or other scripts using `execute_command()` (the CommonServerPython alias). It raises `DemistoException` automatically on error — no manual `isError()` check needed:

```python
# Preferred: execute_command() — raises on error automatically
def enrich_ip(ip: str) -> dict:
    result = execute_command('ip', {'ip': ip})
    return result  # already unwrapped to Contents


# Multiple enrichments
def enrich_multiple(ips: list) -> list:
    results = []
    for ip in ips:
        data = execute_command('ip', {'ip': ip})
        results.append(data)
    return results


# If you need raw control, use demisto.executeCommand() with manual error checking
def enrich_ip_low_level(ip: str) -> dict:
    results = demisto.executeCommand('ip', {'ip': ip})
    if isError(results[0]):
        raise DemistoException(f'ip command failed: {results[0].get("Contents")}')
    return results[0].get('Contents', {})
```

**Notes:**
- `execute_command()` is available via `from CommonServerPython import *`
- Use these in **scripts** to chain integrations; avoid inside integrations themselves
- For XSIAM alert-context scripts, access alert data with `demisto.alert()`, not `demisto.incident()`

---

## Complete Script Example

A minimal but complete script YAML ready for XSIAM import:

```yaml
commonfields:
  id: FormatData
  version: -1

vcShouldKeepItemLegacyProdMachine: false

name: FormatData
script: |-
  import demistomock as demisto
  from CommonServerPython import *
  from CommonServerUserPython import *
  import json


  def main():
      try:
          args = demisto.args()
          input_data = args.get('input_data')
          output_format = args.get('output_format', 'json')

          if not input_data:
              raise ValueError('input_data argument is required')

          if isinstance(input_data, str):
              try:
                  input_data = json.loads(input_data)
              except json.JSONDecodeError:
                  pass

          if output_format == 'json':
              result = json.dumps(input_data, indent=2)
          elif output_format == 'markdown':
              if isinstance(input_data, list):
                  result = tableToMarkdown('Formatted Data', input_data)
              else:
                  result = tableToMarkdown('Formatted Data', [input_data])
          else:
              raise ValueError(f'Unsupported format: {output_format}')

          return_results(CommandResults(
              readable_output=f'### Formatted Output\n```\n{result}\n```',
              outputs_prefix='FormattedData',
              outputs={'Result': result, 'Format': output_format}
          ))

      except Exception as e:
          return_error(f'Error formatting data: {str(e)}')


  if __name__ in ('__main__', '__builtin__', 'builtins'):
      main()

type: python
tags:
  - Utility
  - Data Processing
comment: |
  Formats input data into the specified output format (JSON or Markdown table).
enabled: true

args:
  - supportedModules: []
    name: input_data
    required: true
    description: The data to format (JSON string, dict, or list)
  - supportedModules: []
    name: output_format
    description: Desired output format
    defaultValue: json
    predefined:
      - json
      - markdown

outputs:
  - contextPath: FormattedData.Result
    description: The formatted output string
    type: string
  - contextPath: FormattedData.Format
    description: The format used
    type: string

scripttarget: 0
subtype: python3
pswd: ""
runonce: false
dockerimage: demisto/python3:3.12.12.6947692
runas: DBotWeakRole
engineinfo: {}
mainengineinfo: {}
```

---

## Common Field Definitions

### Context Paths
Format: `Vendor.Object.Field`

Examples: `FormattedData.Result`, `NormalizedTimestamp.Value`, `CaseInfo.URL`

### Output Data Types
`string`, `number`, `boolean`, `date`, `unknown`, `array`, `object`

### Docker Images
Always pin to a specific version — check your XSIAM tenant for the latest available:
- `demisto/python3:3.12.12.6947692` — standard Python 3.12
- Never use `:latest`
