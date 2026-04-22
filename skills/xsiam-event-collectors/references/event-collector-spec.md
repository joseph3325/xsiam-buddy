# XSIAM Event Collector Specification

Event-collector-specific YAML structure and Python patterns. For base integration YAML (commonfields, configuration, parameter types, sectionorder), see `../../xsiam-integrations/references/integration-yaml-spec.md`. For BaseClient, auth, and pagination patterns, see `../../xsiam-integrations/references/integration-patterns.md`.

---

## Event Collector vs Regular Integration

| Aspect | Regular Integration | Event Collector |
|---|---|---|
| YAML flag | `isfetch: true` | `isfetchevents: true` |
| Data destination | Incident queue | XSIAM data lake |
| Output function | `demisto.incidents()` | `send_events_to_xsiam()` |
| Naming | Any PascalCase | Must end with `EventCollector` |
| Required commands | `test-module`, `fetch-incidents` | `test-module`, `fetch-events`, `<prefix>-get-events` |
| Timestamp field | `occurred` on incident dict | `_time` on every event |
| Queryable via | Incident search | XQL queries on `vendor_product_raw` dataset |

---

## YAML Structure

### Top-Level Field Order

Event collector unified YAML follows this exact field order (matching real XSIAM exports):

```yaml
commonfields:
  id: VendorNameEventCollector
  version: -1
vcShouldKeepItemLegacyProdMachine: false
name: Vendor Name Event Collector
display: Vendor Name Event Collector
category: Analytics & SIEM
description: Ingests events from Vendor Name into the XSIAM data lake.
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
- `sectionorder` uses lowercase 'o' (not camelCase `sectionOrder`)
- `sectionorder` is a simple list of section names — each param declares its section via a `section:` field

### Script Section Flags

```yaml
script:
  script: |-
    # Python code here
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.12.12.6947692
  isfetchevents: true
  isfetch: false
  isFetchSamples: false
  runonce: false
  commands:
    # ... command definitions ...
```

### Required Commands

Only the `<prefix>-get-events` debug command is listed in the `commands` array. `test-module` and `fetch-events` are implicit platform commands — they are NOT listed.

```yaml
  commands:
  - supportedModules: []
    name: vendorname-get-events
    description: Manual command to fetch events for debugging. Does not update LastRun or push events to XSIAM.
    arguments:
    - supportedModules: []
      name: should_push_events
      description: If true, pushes fetched events to XSIAM. Use with caution — may create duplicates.
      required: true
      defaultValue: 'false'
      predefined:
      - 'true'
      - 'false'
    - supportedModules: []
      name: limit
      description: Maximum number of events to return.
      required: false
      defaultValue: '100'
    - supportedModules: []
      name: start_time
      description: "Start time for event fetch (ISO 8601 or relative like '1 hour')."
      required: false
```

**`should_push_events` argument:** Production event collectors include this on the debug command. When `true`, events are pushed to XSIAM (useful for testing ingestion). When `false` (default), events are only displayed in the War Room. This follows the pattern used by official Palo Alto event collectors (Mimecast, CrowdStrike, etc.).

### Configuration Parameters

Every configuration parameter must include `supportedModules: []` as its first field and `section:` to assign it to a tab. Fields within each parameter follow this order: `supportedModules` → `section` → `advanced` (if true) → `display` → `displaypassword` (if type 9) → `name` → `type` → `required` → `defaultvalue` → `additionalinfo` → `options` (if select) → `hiddenusername` (if type 9).

Event collectors use `first_fetch` and `max_events` (not `first_fetch` and `max_fetch` like incident-fetch integrations):

#### Connect Section

```yaml
configuration:
- supportedModules: []
  section: Connect
  display: Server URL
  name: server_url
  type: 0
  required: true
  defaultvalue: https://api.example.com
  additionalinfo: Base URL of the vendor API.

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

#### Collect Section

```yaml
- supportedModules: []
  section: Collect
  display: First fetch time
  name: first_fetch
  type: 0
  required: false
  defaultvalue: 24 hours
  additionalinfo: "How far back to fetch on first run (e.g., '24 hours', '3 days')."

- supportedModules: []
  section: Collect
  display: Maximum events per fetch
  name: max_events
  type: 0
  required: false
  defaultvalue: '1000'
  additionalinfo: Maximum number of events to retrieve per fetch cycle.
```

#### Optional: Event Type Selection (Multi-Select)

For collectors that ingest multiple event types:

```yaml
- supportedModules: []
  section: Collect
  display: Event types to fetch
  name: event_types
  type: 14
  required: false
  options:
  - audit
  - security
  - network
  additionalinfo: Select which event types to ingest. Leave empty for all types.
```

#### Optional: Credentials (Type 9)

When the API uses username + password/token:

```yaml
- supportedModules: []
  section: Connect
  display: ''
  displaypassword: API Token
  name: credentials
  type: 9
  required: true
  hiddenusername: true
  additionalinfo: API token for authentication.
```

Access in code: `params.get('credentials', {}).get('password', '')`

---

## `send_events_to_xsiam()` Reference

The primary function for pushing events to the XSIAM data lake:

```python
send_events_to_xsiam(
    events,                              # list[dict] or str (raw CEF/LEEF)
    vendor,                              # str — lowercase vendor name
    product,                             # str — lowercase product name
    data_format=None,                    # None (auto-detect for dicts), 'cef', 'leef'
    url_key='url',                       # param key for XSIAM URL override
    num_of_attempts=3,                   # retry attempts on failure
    chunk_size=XSIAM_EVENT_CHUNK_SIZE,   # Default 1 MiB
    should_update_health_module=True,    # update integration health status
    add_proxy_to_request=False,          # use proxy for API call
    multiple_threads=False,              # enable multi-threaded upload
)
```

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `events` | `list[dict]` or `str` | required | Events to send. Dicts auto-serialized as JSON. Raw strings sent as-is (newline-delimited). |
| `vendor` | `str` | required | Lowercase vendor name (e.g., `'okta'`) — used in dataset naming |
| `product` | `str` | required | Lowercase product name (e.g., `'auth'`) — used in dataset naming |
| `data_format` | `str` or `None` | `None` | Auto-detected for dicts. Use `'cef'` or `'leef'` for raw string events. |
| `url_key` | `str` | `'url'` | Config param key for XSIAM URL override (rarely changed) |
| `num_of_attempts` | `int` | `3` | Number of retry attempts on 429 or server errors |
| `chunk_size` | `int` | 1 MiB | Events are split into chunks of this size before upload |
| `should_update_health_module` | `bool` | `True` | Update the integration health module after sending. Set `False` when sending multiple batches, then update manually at the end. |
| `add_proxy_to_request` | `bool` | `False` | Route the data ingestion API call through the configured proxy |
| `multiple_threads` | `bool` | `False` | Enable multi-threaded upload for high-volume collectors |

### Behavior Notes

**Chunking:** Events are automatically split into chunks (default 1 MiB, hard limit 9 MB per API call). For most collectors the default is fine. Constants: `XSIAM_EVENT_CHUNK_SIZE` (1 MiB), `XSIAM_EVENT_CHUNK_SIZE_LIMIT` (9 MB).

**Rate limiting:** The underlying API call retries on 429 automatically with exponential backoff up to `num_of_attempts`.

**Multi-threading:** Only needed for very high-volume collectors. When enabled, set `should_update_health_module=False` and call `demisto.updateModuleHealth('')` manually after all threads complete.

**Health module:** When `should_update_health_module=True` (default), the platform UI shows the number of events sent and the last successful fetch time. For batch sending, disable on intermediate calls and enable on the final call.

### Multiple Event Types

If a vendor has distinct event types (e.g., audit logs and security events), make separate `send_events_to_xsiam()` calls with different `product` values:

```python
if audit_events:
    send_events_to_xsiam(events=audit_events, vendor='vendorname', product='audit')
if security_events:
    send_events_to_xsiam(events=security_events, vendor='vendorname', product='security')
```

Each vendor/product pair creates a distinct dataset in XSIAM: `vendorname_audit_raw`, `vendorname_security_raw`.

---

## `_time` Field Handling

Every event sent to XSIAM must have a `_time` field containing an ISO 8601 timestamp. This is the canonical timestamp used by XSIAM for timeline ordering and XQL queries.

### Mapping from Source Field

If the source API uses a different field name, map it during fetch:

```python
def normalize_events(events, time_field='created_at'):
    """Ensure every event has a _time field."""
    for event in events:
        if '_time' not in event and time_field in event:
            event['_time'] = event[time_field]
    return events
```

### Converting Epoch Timestamps

If the source timestamp is epoch (seconds or milliseconds), convert:

```python
from datetime import datetime, timezone


def epoch_to_iso(epoch_value):
    """Convert epoch seconds or milliseconds to ISO 8601."""
    if epoch_value > 1e12:  # milliseconds
        epoch_value = epoch_value / 1000
    return datetime.fromtimestamp(epoch_value, tz=timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ')
```

Use in `normalize_events`:

```python
def normalize_events(events, time_field='created_at'):
    for event in events:
        if '_time' not in event and time_field in event:
            raw_time = event[time_field]
            if isinstance(raw_time, (int, float)):
                event['_time'] = epoch_to_iso(raw_time)
            else:
                event['_time'] = raw_time
    return events
```

---

## fetch-events Command Pattern

### Single Event Type

```python
def fetch_events(client, params, last_run):
    """Fetch events and send to XSIAM data lake."""
    first_fetch = params.get('first_fetch', '24 hours')
    max_events = int(params.get('max_events', 1000))

    last_time = last_run.get('last_time')
    if not last_time:
        last_time = dateparser.parse(first_fetch).strftime('%Y-%m-%dT%H:%M:%SZ')

    raw_events = client.get_events(since=last_time, limit=max_events)
    events = normalize_events(raw_events, time_field='created_at')

    if events:
        send_events_to_xsiam(
            events=events,
            vendor='vendorname',
            product='productname',
        )
        # Track the latest event timestamp for the next fetch cycle
        latest_time = max(e['_time'] for e in events)
        last_run['last_time'] = latest_time

    demisto.setLastRun(last_run)


# In main():
if command == 'fetch-events':
    fetch_events(client, params, demisto.getLastRun())
```

### Multiple Event Types

When fetching distinct event types with separate vendor/product pairs:

```python
def fetch_events(client, params, last_run):
    first_fetch = params.get('first_fetch', '24 hours')
    max_events = int(params.get('max_events', 1000))

    for event_type in ['audit', 'security']:
        last_time = last_run.get(f'{event_type}_last_time')
        if not last_time:
            last_time = dateparser.parse(first_fetch).strftime('%Y-%m-%dT%H:%M:%SZ')

        raw_events = client.get_events(event_type=event_type, since=last_time, limit=max_events)
        events = normalize_events(raw_events, time_field='timestamp')

        if events:
            send_events_to_xsiam(
                events=events,
                vendor='vendorname',
                product=event_type,
            )
            latest_time = max(e['_time'] for e in events)
            last_run[f'{event_type}_last_time'] = latest_time

    demisto.setLastRun(last_run)
```

### LastRun Object Structure

```python
# Single event type
{'last_time': '2024-01-15T10:30:00Z'}

# Multiple event types
{
    'audit_last_time': '2024-01-15T10:30:00Z',
    'security_last_time': '2024-01-15T10:28:00Z',
}
```

---

## `<prefix>-get-events` Debug Command Pattern

This command is for manual debugging in the War Room. It fetches events and returns them as readable output. The `should_push_events` argument optionally sends events to XSIAM (useful for testing the ingestion pipeline).

```python
def get_events_command(client, args):
    """Manual command to fetch and display events (debugging only)."""
    should_push_events = argToBoolean(args.get('should_push_events', 'false'))
    limit = arg_to_number(args.get('limit')) or 100
    start_time = args.get('start_time')
    if not start_time:
        start_time = dateparser.parse('1 hour').strftime('%Y-%m-%dT%H:%M:%SZ')

    events = client.get_events(since=start_time, limit=limit)
    events = normalize_events(events, time_field='created_at')

    if should_push_events and events:
        send_events_to_xsiam(
            events=events,
            vendor='vendorname',
            product='productname',
        )

    return CommandResults(
        readable_output=tableToMarkdown(
            f'Events (showing {len(events)} of {limit} max)',
            events,
            headers=['_time', 'event_type', 'description'],
        ),
        outputs_prefix='VendorName.Events',
        outputs_key_field='id',
        outputs=events,
    )


# In main():
elif command == 'vendorname-get-events':
    return_results(get_events_command(client, demisto.args()))
```

---

## Complete Event Collector Example

A production-ready event collector unified YAML matching real XSIAM export structure:

```yaml
commonfields:
  id: ExampleEventCollector
  version: -1
vcShouldKeepItemLegacyProdMachine: false
name: Example Event Collector
display: Example Event Collector
category: Analytics & SIEM
description: Ingests events from Example vendor into the XSIAM data lake.
sectionorder:
- Connect
- Collect
configuration:
- supportedModules: []
  section: Connect
  display: Server URL
  name: server_url
  type: 0
  required: true
  defaultvalue: https://api.example.com
  additionalinfo: Base URL of the Example API.

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

- supportedModules: []
  section: Collect
  display: First fetch time
  name: first_fetch
  type: 0
  required: false
  defaultvalue: 24 hours
  additionalinfo: "How far back to fetch on first run (e.g., '24 hours', '3 days')."

- supportedModules: []
  section: Collect
  display: Maximum events per fetch
  name: max_events
  type: 0
  required: false
  defaultvalue: '1000'
  additionalinfo: Maximum number of events to retrieve per fetch cycle.

script:
  script: |-
    register_module_line('ExampleEventCollector', 'start', __line__())


    import dateparser


    VENDOR = 'example'
    PRODUCT = 'events'


    class ExampleClient(BaseClient):
        def __init__(self, base_url, api_key, verify=True, proxy=False):
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

        def get_events(self, since, limit=1000):
            return self._http_request(
                method='GET',
                url_suffix='/api/v1/events',
                params={'since': since, 'limit': limit}
            ).get('events', [])

        def test_connection(self):
            self._http_request(method='GET', url_suffix='/api/v1/health')


    def normalize_events(events, time_field='created_at'):
        """Ensure every event has a _time field."""
        for event in events:
            if '_time' not in event and time_field in event:
                event['_time'] = event[time_field]
        return events


    def fetch_events(client, params, last_run):
        """Fetch events and send to XSIAM data lake."""
        first_fetch = params.get('first_fetch', '24 hours')
        max_events = int(params.get('max_events', 1000))

        last_time = last_run.get('last_time')
        if not last_time:
            last_time = dateparser.parse(first_fetch).strftime('%Y-%m-%dT%H:%M:%SZ')

        events = client.get_events(since=last_time, limit=max_events)
        events = normalize_events(events, time_field='created_at')

        if events:
            send_events_to_xsiam(
                events=events,
                vendor=VENDOR,
                product=PRODUCT,
            )
            last_run['last_time'] = max(e['_time'] for e in events)

        demisto.setLastRun(last_run)


    def get_events_command(client, args):
        """Manual command to fetch and display events (debugging only)."""
        should_push_events = argToBoolean(args.get('should_push_events', 'false'))
        limit = arg_to_number(args.get('limit')) or 100
        start_time = args.get('start_time')
        if not start_time:
            start_time = dateparser.parse('1 hour').strftime('%Y-%m-%dT%H:%M:%SZ')

        events = client.get_events(since=start_time, limit=limit)
        events = normalize_events(events, time_field='created_at')

        if should_push_events and events:
            send_events_to_xsiam(
                events=events,
                vendor=VENDOR,
                product=PRODUCT,
            )

        return CommandResults(
            readable_output=tableToMarkdown(
                f'Example Events ({len(events)} results)',
                events,
                headers=['_time', 'event_type', 'description'],
            ),
            outputs_prefix='Example.Events',
            outputs_key_field='id',
            outputs=events,
        )


    def test_module(client):
        """Test connectivity and authentication."""
        try:
            client.test_connection()
            return 'ok'
        except DemistoException as e:
            if e.res is not None:
                if e.res.status_code == 401:
                    return 'Authorization failed — check API key'
                if e.res.status_code == 403:
                    return 'Forbidden — check API key permissions'
            if 'Connection' in str(e):
                return f'Connection failed — verify server URL: {e}'
            raise


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
            elif command == 'fetch-events':
                fetch_events(client, params, demisto.getLastRun())
            elif command == 'example-get-events':
                return_results(get_events_command(client, demisto.args()))
            else:
                raise NotImplementedError(f'Command {command} is not implemented')

        except Exception as e:
            return_error(f'Failed to execute {demisto.command()}: {str(e)}')


    if __name__ in ('__main__', '__builtin__', 'builtins'):
        main()

    register_module_line('ExampleEventCollector', 'end', __line__())
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.12.12.6947692
  isfetchevents: true
  isfetch: false
  isFetchSamples: false
  runonce: false
  commands:
  - supportedModules: []
    name: example-get-events
    description: Manual command to fetch and display events for debugging. Does not push events to XSIAM unless should_push_events is set to true.
    arguments:
    - supportedModules: []
      name: should_push_events
      description: If true, pushes fetched events to XSIAM. Use with caution — may create duplicates.
      required: true
      defaultValue: 'false'
      predefined:
      - 'true'
      - 'false'
    - supportedModules: []
      name: limit
      description: Maximum number of events to return.
      required: false
      defaultValue: '100'
    - supportedModules: []
      name: start_time
      description: "Start time for event fetch (ISO 8601 or relative like '1 hour')."
      required: false
```
