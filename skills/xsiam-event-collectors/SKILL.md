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
