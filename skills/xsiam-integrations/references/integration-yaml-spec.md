# XSIAM Integration YAML Specification

Integrations use **unified YAML** for import: a single `.yml` file containing metadata AND embedded Python code. This is the only format XSIAM accepts for direct import.

---

## Integration Structure

For integrations, the `script` key is a **mapping** (not a string). The Python code lives inside `script.script`.

### Top-Level Fields

```yaml
commonfields:
  id: IntegrationName      # Unique identifier — matches the integration name
  version: -1              # Always -1

name: Integration Display Name
display: Integration Display Name
category: Utilities        # See category values below
description: One-line description of what this integration does.
```

**Common `category` values:** `Authentication`, `Utilities`, `Messaging`, `Network Security`, `Vulnerability Management`, `Case Management`, `Endpoint Detection and Response`, `SIEM`, `Forensics`, `Data Enrichment/Lookup`

### `configuration` Section

Array of parameters shown in the integration instance settings UI:

```yaml
configuration:
  - display: Server URL
    name: server_url
    type: 0
    required: true
    defaultvalue: https://api.example.com
    additionalinfo: Base URL of the API server

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
    defaultvalue: 'false'
```

Always include `insecure` and `proxy` params — they are expected by `BaseClient`.

### `script` Section (Integration Format)

The `script` key is a **mapping** with nested fields:

```yaml
script:
  script: |-
    import demistomock as demisto
    from CommonServerPython import *
    from CommonServerUserPython import *

    class VendorClient(BaseClient):
        def __init__(self, base_url, api_key, verify=True, proxy=False):
            super().__init__(base_url=base_url, verify=verify, proxy=proxy)
            self.api_key = api_key

        def _get_headers(self):
            return {'Authorization': f'Bearer {self.api_key}'}

        def get_data(self, query, limit=50):
            return self._http_request(
                method='GET',
                url_suffix='/api/v1/data',
                headers=self._get_headers(),
                params={'q': query, 'limit': limit}
            )

    def get_data_command(client, args):
        query = args.get('query')
        limit = arg_to_number(args.get('limit')) or 50
        data = client.get_data(query=query, limit=limit)
        items = data.get('items', [])
        return CommandResults(
            readable_output=tableToMarkdown('Results', items),
            outputs_prefix='Vendor.Data',
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
            client = VendorClient(
                base_url=params.get('server_url'),
                api_key=params.get('api_key'),
                verify=not params.get('insecure', False),
                proxy=params.get('proxy', False)
            )
            command = demisto.command()
            demisto.debug(f'Command being called is {command}')

            if command == 'test-module':
                return_results(test_module(client))
            elif command == 'vendor-get-data':
                return_results(get_data_command(client, demisto.args()))
            else:
                raise NotImplementedError(f'Command {command} is not implemented')

        except Exception as e:
            return_error(f'Failed to execute {demisto.command()}: {str(e)}')

    if __name__ in ('__main__', '__builtin__', 'builtins'):
        main()
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.12.12.6947692
  isfetch: false
  isfetchevents: false
  runonce: false

  commands:
    - name: vendor-get-data
      description: Retrieves data from the Vendor API
      arguments:
        - name: query
          description: Search query string
          required: true
        - name: limit
          description: Maximum number of results to return (default 50)
          required: false
          defaultValue: '50'
      outputs:
        - contextPath: Vendor.Data.ID
          description: Result ID
          type: string
        - contextPath: Vendor.Data.Name
          description: Result name
          type: string
```

**Key points about the `script` section:**
- `script.script` contains the full Python code as a `|-` literal block scalar
- `script.type` is `python` (not `python3` — subtype handles the version)
- `script.subtype` is `python3`
- `script.dockerimage` must use a pinned version, not `:latest`
- `script.commands` lists all available commands with their arguments and outputs
- `script.isfetch: true` required if the integration fetches incidents

### Command Definition Fields

Each command in `script.commands`:

```yaml
commands:
  - name: vendor-action-name    # kebab-case, vendor-prefixed
    description: What this command does
    polling: true               # Only for ScheduledCommand polling commands
    arguments:
      - name: param_name
        description: Parameter description
        required: true          # Omit if optional
        defaultValue: '50'      # String default; key is defaultValue
        isArray: true           # Set true if accepts comma-separated list
        predefined:             # Allowed values for dropdown
          - option1
          - option2
        secret: true            # Mask this value in logs
    outputs:
      - contextPath: Vendor.Object.Field
        description: Field description
        type: string            # string | number | boolean | date | unknown | array | object
```

---

## Configuration Parameter Types

| Type | Name | Description | Example use |
|------|------|-------------|-------------|
| 0 | Short Text | Single-line text input | `server_url`, `username` |
| 4 | Encrypted | Password field (encrypted at rest) | `api_key`, `password` |
| 8 | Boolean | Checkbox (true/false) | `insecure`, `proxy` |
| 9 | Authentication | Username + password pair | Auth credential pair |
| 12 | Single Select | Dropdown with one choice | `log_level`, `region` |
| 13 | Text Area | Multi-line text input | `ca_certificate`, `private_key` |
| 15 | Multi Select | Multiple selections allowed | `incident_types`, `severities` |

### Select Parameter Example (Type 12)

```yaml
- display: Log Level
  name: log_level
  type: 12
  required: false
  defaultvalue: INFO
  options:
    - DEBUG
    - INFO
    - WARNING
    - ERROR
```

---

## Content-Pack-Only Fields (Never Include for Direct XSIAM Import)

These are CI/content-pack conventions only — real XSIAM tenant imports do not include them:
- `fromversion` — content pack minimum version requirement
- `marketplaces` — marketplace targeting
- `tests` — CI test configuration
- `timeout` — not present in real XSIAM exports

---

## Complete Integration Example

A minimal but complete integration YAML ready for XSIAM import:

```yaml
commonfields:
  id: ExampleAPI
  version: -1

name: Example API
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
    defaultvalue: 'false'

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


    def get_data_command(client, args):
        query = args.get('query')
        limit = arg_to_number(args.get('limit')) or 50

        data = client.get_data(query=query, limit=limit)
        items = data.get('items', [])

        return CommandResults(
            readable_output=tableToMarkdown(
                'Example API - Data Results',
                items,
                headers=['ID', 'Name', 'Status']
            ),
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
  dockerimage: demisto/python3:3.12.12.6947692
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
          description: Maximum number of results to return
          required: false
          defaultValue: '50'
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
```

---

## Common Field Definitions

### Context Paths
Format: `Vendor.Object.Field`

Examples: `PaloAltoNetworksXDR.Endpoint.ID`, `Example.Data.Status`

### Output Data Types
`string`, `number`, `boolean`, `date`, `unknown`, `array`, `object`

### Docker Images
Always pin to a specific version — check your XSIAM tenant for the latest available:
- `demisto/python3:3.12.12.6947692` — standard Python 3.12
- Never use `:latest`
