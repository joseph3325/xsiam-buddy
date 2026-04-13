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

scripttarget: 0      # 0 = server-side (default); 1 = endpoint-side (XDR agent)
subtype: python3
pswd: ""             # Required — always empty string
runonce: false
dockerimage: demisto/python3:3.12.12.6947692   # Use latest 3.12 available in your tenant
runas: DBotWeakRole  # DBotWeakRole (default) or DBotRole (elevated permissions)
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
- `scripttarget: 0` runs on the XSIAM server (default for all automation scripts). Use `scripttarget: 1` only for scripts that must execute directly on an endpoint via the XDR agent (e.g., forensic artifact collection, OS-level commands)
- `runas: DBotWeakRole` is the default — restricted permissions (read context, return results). Use `runas: DBotRole` for scripts that need elevated permissions (create/modify incidents, change integration settings, manage context keys)

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

### register_module_line (Required)

Every script must include `register_module_line()` calls as the **first and last lines** of the embedded Python:

```python
register_module_line('ScriptName', 'start', __line__())

import demistomock as demisto
from CommonServerPython import *
from CommonServerUserPython import *

# ... all code ...

if __name__ in ('__main__', '__builtin__', 'builtins'):
    main()

register_module_line('ScriptName', 'end', __line__())
```

- The string passed must match the script's `name` field exactly.
- These are **not** imports — they are bare function calls provided by the platform runtime.
- Do **not** add an `if __name__` guard around them; they must always execute at module load time.

### Platform-Generated Fields

These appear in XSIAM **exports** — do not write them by hand:
- `restrictioncenter: {}` and `signature: ""` — conditionally present; always empty

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
register_module_line('ScriptName', 'start', __line__())

import demistomock as demisto
from CommonServerPython import *
from CommonServerUserPython import *


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

register_module_line('ScriptName', 'end', __line__())
```

---

## Execute Command Pattern (Scripts Only)

Scripts chain to integrations or other scripts using `demisto.executeCommand()`. It returns a list of result entries — check for errors with `isError()` before accessing `Contents`:

```python
def enrich_ip(ip: str) -> dict:
    results = demisto.executeCommand('ip', {'ip': ip})
    if isError(results[0]):
        raise DemistoException(f'ip command failed: {results[0].get("Contents")}')
    return results[0].get('Contents', {})


# Allow partial success across multiple IPs
def enrich_multiple(ips: list) -> list:
    enriched = []
    for ip in ips:
        results = demisto.executeCommand('ip', {'ip': ip})
        if isError(results[0]):
            demisto.error(f'Enrichment failed for {ip}: {results[0].get("Contents")}')
            continue
        enriched.append(results[0].get('Contents', {}))
    return enriched
```

**Notes:**
- `demisto.executeCommand()` returns a list of entry dicts; always check `isError(results[0])` before using `Contents`
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
  register_module_line('FormatData', 'start', __line__())

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

  register_module_line('FormatData', 'end', __line__())

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

---

## Tags Reference

Tags serve two purposes: **functional** tags that enable a script to appear in specific XSIAM UI locations, and **informational** tags that are purely organizational.

### Functional Tags

These tags wire the script into specific platform features. A script can carry multiple functional tags.

| Tag | Effect |
|-----|--------|
| `transformer` | Script appears as a transformer option in playbook task input mappings |
| `filter` | Script appears as a filter/condition option in playbook task conditions |
| `field-change-triggered` | Script can be selected as a field-change trigger on layout fields |
| `field-display` | Script can be selected to dynamically render the value of a specific layout field |
| `dynamic-section` | Script can be used as a general-purpose dynamic display section in layouts |
| `widget` | Script appears as a selectable widget in dashboards and reports |

### Informational Tags

Tags like `Utility`, `Data Processing`, `Enrichment`, etc. are organizational only — they do not affect where the script appears in the UI. Use them to make scripts easier to find in the content library.

### Example: multi-purpose script

```yaml
tags:
  - transformer
  - Utility
```

### Notes

- A script with no functional tag is only accessible by name (e.g., via an Execute Script task in a playbook, or `demisto.executeCommand('ScriptName', args)` from another script).
- `transformer` and `filter` scripts receive their input via the `value` argument by convention.
- `field-display` and `dynamic-section` scripts typically return HTML or Markdown for rendering.
- `widget` scripts must return a structure understood by the dashboard renderer.

### Field-Change-Triggered Script: Arg Structure

When a script is assigned as a `field-change-triggered` handler for a layout field, XSIAM passes the field change event data as the script's arguments. Use `demisto.args()` to access the payload.

**Key fields:**

| Field | Description |
|-------|-------------|
| `name` | Human-readable field name (e.g., `"Resolution Reason"`) |
| `cliName` | CLI path to the field (e.g., `"status.resolution_reason"`) |
| `new` | The new value after the change |
| `old` | The previous value (`null` if the field was previously unset) |
| `type` | How XSIAM stores the field internally (e.g., `"multiSelect"`, `"shortText"`) |
| `selectValues` | All possible option values for select-type fields |

**Example payload** (Resolution Reason field — a single-select field in the UI):

```json
{
  "associatedToAll": true,
  "associatedTypes": null,
  "cliName": "status.resolution_reason",
  "content": true,
  "description": "",
  "isReadOnly": false,
  "name": "Resolution Reason",
  "new": "STATUS_070_RESOLVED_OTHER",
  "old": null,
  "ownerOnly": false,
  "placeholder": "",
  "required": false,
  "selectValues": [
    "STATUS_040_RESOLVED_KNOWN_ISSUE",
    "STATUS_050_RESOLVED_DUPLICATE",
    "STATUS_060_RESOLVED_FALSE_POSITIVE",
    "STATUS_070_RESOLVED_OTHER",
    "STATUS_090_RESOLVED_TRUE_POSITIVE",
    "STATUS_100_RESOLVED_SECURITY_TESTING",
    "STATUS_080_RESOLVED_AUTO",
    "STATUS_030_RESOLVED_THREAT_HANDLED",
    "STATUS_240_RESOLVED_DISMISSED",
    "STATUS_250_RESOLVED_FIXED",
    "STATUS_130_RESOLVED_RISK_ACCEPTED"
  ],
  "system": false,
  "type": "multiSelect",
  "unmapped": false,
  "useAsKpi": false,
  "validationRegex": ""
}
```

**Notes:**
- The most useful fields are `new` (new value), `old` (previous value — `null` if first set), `name`, and `cliName`
- `type` in the payload reflects XSIAM's internal field storage type; `"multiSelect"` may appear even for fields that are functionally single-select in the UI
- `selectValues` lists all configured options for select-type fields — useful for validation or value mapping
- `new` is `null` when the field is cleared

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
