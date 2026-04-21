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
version: 2.0.0
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

### 2. Generate the Unified YAML

Build a single `.yml` file following these ordered sub-steps:

1. **Top-level metadata** — `commonfields` (`id`, `version: -1`), then `vcShouldKeepItemLegacyProdMachine: false`, then `name`, `display`, `category`, `description`
2. **`sectionorder`** — lowercase 'o', simple list: `- Connect`, `- Collect` (if fetching), `- Optimize` (optional)
3. **`configuration`** — each param starts with `supportedModules: []`, then `section:`, then remaining fields. Auth and connection params in Connect; fetch params in Collect; `insecure`/`proxy` in Connect with `advanced: true`. Include `additionalinfo:` tooltips.
4. **`script` mapping** — set flags: `type: python`, `subtype: python3`, `dockerimage` (pinned `3.12.x`), `isfetch` (true if fetching incidents), `isfetchevents: false`, `isFetchSamples` (true if fetching), `runonce: false`
5. **Command definitions** — each command and argument starts with `supportedModules: []`. Full argument specs (`defaultValue`, `isArray`, `predefined`) and output specs (`contextPath`, `type`)
6. **Embed Python code** — insert into `script.script: |-` last, with `register_module_line()` as the first and last lines

**Key structural difference from scripts:** For integrations, `script` is a **mapping** with nested fields. The Python code lives in `script.script`, not at the top level.

### 3. Python Code Conventions

The embedded Python follows XSOAR/XSIAM conventions:
- Standard imports: `CommonServerPython`, `CommonServerUserPython`. The `demistomock` import (`import demistomock as demisto`) may be present during development/testing but **must be removed** from the final unified YAML — the platform provides the `demisto` object natively. Before delivering the YAML, scan the embedded Python and strip any `demistomock` import line.
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
- [ ] Standard imports present (`CommonServerPython`, `CommonServerUserPython`) — `demistomock` import **removed** from final output
- [ ] `main()` has `try/except` with `return_error()`
- [ ] `BaseClient` subclass used for all HTTP calls via `_http_request()`
- [ ] `test-module` command is implemented and routes correctly in `main()`
- [ ] `script.type` is `python` (not `python3`); `script.subtype` is `python3`
- [ ] Docker image is a pinned `3.12.x` version (not `:latest`)
- [ ] `configuration` section includes server URL, credentials, `insecure`, and `proxy` params
- [ ] All command arguments have descriptions
- [ ] All outputs have `contextPath`, `description`, and `type`
- [ ] **Do not include** `fromversion`, `marketplaces`, `tests`, `timeout` — content-pack CI fields only
- [ ] **Always include** `register_module_line()` calls as first and last lines of embedded Python
- [ ] No tab characters; consistent YAML indentation throughout
- [ ] Arg parsing uses helpers (`argToList`, `argToBoolean`, etc.), not raw casting
- [ ] Fetch integrations: `isfetch: true` in script section; `fetch-incidents` routes to `demisto.incidents()`
- [ ] Polling commands: `ScheduledCommand` used in Python and `polling: true` set in YAML command definition
- [ ] Indicator commands: `DBotScore` + `Common.*` indicator object passed to `CommandResults`
- [ ] Token-caching integrations: integration context used, not global variables
- [ ] Sensitive config params use `type: 4` (encrypted) or `type: 9` (credential vault pair)
- [ ] `vcShouldKeepItemLegacyProdMachine: false` present after `commonfields`
- [ ] `sectionorder` (lowercase 'o') present when integration has 4+ configuration parameters
- [ ] Every configuration parameter has `supportedModules: []` as its first field
- [ ] Every command and argument has `supportedModules: []` as its first field
- [ ] Configuration params have `section: Connect`/`Collect`, `advanced: true` where appropriate, and `additionalinfo:` tooltips
- [ ] Fetch incidents: dedup IDs tracked in `lastRun`, `isFetchSamples: true` set in script section
- [ ] Polling: `@polling_function` decorator or `ScheduledCommand` used — never `time.sleep()` in regular commands
- [ ] Proxy + insecure params have `section: Connect` and `advanced: true`

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
