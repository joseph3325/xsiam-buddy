# XSIAM Integration YAML Specification

Integrations use **unified YAML** for import: a single `.yml` file containing metadata AND embedded Python code. This is the only format XSIAM accepts for direct import.

---

## Integration Structure

For integrations, the `script` key is a **mapping** (not a string). The Python code lives inside `script.script`.

### Top-Level Field Order

Unified YAML follows this exact field order (matching real XSIAM exports):

```yaml
commonfields:
  id: IntegrationName
  version: -1
vcShouldKeepItemLegacyProdMachine: false
name: Integration Display Name
display: Integration Display Name
category: Utilities
description: One-line description of what this integration does.
sectionorder:
- Connect
- Collect
configuration:
  # ... params ...
script:
  # ... code, flags, commands ...
```

**Key structural notes:**
- `vcShouldKeepItemLegacyProdMachine: false` goes between `commonfields` and `name`
- `sectionorder` uses lowercase 'o' (not camelCase `sectionOrder`) — this matches real XSIAM export format
- `sectionorder` is a simple list of section names; each param declares its section via the `section:` field

**Common `category` values:** `Authentication`, `Utilities`, `Messaging`, `Network Security`, `Vulnerability Management`, `Case Management`, `Endpoint Detection and Response`, `Analytics & SIEM`, `Forensics`, `Data Enrichment/Lookup`

### `configuration` Section

Array of parameters shown in the integration instance settings UI. Each parameter starts with `supportedModules: []` and includes a `section:` field. Fields follow this order:

```
supportedModules → section → advanced (if true) → display → displaypassword (if type 9) → name → type → required → defaultvalue → additionalinfo → options (if select) → hiddenusername (if type 9)
```

```yaml
configuration:
- supportedModules: []
  section: Connect
  display: Server URL
  name: server_url
  type: 0
  required: true
  defaultvalue: https://api.example.com
  additionalinfo: Base URL of the API server.

- supportedModules: []
  section: Connect
  display: API Key
  name: api_key
  type: 4
  required: true
  additionalinfo: API key for authentication.

- supportedModules: []
  section: Connect
  advanced: true
  display: Trust any certificate (not secure)
  name: insecure
  type: 8
  required: false
  defaultvalue: 'false'

- supportedModules: []
  section: Connect
  advanced: true
  display: Use system proxy settings
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
    register_module_line('IntegrationName', 'start', __line__())



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

    register_module_line('IntegrationName', 'end', __line__())
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.12.12.6947692
  isfetch: false
  isfetchevents: false
  runonce: false
  commands:
  - supportedModules: []
    name: vendor-get-data
    description: Retrieves data from the Vendor API
    arguments:
    - supportedModules: []
      name: query
      description: Search query string
      required: true
    - supportedModules: []
      name: limit
      description: Maximum number of results to return
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
- `register_module_line()` calls go as the first and last lines of the Python code
- `script.type` is `python` (not `python3` — subtype handles the version)
- `script.subtype` is `python3`
- `script.dockerimage` must use a pinned version, not `:latest`
- `script.commands` lists all available commands with their arguments and outputs
- `script.isfetch: true` required if the integration fetches incidents
- Every command starts with `supportedModules: []`
- Every argument starts with `supportedModules: []`

### Command Definition Fields

Each command in `script.commands`:

```yaml
  commands:
  - supportedModules: []
    name: vendor-action-name    # kebab-case, vendor-prefixed
    description: What this command does
    polling: true               # Only for ScheduledCommand polling commands
    arguments:
    - supportedModules: []
      name: param_name
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
| 0 | Short Text | Single-line text input | `server_url`, `username`, `max_fetch` |
| 1 | Long Text | Multi-line text (no encryption) | `certificate_pem`, `query_filter` |
| 4 | Encrypted | Password field (encrypted at rest) | `api_key`, `password`, `client_secret` |
| 8 | Boolean | Checkbox (true/false) | `insecure`, `proxy`, `isFetch` |
| 9 | Credentials | Username + password pair (vault-compatible) | Auth credential pair; platform resolves vault references at runtime |
| 12 | Single Select | Dropdown — one choice from `options` list | `log_level`, `region` |
| 13 | Text Area (alt) | Multi-line text input (alternate rendering) | `private_key`, `ca_bundle` |
| 14 | Multi Select | Multiple selections from `options` list | `event_types`, `severities` |
| 15 | Single Select (alt) | Predefined dropdown (alternate) | `incident_type_filter` |
| 16 | Incident Type | Incident type selector dropdown (platform UI) | `incident_type` for fetch integrations |

### Select Parameter Examples

**Single Select (Type 15):**

```yaml
- supportedModules: []
  section: Connect
  display: API Version
  name: api_version
  type: 15
  required: true
  defaultvalue: '2.1'
  options:
  - '2.0'
  - '2.1'
```

**Multi Select (Type 14):**

```yaml
- supportedModules: []
  section: Collect
  display: Event Types to Fetch
  name: event_types
  type: 14
  required: false
  options:
  - authentication
  - network
  - endpoint
  - cloud
  additionalinfo: Select which event types to ingest. Leave empty for all.
```

**Credentials (Type 9):**

```yaml
- supportedModules: []
  section: Connect
  display: ''
  displaypassword: API Token
  name: credentials
  type: 9
  required: true
  hiddenusername: true
  additionalinfo: API token for authentication. Supports credential vault references.
```

---

## sectionorder and Tabbed Configuration UI

Integrations with 4+ parameters should use `sectionorder` to organize the configuration UI into tabs:

```yaml
sectionorder:
- Connect
- Collect
```

Each parameter references its section with `section:` and optionally `advanced: true` to collapse it under "Advanced":

**Tab assignment rules:**
- **Connect** — server URL, credentials, proxy, insecure, API version
- **Collect** — fetch toggle, incident type, max fetch, first fetch, event type filters
- **Optimize** (optional) — rarely needed; for log level, caching, batch size

---

## Fetch-Incidents YAML Configuration

Integrations that ingest alerts require these `script` flags:

```yaml
script:
  isfetch: true
  isFetchSamples: true
```

And these configuration parameters (in the `Collect` section):

| Parameter | Type | Purpose |
|---|---|---|
| `incident_type` | 16 | Incident type selector |
| `max_fetch` | 0 | Max incidents per fetch cycle (default `50`) |
| `first_fetch` | 0 | How far back to fetch on first run (default `3 days`) |
| `look_back` | 0 | Lookback window in minutes for late-arriving incidents (default `10`) |

The fetch interval is controlled by the platform (configured when creating the integration instance), not by a YAML parameter.

---

## Credential Vault Configuration

For integrations consuming vault-managed credentials, use a type 9 parameter and set `isFetchCredentials: true`:

```yaml
configuration:
- supportedModules: []
  section: Connect
  display: ''
  displaypassword: API Token
  name: credentials
  type: 9
  required: true
  hiddenusername: true
  additionalinfo: Username and password. Supports credential vault references.

script:
  isFetchCredentials: true
```

At runtime, the platform resolves vault references and passes the resolved `identifier` (username) and `password` to `demisto.params()`.

---

## Content-Pack-Only Fields (Never Include for Direct XSIAM Import)

These are CI/content-pack conventions only — real XSIAM tenant imports do not include them:
- `fromversion` — content pack minimum version requirement
- `marketplaces` — marketplace targeting
- `tests` — CI test configuration
- `timeout` — not present in real XSIAM exports

---

## Complete Integration Example

A production-ready integration YAML with fetch-incidents, sectionorder, and credential vault support:

```yaml
commonfields:
  id: ExampleAPI
  version: -1
vcShouldKeepItemLegacyProdMachine: false
name: Example API
display: Example API
category: Utilities
description: Integrates with the Example API to fetch alerts and query data.
sectionorder:
- Connect
- Collect
configuration:
- supportedModules: []
  section: Connect
  display: API Server URL
  name: server_url
  type: 0
  required: true
  defaultvalue: https://api.example.com
  additionalinfo: Base URL of the Example API server.

- supportedModules: []
  section: Connect
  display: ''
  displaypassword: API Key
  name: credentials
  type: 9
  required: true
  hiddenusername: true
  additionalinfo: API key for authentication. Supports credential vault references.

- supportedModules: []
  section: Connect
  advanced: true
  display: Trust any certificate (not secure)
  name: insecure
  type: 8
  required: false
  defaultvalue: 'false'

- supportedModules: []
  section: Connect
  advanced: true
  display: Use system proxy settings
  name: proxy
  type: 8
  required: false
  defaultvalue: 'false'

- supportedModules: []
  section: Collect
  display: Fetch incidents
  name: isFetch
  type: 8
  required: false
  defaultvalue: 'true'

- supportedModules: []
  section: Collect
  display: Incident type
  name: incident_type
  type: 16
  required: false

- supportedModules: []
  section: Collect
  display: Maximum incidents per fetch
  name: max_fetch
  type: 0
  required: false
  defaultvalue: '50'
  additionalinfo: Maximum number of incidents to fetch per cycle.

- supportedModules: []
  section: Collect
  display: First fetch timestamp
  name: first_fetch
  type: 0
  required: false
  defaultvalue: 3 days
  additionalinfo: "Date or relative time (e.g., '3 days', '2024-01-01T00:00:00Z')."

script:
  script: |-
    register_module_line('ExampleAPI', 'start', __line__())


    import json


    class ExampleClient(BaseClient):
        def __init__(self, base_url, username, api_key, verify=True, proxy=False):
            super().__init__(
                base_url=base_url,
                verify=verify,
                proxy=proxy,
                headers={
                    'Authorization': f'Bearer {api_key}',
                    'Content-Type': 'application/json'
                },
                retries=3,
                backoff_factor=1.0,
            )
            self.username = username

        def get_data(self, query, limit=50):
            return self._http_request(
                method='GET',
                url_suffix='/api/v1/data',
                params={'q': query, 'limit': limit}
            )

        def list_alerts(self, start_time, end_time, limit=50):
            return self._http_request(
                method='GET',
                url_suffix='/api/v1/alerts',
                params={'since': start_time, 'until': end_time, 'limit': limit}
            )

        def test_connection(self):
            self._http_request(method='GET', url_suffix='/api/v1/health')


    def get_data_command(client, args):
        query = args.get('query')
        limit = arg_to_number(args.get('limit')) or 50
        data = client.get_data(query=query, limit=limit)
        items = data.get('items', [])
        return CommandResults(
            readable_output=tableToMarkdown('Example API - Data Results', items, headers=['ID', 'Name', 'Status']),
            outputs_prefix='Example.Data',
            outputs_key_field='id',
            outputs=items
        )


    def fetch_incidents(client, params, last_run):
        start_time, end_time = get_fetch_run_time_range(
            last_run=last_run,
            first_fetch=params.get('first_fetch', '3 days'),
            look_back=int(params.get('look_back', 1)),
            date_format='%Y-%m-%dT%H:%M:%SZ',
        )
        max_fetch = int(params.get('max_fetch', 50))
        raw_alerts = client.list_alerts(start_time=start_time, end_time=end_time, limit=max_fetch)
        alerts = raw_alerts.get('alerts', [])

        incidents, new_last_run = filter_incidents_by_duplicates_and_limit(
            incidents_res=alerts,
            last_run=last_run,
            fetch_limit=max_fetch,
            id_field='id',
        )

        formatted = []
        for alert in incidents:
            formatted.append({
                'name': alert.get('title', 'Unnamed Alert'),
                'occurred': alert.get('created_at', ''),
                'rawJSON': json.dumps(alert),
                'type': params.get('incident_type', 'Example Alert'),
                'severity': map_severity(alert.get('severity')),
            })

        demisto.incidents(formatted)
        demisto.setLastRun(update_last_run_object(
            last_run=new_last_run,
            incidents=incidents,
            fetch_limit=max_fetch,
            id_field='id',
            date_field='created_at',
        ))


    def map_severity(api_severity):
        return {
            'informational': 0, 'low': 1, 'medium': 2, 'high': 3, 'critical': 4
        }.get((api_severity or '').lower(), 0)


    def test_module(client):
        try:
            client.test_connection()
            return 'ok'
        except DemistoException as e:
            if e.res is not None:
                if e.res.status_code == 401:
                    return 'Authorization failed — check credentials'
                if e.res.status_code == 403:
                    return 'Forbidden — check API key permissions'
            if 'Connection' in str(e):
                return f'Connection failed — verify server URL: {e}'
            raise


    def main():
        try:
            params = demisto.params()
            credentials = params.get('credentials', {})
            client = ExampleClient(
                base_url=params.get('server_url'),
                username=credentials.get('identifier', ''),
                api_key=credentials.get('password', ''),
                verify=not params.get('insecure', False),
                proxy=params.get('proxy', False)
            )

            command = demisto.command()
            demisto.debug(f'Command being called is {command}')

            if command == 'test-module':
                return_results(test_module(client))
            elif command == 'fetch-incidents':
                fetch_incidents(client, params, demisto.getLastRun())
            elif command == 'example-get-data':
                return_results(get_data_command(client, demisto.args()))
            else:
                raise NotImplementedError(f'Command {command} is not implemented')

        except Exception as e:
            return_error(f'Failed to execute {demisto.command()}: {str(e)}')


    if __name__ in ('__main__', '__builtin__', 'builtins'):
        main()

    register_module_line('ExampleAPI', 'end', __line__())
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.12.12.6947692
  isfetch: true
  isFetchSamples: true
  isfetchevents: false
  isFetchCredentials: true
  runonce: false
  commands:
  - supportedModules: []
    name: example-get-data
    description: Retrieves data from the Example API
    arguments:
    - supportedModules: []
      name: query
      description: Search query string
      required: true
    - supportedModules: []
      name: limit
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
