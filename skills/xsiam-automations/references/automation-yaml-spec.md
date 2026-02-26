# XSOAR/XSIAM Automation YAML Specification — Unified Format

XSIAM requires **unified YAML** for import: a single `.yml` file containing both metadata AND embedded Python code. This is the only format that can be imported directly into XSIAM.

## Table of Contents

1. [Integration Unified YAML](#integration-unified-yaml)
2. [Script Unified YAML](#script-unified-yaml)
3. [Configuration Parameter Types](#configuration-parameter-types)
4. [Complete Integration Example](#complete-integration-example)
5. [Complete Script Example](#complete-script-example)
6. [Common Field Definitions](#common-field-definitions)

---

## Integration Unified YAML

Integration YAML files define custom integrations that connect XSOAR/XSIAM to external systems. The Python code is embedded inside the `script.script` field.

### Top-Level Fields

#### `commonfields`
```yaml
commonfields:
  id: IntegrationName
  version: -1
```
- `id`: Unique identifier (matches the integration name)
- `version`: Always -1

#### `name` and `display`
```yaml
name: Integration Display Name
display: Integration Display Name
```

#### `category`
Common values: Authentication, Utilities, Messaging, Network Security, Vulnerability Management, Case Management, Endpoint Detection and Response, SIEM, Forensics, Data Enrichment/Lookup

#### `description`
```yaml
description: Integrates with Example API to fetch and query data.
```

#### `configuration`
Array of configuration parameters:
```yaml
configuration:
  - display: Server URL
    name: server_url
    defaultvalue: https://api.example.com
    type: 0
    required: true
    additionalinfo: The base URL of the API server

  - display: API Key
    name: api_key
    type: 4
    required: true

  - display: Trust any certificate (not secure)
    name: insecure
    type: 8
    required: false
    defaultvalue: 'false'

  - display: Use system proxy settings
    name: proxy
    type: 8
    required: false
    defaultvalue: 'true'
```

#### `script` (Integration Format)

This is the critical section. For integrations, the `script` key is a mapping with nested fields including `script` (the code), `type`, `subtype`, `dockerimage`, and `commands`.

```yaml
script:
  script: |-
    import demistomock as demisto
    from CommonServerPython import *
    from CommonServerUserPython import *

    class ExampleClient(BaseClient):
        def __init__(self, base_url, api_key, verify=True):
            super().__init__(base_url=base_url, verify=verify)
            self.api_key = api_key

        def get_data(self, query, limit=50):
            return self._http_request(
                method='GET',
                url_suffix='/api/v1/data',
                headers={'Authorization': f'Bearer {self.api_key}'},
                params={'q': query, 'limit': limit}
            )

    def get_data_command(client, args):
        query = args.get('query')
        limit = int(args.get('limit', 50))
        data = client.get_data(query=query, limit=limit)
        items = data.get('items', [])
        return CommandResults(
            readable_output=tableToMarkdown('Results', items),
            outputs_prefix='Example.Data',
            outputs_key_field='id',
            outputs=items
        )

    def test_module(client):
        try:
            client.get_data(query='test', limit=1)
            return 'ok'
        except Exception as e:
            return f'Test failed: {str(e)}'

    def main():
        try:
            params = demisto.params()
            client = ExampleClient(
                base_url=params.get('server_url'),
                api_key=params.get('api_key'),
                verify=not params.get('insecure', False)
            )
            command = demisto.command()
            if command == 'test-module':
                return_results(test_module(client))
            elif command == 'example-get-data':
                return_results(get_data_command(client, demisto.args()))
            else:
                return_error(f'Unknown command: {command}')
        except Exception as e:
            return_error(f'Failed to execute {demisto.command()}: {str(e)}')

    if __name__ in ('__main__', '__builtin__', 'builtins'):
        main()
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.10.14.100715
  isfetch: false
  isfetchevents: false
  runonce: false

  commands:
    - name: example-get-data
      description: Retrieves data from the Example API
      arguments:
        - name: query
          description: Search query string
          required: true
        - name: limit
          description: Maximum number of results (default 50)
          required: false
          default: 50
      outputs:
        - contextPath: Example.Data.ID
          description: Result ID
          type: string
        - contextPath: Example.Data.Name
          description: Result name
          type: string
```

**Key points about the `script` section:**
- `script.script` contains the full Python code as a `|-` literal block scalar
- `script.type` is `python` (not `python3` — the subtype handles the version)
- `script.subtype` is `python3`
- `script.dockerimage` should use a pinned version, not `:latest`
- `script.commands` lists all available commands with their arguments and outputs

#### `fromversion` and `marketplaces`
```yaml
fromversion: 6.5.0
marketplaces:
  - xsoar
  - marketplacev2
```

#### `tests`
```yaml
tests:
  - No tests
```

---

## Script Unified YAML

Scripts are simpler than integrations. The Python code goes in a top-level `script` field (a string, not a mapping).

### Script Structure

```yaml
commonfields:
  id: ScriptName
  version: -1

name: Script Display Name
comment: |
  Description of what this script does.

script: |-
  import demistomock as demisto
  from CommonServerPython import *
  from CommonServerUserPython import *

  def main():
      try:
          args = demisto.args()
          input_data = args.get('input_data')
          output_format = args.get('output_format', 'json')

          # Process the data
          result = process_data(input_data, output_format)

          return_results(CommandResults(
              readable_output=tableToMarkdown('Results', result),
              outputs_prefix='ProcessedData',
              outputs=result
          ))
      except Exception as e:
          return_error(f'Error: {str(e)}')

  def process_data(data, fmt):
      # Implementation here
      return {'result': data, 'format': fmt}

  if __name__ in ('__main__', '__builtin__', 'builtins'):
      main()

type: python
subtype: python3
dockerimage: demisto/python3:3.10.14.100715

tags:
  - Utility

timeout: '0'

args:
  - name: input_data
    description: The data to process
    required: true
  - name: output_format
    description: Output format
    required: false
    default: json
    predefined:
      - json
      - csv
      - markdown

outputs:
  - contextPath: ProcessedData.Result
    description: The processed result
    type: string

fromversion: 6.5.0
tests:
  - No tests

marketplaces:
  - xsoar
  - marketplacev2
```

**Key differences from integration YAML:**
- `script` is a **string** (top-level), not a mapping — code goes directly under `script: |-`
- `type` and `subtype` are **top-level** fields, not nested under `script`
- `dockerimage` is **top-level**, not nested under `script`
- No `configuration` section
- No `commands` array — the script itself is a single command
- `comment` instead of `description`
- `args` and `outputs` are top-level (not nested under a command)

---

## Configuration Parameter Types

| Type | Name | Description | Example |
|------|------|-------------|---------|
| 0 | Short Text | Single-line text input | `server_url`, `username` |
| 4 | Encrypted | Password field (encrypted) | `api_key`, `password` |
| 8 | Boolean | Checkbox (true/false) | `insecure`, `proxy` |
| 9 | Authentication | Username + password pair | Auth parameter |
| 12 | Single Select | Dropdown with one choice | `log_level` |
| 13 | Text Area | Multi-line text input | `ca_certificate` |
| 15 | Multi Select | Multiple selections | `incident_types` |

### Select Example (Type 12)
```yaml
- display: Authentication Type
  name: auth_type
  type: 12
  required: true
  options:
    - API Key
    - Basic Auth
    - OAuth
```

---

## Complete Integration Example

A minimal but complete integration YAML ready for XSIAM import:

```yaml
commonfields:
  id: ExampleAPI
  version: -1

name: Example API Integration
display: Example API
category: Utilities
description: Integrates with the Example API to fetch and query data.

configuration:
  - display: API Server URL
    name: server_url
    type: 0
    required: true
    defaultvalue: https://api.example.com
    additionalinfo: Base URL of the Example API server

  - display: API Key
    name: api_key
    type: 4
    required: true

  - display: Trust any certificate (not secure)
    name: insecure
    type: 8
    required: false
    defaultvalue: 'false'

  - display: Use system proxy settings
    name: proxy
    type: 8
    required: false
    defaultvalue: 'true'

script:
  script: |-
    import demistomock as demisto
    from CommonServerPython import *
    from CommonServerUserPython import *


    class ExampleClient(BaseClient):
        def __init__(self, base_url, api_key, verify=True, proxy=False):
            super().__init__(base_url=base_url, verify=verify, proxy=proxy)
            self.api_key = api_key

        def _get_headers(self):
            return {
                'Authorization': f'Bearer {self.api_key}',
                'Content-Type': 'application/json'
            }

        def get_data(self, query, limit=50):
            return self._http_request(
                method='GET',
                url_suffix='/api/v1/data',
                headers=self._get_headers(),
                params={'q': query, 'limit': limit}
            )

        def test_connection(self):
            self._http_request(
                method='GET',
                url_suffix='/api/v1/health',
                headers=self._get_headers()
            )
            return True


    def get_data_command(client, args):
        query = args.get('query')
        limit = int(args.get('limit', 50))

        data = client.get_data(query=query, limit=limit)
        items = data.get('items', [])

        human_readable = tableToMarkdown(
            'Example API - Data Results',
            items,
            headers=['ID', 'Name', 'Status']
        )

        return CommandResults(
            readable_output=human_readable,
            outputs_prefix='Example.Data',
            outputs_key_field='id',
            outputs=items
        )


    def test_module(client):
        try:
            client.test_connection()
            return 'ok'
        except DemistoException as e:
            return f'Connection failed: {str(e)}'


    def main():
        try:
            params = demisto.params()
            client = ExampleClient(
                base_url=params.get('server_url'),
                api_key=params.get('api_key'),
                verify=not params.get('insecure', False),
                proxy=params.get('proxy', False)
            )

            command = demisto.command()
            demisto.debug(f'Command being called is {command}')

            if command == 'test-module':
                return_results(test_module(client))
            elif command == 'example-get-data':
                return_results(get_data_command(client, demisto.args()))
            else:
                raise NotImplementedError(f'Command {command} is not implemented')

        except Exception as e:
            return_error(f'Failed to execute {demisto.command()}: {str(e)}')


    if __name__ in ('__main__', '__builtin__', 'builtins'):
        main()
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.10.14.100715
  isfetch: false
  runonce: false

  commands:
    - name: example-get-data
      description: Retrieves data from the Example API
      arguments:
        - name: query
          description: Search query string
          required: true
        - name: limit
          description: Maximum number of results to return
          required: false
          default: 50
      outputs:
        - contextPath: Example.Data.ID
          description: The unique ID of the result
          type: string
        - contextPath: Example.Data.Name
          description: The name of the result
          type: string
        - contextPath: Example.Data.Status
          description: The status of the result
          type: string

fromversion: 6.5.0
tests:
  - No tests

marketplaces:
  - xsoar
  - marketplacev2
```

---

## Complete Script Example

A minimal but complete script YAML ready for XSIAM import:

```yaml
commonfields:
  id: FormatData
  version: -1

name: FormatData
comment: |
  Formats input data into the specified output format (JSON, CSV, or Markdown table).
  Accepts raw data and returns it in a structured format.

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

          # Parse input if it's a JSON string
          if isinstance(input_data, str):
              try:
                  input_data = json.loads(input_data)
              except json.JSONDecodeError:
                  pass

          # Format based on requested output
          if output_format == 'json':
              result = json.dumps(input_data, indent=2)
          elif output_format == 'markdown':
              if isinstance(input_data, list):
                  result = tableToMarkdown('Formatted Data', input_data)
              else:
                  result = tableToMarkdown('Formatted Data', [input_data])
          elif output_format == 'csv':
              if isinstance(input_data, list) and input_data:
                  headers = list(input_data[0].keys())
                  lines = [','.join(headers)]
                  for item in input_data:
                      lines.append(','.join(str(item.get(h, '')) for h in headers))
                  result = '\n'.join(lines)
              else:
                  result = str(input_data)
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
subtype: python3
dockerimage: demisto/python3:3.10.14.100715

tags:
  - Utility
  - Data Processing

timeout: '0'

args:
  - name: input_data
    description: The data to format (JSON string, dict, or list)
    required: true
  - name: output_format
    description: Desired output format
    required: false
    default: json
    predefined:
      - json
      - csv
      - markdown

outputs:
  - contextPath: FormattedData.Result
    description: The formatted output string
    type: string
  - contextPath: FormattedData.Format
    description: The format used
    type: string

fromversion: 6.5.0
tests:
  - No tests

marketplaces:
  - xsoar
  - marketplacev2
```

---

## Common Field Definitions

### Context Paths
Format: `Vendor.Object.Field`

Examples: `IP.Address`, `Domain.Name`, `File.MD5`, `Example.Data.ID`

### Data Types
`string`, `number`, `boolean`, `date`, `unknown`, `array`, `object`

### Marketplace Targeting
- `marketplacev2` — available in XSIAM
- `xsoar` — available in XSOAR on-prem
- Include both for maximum compatibility

### Version
- `fromversion: 6.5.0` minimum for XSIAM support
- Use semantic versioning: `X.Y.Z`

### Docker Images
Always pin to a specific version:
- `demisto/python3:3.10.14.100715` (standard Python 3)
- `demisto/python3:3.10.13.86450` (alternative pinned)
- Never use `:latest` in production
