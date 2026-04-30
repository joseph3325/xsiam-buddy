# xsiam-event-collectors Skill — Design Spec

**Date:** 2026-04-16
**Scope:** New skill for building XSIAM event collector integrations. Shares existing integration references for BaseClient/auth/pagination; adds one new reference file for event-collector-specific content.
**Approach:** Single new reference file (Approach A). Cross-references existing integration references.

---

## Source Material

Knowledge vault files under `wiki/`:
- `xsiam-developer-guide/event-collectors.md` — naming, YAML flags, required commands, _time field, sectionOrder
- `xsiam-common-server-python/xsiam-data-ingestion.md` — send_events_to_xsiam(), send_data_to_xsiam(), chunking, threading, retry
- `xsiam-developer-guide/integration-structure.md` — parameter types, sectionOrder, script flags
- `raw/processed/2026-04-13-Mimecast Event Collector v2.md` — real-world event collector example

---

## File Structure

```
skills/xsiam-event-collectors/
  SKILL.md                              # Skill definition (~150 lines)
  references/
    event-collector-spec.md             # YAML spec + Python patterns (~400 lines)
```

**References loaded by SKILL.md (4 total):**
1. `references/event-collector-spec.md` — event-collector-specific YAML, send_events_to_xsiam(), _time handling, fetch-events flow, complete example
2. `../xsiam-integrations/references/integration-yaml-spec.md` — base integration YAML structure, parameter types, sectionOrder
3. `../xsiam-integrations/references/integration-patterns.md` — BaseClient, auth, pagination, rate limiting, proxy/SSL
4. `../xsiam-shared/references/common-patterns.md` — shared patterns, demisto API, arg parsing

**Marketplace registration:**
- Add `./skills/xsiam-event-collectors` to `marketplace.json` skills array
- Also add `./skills/xsiam-integrations` (currently missing from the array)

---

## SKILL.md (~150 lines)

### Frontmatter

```yaml
---
name: xsiam-event-collectors
description: >
  This skill should be used when the user asks to "create an event collector",
  "build an event collector", "write an event collector", "ingest events into XSIAM",
  "fetch events", "send_events_to_xsiam", "isfetchevents", "XSIAM event ingestion",
  or needs to build a specialized integration that ingests vendor events into the
  XSIAM data lake. For standard API integrations that create incidents, use the
  xsiam-integrations skill instead. For standalone data-processing scripts, use
  xsiam-scripts.
version: 1.0.0
---
```

### Section Structure

**Intro paragraph:**
Event collectors are specialized integrations that ingest vendor events directly into the XSIAM data lake via `send_events_to_xsiam()`. Unlike regular fetch-incidents integrations (which create incidents in the queue), event collector data lands in the data lake and is queryable via XQL. The Python code is embedded in a unified `.yml` file — the only format XSIAM accepts for import.

**Before Starting:**
References the 4 files listed above with relative paths and one-line descriptions.

**What is an Event Collector?**
Explains the concept and provides a decision tree:
- Events → data lake → XQL queryable → **event collector** (this skill)
- Alerts → incident queue → playbook-actionable → **integration with isfetch** (xsiam-integrations skill)
- Standalone processing → **script** (xsiam-scripts skill)

Use an event collector when:
- Ingesting logs, audit trails, or telemetry into the XSIAM data lake
- Vendor has an events/logs API
- Data needs to be available for XQL hunting and correlation rules
- Events need `_time` normalization for XSIAM timeline

**Workflow (5 steps):**

### 1. Gather Requirements

Determine:
- Target vendor/API and documentation URL
- Event types to ingest (audit logs, security events, network telemetry, etc.)
- `vendor` and `product` strings for XSIAM (lowercase, used in dataset naming: `vendor_product_raw`)
- Timestamp field name in source events (must be mapped to `_time`)
- Whether multiple event types need separate streams (separate `send_events_to_xsiam()` calls per vendor/product pair)

Authentication method — same decision tree as xsiam-integrations:

| Auth scheme | Pattern |
|---|---|
| API key in header | Bearer token via `_get_headers()` |
| Username + password | `HTTPBasicAuth` via `_http_request(auth=)` |
| OAuth2 client credentials | Token caching with `getIntegrationContext()` + expiry |
| OAuth2 authorization code | Device/auth code flow (rare for server-to-server) |
| Certificate-based | Mutual TLS via `_http_request()` cert params |

### 2. Generate the Unified YAML

Build a single `.yml` file following these ordered sub-steps:

1. **Top-level metadata** — `commonfields` (`id` must end with `EventCollector`, `version: -1`), `name`, `display`, `category`, `description`
2. **`sectionOrder`** — define UI tabs: `Connect`, `Collect`
3. **`configuration`** — auth params in Connect section; `first_fetch`, `max_events` in Collect section; `insecure`/`proxy` in Connect with `advanced: true`
4. **`script` mapping** — set flags: `type: python`, `subtype: python3`, `dockerimage` (pinned `3.12.x`), `isfetchevents: true`, `isfetch: false`, `runonce: false`
5. **Command definitions** — three required commands:
   - `test-module` (no arguments, no outputs)
   - `fetch-events` (no arguments — called by platform on schedule)
   - `<prefix>-get-events` with `limit` and `start_time` arguments (manual debug command)
6. **Embed Python code** — insert into `script.script: |-` last

**Key structural difference from regular integrations:** `isfetchevents: true` instead of `isfetch: true`. Events go to the data lake via `send_events_to_xsiam()`, not the incident queue via `demisto.incidents()`.

### 3. Python Code Conventions

Same BaseClient pattern as integrations, with these differences:
- `fetch-events` command calls `send_events_to_xsiam(events, vendor, product)` — never `demisto.incidents()`
- `<prefix>-get-events` is a debug command: returns `CommandResults` with events as readable output (does NOT send to XSIAM)
- Every event must have a `_time` field in ISO 8601 format — map from the source timestamp field during fetch
- `demisto.getLastRun()` / `demisto.setLastRun()` tracks last event timestamp per event type
- For multiple event types, call `send_events_to_xsiam()` separately for each vendor/product pair

### 4. File Output

Generate a single file:
- `VendorEventCollector.yml` — the unified YAML ready for import into XSIAM

Optionally also generate:
- `VendorEventCollector_test.py` — pytest test file (separate, not for import)
- `README.md` — command reference documentation (separate)

### 5. Validation Checklist

Before delivering, verify:
- [ ] Integration name ends with `EventCollector` in `commonfields.id`, `name`, and `display`
- [ ] `script.isfetchevents` is `true`
- [ ] `script.isfetch` is `false`
- [ ] Three commands present: `test-module`, `fetch-events`, `<prefix>-get-events`
- [ ] `fetch-events` calls `send_events_to_xsiam(events, vendor, product)` — not `demisto.incidents()`
- [ ] `<prefix>-get-events` returns `CommandResults` — does NOT call `send_events_to_xsiam()`
- [ ] `vendor` and `product` strings are lowercase
- [ ] Every event has a `_time` field in ISO 8601 format
- [ ] `demisto.setLastRun()` tracks last event timestamp
- [ ] Python code is embedded in `script.script: |-` (nested, not top-level)
- [ ] Python indentation is consistent within the YAML block
- [ ] Standard imports present (`CommonServerPython`, `CommonServerUserPython`) — `demistomock` omitted unless user requested it
- [ ] `main()` has `try/except` with `return_error()`
- [ ] `BaseClient` subclass used for all HTTP calls via `_http_request()`
- [ ] `test-module` command is implemented and routes correctly in `main()`
- [ ] `script.type` is `python` (not `python3`); `script.subtype` is `python3`
- [ ] Docker image is a pinned `3.12.x` version (not `:latest`)
- [ ] `sectionOrder` present with Connect/Collect tabs
- [ ] Configuration params have `section` and `advanced` where appropriate
- [ ] **Do not include** `fromversion`, `marketplaces`, `tests`, `timeout` — content-pack CI fields only
- [ ] **Do not include** `register_module_line()` calls — platform-injected on export
- [ ] No tab characters; consistent YAML indentation throughout

**Key Conventions:**
- Integration name: PascalCase ending with `EventCollector` (e.g., `VendorNameEventCollector`)
- Command prefix: lowercase vendor name (e.g., `vendorname-get-events`)
- Vendor/product strings: lowercase (e.g., `vendor='vendorname'`, `product='audit'`)
- Python functions and variables: `snake_case`; class names: `PascalCase`
- Docker image: pinned `3.12.x`
- Always include `proxy` and `insecure` configuration params
- Use `@logger` decorator on command functions for debug logging

---

## event-collector-spec.md (~400 lines)

### Section 1: Event Collector vs Regular Integration (~20 lines)

Comparison table:

| Aspect | Regular Integration | Event Collector |
|---|---|---|
| YAML flag | `isfetch: true` | `isfetchevents: true` |
| Data destination | Incident queue | XSIAM data lake |
| Output function | `demisto.incidents()` | `send_events_to_xsiam()` |
| Naming | Any PascalCase | Must end with `EventCollector` |
| Required commands | `test-module`, `fetch-incidents` | `test-module`, `fetch-events`, `<prefix>-get-events` |
| Timestamp field | `occurred` on incident dict | `_time` on every event |
| Queryable via | Incident search | XQL queries on `vendor_product_raw` dataset |

### Section 2: YAML Differences (~60 lines)

Only the fields that differ from a standard integration. The base YAML structure (commonfields, name, display, category, description, sectionOrder, configuration) is identical — refer to `integration-yaml-spec.md` for those.

Script section differences:

```yaml
script:
  isfetchevents: true
  isfetch: false
  isFetchSamples: false
```

Required commands array:

```yaml
  commands:
    - name: vendorname-get-events
      description: Manual command to fetch events for debugging. Does not update LastRun or push events to XSIAM.
      arguments:
        - name: limit
          description: Maximum number of events to return
          required: false
          defaultValue: '100'
        - name: start_time
          description: Start time for event fetch (ISO 8601 or relative like '1 hour')
          required: false
```

Note: `test-module` and `fetch-events` are implicit platform commands — they don't need entries in the `commands` array. Only the `<prefix>-get-events` debug command is listed.

### Section 3: `send_events_to_xsiam()` Reference (~80 lines)

Complete function reference:

```python
send_events_to_xsiam(
    events,              # list[dict] or str (raw CEF/LEEF)
    vendor,              # str — lowercase vendor name
    product,             # str — lowercase product name
    data_format=None,    # None (auto-detect for dicts), 'cef', 'leef'
    chunk_size=XSIAM_EVENT_CHUNK_SIZE,  # Default 1 MiB
    multiple_threads=False,             # Enable multi-threaded upload
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `events` | `list[dict]` or `str` | required | Events to send. Dicts auto-serialized as JSON. Raw strings sent as-is. |
| `vendor` | `str` | required | Lowercase vendor name (e.g., `'okta'`) — used in dataset naming |
| `product` | `str` | required | Lowercase product name (e.g., `'auth'`) — used in dataset naming |
| `data_format` | `str` or `None` | `None` | Auto-detected for dicts. Use `'cef'` or `'leef'` for raw string events. |
| `chunk_size` | `int` | 1 MiB | Events are split into chunks of this size before upload |
| `multiple_threads` | `bool` | `False` | Enable multi-threaded upload for high-volume collectors |

**Chunking:** Events are automatically split into chunks (default 1 MiB, hard limit 9 MB per API call). For most collectors the default is fine.

**Rate limiting:** The underlying API call retries on 429 automatically with exponential backoff.

**Multi-threading:** Only needed for very high-volume collectors. When enabled, call `demisto.updateModuleHealth('')` manually after completion to update the integration health status.

**Multiple event types:** If a vendor has distinct event types (e.g., audit logs and security events), make separate `send_events_to_xsiam()` calls with different `product` values:

```python
if audit_events:
    send_events_to_xsiam(events=audit_events, vendor='vendorname', product='audit')
if security_events:
    send_events_to_xsiam(events=security_events, vendor='vendorname', product='security')
```

### Section 4: `_time` Field Handling (~40 lines)

Every event sent to XSIAM must have a `_time` field containing an ISO 8601 timestamp. This is the canonical timestamp used by XSIAM for timeline ordering and XQL queries.

If the source API uses a different field name, map it during fetch:

```python
def normalize_events(events, time_field='created_at'):
    """Ensure every event has a _time field."""
    for event in events:
        if '_time' not in event and time_field in event:
            event['_time'] = event[time_field]
    return events
```

If the source timestamp is epoch (seconds or milliseconds), convert:

```python
from datetime import datetime, timezone

def epoch_to_iso(epoch_value):
    """Convert epoch seconds or milliseconds to ISO 8601."""
    if epoch_value > 1e12:  # milliseconds
        epoch_value = epoch_value / 1000
    return datetime.fromtimestamp(epoch_value, tz=timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ')
```

### Section 5: fetch-events Command Pattern (~80 lines)

Production-grade fetch-events function:

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

**Multi-event-type variant** — when fetching distinct event types:

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

**LastRun object structure:**

```python
# Single event type
{'last_time': '2024-01-15T10:30:00Z'}

# Multiple event types
{
    'audit_last_time': '2024-01-15T10:30:00Z',
    'security_last_time': '2024-01-15T10:28:00Z',
}
```

### Section 6: `<prefix>-get-events` Debug Command Pattern (~40 lines)

This command is for manual debugging in the War Room. It fetches events and returns them as readable output — it does NOT send them to XSIAM.

```python
def get_events_command(client, args):
    """Manual command to fetch and display events (debugging only)."""
    limit = arg_to_number(args.get('limit')) or 100
    start_time = args.get('start_time')
    if not start_time:
        start_time = dateparser.parse('1 hour').strftime('%Y-%m-%dT%H:%M:%SZ')

    events = client.get_events(since=start_time, limit=limit)
    events = normalize_events(events, time_field='created_at')

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

### Section 7: Complete Event Collector Example (~80 lines)

Full working YAML for a minimal event collector:

```yaml
commonfields:
  id: ExampleEventCollector
  version: -1

name: Example Event Collector
display: Example Event Collector
category: Analytics & SIEM
description: Ingests events from Example vendor into the XSIAM data lake.

sectionOrder:
  - Connect
  - Collect

configuration:
  - display: Server URL
    name: server_url
    type: 0
    required: true
    defaultvalue: https://api.example.com
    section: Connect

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

  - display: First fetch time
    name: first_fetch
    type: 0
    required: false
    defaultvalue: 24 hours
    section: Collect
    additionalinfo: "How far back to fetch on first run (e.g., '24 hours', '3 days')"

  - display: Maximum events per fetch
    name: max_events
    type: 0
    required: false
    defaultvalue: '1000'
    section: Collect

script:
  script: |-
    from CommonServerPython import *
    from CommonServerUserPython import *
    import dateparser


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
        for event in events:
            if '_time' not in event and time_field in event:
                event['_time'] = event[time_field]
        return events


    def fetch_events(client, params, last_run):
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
                vendor='example',
                product='events',
            )
            last_run['last_time'] = max(e['_time'] for e in events)

        demisto.setLastRun(last_run)


    def get_events_command(client, args):
        limit = arg_to_number(args.get('limit')) or 100
        start_time = args.get('start_time')
        if not start_time:
            start_time = dateparser.parse('1 hour').strftime('%Y-%m-%dT%H:%M:%SZ')

        events = client.get_events(since=start_time, limit=limit)
        events = normalize_events(events, time_field='created_at')

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
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.12.12.6947692
  isfetchevents: true
  isfetch: false
  runonce: false

  commands:
    - name: example-get-events
      description: Manual command to fetch and display events for debugging. Does not push events to XSIAM.
      arguments:
        - name: limit
          description: Maximum number of events to return
          required: false
          defaultValue: '100'
        - name: start_time
          description: Start time for event fetch (ISO 8601 or relative like '1 hour')
          required: false
```

---

## What's NOT In Scope

- Feed integrations (`demisto.createIndicators()`) — future separate skill
- Long-running container integrations — future separate skill
- Changes to existing integration references — they already have what event collectors need
- Changes to common-patterns.md — `send_events_to_xsiam()` is event-collector-only

## Marketplace Registration

Add both missing skills to `.claude-plugin/marketplace.json`:

```json
"skills": [
    "./skills/xsiam-scripts",
    "./skills/xsiam-integrations",
    "./skills/xsiam-event-collectors",
    "./skills/xsiam-xql",
    "./skills/xsiam-correlations",
    "./skills/xsiam-splunk-to-xql",
    "./skills/xsiam-playbooks",
    "./skills/xsiam-docs",
    "./skills/xsiam-shared"
]
```

## CLAUDE.md Update

Add event collectors to the plugin architecture table and the scripts-vs-integrations comparison:

- Add `xsiam-event-collectors/` to the directory tree
- Add a row to the "XQL Skill Family" or create a new "Integration Skill Family" table showing the relationship between xsiam-integrations and xsiam-event-collectors
- Update "Scripts vs Integrations" to mention event collectors as a third category
