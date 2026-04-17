# xsiam-event-collectors Skill — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a new xsiam-event-collectors skill for building XSIAM event collector integrations, sharing existing integration references for BaseClient/auth/pagination.

**Architecture:** New skill directory with SKILL.md and one reference file (event-collector-spec.md). Cross-references existing integration references. Also registers both xsiam-integrations and xsiam-event-collectors in marketplace.json and updates CLAUDE.md.

**Tech Stack:** Markdown with embedded YAML and Python code blocks.

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `skills/xsiam-event-collectors/SKILL.md` | Create | Skill definition: frontmatter, workflow, validation checklist |
| `skills/xsiam-event-collectors/references/event-collector-spec.md` | Create | YAML spec deltas, send_events_to_xsiam(), _time handling, fetch-events pattern, complete example |
| `.claude-plugin/marketplace.json` | Modify | Add xsiam-integrations and xsiam-event-collectors to skills array |
| `CLAUDE.md` | Modify | Add event collectors to directory tree and comparison tables |

---

### Task 1: Create SKILL.md

**Files:**
- Create: `skills/xsiam-event-collectors/SKILL.md`

- [ ] **Step 1: Create the skill directory**

```bash
mkdir -p skills/xsiam-event-collectors/references
```

- [ ] **Step 2: Write SKILL.md**

Write the following content to `skills/xsiam-event-collectors/SKILL.md`:

```markdown
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

# XSIAM Event Collector Development

Generate importable unified YAML files for Cortex XSIAM event collector integrations. Event collectors ingest vendor events directly into the XSIAM data lake via `send_events_to_xsiam()`. Unlike regular fetch-incidents integrations (which create incidents in the queue), event collector data lands in the data lake and is queryable via XQL. The Python code MUST be embedded directly inside the YAML file — this is the only format XSIAM accepts for import.

## Before Starting

Read the reference files:
- `references/event-collector-spec.md` — Event collector YAML differences, `send_events_to_xsiam()`, `_time` handling, fetch-events flow, complete example
- `../xsiam-integrations/references/integration-yaml-spec.md` — Base integration YAML structure, configuration parameter types, sectionOrder
- `../xsiam-integrations/references/integration-patterns.md` — Python patterns: BaseClient, auth, pagination, rate limiting, proxy/SSL
- `../xsiam-shared/references/common-patterns.md` — Shared patterns: outputs, demisto API, arg parsing

## What is an Event Collector?

Event collectors are **specialized integrations** that ingest vendor events into the XSIAM data lake. They authenticate to a vendor's events/logs API, fetch events on a schedule, and push them to XSIAM via `send_events_to_xsiam()`.

**When to use which skill:**
- Events → data lake → XQL queryable → **event collector** (this skill)
- Alerts → incident queue → playbook-actionable → **integration with isfetch** (xsiam-integrations skill)
- Standalone data processing → **script** (xsiam-scripts skill)

Use an event collector when:
- Ingesting logs, audit trails, or telemetry into the XSIAM data lake
- Vendor has an events/logs API
- Data needs to be available for XQL hunting and correlation rules
- Events need `_time` normalization for XSIAM timeline

## Workflow

### 1. Gather Requirements

Determine:
- Target vendor/API and documentation URL
- Event types to ingest (audit logs, security events, network telemetry, etc.)
- `vendor` and `product` strings for XSIAM (lowercase, used in dataset naming: `vendor_product_raw`)
- Timestamp field name in source events (must be mapped to `_time`)
- Whether multiple event types need separate streams (separate `send_events_to_xsiam()` calls per vendor/product pair)

**Authentication method** — match the API's auth scheme to the correct pattern:

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
   - `test-module` (no arguments, no outputs — implicit platform command)
   - `fetch-events` (no arguments — implicit platform command, called on schedule)
   - `<prefix>-get-events` with `limit` and `start_time` arguments (manual debug command — listed in `commands` array)
6. **Embed Python code** — insert into `script.script: |-` last

**Key structural difference from regular integrations:** `isfetchevents: true` instead of `isfetch: true`. Events go to the data lake via `send_events_to_xsiam()`, not the incident queue via `demisto.incidents()`.

### 3. Python Code Conventions

Same BaseClient pattern as integrations, with these event-collector-specific differences:
- `fetch-events` command calls `send_events_to_xsiam(events, vendor, product)` — never `demisto.incidents()`
- `<prefix>-get-events` is a debug command: returns `CommandResults` with events as readable output (does NOT send to XSIAM)
- Every event must have a `_time` field in ISO 8601 format — map from the source timestamp field during fetch
- `demisto.getLastRun()` / `demisto.setLastRun()` tracks last event timestamp per event type
- For multiple event types, call `send_events_to_xsiam()` separately for each vendor/product pair
- Standard imports: `CommonServerPython`, `CommonServerUserPython`, `dateparser` (do **not** include `demistomock` unless user requests it)

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

## Key Conventions

- Integration name: PascalCase ending with `EventCollector` (e.g., `VendorNameEventCollector`)
- Command prefix: lowercase vendor name (e.g., `vendorname-get-events`)
- Vendor/product strings: lowercase (e.g., `vendor='vendorname'`, `product='audit'`)
- Python functions and variables: `snake_case`; class names: `PascalCase`
- Docker image: pinned `3.12.x` (e.g., `demisto/python3:3.12.12.6947692`) — check your tenant for the latest available build
- Always include `proxy` and `insecure` configuration params
- Use `@logger` decorator on command functions for debug logging
- Handle pagination — never assume single-page API results
```

- [ ] **Step 3: Commit**

```bash
git add skills/xsiam-event-collectors/SKILL.md
git commit -m "feat(xsiam-event-collectors): add SKILL.md for event collector skill"
```

---

### Task 2: Create event-collector-spec.md

**Files:**
- Create: `skills/xsiam-event-collectors/references/event-collector-spec.md`

- [ ] **Step 1: Write event-collector-spec.md**

Write the following content to `skills/xsiam-event-collectors/references/event-collector-spec.md`:

````markdown
# XSIAM Event Collector Specification

Event-collector-specific YAML structure and Python patterns. For base integration YAML (commonfields, configuration, parameter types, sectionOrder), see `../../xsiam-integrations/references/integration-yaml-spec.md`. For BaseClient, auth, and pagination patterns, see `../../xsiam-integrations/references/integration-patterns.md`.

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

## YAML Differences

Only the fields that differ from a standard integration. The base YAML structure (commonfields, name, display, category, description, sectionOrder, configuration) is identical — refer to `integration-yaml-spec.md` for those.

### Script Section Flags

```yaml
script:
  isfetchevents: true
  isfetch: false
  isFetchSamples: false
```

### Required Commands

Only the `<prefix>-get-events` debug command is listed in the `commands` array. `test-module` and `fetch-events` are implicit platform commands.

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

### Configuration Parameters (Collect Section)

Event collectors use `first_fetch` and `max_events` instead of `first_fetch` and `max_fetch`:

```yaml
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
```

---

## `send_events_to_xsiam()` Reference

The primary function for pushing events to the XSIAM data lake:

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

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `events` | `list[dict]` or `str` | required | Events to send. Dicts auto-serialized as JSON. Raw strings sent as-is. |
| `vendor` | `str` | required | Lowercase vendor name (e.g., `'okta'`) — used in dataset naming |
| `product` | `str` | required | Lowercase product name (e.g., `'auth'`) — used in dataset naming |
| `data_format` | `str` or `None` | `None` | Auto-detected for dicts. Use `'cef'` or `'leef'` for raw string events. |
| `chunk_size` | `int` | 1 MiB | Events are split into chunks of this size before upload |
| `multiple_threads` | `bool` | `False` | Enable multi-threaded upload for high-volume collectors |

### Behavior Notes

**Chunking:** Events are automatically split into chunks (default 1 MiB, hard limit 9 MB per API call). For most collectors the default is fine.

**Rate limiting:** The underlying API call retries on 429 automatically with exponential backoff.

**Multi-threading:** Only needed for very high-volume collectors. When enabled, call `demisto.updateModuleHealth('')` manually after completion to update the integration health status.

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

This command is for manual debugging in the War Room. It fetches events and returns them as readable output — it does **NOT** send them to XSIAM.

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

---

## Complete Event Collector Example

A production-ready event collector YAML ready for XSIAM import:

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
                vendor='example',
                product='events',
            )
            last_run['last_time'] = max(e['_time'] for e in events)

        demisto.setLastRun(last_run)


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
````

- [ ] **Step 2: Commit**

```bash
git add skills/xsiam-event-collectors/references/event-collector-spec.md
git commit -m "feat(xsiam-event-collectors): add event-collector-spec.md reference"
```

---

### Task 3: Update marketplace.json

**Files:**
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Add xsiam-integrations and xsiam-event-collectors to the skills array**

Find this text in `.claude-plugin/marketplace.json`:

```json
      "skills": [
        "./skills/xsiam-scripts",
        "./skills/xsiam-xql",
        "./skills/xsiam-correlations",
        "./skills/xsiam-splunk-to-xql",
        "./skills/xsiam-playbooks",
        "./skills/xsiam-docs",
        "./skills/xsiam-shared"
      ]
```

Replace with:

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

- [ ] **Step 2: Commit**

```bash
git add .claude-plugin/marketplace.json
git commit -m "chore: register xsiam-integrations and xsiam-event-collectors in marketplace.json"
```

---

### Task 4: Update CLAUDE.md

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add xsiam-event-collectors to the directory tree**

Find this text in `CLAUDE.md`:

```
  xsiam-integrations/  # Multi-command integrations with BaseClient
    SKILL.md
    references/integration-yaml-spec.md
    references/integration-patterns.md
```

Replace with:

```
  xsiam-integrations/  # Multi-command integrations with BaseClient
    SKILL.md
    references/integration-yaml-spec.md
    references/integration-patterns.md
  xsiam-event-collectors/ # Event collector integrations (data lake ingestion)
    SKILL.md
    references/event-collector-spec.md
```

- [ ] **Step 2: Update the "Scripts vs Integrations" section to include event collectors**

Find this text:

```markdown
## Scripts vs Integrations

The key structural difference between the two Python skill types:

| | xsiam-scripts | xsiam-integrations |
|---|---|---|
| Use when | Standalone data processing | Connecting to external APIs |
| Python location in YAML | Top-level `script: |-` | Nested `script.script: |-` |
| Base class | None (direct demisto calls) | `BaseClient` subclass |
| `demisto.alert()` normalization | Required in XSIAM context | N/A |
```

Replace with:

```markdown
## Scripts vs Integrations vs Event Collectors

The key structural differences between the three Python skill types:

| | xsiam-scripts | xsiam-integrations | xsiam-event-collectors |
|---|---|---|---|
| Use when | Standalone data processing | Connecting to external APIs | Ingesting events into data lake |
| Python location in YAML | Top-level `script: |-` | Nested `script.script: |-` | Nested `script.script: |-` |
| Base class | None (direct demisto calls) | `BaseClient` subclass | `BaseClient` subclass |
| YAML fetch flag | N/A | `isfetch: true` | `isfetchevents: true` |
| Data output | `return_results()` | `demisto.incidents()` (fetch) | `send_events_to_xsiam()` |
| Data destination | War room / context | Incident queue | XSIAM data lake (XQL queryable) |
| Naming convention | Any | PascalCase | Must end with `EventCollector` |
```

- [ ] **Step 3: Add event collectors to the "How Skills Work" section**

Find this text:

```
Skills never generate code outside of YAML/JSON output files. All XSIAM content is delivered as unified `.yml` files (Python embedded inside YAML via `script: |-` for scripts, `script.script: |-` for integrations), except correlation rules which are delivered as `.json` files matching the XSIAM export/import format.
```

Replace with:

```
Skills never generate code outside of YAML/JSON output files. All XSIAM content is delivered as unified `.yml` files (Python embedded inside YAML via `script: |-` for scripts, `script.script: |-` for integrations and event collectors), except correlation rules which are delivered as `.json` files matching the XSIAM export/import format.
```

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add xsiam-event-collectors to CLAUDE.md architecture docs"
```

---

### Task 5: Version Bump

**Files:**
- Modify: `.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Bump plugin version in plugin.json**

Find in `.claude-plugin/plugin.json`:

```json
"version": "0.6.0"
```

Replace with:

```json
"version": "0.7.0"
```

- [ ] **Step 2: Bump plugin version in marketplace.json**

Find in `.claude-plugin/marketplace.json`:

```json
"version": "0.6.0"
```

Replace with:

```json
"version": "0.7.0"
```

- [ ] **Step 3: Read all new files end-to-end to verify no broken Markdown**

Read each file and verify:
- No unclosed code fences
- No broken table formatting
- Section flow is logical
- YAML examples have correct indentation

Files to verify:
- `skills/xsiam-event-collectors/SKILL.md`
- `skills/xsiam-event-collectors/references/event-collector-spec.md`

- [ ] **Step 4: Commit version bump**

```bash
git add .claude-plugin/plugin.json .claude-plugin/marketplace.json
git commit -m "chore: bump plugin to v0.7.0 for xsiam-event-collectors skill"
```
