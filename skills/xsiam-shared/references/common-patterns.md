# XSIAM/XSOAR Common Python Patterns

Shared patterns used in both scripts and integrations.

---

## Standard Imports

Every XSIAM/XSOAR automation starts with:

```python
import demistomock as demisto
from CommonServerPython import *
from CommonServerUserPython import *
```

- **`demistomock`**: Provides the `demisto` object — the interface to the platform
- **`CommonServerPython`**: Core utilities: `BaseClient`, `CommandResults`, `DemistoException`, `tableToMarkdown`, `return_results`, `return_error`, argument helpers, indicator types, etc.
- **`CommonServerUserPython`**: Integration-specific additions (usually empty or auto-generated)

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
    removeNull=True                    # optional: omit None values
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

## Demisto API Reference

The `demisto` object is the interface between your code and the platform. Import via `import demistomock as demisto`.

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

### Integration Context (Persistent Cache)

| Method | Returns | Description |
|--------|---------|-------------|
| `demisto.getIntegrationContext()` | `dict` | Persistent store for the integration instance |
| `demisto.setIntegrationContext(context)` | `None` | Save integration context |
| `demisto.getIntegrationContextVersioned(refresh)` | `dict` | Versioned context (includes `version` key for optimistic locking) |
| `demisto.setIntegrationContextVersioned(context, version, sync)` | `None` | Save with optimistic locking |

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

**Available `Common.*` indicator types:**
- `Common.IP(ip, dbot_score, asn, organization, geo_country, ...)`
- `Common.Domain(domain, dbot_score, dns_records, ...)`
- `Common.URL(url, dbot_score, ...)`
- `Common.File(md5, sha1, sha256, dbot_score, name, size, ...)`
- `Common.CVE(id, cvss, description, ...)`
- `Common.Account(id, dbot_score, ...)`
- `Common.Email(address, dbot_score, ...)`

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
