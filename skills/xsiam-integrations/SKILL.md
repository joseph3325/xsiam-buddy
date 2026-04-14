---
name: xsiam-integrations
description: >
  This skill should be used when the user asks to "build an integration",
  "create an integration", "write an XSIAM integration", "XSOAR integration",
  "demisto integration", "generate integration YAML", "connect to an API",
  "write an integration for", or needs to develop a Python integration with
  authentication, multiple commands, and a corresponding YAML metadata file
  for Cortex XSIAM or XSOAR. For standalone data-processing scripts, use the
  xsiam-scripts skill instead.
version: 1.0.0
---

# XSIAM Integration Development

Generate importable unified YAML files for Cortex XSIAM/XSOAR integrations. The Python code MUST be embedded directly inside the YAML file — this is the only format XSIAM accepts for import.

## Before Starting

Read the reference files:
- `references/integration-yaml-spec.md` — Integration YAML structure, configuration types, and complete example
- `references/integration-patterns.md` — Python patterns: BaseClient, auth, fetch-incidents, polling, integration context
- `../xsiam-shared/references/common-patterns.md` — Shared patterns: outputs, demisto API, arg parsing, indicators

## What is an Integration?

Integrations **connect XSIAM to external products** — they authenticate to an API, expose multiple commands, and optionally fetch events or incidents. The Python code is embedded in a nested `script.script: |-` field.

Use an integration when:
- Connecting to an external product API (REST, GraphQL, SOAP)
- Multiple related commands are needed (get, list, create, delete)
- Authentication is required (API key, OAuth, basic auth)
- Fetching incidents or events into XSIAM
- Mirroring incident state to/from an external system

Use the **xsiam-scripts** skill instead for standalone data processing without external API calls.

## Workflow

### 1. Gather Requirements

Determine:
- Target product/vendor and API documentation URL
- Authentication method (API key, OAuth, basic auth, certificate)
- Commands needed — what operations to support
- For each command: required/optional arguments and expected outputs
- Whether it fetches incidents/events into XSIAM (`isfetch: true`)
- Whether any commands are long-running and need polling (`ScheduledCommand`)

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

### 3. Python Code Conventions

The embedded Python follows XSOAR/XSIAM conventions:
- Standard imports: `CommonServerPython`, `CommonServerUserPython` (do **not** include `demistomock` unless user requests it)
- `BaseClient` subclass with `_http_request()` for all API calls
- Command routing in `main()`: `if command == 'test-module': ... elif command == 'vendor-action': ...`
- Parse args with `argToList()`, `argToBoolean()`, `arg_to_number()`, `arg_to_datetime()` — never raw casting
- Return results with `CommandResults` and `return_results()`
- Always implement `test-module` to validate connectivity
- Use `demisto.getIntegrationContext()` / `demisto.setIntegrationContext()` for token caching
- Long-running operations: use `ScheduledCommand` and set `polling: true` on the YAML command
- Indicator enrichment: use `DBotScore` + `Common.IP` / `Common.Domain` / etc.

### 4. File Output

Generate a single file:
- `IntegrationName.yml` — the unified YAML ready for import into XSIAM

Optionally also generate:
- `IntegrationName_test.py` — pytest test file (separate, not for import)
- `README.md` — command reference documentation (separate)

### 5. Validation Checklist

Before delivering, verify:
- [ ] Python code is embedded in `script.script: |-` (nested, not top-level)
- [ ] Python indentation is consistent within the YAML block
- [ ] Standard imports present (`CommonServerPython`, `CommonServerUserPython`) — `demistomock` omitted unless user requested it
- [ ] `main()` has `try/except` with `return_error()`
- [ ] `BaseClient` subclass used for all HTTP calls via `_http_request()`
- [ ] `test-module` command is implemented and routes correctly in `main()`
- [ ] `script.type` is `python` (not `python3`); `script.subtype` is `python3`
- [ ] Docker image is a pinned `3.12.x` version (not `:latest`)
- [ ] `configuration` section includes server URL, credentials, `insecure`, and `proxy` params
- [ ] All command arguments have descriptions
- [ ] All outputs have `contextPath`, `description`, and `type`
- [ ] **Do not include** `fromversion`, `marketplaces`, `tests`, `timeout` — content-pack CI fields only
- [ ] **Do not include** `register_module_line()` calls — platform-injected on export
- [ ] No tab characters; consistent YAML indentation throughout
- [ ] Arg parsing uses helpers (`argToList`, `argToBoolean`, etc.), not raw casting
- [ ] Fetch integrations: `isfetch: true` in script section; `fetch-incidents` routes to `demisto.incidents()`
- [ ] Polling commands: `ScheduledCommand` used in Python and `polling: true` set in YAML command definition
- [ ] Indicator commands: `DBotScore` + `Common.*` indicator object passed to `CommandResults`
- [ ] Token-caching integrations: integration context used, not global variables
- [ ] Sensitive config params use `type: 4` (encrypted)

## Key Conventions

- Integration names: `PascalCase` for `commonfields.id` and `name`; `display` can include spaces
- Command names: `kebab-case` prefixed with vendor (e.g., `vendor-get-endpoints`, `xdr-list-alerts`)
- Python functions and variables: `snake_case`; class names: `PascalCase`
- Context prefix format: `Vendor.Object` (e.g., `PaloAltoNetworksXDR.Endpoint`)
- Docker image: pinned `3.12.x` (e.g., `demisto/python3:3.12.12.6947692`) — check your tenant for the latest available build
- Always include `proxy` and `insecure` configuration params
- Use `@logger` decorator on command functions for debug logging
- Handle pagination — never assume single-page API results
- Use `demisto.getIntegrationContext()` / `demisto.setIntegrationContext()` for state; never global variables
