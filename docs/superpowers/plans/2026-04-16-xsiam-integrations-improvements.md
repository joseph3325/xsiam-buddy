# xsiam-integrations Skill Improvements — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deepen the xsiam-integrations skill with production-grade patterns from the RagVault — fetch-incident dedup/lookback, @polling_function, rate limiting, credential vault, sectionOrder, complete parameter types, and expanded indicator enrichment.

**Architecture:** Expand four existing Markdown reference files in-place. No new files. Content sourced from the RagVault at `/Users/joeplunkett/Documents/RagVault/wiki/`. Each task modifies exactly one file and can be committed independently.

**Tech Stack:** Markdown with embedded YAML and Python code blocks.

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `skills/xsiam-integrations/SKILL.md` | Modify (lines 40-115) | Workflow: requirements gathering, YAML generation steps, validation checklist |
| `skills/xsiam-integrations/references/integration-yaml-spec.md` | Modify (lines 190-399) | YAML spec: parameter types, sectionOrder, fetch config, credential vault, complete example |
| `skills/xsiam-integrations/references/integration-patterns.md` | Modify (lines 240-408) | Python patterns: @polling_function, fetch dedup, rate limiting, credentials, proxy, error handling |
| `skills/xsiam-shared/references/common-patterns.md` | Modify (lines 156-379) | Shared patterns: demisto API examples, CommonServerPython helpers, indicator enrichment, searchIndicators pagination |

---

### Task 1: Expand SKILL.md — Requirements Gathering & Validation

**Files:**
- Modify: `skills/xsiam-integrations/SKILL.md:40-115`

- [ ] **Step 1: Replace Section 1 (Gather Requirements) at lines 40-48**

Replace the current requirements section with an expanded version that includes the auth decision tree and conditional requirement questions. Find this text:

```
### 1. Gather Requirements

Determine:
- Target product/vendor and API documentation URL
- Authentication method (API key, OAuth, basic auth, certificate)
- Commands needed — what operations to support
- For each command: required/optional arguments and expected outputs
- Whether it fetches incidents/events into XSIAM (`isfetch: true`)
- Whether any commands are long-running and need polling (`ScheduledCommand`)
```

Replace with:

```markdown
### 1. Gather Requirements

Determine:
- Target product/vendor and API documentation URL
- Commands needed — what operations to support
- For each command: required/optional arguments and expected outputs

**Authentication method** — match the API's auth scheme to the correct pattern:

| Auth scheme | Integration pattern |
|---|---|
| API key in header | Bearer token via `_get_headers()` |
| Username + password | `HTTPBasicAuth` via `_http_request(auth=)` |
| OAuth2 client credentials | Token caching with `getIntegrationContext()` + expiry |
| OAuth2 authorization code | Device/auth code flow (rare for server-to-server) |
| Certificate-based | Mutual TLS via `_http_request()` cert params |

**Conditional requirements** — ask only when applicable:
- **Fetch incidents?** → Gather: alert endpoint, time field name, severity mapping, dedup strategy (ID-based vs. time-based), lookback window needs
- **Polling commands?** → Which commands are long-running async? Use `@polling_function` by default
- **Indicator enrichment?** → Which entity types (IP, Domain, URL, File, CVE)? What vendor score → DBotScore mapping?
- **Credential vault?** → Does target system use rotating credentials? → `isFetchCredentials` pattern with type 9 params
```

- [ ] **Step 2: Replace Section 2 (Generate the Unified YAML) at lines 50-69**

Find this text:

```
### 2. Generate the Unified YAML

Build a single `.yml` file with this top-level structure:

1. `commonfields` — `id` and `version: -1`
2. `name` and `display` — integration name
3. `category` — product category
4. `description` — one-line integration description
5. `configuration` — auth and connection parameters
6. `script` — nested mapping containing:
   - `script: |-` — embedded Python code
   - `type: python`
   - `subtype: python3`
   - `dockerimage` — pinned `3.12.x` version
   - `isfetch: false` (or `true` if fetching incidents)
   - `isfetchevents: false`
   - `runonce: false`
   - `commands` — array of command definitions

**Key structural difference from scripts:** For integrations, `script` is a **mapping** with nested fields. The Python code lives in `script.script`, not at the top level.
```

Replace with:

```markdown
### 2. Generate the Unified YAML

Build a single `.yml` file following these ordered sub-steps:

1. **Top-level metadata** — `commonfields` (`id`, `version: -1`), `name`, `display`, `category`, `description`
2. **`sectionOrder`** — define UI tabs: `Connect`, `Collect` (if fetching), `Optimize` (optional)
3. **`configuration`** — auth and connection parameters; assign each param a `section` and optionally `advanced: true`
4. **`script` mapping** — set flags: `type: python`, `subtype: python3`, `dockerimage` (pinned `3.12.x`), `isfetch` (true if fetching incidents), `isfetchevents: false`, `isFetchSamples` (true if fetching), `runonce: false`
5. **Command definitions** — full argument specs (`supportedModules`, `defaultValue`, `isArray`, `predefined`) and output specs (`contextPath`, `type`)
6. **Embed Python code** — insert into `script.script: |-` last

**Key structural difference from scripts:** For integrations, `script` is a **mapping** with nested fields. The Python code lives in `script.script`, not at the top level.
```

- [ ] **Step 3: Add new validation checklist items at line 115 (before the closing of Section 5)**

Find this text (the last items of the existing checklist):

```
- [ ] Token-caching integrations: integration context used, not global variables
- [ ] Sensitive config params use `type: 4` (encrypted)
```

Replace with:

```markdown
- [ ] Token-caching integrations: integration context used, not global variables
- [ ] Sensitive config params use `type: 4` (encrypted) or `type: 9` (credential vault pair)
- [ ] `sectionOrder` present when integration has 4+ configuration parameters
- [ ] Configuration params have `section: Connect`/`Collect` and `advanced: true` where appropriate
- [ ] Fetch incidents: dedup IDs tracked in `lastRun`, `isFetchSamples: true` set in script section
- [ ] Polling: `@polling_function` decorator or `ScheduledCommand` used — never `time.sleep()` in regular commands
- [ ] Proxy + insecure params have `section: Connect` and `advanced: true`
```

- [ ] **Step 4: Commit**

```bash
git add skills/xsiam-integrations/SKILL.md
git commit -m "feat(xsiam-integrations): expand requirements gathering, YAML generation steps, and validation checklist"
```

---

### Task 2: Expand integration-yaml-spec.md — Parameter Types, sectionOrder, Fetch Config

**Files:**
- Modify: `skills/xsiam-integrations/references/integration-yaml-spec.md:190-399`

- [ ] **Step 1: Replace the Configuration Parameter Types section (lines 190-215)**

Find this text:

```markdown
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
```

Replace with:

````markdown
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

**Single Select (Type 12):**

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

**Multi Select (Type 14):**

```yaml
- display: Event Types to Fetch
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
- display: Credentials
  name: credentials
  type: 9
  required: true
  section: Connect
  additionalinfo: Username and password. Supports credential vault references.
```
````

- [ ] **Step 2: Add new sectionOrder section after the parameter types section (after the select parameter examples, before the Content-Pack-Only Fields heading)**

Insert this new section before the line `## Content-Pack-Only Fields (Never Include for Direct XSIAM Import)`:

````markdown
---

## sectionOrder and Tabbed Configuration UI

Integrations with 4+ parameters should use `sectionOrder` to organize the configuration UI into tabs:

```yaml
sectionOrder:
  - Connect
  - Collect
```

Each parameter references its section with `section:` and optionally `advanced: true` to collapse it under "Advanced":

```yaml
configuration:
  - display: Server URL
    name: server_url
    type: 0
    required: true
    section: Connect
    additionalinfo: Base URL of the API server

  - display: API Key
    name: api_key
    type: 4
    required: true
    section: Connect

  - display: Trust any certificate (not secure)
    name: insecure
    type: 8
    required: false
    defaultvalue: 'false'
    section: Connect
    advanced: true

  - display: Use system proxy settings
    name: proxy
    type: 8
    required: false
    defaultvalue: 'false'
    section: Connect
    advanced: true

  - display: Incident type
    name: incident_type
    type: 16
    required: false
    section: Collect

  - display: Maximum incidents per fetch
    name: max_fetch
    type: 0
    required: false
    defaultvalue: '50'
    section: Collect

  - display: First fetch timestamp
    name: first_fetch
    type: 0
    required: false
    defaultvalue: 3 days
    section: Collect
    additionalinfo: "Date or relative time (e.g., '3 days', '2024-01-01T00:00:00Z')"
```

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
  - display: Credentials
    name: credentials
    type: 9
    required: true
    section: Connect
    additionalinfo: Username and password. Supports credential vault references.

script:
  isFetchCredentials: true
```

At runtime, the platform resolves vault references and passes the resolved `identifier` (username) and `password` to `demisto.params()`.
````

- [ ] **Step 3: Replace the Complete Integration Example (lines 229-382)**

Find the line `## Complete Integration Example` and replace everything from that heading through the closing ` ``` ` of the YAML block (before `---` and `## Common Field Definitions`).

Replace with this updated example that includes sectionOrder, fetch-incidents, and section assignments:

````markdown
## Complete Integration Example

A production-ready integration YAML with fetch-incidents, sectionOrder, and credential vault support:

```yaml
commonfields:
  id: ExampleAPI
  version: -1

name: Example API
display: Example API
category: Utilities
description: Integrates with the Example API to fetch alerts and query data.

sectionOrder:
  - Connect
  - Collect

configuration:
  - display: API Server URL
    name: server_url
    type: 0
    required: true
    defaultvalue: https://api.example.com
    additionalinfo: Base URL of the Example API server
    section: Connect

  - display: Credentials
    name: credentials
    type: 9
    required: true
    section: Connect
    additionalinfo: Username and API key. Supports credential vault references.

  - display: Trust any certificate (not secure)
    name: insecure
    type: 8
    required: false
    defaultvalue: 'false'
    section: Connect
    advanced: true

  - display: Use system proxy settings
    name: proxy
    type: 8
    required: false
    defaultvalue: 'false'
    section: Connect
    advanced: true

  - display: Fetch incidents
    name: isFetch
    type: 8
    required: false
    defaultvalue: 'true'
    section: Collect

  - display: Incident type
    name: incident_type
    type: 16
    required: false
    section: Collect

  - display: Maximum incidents per fetch
    name: max_fetch
    type: 0
    required: false
    defaultvalue: '50'
    section: Collect

  - display: First fetch timestamp
    name: first_fetch
    type: 0
    required: false
    defaultvalue: 3 days
    section: Collect
    additionalinfo: "Date or relative time (e.g., '3 days', '2024-01-01T00:00:00Z')"

script:
  script: |-
    from CommonServerPython import *
    from CommonServerUserPython import *
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
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.12.12.6947692
  isfetch: true
  isFetchSamples: true
  isfetchevents: false
  isFetchCredentials: true
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
````

- [ ] **Step 4: Commit**

```bash
git add skills/xsiam-integrations/references/integration-yaml-spec.md
git commit -m "feat(xsiam-integrations): add complete parameter types, sectionOrder, fetch-incidents config, credential vault, and expanded example"
```

---

### Task 3: Expand integration-patterns.md — Polling, Fetch Dedup, Rate Limiting, Credentials, Error Handling

**Files:**
- Modify: `skills/xsiam-integrations/references/integration-patterns.md:240-408`

- [ ] **Step 1: Replace the Fetch Incidents Pattern section (lines 242-296)**

Find the section starting with `## Fetch Incidents Pattern` and ending before `## Integration Context Pattern` (line 300).

Replace with:

````markdown
## Fetch Incidents Pattern

For integrations that ingest alerts into XSIAM. Uses CommonServerPython helpers for deduplication and lookback windows to handle late-arriving incidents.

**YAML:** Set `isfetch: true` and `isFetchSamples: true` in the `script` section.

### Production-Grade Fetch with Dedup & Lookback

```python
def fetch_incidents(client, params, last_run):
    """Fetch new incidents with deduplication and lookback window."""
    start_time, end_time = get_fetch_run_time_range(
        last_run=last_run,
        first_fetch=params.get('first_fetch', '3 days'),
        look_back=int(params.get('look_back', 1)),
        date_format='%Y-%m-%dT%H:%M:%SZ',
    )

    max_fetch = int(params.get('max_fetch', 50))
    raw_alerts = client.list_alerts(start_time=start_time, end_time=end_time)

    incidents, new_last_run = filter_incidents_by_duplicates_and_limit(
        incidents_res=raw_alerts,
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
            'type': params.get('incident_type', 'Vendor Alert'),
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


def map_severity(api_severity: str) -> int:
    """Map vendor severity string to Demisto severity level (0-4)."""
    return {
        'informational': 0,
        'low': 1,
        'medium': 2,
        'high': 3,
        'critical': 4
    }.get((api_severity or '').lower(), 0)


# In main():
if command == 'fetch-incidents':
    fetch_incidents(client, params, demisto.getLastRun())
```

### Helper Functions

| Function | Purpose |
|---|---|
| `get_fetch_run_time_range(last_run, first_fetch, look_back, date_format)` | Calculates `(start_time, end_time)` accounting for lookback window |
| `filter_incidents_by_duplicates_and_limit(incidents_res, last_run, fetch_limit, id_field)` | Removes already-seen IDs, enforces fetch limit |
| `update_last_run_object(last_run, incidents, fetch_limit, id_field, date_field)` | Updates LastRun with new timestamps and found IDs |

### LastRun Object Structure

```python
{
    'time': '2024-01-15T10:00:00Z',       # Timestamp of last fetched incident
    'limit': 50,                            # Current fetch limit
    'found_incident_ids': ['id1', 'id2']   # IDs seen in last window (for dedup)
}
```

**Lookback window:** The `look_back` parameter (minutes) slides the fetch window backwards to catch incidents that arrived late. `filter_incidents_by_duplicates_and_limit` prevents re-ingesting incidents already seen in the overlap.
````

- [ ] **Step 2: Add @polling_function decorator section after the existing Polling Pattern section (after line 408, at the end of the file)**

Append to the end of the file:

````markdown

---

## @polling_function Decorator Pattern

A simpler alternative to raw `ScheduledCommand` for most polling use cases:

```python
@polling_function(
    name='vendor-scan-file',
    interval=30,
    timeout=600,
)
def scan_file_command(args: dict, **kwargs) -> PollResult:
    scan_id = args.get('scan_id') or client.submit_scan(args['file_id'])
    result = client.get_scan_status(scan_id)

    if result['status'] == 'pending':
        return PollResult(
            response=None,
            continue_to_poll=True,
            args_for_next_run={'scan_id': scan_id, **args},
        )

    return PollResult(
        response=CommandResults(
            outputs_prefix='Vendor.Scan',
            outputs_key_field='id',
            outputs=result,
        ),
        continue_to_poll=False,
    )
```

**`PollResult` parameters:**

| Parameter | Type | Description |
|---|---|---|
| `response` | `CommandResults` or `None` | Final result when done; `None` while polling |
| `continue_to_poll` | `bool` | `True` to keep polling, `False` when complete |
| `args_for_next_run` | `dict` | Arguments passed to the next poll iteration |
| `partial_result` | `CommandResults` or `None` | Intermediate result shown while polling |

**When to use which:**
- **`@polling_function`** — most cases. Simpler, handles scheduling automatically.
- **Raw `ScheduledCommand`** — when you need variable intervals, conditional rescheduling, or custom timeout logic.

---

## Rate Limiting & Retry

Configure automatic retries with exponential backoff in the BaseClient constructor:

```python
class VendorClient(BaseClient):
    def __init__(self, base_url, api_key, verify=True, proxy=False):
        super().__init__(
            base_url=base_url,
            verify=verify,
            proxy=proxy,
            headers={'Authorization': f'Bearer {api_key}'},
            retries=3,
            backoff_factor=1.0,  # waits 1s, 2s, 4s between retries
        )
```

**Retry behavior:** When `retries` is set, `_http_request()` automatically retries on status codes 429, 500, 502, 503, 504. The `backoff_factor` controls wait time between attempts (factor × 2^retry).

To customize which status codes trigger retries:

```python
super().__init__(
    base_url=base_url,
    verify=verify,
    proxy=proxy,
    retries=3,
    backoff_factor=2.0,
    status_list_to_retry=[429, 500, 502, 503],
)
```

---

## Credential Vault Pattern

Most integrations are **consumers** — they accept vault-managed credentials via type 9 parameters. Only credential-management integrations are **providers** (implement `fetch-credentials`).

### Consumer Pattern (Common)

```python
# In main() — type 9 params arrive as {'identifier': 'user', 'password': 'pass'}
credentials = params.get('credentials', {})
username = credentials.get('identifier', '')
password = credentials.get('password', '')

client = VendorClient(
    base_url=params.get('server_url'),
    username=username,
    password=password,
    verify=not params.get('insecure', False),
    proxy=params.get('proxy', False)
)
```

### Provider Pattern (Rare — Vault Integrations Only)

```python
def fetch_credentials_command(client):
    """Return credentials for other integrations to consume."""
    creds = client.get_credentials()
    demisto.credentials([
        {'user': c['username'], 'password': c['password'], 'name': c['name']}
        for c in creds
    ])
```

---

## Proxy & SSL Handling

Standard proxy and SSL setup in `main()`:

```python
def main():
    params = demisto.params()
    verify = not params.get('insecure', False)
    proxy = params.get('proxy', False)

    client = VendorClient(
        base_url=params.get('server_url', '').rstrip('/'),
        api_key=params.get('api_key'),
        verify=verify,
        proxy=proxy,
    )
```

When you need the explicit proxy URL (rare — usually for non-BaseClient HTTP calls):

```python
from CommonServerPython import handle_proxy

proxies = handle_proxy(proxy_param_name='proxy', checkbox_default_value=False)
# Returns {'http': 'http://proxy:8080', 'https': 'http://proxy:8080'} or {}
```

`handle_proxy()` also unsets `REQUESTS_CA_BUNDLE` and `CURL_CA_BUNDLE` when SSL verification is disabled.

---

## Error Handling — Detailed Patterns

### test_module with Status Code Differentiation

```python
def test_module(client):
    try:
        client.list_items(limit=1)
        return 'ok'
    except DemistoException as e:
        if e.res is not None:
            if e.res.status_code == 401:
                return 'Authorization failed — check API key'
            if e.res.status_code == 403:
                return 'Forbidden — check API key permissions'
            if e.res.status_code == 429:
                return 'Rate limited — reduce fetch frequency or try again later'
        if 'Connection' in str(e):
            return f'Connection failed — verify server URL: {e}'
        raise
```

### Command Error Handling

```python
def main():
    try:
        # ... client setup and command routing ...
        pass
    except DemistoException as e:
        if e.res is not None and e.res.status_code == 404:
            return_error(f'Resource not found: {str(e)}')
        else:
            return_error(f'API error in {demisto.command()}: {str(e)}')
    except ValueError as e:
        return_error(f'Invalid input: {str(e)}')
    except Exception as e:
        return_error(f'Failed to execute {demisto.command()}: {str(e)}')
```
````

- [ ] **Step 3: Commit**

```bash
git add skills/xsiam-integrations/references/integration-patterns.md
git commit -m "feat(xsiam-integrations): add @polling_function, fetch dedup/lookback, rate limiting, credential vault, proxy, and error handling patterns"
```

---

### Task 4: Expand common-patterns.md — Demisto API Examples, Helpers, Indicator Enrichment

**Files:**
- Modify: `skills/xsiam-shared/references/common-patterns.md:156-379`

- [ ] **Step 1: Add examples to the Demisto API Reference section**

Find the State Management table (lines 194-201):

```markdown
### State Management (Fetch / Mirror)

| Method | Returns | Description |
|--------|---------|-------------|
| `demisto.getLastRun()` | `dict` | LastRun object (used by `fetch-incidents`) |
| `demisto.setLastRun(obj)` | `None` | Store LastRun for next fetch cycle |
| `demisto.getLastMirrorRun()` | `dict` | LastMirrorRun object |
| `demisto.setLastMirrorRun(obj)` | `None` | Store LastMirrorRun |

### Integration Context (Persistent Cache)
```

Replace with:

````markdown
### State Management (Fetch / Mirror)

| Method | Returns | Description |
|--------|---------|-------------|
| `demisto.getLastRun()` | `dict` | LastRun object (used by `fetch-incidents`) |
| `demisto.setLastRun(obj)` | `None` | Store LastRun for next fetch cycle |
| `demisto.getLastMirrorRun()` | `dict` | LastMirrorRun object |
| `demisto.setLastMirrorRun(obj)` | `None` | Store LastMirrorRun |

```python
# State management examples
last_run = demisto.getLastRun()  # Returns dict, empty {} on first run
demisto.setLastRun({'last_fetch': '2024-01-01T00:00:00Z', 'found_ids': ['id1', 'id2']})
```

### Integration Context (Persistent Cache)
````

- [ ] **Step 2: Add examples after the Integration Context table (after line 210)**

Find:

```markdown
| `demisto.setIntegrationContextVersioned(context, version, sync)` | `None` | Save with optimistic locking |

### Indicators
```

Replace with:

````markdown
| `demisto.setIntegrationContextVersioned(context, version, sync)` | `None` | Save with optimistic locking |

```python
# Integration context examples — persistent across commands within same instance
ctx = demisto.getIntegrationContext()  # Returns dict, empty {} initially
demisto.setIntegrationContext({'token': 'abc123', 'expires': 1700000000})

# Create incidents programmatically (outside fetch-incidents)
demisto.createIncidents([{'name': 'Alert 1', 'occurred': '2024-01-01T00:00:00Z', 'rawJSON': '{}'}])
```

### Indicators
````

- [ ] **Step 3: Add CommonServerPython Helpers section after the Argument Parsing Utilities section (after line 260)**

Find:

```markdown
if since:
    since_str = since.strftime('%Y-%m-%dT%H:%M:%SZ')
```

(this is the last line of the Argument Parsing Utilities section)

After it, and before the line `---` that precedes `## Indicator Enrichment Pattern`, insert:

````markdown

### Additional CommonServerPython Helpers

**`assign_params()`** — build request parameter dicts, automatically stripping `None` values:

```python
params = assign_params(page=page, page_size=page_size, query=query)
# Only includes keys with non-None values — clean API call params
```

**Time conversion utilities:**

```python
date_to_timestamp('2024-01-15T10:30:00Z')  # → epoch milliseconds
timestamp_to_datestring(1705312200000)       # → '2024-01-15T10:30:00Z'
```

**`CommandRunner`** — batch execution of multiple sub-commands with error isolation:

```python
runner = CommandRunner()
runner.add_command(ip_command, args={'ip': '1.2.3.4'}, name='ip')
runner.add_command(domain_command, args={'domain': 'example.com'}, name='domain')
results = runner.execute()
return_results(results)
```

**`IndicatorsTimeline`** — add timeline entries to indicator enrichment results:

```python
timeline = IndicatorsTimeline(
    indicators=['1.2.3.4'],
    message='First seen by VendorName',
    category='Integration Update',
)
# Pass to CommandResults via indicators_timeline=timeline
```
````

- [ ] **Step 4: Expand the Indicator Enrichment Pattern section — add Domain, File, URL examples**

Find the end of the indicator enrichment section, at the list of `Common.*` types (lines 330-337):

```markdown
**Available `Common.*` indicator types:**
- `Common.IP(ip, dbot_score, asn, organization, geo_country, ...)`
- `Common.Domain(domain, dbot_score, dns_records, ...)`
- `Common.URL(url, dbot_score, ...)`
- `Common.File(md5, sha1, sha256, dbot_score, name, size, ...)`
- `Common.CVE(id, cvss, description, ...)`
- `Common.Account(id, dbot_score, ...)`
- `Common.Email(address, dbot_score, ...)`
```

Replace with:

````markdown
**Available `Common.*` indicator types** with key fields:

```python
# IP enrichment (shown in full example above)
Common.IP(ip, dbot_score, asn, organization, geo_country)

# Domain enrichment
dbot_score = Common.DBotScore('example.com', DBotScoreType.DOMAIN, 'VendorName', Common.DBotScore.GOOD)
domain = Common.Domain(
    domain='example.com',
    dbot_score=dbot_score,
    registrar_name='RegistrarCo',
    creation_date='2020-01-01',
    expiration_date='2025-01-01',
    name_servers='ns1.example.com,ns2.example.com',
)

# File enrichment
dbot_score = Common.DBotScore('abc123hash', DBotScoreType.FILE, 'VendorName', Common.DBotScore.BAD)
file_indicator = Common.File(
    md5='abc123',
    sha1='def456',
    sha256='ghi789',
    dbot_score=dbot_score,
    size=1024,
    file_type='PE',
    name='malware.exe',
)

# URL enrichment
dbot_score = Common.DBotScore('https://evil.com', DBotScoreType.URL, 'VendorName', Common.DBotScore.SUSPICIOUS)
url = Common.URL(
    url='https://evil.com',
    dbot_score=dbot_score,
    detection_engines=10,
    positive_detections=3,
)

# CVE enrichment
Common.CVE(id='CVE-2024-1234', cvss='9.8', published='2024-01-15', modified='2024-02-01',
           description='Remote code execution vulnerability')

# Also available: Common.Account, Common.Email
```
````

- [ ] **Step 5: Add searchIndicators pagination note after the Entity Relationship Pattern section**

Find the end of the Entity Relationship section (lines 370-378):

```markdown
**Common relationship names (`EntityRelationship.Relationships.*`):**
- `RESOLVES_TO`
- `INDICATOR_OF`
- `RELATED_TO`
- `HOSTED_ON`
- `COMMUNICATES_WITH`
- `USES`
- `ATTRIBUTED_TO`
```

After this block, before the `---` and `## Best Practices`, insert:

````markdown

### searchIndicators Pagination

For large indicator queries, use `searchAfter` to paginate:

```python
query = 'type:IP'
search_after = None
all_indicators = []

while True:
    result = demisto.searchIndicators(query=query, size=100, searchAfter=search_after)
    indicators = result.get('iocs', [])
    if not indicators:
        break
    all_indicators.extend(indicators)
    search_after = result.get('searchAfter')
```
````

- [ ] **Step 6: Commit**

```bash
git add skills/xsiam-shared/references/common-patterns.md
git commit -m "feat(xsiam-shared): add demisto API examples, CommonServerPython helpers, expanded indicator enrichment, and searchIndicators pagination"
```

---

### Task 5: Final Review & Version Bump

**Files:**
- Modify: `skills/xsiam-integrations/SKILL.md:11` (version)
- Modify: `.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Bump the skill version in SKILL.md frontmatter**

Find:

```
version: 1.0.0
```

Replace with:

```
version: 2.0.0
```

- [ ] **Step 2: Read plugin.json and marketplace.json to check current versions**

Read `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` to determine the current plugin version before bumping.

- [ ] **Step 3: Bump plugin version in both files**

Update the `version` field in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` from the current version to the next minor version (e.g., `0.4.0` → `0.5.0`).

- [ ] **Step 4: Read all four modified reference files end-to-end to verify no broken Markdown**

Read each file and verify:
- No unclosed code fences
- No broken table formatting
- No orphaned section headers
- Section flow is logical (no duplicated headings, no missing `---` separators)

- [ ] **Step 5: Commit version bump**

```bash
git add skills/xsiam-integrations/SKILL.md .claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "chore: bump plugin to v0.5.0, xsiam-integrations skill to v2.0.0"
```
