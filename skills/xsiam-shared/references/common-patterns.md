# XSIAM/XSOAR Common Python Patterns

Shared patterns used in both scripts and integrations.

---

## Standard Imports

Every XSIAM/XSOAR automation starts with:

```python
from CommonServerPython import *
from CommonServerUserPython import *
```

- **`CommonServerPython`**: Core utilities: `BaseClient`, `CommandResults`, `DemistoException`, `tableToMarkdown`, `return_results`, `return_error`, argument helpers, indicator types, etc. Also provides the `demisto` object at runtime.
- **`CommonServerUserPython`**: Integration-specific additions (usually empty or auto-generated)

> **Note:** Do not include `import demistomock as demisto` by default. The platform provides the `demisto` object at runtime. Only add the `demistomock` import if the user explicitly requests it (e.g., for local testing with `demisto-sdk` and `pytest`).

---

## Output Patterns

### CommandResults

The standard way to return structured output:

```python
return CommandResults(
    readable_output=tableToMarkdown('Results', items, headers=['ID', 'Name', 'Status']),
    outputs_prefix='Vendor.Object',     # Context path prefix
    outputs_key_field='id',             # Field used as the dedup key
    outputs=items                       # List or dict to store in context
)
```

Pass a list of `CommandResults` to `return_results()` to return multiple results at once:

```python
results = [
    CommandResults(outputs_prefix='Vendor.IP', outputs=ip_data, indicator=ip_obj),
    CommandResults(outputs_prefix='Vendor.Domain', outputs=domain_data)
]
return_results(results)
```

### tableToMarkdown

```python
human_readable = tableToMarkdown(
    'Table Title',
    data,                              # list of dicts, or single dict
    headers=['ID', 'Name', 'Status'],  # subset/order of columns to show
    headerTransform=pascalToSpace,     # optional: transform header names
    removeNull=True,                   # optional: omit columns where all values are None
    url_keys=['link', 'profile_url'],  # optional: auto-format as clickable markdown links
    date_fields=['created', 'updated'],# optional: auto-format date strings
    is_auto_json_transform=True        # optional: pretty-print nested dicts/lists
)
```

### File Output

```python
import json

def export_command(client, args):
    data = client.get_data(args.get('query'))
    file_content = json.dumps(data, indent=2)
    return fileResult('results.json', file_content)
```

---

## Error Handling

### Standard Pattern

```python
def main():
    try:
        # ... command logic ...
    except DemistoException as e:
        return_error(f'API error: {str(e)}')
    except ValueError as e:
        return_error(f'Invalid input: {str(e)}')
    except Exception as e:
        return_error(f'Failed to execute {demisto.command()}: {str(e)}')
```

### Input Validation

```python
def validate_inputs(args: dict) -> None:
    if not args.get('query'):
        raise ValueError('query argument is required')
    limit = arg_to_number(args.get('limit'))
    if limit is not None and (limit < 1 or limit > 1000):
        raise ValueError('limit must be between 1 and 1000')
```

---

## Async Operations (Polling)

**Never use `time.sleep()`** — the platform will kill long-running scripts. For operations that take time to complete (scans, exports, batch jobs), use the `ScheduledCommand` pattern. The script exits after each check and the platform re-invokes it at the specified interval.

```python
from CommonServerPython import ScheduledCommand

def check_scan_command(args):
    scan_id = args.get('scan_id')

    if not scan_id:
        # First call: start the operation
        scan_id = start_scan(args.get('target'))
        scheduled_command = ScheduledCommand(
            command='!my-check-scan',
            args={'scan_id': scan_id},
            next_run_in_seconds=30,
            timeout_in_seconds=600
        )
        return CommandResults(
            readable_output=f'Scan started: {scan_id}. Polling every 30s...',
            scheduled_command=scheduled_command
        )

    # Subsequent calls: check status
    result = get_scan_status(scan_id)

    if result['status'] != 'completed':
        scheduled_command = ScheduledCommand(
            command='!my-check-scan',
            args={'scan_id': scan_id},
            next_run_in_seconds=30,
            timeout_in_seconds=int(args.get('timeout', 600))
        )
        return CommandResults(
            readable_output=f'Scan {scan_id} still running...',
            scheduled_command=scheduled_command
        )

    # Done
    return CommandResults(
        readable_output=tableToMarkdown('Scan Results', [result]),
        outputs_prefix='Vendor.Scan',
        outputs_key_field='id',
        outputs=result
    )
```

**How it works:** When `CommandResults` includes a `scheduled_command`, the platform schedules a re-invocation with the specified args after `next_run_in_seconds`. The script is stateless between invocations — pass any needed state via the args dict. In scripts, wrap the call in `main()`: `return_results(check_scan_command(demisto.args()))`.

---

## Demisto API Reference

The `demisto` object is the interface between your code and the platform. It is available at runtime without any import.

### Inputs

| Method | Returns | Description |
|--------|---------|-------------|
| `demisto.args()` | `dict` | All command/script arguments |
| `demisto.params()` | `dict` | Integration configuration parameters |
| `demisto.command()` | `str` | Current command name being executed |
| `demisto.getArg(arg)` | `str` | Single argument value |
| `demisto.getParam(param)` | `str` | Single parameter value |

### Outputs & Logging

| Method | Returns | Description |
|--------|---------|-------------|
| `demisto.results(results)` | `None` | Output to war room (prefer `return_results()`) |
| `demisto.log(msg)` | `None` | Write to war room log |
| `demisto.debug(msg, *args)` | `None` | Debug log (only visible at debug level) |
| `demisto.info(msg, *args)` | `None` | Info log |
| `demisto.error(msg, *args)` | `None` | Error log |

### Incident & Investigation

| Method | Returns | Description |
|--------|---------|-------------|
| `demisto.alert()` | `dict` | **XSIAM**: Current alert object. Use this instead of `demisto.incident()` in XSIAM alert-context scripts |
| `demisto.incident()` | `dict` | Current incident with all fields (XSOAR; use `demisto.alert()` on XSIAM) |
| `demisto.incidents(incidents)` | `None` | Set incidents (used by `fetch-incidents`) |
| `demisto.createIncidents(incidents, lastRun, userID)` | `list` | Create incidents programmatically |
| `demisto.investigation()` | `dict` | Current investigation ID and metadata |
| `demisto.context()` | `dict` | Current incident context data |
| `demisto.setContext(contextPath, value)` | `None` | Set a context key directly |
| `demisto.addEntry(id, entry, username, email, footer)` | `None` | Add war room entry to an incident |

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

| Method | Returns | Description |
|--------|---------|-------------|
| `demisto.getIntegrationContext()` | `dict` | Persistent store for the integration instance |
| `demisto.setIntegrationContext(context)` | `None` | Save integration context |
| `demisto.getIntegrationContextVersioned(refresh)` | `dict` | Versioned context (includes `version` key for optimistic locking) |
| `demisto.setIntegrationContextVersioned(context, version, sync)` | `None` | Save with optimistic locking |

```python
# Integration context examples — persistent across commands within same instance
ctx = demisto.getIntegrationContext()  # Returns dict, empty {} initially
demisto.setIntegrationContext({'token': 'abc123', 'expires': 1700000000})

# Create incidents programmatically (outside fetch-incidents)
demisto.createIncidents([{'name': 'Alert 1', 'occurred': '2024-01-01T00:00:00Z', 'rawJSON': '{}'}])
```

### Indicators

| Method | Returns | Description |
|--------|---------|-------------|
| `demisto.searchIndicators(fromDate, query, size, page, toDate, value, searchAfter, populateFields)` | `dict` | Search TIM indicators; returns `iocs`, `total`, `searchAfter` |
| `demisto.createIndicators(indicators_batch, noUpdate)` | `None` | Bulk create/update indicators |
| `demisto.searchRelationships(args)` | `dict` | Search entity relationships |

### Utilities

| Method | Returns | Description |
|--------|---------|-------------|
| `demisto.executeCommand(command, args)` | `list` | Execute an XSOAR command or script |
| `demisto.executeCommandBatch(commands_list)` | `list` | Execute multiple commands |
| `demisto.internalHttpRequest(method, uri, body)` | `dict` | HTTP request to the XSOAR server itself |
| `demisto.getFilePath(id)` | `dict` | File path/name by entry ID; returns `id`, `path`, `name` |
| `demisto.uniqueFile()` | `str` | Generate a random UUID filename |
| `demisto.demistoUrls()` | `dict` | Platform URLs: `warRoom`, `investigation`, `server`, etc. |
| `demisto.demistoVersion()` | `dict` | Server version and build number |
| `demisto.integrationInstance()` | `str` | Current integration instance name |
| `demisto.updateModuleHealth(data, is_error)` | `None` | Update integration health/status in the UI |
| `demisto.mapObject(obj, mapper, mapper_type)` | `dict` | Map object using an XSOAR mapper |
| `demisto.dt(obj, trnsfrm)` | `str` | Evaluate a DT (Demisto Transform) expression |
| `demisto.findUser(username, email)` | `dict` | Look up a workspace user |
| `demisto.mirrorInvestigation(id, mirrorType, autoClose)` | `None` | Configure investigation mirroring |
| `demisto.getAllSupportedCommands()` | `dict` | Available integrations and scripts |
| `demisto.parentEntry()` | `dict` | War room entry that triggered this command |

---

## Argument Parsing Utilities

Use these `CommonServerPython` helpers — never raw `args.get()` + casting:

```python
# argToList: "a,b,c" → ['a', 'b', 'c']; list passes through; None → []
ids = argToList(args.get('ids'))

# argToBoolean: handles "true"/"false"/"yes"/"no"/True/False
verify_ssl = argToBoolean(args.get('verify', 'true'))

# arg_to_number: converts to int/float; raises ValueError with arg_name in message if invalid
limit = arg_to_number(args.get('limit'), arg_name='limit', required=False) or 50

# arg_to_datetime: parses ISO strings, epoch ints, and relative strings ("3 days ago")
since = arg_to_datetime(args.get('since'), arg_name='since', required=False)
if since:
    since_str = since.strftime('%Y-%m-%dT%H:%M:%SZ')
```

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

---

## Indicator Enrichment Pattern

For commands that enrich indicators (IP, domain, URL, file hash):

```python
from CommonServerPython import DBotScore, Common, argToList


def ip_command(client, args):
    """Enrich one or more IP addresses."""
    ips = argToList(args.get('ip'))
    results = []

    for ip in ips:
        data = client.get_ip_reputation(ip)
        score_value = map_vendor_score(data.get('risk'))

        dbot_score = DBotScore(
            indicator=ip,
            indicator_type=DBotScore.IP,
            integration_name='VendorName',
            score=score_value,
            reliability=DBotScore.Reliability.B_USUALLY_RELIABLE
        )

        ip_indicator = Common.IP(
            ip=ip,
            dbot_score=dbot_score,
            asn=data.get('asn'),
            organization=data.get('org'),
            geo_country=data.get('country')
        )

        results.append(CommandResults(
            readable_output=tableToMarkdown(f'IP Reputation: {ip}', [data]),
            outputs_prefix='Vendor.IP',
            outputs_key_field='ip',
            outputs=data,
            indicator=ip_indicator
        ))

    return results  # return_results() accepts a list


def map_vendor_score(risk_level: str) -> int:
    return {
        'none': DBotScore.GOOD,
        'low': DBotScore.SUSPICIOUS,
        'medium': DBotScore.SUSPICIOUS,
        'high': DBotScore.BAD,
        'critical': DBotScore.BAD,
    }.get((risk_level or '').lower(), DBotScore.NONE)
```

**DBotScore constants:**
- `DBotScore.NONE` = 0 (Unknown)
- `DBotScore.GOOD` = 1
- `DBotScore.SUSPICIOUS` = 2
- `DBotScore.BAD` = 3

**DBotScore reliability constants:**
- `DBotScore.Reliability.A_VERY_RELIABLE`
- `DBotScore.Reliability.B_USUALLY_RELIABLE`
- `DBotScore.Reliability.C_FAIRLY_RELIABLE`
- `DBotScore.Reliability.D_NOT_USUALLY_RELIABLE`

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

---

## Entity Relationship Pattern

Use `EntityRelationship` to define relationships between indicators:

```python
from CommonServerPython import EntityRelationship, FeedIndicatorType, DBotScore


def build_relationships(domain: str, ip: str) -> list:
    return [
        EntityRelationship(
            name=EntityRelationship.Relationships.RESOLVES_TO,
            entity_a=domain,
            entity_a_type=FeedIndicatorType.Domain,
            entity_b=ip,
            entity_b_type=FeedIndicatorType.IP,
            source_reliability=DBotScore.Reliability.B_USUALLY_RELIABLE,
            brand='VendorName'
        )
    ]


# Pass to CommandResults
return CommandResults(
    outputs_prefix='Vendor.Data',
    outputs=data,
    relationships=build_relationships(domain, ip)
)
```

**Common relationship names (`EntityRelationship.Relationships.*`):**
- `RESOLVES_TO`
- `INDICATOR_OF`
- `RELATED_TO`
- `HOSTED_ON`
- `COMMUNICATES_WITH`
- `USES`
- `ATTRIBUTED_TO`

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

---

## Best Practices

### Security
- Never log credentials: `demisto.debug('Authenticating...')` not `demisto.debug(f'Key: {api_key}')`
- Store sensitive config in `type: 4` (encrypted) parameters
- Validate user input at the command boundary

### Code Organization
- Keep API client logic separate from command handlers
- Use `snake_case` for functions and variables; `PascalCase` for class names
- One command function per XSIAM command

### Compatibility
- Use `demisto.alert()` in XSIAM alert-context scripts; `demisto.incident()` in XSOAR
- Python 3.8+ compatible code; prefer standard library over third-party packages
- Pin docker images to specific versions — never `:latest`

---

## Unit Testing (On Request Only)

Unit test files are only generated when the user explicitly requests them. Tests use `pytest` and the `demistomock` layer to run scripts outside Docker.

When requested, the test file **does** include `import demistomock as demisto` — this is the one place that import is needed, since tests run outside the platform where `demisto` is not provided automatically.

```python
# FormatData_test.py
import demistomock as demisto
from FormatData import main


def test_format_json(mocker):
    mocker.patch.object(demisto, 'args', return_value={
        'input_data': '{"key": "value"}',
        'output_format': 'json'
    })
    mock_return = mocker.patch('FormatData.return_results')

    main()

    result = mock_return.call_args[0][0]
    assert result.outputs['Format'] == 'json'
    assert '"key"' in result.outputs['Result']


def test_format_markdown(mocker):
    mocker.patch.object(demisto, 'args', return_value={
        'input_data': '[{"id": 1, "name": "test"}]',
        'output_format': 'markdown'
    })
    mock_return = mocker.patch('FormatData.return_results')

    main()

    result = mock_return.call_args[0][0]
    assert result.outputs['Format'] == 'markdown'


def test_missing_input(mocker):
    mocker.patch.object(demisto, 'args', return_value={
        'input_data': '',
        'output_format': 'json'
    })
    mock_error = mocker.patch('FormatData.return_error')

    main()

    assert mock_error.called
```

**Key patterns:**
- Import `demistomock as demisto` in the test file — needed for local testing outside the platform
- `mocker.patch.object(demisto, 'args', ...)` to control input
- `mocker.patch('ScriptName.return_results')` to capture output without platform side effects
- `mocker.patch('ScriptName.return_error')` to verify error handling
- Run with: `pytest ScriptName_test.py -v`
