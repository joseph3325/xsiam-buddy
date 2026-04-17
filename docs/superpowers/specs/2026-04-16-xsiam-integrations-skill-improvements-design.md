# xsiam-integrations Skill Improvements — Design Spec

**Date:** 2026-04-16
**Scope:** Deepen the existing xsiam-integrations skill for standard API integrations. Event collectors, feeds, and long-running containers are out of scope (future separate skills).
**Approach:** Expand existing reference files in-place (Approach A). No new files, no tiered loading.

---

## Source Material

New content from the RagVault at `/Users/joeplunkett/Documents/RagVault/wiki/xsiam-developer-guide/` and `/Users/joeplunkett/Documents/RagVault/wiki/xsiam-common-server-python/`. Key source files:

- `integration-structure.md` — parameter types 0-16, sectionOrder, script flags
- `demisto-class-api.md` — complete demisto object API
- `common-server-functions.md` — CommandRunner, IndicatorsTimeline, arg utilities
- `command-results-and-output.md` — CommandResults, return_results, fileResult
- `http-and-networking.md` — retry, backoff_factor, proxy handling
- `fetch-incident-helpers.md` — dedup, lookback, LastRun management
- `fetch-missing-incidents.md` — late-arriving incident handling
- `scheduled-commands.md` — @polling_function decorator, PollResult
- `credentials.md` — credential vault, isFetchCredentials, type 9
- `context-standards.md` — mandatory entity fields, DBotScore scoring
- `sample-integration.md` — end-to-end integration tutorial
- `integration-yaml-example-xql-http-collector.md` — real YAML example

---

## File-by-File Changes

### 1. SKILL.md (~127 → ~180 lines)

**Section 1.1 — Expanded requirements gathering:**

Add structured questions with an auth method decision tree:

| Auth detected | Pattern to use |
|---|---|
| API key in header | Bearer token `_get_headers()` |
| Username + password | `HTTPBasicAuth` via `_http_request(auth=)` |
| OAuth2 client credentials | Token caching with `getIntegrationContext()` + expiry |
| OAuth2 authorization code | Device/auth code flow (rare for server-to-server) |
| Certificate-based | Mutual TLS via `_http_request()` cert params |

Add conditional requirement questions:
- **Fetch incidents?** → Gather: alert endpoint, time field name, severity mapping, dedup strategy (ID-based vs. time-based), lookback window needs
- **Polling commands?** → Which commands are async? Use `@polling_function` by default
- **Indicator enrichment?** → Which entity types (IP, Domain, URL, File, CVE)? What score mapping?
- **Credential vault?** → Does target system rotate credentials? → `isFetchCredentials` pattern

**Section 1.2 — Refined YAML generation sub-steps:**

Break into ordered sub-steps:
1. Build `configuration` section with `sectionOrder` (Connect/Collect/Optimize tabs), assign each param a `section` and optionally `advanced: true`
2. Build `script` mapping with flags: `isfetch`, `isFetchSamples` (when fetch-incidents applies)
3. Build command definitions with full argument specs (`supportedModules`, `defaultValue`, `isArray`, `predefined`) and output specs (`contextPath`, `type`)
4. Embed Python code into `script.script: |-` last

**Section 1.5 — Validation checklist additions:**

Add these items to the existing 24-point list:
- `sectionOrder` present when integration has 4+ configuration parameters
- Fetch incidents: dedup IDs tracked in `lastRun`, `isFetchSamples: true` set in script section
- Polling: `@polling_function` decorator or `ScheduledCommand` used — never `time.sleep()` in regular commands
- Credentials: sensitive params use type 4 (encrypted) or type 9 (credential vault)
- Proxy + insecure params have `section: Connect` and `advanced: true`

---

### 2. integration-yaml-spec.md (~399 → ~550 lines)

**New section: Complete parameter types (replaces current 7-type table):**

| Type | Name | Use Case |
|---|---|---|
| 0 | Short text | Server URL, prefixes |
| 1 | Long text | Certificates, multi-line input |
| 4 | Encrypted | API keys, tokens, passwords |
| 8 | Boolean | Toggles (insecure, proxy) |
| 9 | Credentials | Username + password pair (vault-compatible) |
| 12 | Single select | Dropdown from `options` list |
| 13 | Multi-line text (alt) | Similar to 1, alternate rendering |
| 14 | Multi-select | Multiple values from `options` list |
| 15 | Single select (alt) | Predefined dropdown (alternate) |
| 16 | Incident type | Incident type selector dropdown |

Each type includes a short YAML example.

**New section: `sectionOrder` and tabbed UI:**

```yaml
sectionOrder:
  - Connect
  - Collect
  - Optimize
```

Show how parameters reference sections:
```yaml
configuration:
  - display: Server URL
    name: url
    type: 0
    required: true
    section: Connect
  - display: Fetch incidents
    name: isFetch
    type: 8
    section: Collect
  - display: Use system proxy
    name: proxy
    type: 8
    section: Connect
    advanced: true
```

**New section: Fetch-incidents YAML configuration:**

Show the `script` mapping fields:
```yaml
script:
  isfetch: true
  isFetchSamples: true
```

Show the required configuration parameters for fetch:
- Incident type (type 16, section: Collect)
- Maximum incidents per fetch (type 0, defaultvalue: "50", section: Collect)
- First fetch time (type 0, defaultvalue: "3 days", section: Collect)
- Fetch interval (handled by platform, not a param)

**New section: Credential vault configuration:**

Show `isFetchCredentials: true` in script section and type 9 parameter:
```yaml
configuration:
  - display: Credentials
    name: credentials
    type: 9
    required: true
    section: Connect
script:
  isFetchCredentials: true
```

**Updated complete example:**

Expand the ExampleAPI integration to include:
- `sectionOrder` with Connect/Collect tabs
- `section` and `advanced` on each parameter
- Fetch-incidents configuration
- A credential parameter (type 9)

This replaces the current example rather than adding a second one.

---

### 3. integration-patterns.md (~408 → ~620 lines)

**New section: `@polling_function` decorator pattern:**

```python
from CommonServerPython import *

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

Note: Use `@polling_function` for most cases. Use raw `ScheduledCommand` only when you need custom scheduling logic (variable intervals, conditional rescheduling).

**Expanded section: Fetch incidents with dedup & lookback:**

Replace the current basic fetch pattern with a production-grade version:

```python
def fetch_incidents(client, params, last_run):
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
        date_field='created_at',
    )
    
    formatted = []
    for alert in incidents:
        formatted.append({
            'name': alert['title'],
            'occurred': alert['created_at'],
            'rawJSON': json.dumps(alert),
            'type': params.get('incident_type', 'Vendor Alert'),
            'severity': map_severity(alert.get('severity', 'low')),
        })
    
    demisto.incidents(formatted)
    demisto.setLastRun(update_last_run_object(
        last_run=new_last_run,
        incidents=incidents,
        fetch_limit=max_fetch,
        id_field='id',
        date_field='created_at',
    ))
```

Include `map_severity` helper and `lastRun` object structure documentation.

**New section: Rate limiting & retry:**

```python
class VendorClient(BaseClient):
    def __init__(self, base_url, api_key, verify, proxy):
        super().__init__(
            base_url=base_url,
            verify=verify,
            proxy=proxy,
            headers={'Authorization': f'Bearer {api_key}'},
            retries=3,
            backoff_factor=1.0,  # waits 1s, 2s, 4s
        )
```

Note: `_http_request()` retries on 429, 500, 502, 503, 504 by default when `retries` is set.

**New section: Credential vault pattern:**

Two roles — most integrations are **consumers** (use type 9 to accept vault credentials). Only credential-management integrations are **providers** (implement `fetch-credentials` command).

Consumer pattern (common):
```python
# In main() — type 9 params arrive as {'identifier': 'user', 'password': 'pass'}
credentials = params.get('credentials', {})
username = credentials.get('identifier', '')
password = credentials.get('password', '')
client = VendorClient(base_url=url, username=username, password=password, ...)
```

Provider pattern (rare — only for vault integrations):
```python
def fetch_credentials_command(client):
    creds = client.get_credentials()
    demisto.credentials([{'user': c['username'], 'password': c['password'], 'name': c['name']} for c in creds])
```

**New section: Proxy & SSL handling:**

```python
def main():
    params = demisto.params()
    verify = not params.get('insecure', False)
    proxy = params.get('proxy', False)
    
    client = VendorClient(
        base_url=params.get('url', '').rstrip('/'),
        api_key=params.get('apikey'),
        verify=verify,
        proxy=proxy,
    )
```

Show `handle_proxy()` usage for cases needing explicit proxy URL.

**Expanded section: Error handling:**

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
                return 'Rate limited — reduce fetch frequency'
        if 'Connection' in str(e):
            return f'Connection failed — verify server URL: {e}'
        raise
```

---

### 4. common-patterns.md (~457 → ~550 lines)

**Demisto API examples** — Add one-liner examples for:

```python
# State management
last_run = demisto.getLastRun()  # Returns dict, empty {} on first run
demisto.setLastRun({'last_fetch': '2024-01-01T00:00:00Z', 'found_ids': ['id1', 'id2']})

# Integration context (persistent across commands within same instance)
ctx = demisto.getIntegrationContext()  # Returns dict
demisto.setIntegrationContext({'token': 'abc', 'expires': 1700000000})

# Create incidents programmatically
demisto.createIncidents([{'name': 'Alert 1', 'occurred': '2024-01-01T00:00:00Z', 'rawJSON': '{}'}])
```

**New subsection: CommonServerPython helpers:**

- `CommandRunner` — batch execution of multiple sub-commands with error isolation:
```python
runner = CommandRunner()
runner.add_command(ip_command, args={'ip': '1.2.3.4'}, name='ip')
runner.add_command(domain_command, args={'domain': 'example.com'}, name='domain')
results = runner.execute()
return_results(results)
```

- `IndicatorsTimeline` — add timeline entries to indicator enrichment:
```python
timeline = IndicatorsTimeline(
    indicators=['1.2.3.4'],
    message='First seen by VendorName',
    category='Integration Update',
)
```

- `assign_params()` — clean parameter building (strips None values):
```python
params = assign_params(page=page, page_size=page_size, query=query)
# Only includes keys with non-None values
```

- Time conversion utilities:
```python
date_to_timestamp('2024-01-15T10:30:00Z')  # → epoch ms
timestamp_to_datestring(1705312200000)       # → '2024-01-15T10:30:00Z'
```

**Expanded indicator enrichment** — Add Common entity examples beyond IP:

```python
# Domain enrichment
dbot_score = Common.DBotScore('example.com', DBotScoreType.DOMAIN, 'VendorName', Common.DBotScore.GOOD)
domain = Common.Domain(domain='example.com', dbot_score=dbot_score, registrar_name='RegCo',
                       creation_date='2020-01-01', expiration_date='2025-01-01',
                       name_servers='ns1.example.com,ns2.example.com')

# File enrichment
dbot_score = Common.DBotScore('abc123hash', DBotScoreType.FILE, 'VendorName', Common.DBotScore.BAD)
file = Common.File(md5='abc123', sha1='def456', sha256='ghi789', dbot_score=dbot_score,
                   size=1024, file_type='PE', name='malware.exe')

# URL enrichment
dbot_score = Common.DBotScore('https://evil.com', DBotScoreType.URL, 'VendorName', Common.DBotScore.SUSPICIOUS)
url = Common.URL(url='https://evil.com', dbot_score=dbot_score, detection_engines=10,
                 positive_detections=3)
```

**searchIndicators pagination note:**

```python
# Paginate through large indicator sets
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

---

## What's NOT Changing

- No new files added — all changes are expansions to existing files
- No tiered loading mechanism
- Testing section in `common-patterns.md` stays as-is
- No event collector, feed, or long-running container patterns
- No changes to SKILL.md trigger descriptions or frontmatter scope
- `register_module_line()` exclusion rule stays (integrations don't use it)
- Docker image pinning guidance stays as-is

## Estimated Line Counts

| File | Current | After | Delta |
|---|---|---|---|
| SKILL.md | 127 | ~180 | +53 |
| integration-yaml-spec.md | 399 | ~550 | +151 |
| integration-patterns.md | 408 | ~620 | +212 |
| common-patterns.md | 457 | ~550 | +93 |
| **Total** | **1,391** | **~1,900** | **+509** |
