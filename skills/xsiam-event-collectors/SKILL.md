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
version: 1.1.0
---

# XSIAM Event Collector Development

Generate importable unified YAML files for Cortex XSIAM event collector integrations. Event collectors ingest vendor events directly into the XSIAM data lake via `send_events_to_xsiam()`. Unlike regular fetch-incidents integrations (which create incidents in the queue), event collector data lands in the data lake and is queryable via XQL. The Python code MUST be embedded directly inside the YAML file — this is the only format XSIAM accepts for import.

## Before Starting

Read the reference files:
- `references/event-collector-spec.md` — Event collector YAML structure (field order, `supportedModules`, sectionorder), `send_events_to_xsiam()`, `_time` handling, fetch-events flow, `should_push_events` pattern, complete example
- `../xsiam-integrations/references/integration-yaml-spec.md` — Base integration YAML structure, configuration parameter types, sectionorder
- `../xsiam-integrations/references/integration-patterns.md` — Python patterns: BaseClient, auth, pagination, rate limiting, proxy/SSL
- `../xsiam-shared/references/common-patterns.md` — Shared patterns: outputs, demisto API, arg parsing

## What is an Event Collector?

Event collectors are **specialized integrations** that ingest vendor events into the XSIAM data lake. They authenticate to a vendor's events/logs API, fetch events on a schedule, and push them to XSIAM via `send_events_to_xsiam()`.

**When to use which skill:**
- Events -> data lake -> XQL queryable -> **event collector** (this skill)
- Alerts -> incident queue -> playbook-actionable -> **integration with isfetch** (xsiam-integrations skill)
- Standalone data processing -> **script** (xsiam-scripts skill)

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
| API key (type 4 encrypted) | `params.get('api_key')` directly |
| Credentials (type 9 auth) | `params.get('credentials', {}).get('password', '')` |
| Username + password | `HTTPBasicAuth` via `_http_request(auth=)` |
| OAuth2 client credentials | Token caching with `getIntegrationContext()` + expiry |
| Certificate-based | Mutual TLS via `_http_request()` cert params |

### 2. Generate the Unified YAML

Build a single `.yml` file following these ordered sub-steps. The YAML structure must match real XSIAM export format for successful import.

1. **Top-level metadata** — `commonfields` (`id` must end with `EventCollector`, `version: -1`), then `vcShouldKeepItemLegacyProdMachine: false`, then `name`, `display`, `category`, `description`
2. **`sectionorder`** — lowercase 'o', simple list format: `- Connect`, `- Collect`
3. **`configuration`** — Each parameter starts with `supportedModules: []`, then `section:`, then remaining fields. Auth params in Connect section; `first_fetch`, `max_events` in Collect section; `insecure`/`proxy` in Connect with `advanced: true`. Always include `additionalinfo` tooltips.
4. **`script` mapping** — set flags: `type: python`, `subtype: python3`, `dockerimage` (pinned `3.12.x`), `isfetchevents: true`, `isfetch: false`, `isFetchSamples: false`, `runonce: false`
5. **Command definitions** — three required commands:
   - `test-module` (implicit platform command — NOT listed in `commands` array)
   - `fetch-events` (implicit platform command — NOT listed in `commands` array)
   - `<prefix>-get-events` with `should_push_events`, `limit`, and `start_time` arguments (listed in `commands` array). Each command and argument starts with `supportedModules: []`.
6. **Embed Python code** — insert into `script.script: |-` with `register_module_line()` calls as the first and last lines

### Configuration Parameter Field Order

Each configuration parameter follows this field order (matching real XSIAM exports):

```
supportedModules → section → advanced (if true) → display → displaypassword (if type 9) → name → type → required → defaultvalue → additionalinfo → options (if select) → hiddenusername (if type 9)
```

### Command and Argument Field Order

Commands: `supportedModules` -> `name` -> `description` -> `arguments`

Arguments: `supportedModules` -> `name` -> `description` -> `required` -> `defaultValue` -> `predefined` (if applicable) -> `isArray` (if applicable)

### 3. Python Code Conventions

Same BaseClient pattern as integrations, with these event-collector-specific differences:

- **`register_module_line()`** — include as first and last lines of the embedded Python code, matching the integration name (e.g., `register_module_line('VendorNameEventCollector', 'start', __line__())`)
- **Global constants** — define `VENDOR` and `PRODUCT` as module-level constants, reference them in `send_events_to_xsiam()` calls
- **`fetch-events`** command calls `send_events_to_xsiam(events, vendor, product)` — never `demisto.incidents()`
- **`<prefix>-get-events`** is a debug command: accepts `should_push_events` argument. When false, returns `CommandResults` with events as readable output only. When true, also calls `send_events_to_xsiam()`.
- Every event must have a `_time` field in ISO 8601 format — map from the source timestamp field during fetch
- `demisto.getLastRun()` / `demisto.setLastRun()` tracks last event timestamp per event type
- For multiple event types, call `send_events_to_xsiam()` separately for each vendor/product pair
- **Do not include** `from CommonServerPython import *`, `from CommonServerUserPython import *`, or `import demistomock as demisto` — the platform injects these automatically at runtime. Unified YAML must not contain them. Third-party imports like `import dateparser` are fine.

### 4. File Output

Generate a single file:
- `VendorEventCollector.yml` — the unified YAML ready for import into XSIAM

Optionally also generate:
- `VendorEventCollector_test.py` — pytest test file (separate, not for import)
- `README.md` — command reference documentation (separate)

### 5. Validation Checklist

Before delivering, verify:
- [ ] Integration name ends with `EventCollector` in `commonfields.id`, `name`, and `display`
- [ ] `vcShouldKeepItemLegacyProdMachine: false` present after `commonfields`
- [ ] `sectionorder` uses lowercase 'o' (not camelCase)
- [ ] `script.isfetchevents` is `true`; `script.isfetch` is `false`
- [ ] Every configuration parameter has `supportedModules: []` as its first field
- [ ] Every command definition has `supportedModules: []` as its first field
- [ ] Every argument definition has `supportedModules: []` as its first field
- [ ] Config params have `section:` field assigning them to Connect or Collect
- [ ] Config params have `additionalinfo:` tooltips
- [ ] `insecure` and `proxy` params have `advanced: true`
- [ ] Only `<prefix>-get-events` is listed in `commands` array — `test-module` and `fetch-events` are NOT listed (they are implicit platform commands)
- [ ] `<prefix>-get-events` has `should_push_events` argument with `predefined` values
- [ ] `fetch-events` calls `send_events_to_xsiam(events, vendor, product)` — not `demisto.incidents()`
- [ ] `<prefix>-get-events` respects `should_push_events` flag
- [ ] `VENDOR` and `PRODUCT` defined as module-level constants
- [ ] `vendor` and `product` strings are lowercase
- [ ] Every event has a `_time` field in ISO 8601 format
- [ ] `demisto.setLastRun()` tracks last event timestamp
- [ ] `register_module_line()` present as first and last lines of Python code
- [ ] Python code is embedded in `script.script: |-` (nested, not top-level)
- [ ] Python indentation is consistent within the YAML block
- [ ] **No** `CommonServerPython`, `CommonServerUserPython`, or `demistomock` imports — the platform injects these at runtime
- [ ] `main()` has `try/except` with `return_error()`
- [ ] `BaseClient` subclass used for all HTTP calls via `_http_request()`
- [ ] `test-module` command is implemented and routes correctly in `main()`
- [ ] `script.type` is `python` (not `python3`); `script.subtype` is `python3`
- [ ] Docker image is a pinned `3.12.x` version (not `:latest`)
- [ ] **Do not include** `fromversion`, `marketplaces`, `tests`, `timeout` — content-pack CI fields only
- [ ] No tab characters; consistent YAML indentation throughout

## Key Conventions

- Integration name: PascalCase ending with `EventCollector` (e.g., `VendorNameEventCollector`)
- Command prefix: lowercase vendor name (e.g., `vendorname-get-events`)
- Vendor/product strings: lowercase (e.g., `vendor='vendorname'`, `product='audit'`)
- Python functions and variables: `snake_case`; class names: `PascalCase`
- Docker image: pinned `3.12.x` (e.g., `demisto/python3:3.12.12.6947692`) — check your tenant for the latest available build
- Always include `proxy` and `insecure` configuration params (with `advanced: true`)
- Use `@logger` decorator on command functions for debug logging
- Handle pagination — never assume single-page API results
- Define `VENDOR` and `PRODUCT` as module-level constants for consistency
