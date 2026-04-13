# Specialized Script Types

Standard scripts (no functional tag) follow the main workflow in SKILL.md. This file covers the specialized types that require specific tags and follow distinct patterns.

---

## Transformer Scripts

Transformers appear as options in playbook task input mappings. They receive a value and return a transformed version.

**Tag:** `transformer`

**Conventions:**
- Receive input via the `value` argument (passed automatically by the platform)
- May accept additional configuration arguments
- Return the transformed value via `return_results()`
- No `outputs` needed in YAML — the transformed value flows back into the playbook task input

**Example — Normalize Timestamp:**

```yaml
commonfields:
  id: NormalizeTimestamp
  version: -1

vcShouldKeepItemLegacyProdMachine: false

name: NormalizeTimestamp
script: |-
  register_module_line('NormalizeTimestamp', 'start', __line__())

  import demistomock as demisto
  from CommonServerPython import *
  from CommonServerUserPython import *
  from datetime import datetime, timezone


  def normalize(value: str, output_format: str) -> str:
      dt = arg_to_datetime(value, arg_name='value', required=True)
      if dt is None:
          raise ValueError(f'Could not parse timestamp: {value}')
      dt_utc = dt.astimezone(timezone.utc)
      return dt_utc.strftime(output_format)


  def main():
      try:
          args = demisto.args()
          value = args.get('value', '')
          output_format = args.get('output_format', '%Y-%m-%dT%H:%M:%SZ')

          result = normalize(value, output_format)
          return_results(result)

      except Exception as e:
          return_error(f'Error normalizing timestamp: {str(e)}')


  if __name__ in ('__main__', '__builtin__', 'builtins'):
      main()

  register_module_line('NormalizeTimestamp', 'end', __line__())

type: python
tags:
  - transformer
comment: |
  Normalizes a timestamp string to UTC ISO 8601 format.
  Accepts ISO strings, epoch integers, and relative strings ("3 days ago").
enabled: true

args:
  - supportedModules: []
    name: value
    required: true
    description: The timestamp value to normalize (passed automatically by the platform)
  - supportedModules: []
    name: output_format
    description: strftime format string for the output
    defaultValue: '%Y-%m-%dT%H:%M:%SZ'

scripttarget: 0
subtype: python3
pswd: ""
runonce: false
dockerimage: demisto/python3:3.12.12.6947692
runas: DBotWeakRole
engineinfo: {}
mainengineinfo: {}
```

**Notes:**
- Transformer scripts return a raw value (string, list, dict) via `return_results()` — not wrapped in `CommandResults` with `outputs_prefix`
- The `value` argument is populated automatically when the script is used as a transformer in a playbook task
- `value` can also be passed manually when testing from the CLI: `!NormalizeTimestamp value="2024-01-15T10:30:00+05:00"`

---

## Filter Scripts

Filters appear as condition options in playbook task conditions. They evaluate whether a value meets a condition.

**Tag:** `filter`

**Conventions:**
- Receive the value to test via the `value` argument
- May accept additional arguments for the condition (e.g., a threshold, a pattern)
- Return a boolean-like string (`"true"` or `"false"`) or an entry with `Contents` set to a boolean
- No `outputs` needed in YAML — the boolean result feeds back into the playbook condition

**Example — Check IP in CIDR Range:**

```yaml
commonfields:
  id: IsIPInCIDR
  version: -1

vcShouldKeepItemLegacyProdMachine: false

name: IsIPInCIDR
script: |-
  register_module_line('IsIPInCIDR', 'start', __line__())

  import demistomock as demisto
  from CommonServerPython import *
  from CommonServerUserPython import *
  from ipaddress import ip_address, ip_network


  def main():
      try:
          args = demisto.args()
          value = args.get('value', '')
          cidr_ranges = argToList(args.get('cidr_ranges'))

          if not value or not cidr_ranges:
              return_results(False)
              return

          ip = ip_address(value)
          match = any(ip in ip_network(cidr, strict=False) for cidr in cidr_ranges)
          return_results(match)

      except ValueError:
          return_results(False)
      except Exception as e:
          return_error(f'Error checking IP: {str(e)}')


  if __name__ in ('__main__', '__builtin__', 'builtins'):
      main()

  register_module_line('IsIPInCIDR', 'end', __line__())

type: python
tags:
  - filter
comment: |
  Checks whether an IP address falls within any of the specified CIDR ranges.
  Returns true if the IP matches any range, false otherwise.
enabled: true

args:
  - supportedModules: []
    name: value
    required: true
    description: The IP address to check (passed automatically by the platform)
  - supportedModules: []
    name: cidr_ranges
    required: true
    description: Comma-separated list of CIDR ranges to check against
    isArray: true

scripttarget: 0
subtype: python3
pswd: ""
runonce: false
dockerimage: demisto/python3:3.12.12.6947692
runas: DBotWeakRole
engineinfo: {}
mainengineinfo: {}
```

**Notes:**
- Filter scripts return a bare boolean via `return_results(True)` or `return_results(False)`
- The `value` argument is populated automatically when used as a filter in a playbook condition
- Use standard library modules (`ipaddress`, `re`) to keep filter scripts self-contained

---

## Dynamic Section Scripts

Dynamic sections render read-only display content inside case/issue layout tabs. They re-execute each time the tab is opened.

**Tag:** `dynamic-section`

**Conventions:**
- Typically no arguments — pull data from `demisto.context()` or `demisto.alert()`
- No `outputs` in YAML — display-only
- Return only `readable_output` (markdown or HTML) via `CommandResults`
- Keep execution fast — runs on every tab open

**Example — Display DBot Scores Table:**

```yaml
commonfields:
  id: DisplayDBotScores
  version: -1

vcShouldKeepItemLegacyProdMachine: false

name: DisplayDBotScores
script: |-
  register_module_line('DisplayDBotScores', 'start', __line__())

  import demistomock as demisto
  from CommonServerPython import *
  from CommonServerUserPython import *


  SCORE_LABELS = {0: 'Unknown', 1: 'Good', 2: 'Suspicious', 3: 'Malicious'}


  def normalize_to_list(data):
      if isinstance(data, dict):
          return [data]
      if isinstance(data, list):
          return data
      return []


  def main():
      try:
          dbot_scores = demisto.context().get('DBotScore')
          if not dbot_scores:
              return_results(CommandResults(
                  readable_output='No DBot scores found in context.'
              ))
              return

          entries = normalize_to_list(dbot_scores)
          rows = []
          for entry in entries:
              score = entry.get('Score', 0)
              rows.append({
                  'Indicator': entry.get('Indicator', ''),
                  'Type': entry.get('Type', ''),
                  'Vendor': entry.get('Vendor', ''),
                  'Score': SCORE_LABELS.get(score, str(score)),
                  'Reliability': entry.get('Reliability', '')
              })

          rows.sort(key=lambda r: (
              -max(e.get('Score', 0) for e in entries
                   if e.get('Indicator') == r['Indicator']),
              r['Indicator'],
              r['Vendor']
          ))

          return_results(CommandResults(
              readable_output=tableToMarkdown(
                  'DBot Scores',
                  rows,
                  headers=['Indicator', 'Type', 'Vendor', 'Score', 'Reliability'],
                  removeNull=True
              )
          ))

      except Exception as e:
          return_error(f'Error displaying DBot scores: {str(e)}')


  if __name__ in ('__main__', '__builtin__', 'builtins'):
      main()

  register_module_line('DisplayDBotScores', 'end', __line__())

type: python
tags:
  - dynamic-section
comment: |
  Displays a sorted table of all DBot reputation scores in the current context.
  Designed to be embedded as a dynamic section in case/issue layouts.
enabled: true

scripttarget: 0
subtype: python3
pswd: ""
runonce: false
dockerimage: demisto/python3:3.12.12.6947692
runas: DBotWeakRole
engineinfo: {}
mainengineinfo: {}
```

**Notes:**
- No `args` or `outputs` sections — dynamic sections read from context and display only
- `demisto.context()` returns the full context dict for the current incident/alert
- For XSIAM alert-specific data, use `demisto.alert()` and flatten `CustomFields`:
  ```python
  alert = demisto.alert()
  cf = alert.pop('CustomFields', {})
  alert.update(cf)
  ```
- Keep logic lightweight — this runs every time a user opens the layout tab

---

## Field-Change-Triggered Scripts

These scripts execute automatically when a specific field value changes on a case/issue layout.

**Tag:** `field-change-triggered`

**Conventions:**
- Receive the field change event payload via `demisto.args()`
- Key payload fields: `new` (new value), `old` (previous value, `null` if first set), `name` (field display name), `cliName` (field path), `type` (storage type), `selectValues` (allowed options for select fields)
- See `script-yaml-spec.md` → "Field-Change-Triggered Script: Arg Structure" for the full payload reference
- Typically perform validation, side effects (e.g., update another field), or logging

**Example — Validate Resolution Reason:**

```yaml
commonfields:
  id: ValidateResolutionReason
  version: -1

vcShouldKeepItemLegacyProdMachine: false

name: ValidateResolutionReason
script: |-
  register_module_line('ValidateResolutionReason', 'start', __line__())

  import demistomock as demisto
  from CommonServerPython import *
  from CommonServerUserPython import *


  REQUIRES_COMMENT = {
      'STATUS_070_RESOLVED_OTHER',
      'STATUS_080_RESOLVED_AUTO',
  }


  def main():
      try:
          args = demisto.args()
          new_value = args.get('new')
          old_value = args.get('old')
          field_name = args.get('name', 'Unknown field')

          if not new_value:
              demisto.debug(f'{field_name} was cleared (old={old_value})')
              return

          if new_value in REQUIRES_COMMENT:
              demisto.info(
                  f'{field_name} set to {new_value} — '
                  f'analyst should add a comment explaining the resolution.'
              )

          return_results(CommandResults(
              readable_output=f'{field_name} changed from {old_value} to {new_value}'
          ))

      except Exception as e:
          return_error(f'Error validating field change: {str(e)}')


  if __name__ in ('__main__', '__builtin__', 'builtins'):
      main()

  register_module_line('ValidateResolutionReason', 'end', __line__())

type: python
tags:
  - field-change-triggered
comment: |
  Validates resolution reason field changes. Logs a reminder when the selected
  reason requires an analyst comment.
enabled: true

scripttarget: 0
subtype: python3
pswd: ""
runonce: false
dockerimage: demisto/python3:3.12.12.6947692
runas: DBotWeakRole
engineinfo: {}
mainengineinfo: {}
```

**Notes:**
- No `args` section in YAML — the platform injects the field change payload automatically
- `new` is `null` when the field is cleared
- `old` is `null` when the field is set for the first time
- `type` in the payload reflects the internal storage type (e.g., `"multiSelect"` may appear for functionally single-select fields)
- `selectValues` lists all configured options — useful for validation
- Assign the script to a field in the XSIAM layout editor under the field's trigger settings

---

## Widget Scripts

Widget scripts power dashboard and report widgets. They are the least common script type.

**Tag:** `widget`

**Conventions:**
- The return structure must match what the dashboard renderer expects for the widget type (number, trend, table, pie chart, etc.)
- Typically no arguments — pull data from context, API calls, or `demisto.executeCommand()`
- Consult the XSIAM widget documentation for the expected return schema for each widget type

**Number widget example return** (assumes `import json` in script imports)**:**
```python
return_results(json.dumps({
    "Type": 17,
    "ContentsFormat": "number",
    "Contents": {"stats": 42, "params": {}}
}))
```

**Table widget example return:**
```python
data = [{"name": "Critical", "count": 5}, {"name": "High", "count": 12}]
return_results(json.dumps({
    "Type": 17,
    "ContentsFormat": "table",
    "Contents": data
}))
```

**Notes:**
- Widget scripts are tightly coupled to the widget type — always confirm the expected return schema before building
- Widget results are cached by the dashboard; execution frequency depends on dashboard refresh settings
- For XQL-powered widgets, prefer native XQL widgets over script-backed widgets when possible
