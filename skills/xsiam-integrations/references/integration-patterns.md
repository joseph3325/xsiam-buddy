# XSIAM Integration Python Patterns

Integration-specific Python patterns for connecting to external APIs in XSIAM/XSOAR.

---

## BaseClient Pattern

All integrations that make HTTP calls should extend `BaseClient`:

```python
from CommonServerPython import BaseClient, DemistoException


class VendorClient(BaseClient):
    """Client for interfacing with Vendor API."""

    def __init__(self, base_url: str, api_key: str, verify: bool = True, proxy: bool = False):
        super().__init__(base_url=base_url, verify=verify, proxy=proxy)
        self.api_key = api_key

    def _get_headers(self) -> dict:
        return {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json'
        }

    def get_item(self, item_id: str) -> dict:
        return self._http_request(
            method='GET',
            url_suffix=f'/api/v1/items/{item_id}',
            headers=self._get_headers()
        )

    def list_items(self, query: str = '', limit: int = 50) -> dict:
        return self._http_request(
            method='GET',
            url_suffix='/api/v1/items',
            headers=self._get_headers(),
            params={'q': query, 'limit': limit}
        )

    def create_item(self, payload: dict) -> dict:
        return self._http_request(
            method='POST',
            url_suffix='/api/v1/items',
            headers=self._get_headers(),
            json=payload
        )
```

**`_http_request()` common parameters:**

| Parameter | Description |
|-----------|-------------|
| `method` | HTTP verb: `GET`, `POST`, `PUT`, `DELETE`, `PATCH` |
| `url_suffix` | Path appended to `base_url` |
| `headers` | Request headers dict |
| `params` | Query string parameters |
| `json` | Request body as dict (sets Content-Type: application/json) |
| `data` | Raw request body string |
| `timeout` | Request timeout in seconds |
| `ok_codes` | Tuple of acceptable HTTP status codes (default: 2xx) |
| `resp_type` | `json` (default), `text`, `content`, `response` |

---

## Authentication Patterns

### API Key (Bearer Token)

```python
def _get_headers(self) -> dict:
    return {
        'Authorization': f'Bearer {self.api_key}',
        'Content-Type': 'application/json'
    }
```

### Basic Authentication

```python
from requests.auth import HTTPBasicAuth

class VendorClient(BaseClient):
    def __init__(self, base_url, username, password, verify=True, proxy=False):
        super().__init__(base_url=base_url, verify=verify, proxy=proxy)
        self.auth = HTTPBasicAuth(username, password)

    def get_data(self, query: str) -> dict:
        return self._http_request(
            method='GET',
            url_suffix='/api/v1/data',
            params={'q': query},
            auth=self.auth
        )
```

### OAuth2 / Token with Caching

For APIs that issue short-lived tokens, cache the token in integration context:

```python
import time


class VendorClient(BaseClient):
    def __init__(self, base_url, client_id, client_secret, verify=True, proxy=False):
        super().__init__(base_url=base_url, verify=verify, proxy=proxy)
        self.client_id = client_id
        self.client_secret = client_secret

    def _get_token(self) -> str:
        ctx = demisto.getIntegrationContext()
        token = ctx.get('token')
        expires = ctx.get('token_expires', 0)

        if token and expires > time.time() + 60:  # 60s safety buffer
            return token

        response = self._http_request(
            method='POST',
            url_suffix='/oauth/token',
            json={'client_id': self.client_id, 'client_secret': self.client_secret}
        )
        token = response['access_token']
        expires_in = response.get('expires_in', 3600)
        demisto.setIntegrationContext({
            'token': token,
            'token_expires': time.time() + expires_in
        })
        return token

    def _get_headers(self) -> dict:
        return {'Authorization': f'Bearer {self._get_token()}'}
```

---

## Pagination Handling

Never assume single-page results — always implement pagination:

```python
def get_all_items(self, query: str) -> list:
    """Fetch all items with automatic pagination."""
    all_items = []
    page = 1
    page_size = 100

    while True:
        response = self._http_request(
            method='GET',
            url_suffix='/api/v1/items',
            headers=self._get_headers(),
            params={'q': query, 'page': page, 'page_size': page_size}
        )

        items = response.get('items', [])
        all_items.extend(items)

        if len(items) < page_size or not response.get('has_more'):
            break

        page += 1

    return all_items


# Cursor-based pagination
def get_all_with_cursor(self) -> list:
    all_items = []
    cursor = None

    while True:
        params = {'limit': 100}
        if cursor:
            params['cursor'] = cursor

        response = self._http_request('GET', '/api/v1/items', params=params)
        all_items.extend(response.get('items', []))
        cursor = response.get('next_cursor')
        if not cursor:
            break

    return all_items
```

---

## Test Module Pattern

All integrations must implement `test-module` to validate connectivity and credentials:

```python
def test_module(client: VendorClient) -> str:
    """
    Test connectivity and authentication.

    Returns:
        'ok' if successful, error message string otherwise.
    """
    try:
        client.list_items(limit=1)
        return 'ok'
    except DemistoException as e:
        if 'Unauthorized' in str(e) or '401' in str(e):
            return 'Authentication failed. Check your API key.'
        if 'Connection' in str(e) or '502' in str(e):
            return 'Failed to reach server. Check the Server URL.'
        return f'Test failed: {str(e)}'
    except Exception as e:
        return f'Unexpected error: {str(e)}'


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
            result = test_module(client)
            return_results(result)
        elif command == 'vendor-get-item':
            return_results(get_item_command(client, demisto.args()))
        else:
            raise NotImplementedError(f'Command {command} is not implemented')

    except Exception as e:
        return_error(f'Failed to execute {demisto.command()}: {str(e)}')
```

---

## Fetch Incidents Pattern

For integrations that ingest alerts or events into XSIAM:

**YAML:** Set `isfetch: true` in the `script` section.

```python
def fetch_incidents(client: VendorClient) -> None:
    """Fetch new incidents and store them via demisto.incidents()."""
    last_run = demisto.getLastRun()
    last_fetch_time = last_run.get('last_fetch', 0)

    incidents = []
    newest_time = last_fetch_time

    try:
        response = client.get_events(since=last_fetch_time, limit=100)

        for item in response.get('events', []):
            occurred = item.get('created_at', '')
            incident = {
                'name': item.get('title', 'Unnamed Event'),
                'occurred': occurred,
                'rawJSON': json.dumps(item),
                'type': 'Vendor Event',
                'severity': map_severity(item.get('severity'))
            }
            incidents.append(incident)

            created_ts = int(parse_date_string(occurred).timestamp()) if occurred else 0
            if created_ts > newest_time:
                newest_time = created_ts

    except Exception as e:
        demisto.error(f'Error fetching incidents: {str(e)}')

    demisto.setLastRun({'last_fetch': newest_time})
    demisto.incidents(incidents)


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
    fetch_incidents(client)
```

---

## Integration Context Pattern

Use integration context for persistent per-instance state (tokens, cursors, configuration cache). Unlike global variables, this persists across command invocations.

```python
import time


def get_cached_token(client) -> str:
    """Return a cached access token, fetching a new one if expired."""
    ctx = demisto.getIntegrationContext()
    token = ctx.get('token')
    expires = ctx.get('token_expires', 0)

    if token and expires > time.time() + 60:  # 60s safety buffer
        return token

    response = client.authenticate()
    token = response['access_token']
    expires_in = response.get('expires_in', 3600)

    demisto.setIntegrationContext({
        'token': token,
        'token_expires': time.time() + expires_in
    })
    return token
```

For concurrent integrations (long-running fetch loops), use **versioned context** to prevent race conditions:

```python
def update_context_safe(new_data: dict) -> None:
    """Update integration context with optimistic locking."""
    ctx = demisto.getIntegrationContextVersioned(refresh=True)
    version = ctx.get('version', -1)
    data = ctx.get('context', {})
    data.update(new_data)
    demisto.setIntegrationContextVersioned(data, version=version, sync=True)
```

---

## Polling Pattern (ScheduledCommand)

For long-running operations: submit a job, then schedule polling until it completes.

**YAML:** Set `polling: true` on the polling command definition.

```python
from CommonServerPython import ScheduledCommand

POLLING_INTERVAL_SECS = 30
POLLING_TIMEOUT_SECS = 600


def submit_job_command(client, args):
    """Submit a job and schedule the first poll."""
    query = args.get('query')
    job_id = client.submit_job(query)

    scheduled = ScheduledCommand(
        command='vendor-get-job-results',
        next_run_in_seconds=POLLING_INTERVAL_SECS,
        args={'job_id': job_id},
        timeout_in_seconds=POLLING_TIMEOUT_SECS
    )
    return CommandResults(
        readable_output=f'Job `{job_id}` submitted. Polling for results...',
        scheduled_command=scheduled
    )


def get_job_results_command(client, args):
    """Poll for job results; reschedule if not complete."""
    job_id = args.get('job_id')
    result = client.get_job(job_id)

    if result.get('status') in ('pending', 'running'):
        scheduled = ScheduledCommand(
            command='vendor-get-job-results',
            next_run_in_seconds=POLLING_INTERVAL_SECS,
            args={'job_id': job_id},
            timeout_in_seconds=POLLING_TIMEOUT_SECS
        )
        return CommandResults(
            readable_output=f'Job `{job_id}` still running...',
            scheduled_command=scheduled
        )

    return CommandResults(
        readable_output=tableToMarkdown('Job Results', result.get('data', [])),
        outputs_prefix='Vendor.Job',
        outputs_key_field='id',
        outputs=result.get('data', [])
    )
```

**YAML command definition for the polling command:**

```yaml
commands:
  - name: vendor-get-job-results
    polling: true
    description: Poll for asynchronous job results
    arguments:
      - name: job_id
        required: true
        description: Job ID returned by the submit command
```
