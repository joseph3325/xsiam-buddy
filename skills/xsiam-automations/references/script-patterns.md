# XSOAR/XSIAM Python Automation Script Patterns

This document provides comprehensive patterns and best practices for writing Python automation scripts in XSOAR and XSIAM.

## Table of Contents

1. [Standard Imports and Structure](#standard-imports-and-structure)
2. [Main Function Pattern](#main-function-pattern)
3. [Common API Patterns](#common-api-patterns)
4. [Output Patterns](#output-patterns)
5. [Fetch Incidents Pattern](#fetch-incidents-pattern)
6. [Test Module Pattern](#test-module-pattern)
7. [Error Handling](#error-handling)
8. [Directory Structure Convention](#directory-structure-convention)
9. [Best Practices](#best-practices)

---

## Standard Imports and Structure

Every XSOAR/XSIAM automation script should start with these imports:

```python
import demistomock as demisto
from CommonServerPython import *
from CommonServerUserPython import *
```

### Import Breakdown

- **`demistomock`**: Provides the `demisto` object with methods for accessing:
  - Arguments: `demisto.args()`
  - Parameters: `demisto.params()`
  - Last run data: `demisto.getLastRun()`
  - Incident manipulation: `demisto.incidents()`

- **`CommonServerPython`**: Provides core utilities:
  - `BaseClient`: HTTP client base class for API integrations
  - `CommandResults`: Standard output format
  - `DemistoException`: Standard exception class
  - `tableToMarkdown()`: Convert data to markdown table
  - `createContext()`: Create context output
  - `fileResult()`: Create file output
  - `return_results()`: Return command results
  - `return_error()`: Return error response

- **`CommonServerUserPython`**: Additional utilities specific to your integration (usually empty or integration-specific)

---

## Main Function Pattern

This is the standard pattern for all automation scripts:

```python
def main():
    try:
        # Get command arguments
        args = demisto.args()

        # Get integration configuration parameters
        params = demisto.params()

        # Initialize client or handler
        client = ExampleClient(
            base_url=params.get('server_url'),
            api_key=params.get('api_key'),
            verify=not params.get('insecure')
        )

        # Execute the appropriate command
        command = demisto.command()

        if command == 'test-module':
            return_results(test_module(client))

        elif command == 'example-get-data':
            return_results(get_data_command(client, args))

        elif command == 'example-query':
            return_results(query_command(client, args))

        else:
            return_error(f'Unknown command: {command}')

    except Exception as e:
        return_error(f'Failed to execute {demisto.command()}. Error: {str(e)}')


if __name__ in ('__main__', '__builtin__', 'builtins'):
    main()
```

### Key Points

1. **Try-Except Wrapper**: Always wrap in try-except to catch unexpected errors
2. **Get Args and Params**: Retrieve both arguments and configuration
3. **Command Router**: Use `demisto.command()` to route to appropriate handler
4. **Error Handling**: Return errors with `return_error()` for proper error reporting
5. **Module Guard**: `if __name__ in ...` allows testing without executing main

---

## Common API Patterns

### BaseClient Class Pattern

For integrations that communicate with external APIs, extend `BaseClient`:

```python
from CommonServerPython import BaseClient, DemistoException
import requests


class ExampleClient(BaseClient):
    """Client for interfacing with Example API."""

    def __init__(self, base_url: str, api_key: str, verify: bool = True):
        """
        Initialize the client.

        Args:
            base_url: Base URL of the API
            api_key: API key for authentication
            verify: Whether to verify SSL certificates
        """
        super().__init__(base_url=base_url, verify=verify)
        self.api_key = api_key

    def _get_headers(self) -> dict:
        """Get standard headers for API requests."""
        return {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json',
            'User-Agent': 'Demisto/1.0'
        }

    def get_data(self, query: str, limit: int = 50) -> dict:
        """
        Get data from the API.

        Args:
            query: Search query string
            limit: Maximum number of results

        Returns:
            dict: API response data

        Raises:
            DemistoException: If API returns error
        """
        try:
            response = self._http_request(
                method='GET',
                url_suffix='/api/v1/data',
                headers=self._get_headers(),
                params={'q': query, 'limit': limit}
            )
            return response
        except requests.exceptions.RequestException as e:
            raise DemistoException(f'API request failed: {str(e)}')

    def test_connection(self) -> bool:
        """Test connectivity to the API."""
        try:
            self._http_request(
                method='GET',
                url_suffix='/api/v1/health',
                headers=self._get_headers()
            )
            return True
        except Exception:
            return False
```

### Authentication Patterns

#### API Key Authentication
```python
def _get_headers(self) -> dict:
    return {
        'Authorization': f'Bearer {self.api_key}',
        'Content-Type': 'application/json'
    }
```

#### Basic Authentication
```python
from requests.auth import HTTPBasicAuth

def _get_auth(self) -> HTTPBasicAuth:
    return HTTPBasicAuth(self.username, self.password)

def get_data(self, query: str) -> dict:
    response = self._http_request(
        method='GET',
        url_suffix='/api/v1/data',
        params={'q': query},
        auth=self._get_auth()
    )
    return response
```

#### Token Authentication
```python
def get_token(self) -> str:
    """Obtain and cache authentication token."""
    if hasattr(self, '_token') and self._token:
        return self._token

    response = self._http_request(
        method='POST',
        url_suffix='/auth/token',
        json={'username': self.username, 'password': self.password}
    )
    self._token = response.get('token')
    return self._token

def _get_headers(self) -> dict:
    return {
        'Authorization': f'Token {self.get_token()}',
        'Content-Type': 'application/json'
    }
```

### HTTP Request Patterns

```python
# GET request
response = self._http_request(
    method='GET',
    url_suffix='/endpoint',
    params={'key': 'value'},
    headers=self._get_headers()
)

# POST request
response = self._http_request(
    method='POST',
    url_suffix='/endpoint',
    json={'field': 'value'},
    headers=self._get_headers()
)

# PUT request with data
response = self._http_request(
    method='PUT',
    url_suffix='/endpoint/123',
    data='raw data',
    headers=self._get_headers()
)

# DELETE request
response = self._http_request(
    method='DELETE',
    url_suffix='/endpoint/123',
    headers=self._get_headers()
)

# Request with timeout
response = self._http_request(
    method='GET',
    url_suffix='/endpoint',
    timeout=30
)
```

### Pagination Handling

```python
def get_all_items(self, base_query: str) -> list:
    """Fetch all items with automatic pagination."""
    all_items = []
    page = 1
    page_size = 50

    while True:
        response = self._http_request(
            method='GET',
            url_suffix='/api/v1/items',
            headers=self._get_headers(),
            params={
                'q': base_query,
                'page': page,
                'page_size': page_size
            }
        )

        items = response.get('items', [])
        if not items:
            break

        all_items.extend(items)

        # Check if there are more pages
        if len(items) < page_size:
            break

        page += 1

    return all_items
```

---

## Output Patterns

### CommandResults Object Pattern

The standard way to return command results:

```python
def get_data_command(client: ExampleClient, args: dict) -> CommandResults:
    """
    Get data and return formatted results.

    Args:
        client: API client instance
        args: Command arguments containing 'query' and optional 'limit'

    Returns:
        CommandResults: Formatted command output
    """
    query = args.get('query')
    limit = int(args.get('limit', 50))

    # Call API
    data = client.get_data(query=query, limit=limit)

    # Format for human-readable output
    human_readable = tableToMarkdown(
        'Data Results',
        data.get('items', []),
        headers=['ID', 'Name', 'Status'],
        headerTransform=pascalToSpace
    )

    # Return formatted results
    return CommandResults(
        content_type=outputs.ContentType.MARKDOWN,
        readable_output=human_readable,
        outputs_prefix='Example.Data',
        outputs_key_field='id',
        outputs=data.get('items', [])
    )
```

### Markdown Table Output

```python
def format_results(items: list) -> str:
    """Convert items to markdown table."""
    return tableToMarkdown(
        'Search Results',
        items,
        headers=['ID', 'Name', 'Type', 'Status'],
        headerTransform=pascalToSpace,
        removeNull=True
    )

# Usage
human_readable = format_results(results)
return CommandResults(
    readable_output=human_readable,
    outputs_prefix='Example.Result',
    outputs=results
)
```

### Context Creation

```python
def create_context_output(data: dict) -> dict:
    """Create context output from raw data."""
    context = createContext(
        data,
        keyTransform=underscoreToCamelCase
    )
    return context
```

### File Output

```python
def export_results_command(client: ExampleClient, args: dict) -> CommandResults:
    """Export results to a file."""
    query = args.get('query')
    file_format = args.get('format', 'json')

    data = client.get_data(query=query)

    if file_format == 'json':
        import json
        file_content = json.dumps(data, indent=2)
        filename = f'results_{query}.json'
    elif file_format == 'csv':
        import csv
        from io import StringIO

        output = StringIO()
        writer = csv.DictWriter(output, fieldnames=data[0].keys())
        writer.writeheader()
        writer.writerows(data)
        file_content = output.getvalue()
        filename = f'results_{query}.csv'

    return fileResult(filename, file_content)
```

---

## Fetch Incidents Pattern

For integrations that fetch incidents/events from external systems:

```python
def fetch_incidents(client: ExampleClient) -> tuple[list, dict]:
    """
    Fetch incidents from the API.

    Returns:
        tuple: (incidents list, last_run dict for next fetch)
    """
    # Get last run time
    last_run = demisto.getLastRun()
    last_fetch_time = last_run.get('last_fetch', 0)

    # Fetch incidents from API
    incidents = []
    newest_time = last_fetch_time

    try:
        response = client.get_incidents(since=last_fetch_time)

        for item in response.get('items', []):
            # Convert API item to incident
            incident = {
                'name': item.get('title'),
                'occurred': item.get('created_at'),
                'rawJSON': json.dumps(item),
                'type': 'Example Incident',
                'severity': map_severity(item.get('severity'))
            }

            incidents.append(incident)

            # Track newest timestamp for next fetch
            created_time = int(parse_datetime(item.get('created_at')).timestamp())
            if created_time > newest_time:
                newest_time = created_time

        # Deduplication: filter out previously fetched incidents
        incidents = dedup_incidents(incidents, last_run.get('dedup_ids', []))

    except Exception as e:
        demisto.error(f'Error fetching incidents: {str(e)}')

    # Store last run data for next fetch
    next_last_run = {
        'last_fetch': newest_time,
        'dedup_ids': [inc['name'] for inc in incidents]
    }
    demisto.setLastRun(next_last_run)

    return incidents, next_last_run


def dedup_incidents(incidents: list, seen_ids: list) -> list:
    """Remove duplicates based on incident ID."""
    return [
        inc for inc in incidents
        if inc.get('name') not in seen_ids
    ]


def map_severity(api_severity: str) -> int:
    """Map API severity to Demisto severity level."""
    severity_map = {
        'low': 1,
        'medium': 2,
        'high': 3,
        'critical': 4
    }
    return severity_map.get(api_severity.lower(), 0)
```

### Using Fetch in Main

```python
def main():
    try:
        params = demisto.params()
        client = ExampleClient(
            base_url=params.get('server_url'),
            api_key=params.get('api_key')
        )

        command = demisto.command()

        if command == 'fetch-incidents':
            incidents, _ = fetch_incidents(client)
            demisto.incidents(incidents)

        elif command == 'test-module':
            return_results(test_module(client))

    except Exception as e:
        return_error(f'Error: {str(e)}')


if __name__ in ('__main__', '__builtin__', 'builtins'):
    main()
```

---

## Test Module Pattern

All integrations should implement a `test-module` command:

```python
def test_module(client: ExampleClient) -> str:
    """
    Test connectivity to the external API.

    Args:
        client: API client instance

    Returns:
        str: 'ok' if successful, error message otherwise
    """
    try:
        # Attempt a simple API call
        if client.test_connection():
            return 'ok'
        else:
            return 'Failed to connect to API'
    except DemistoException as e:
        return f'Authentication failed: {str(e)}'
    except requests.exceptions.ConnectionError:
        return 'Failed to reach API server. Check server URL.'
    except requests.exceptions.Timeout:
        return 'API request timeout. Check server connectivity.'
    except Exception as e:
        return f'Unexpected error: {str(e)}'


def main():
    try:
        params = demisto.params()
        client = ExampleClient(
            base_url=params.get('server_url'),
            api_key=params.get('api_key'),
            verify=not params.get('insecure')
        )

        if demisto.command() == 'test-module':
            result = test_module(client)
            if result == 'ok':
                return_results(result)
            else:
                return_error(result)

    except Exception as e:
        return_error(f'Test failed: {str(e)}')
```

---

## Error Handling

### Standard Exception Handling

```python
from CommonServerPython import DemistoException

try:
    result = client.get_data(query)
except DemistoException as e:
    return_error(f'API error: {str(e)}')
except requests.exceptions.RequestException as e:
    return_error(f'Network error: {str(e)}')
except ValueError as e:
    return_error(f'Invalid input: {str(e)}')
except Exception as e:
    return_error(f'Unexpected error: {str(e)}')
```

### Custom Exception Handling

```python
def validate_args(args: dict) -> dict:
    """Validate and normalize command arguments."""
    required_fields = ['query']

    for field in required_fields:
        if field not in args or not args[field]:
            raise ValueError(f'Missing required field: {field}')

    # Type validation
    try:
        limit = int(args.get('limit', 50))
        if limit < 1 or limit > 1000:
            raise ValueError('Limit must be between 1 and 1000')
    except ValueError as e:
        raise ValueError(f'Invalid limit value: {str(e)}')

    return {
        'query': args['query'],
        'limit': limit
    }
```

---

## Output Format: Unified YAML

XSIAM requires **unified YAML** for import — a single `.yml` file that contains both the metadata AND the Python code embedded in the `script` field. Always generate this format.

### For Scripts

The Python code goes in a **top-level** `script` field as a literal block scalar (`|-`):
```yaml
script: |-
  import demistomock as demisto
  from CommonServerPython import *
  # ... all Python code here ...
type: python
subtype: python3
dockerimage: demisto/python3:3.10.14.100715
```

### For Integrations

The Python code goes inside **`script.script`** (nested):
```yaml
script:
  script: |-
    import demistomock as demisto
    from CommonServerPython import *
    # ... all Python code here ...
  type: python
  subtype: python3
  dockerimage: demisto/python3:3.10.14.100715
  commands:
    - name: my-command
      ...
```

### YAML Block Scalar Rules

- Use `|-` (literal block, strip trailing newline)
- Indent all code lines consistently (typically 4 spaces from the YAML key)
- Do not use tab characters — YAML only allows spaces
- Empty lines in code are fine — just leave them blank at the correct indentation level

### Optional Companion Files

These are NOT part of the import but useful for development:
- **`_test.py`**: Unit tests (pytest + mocker)
- **`README.md`**: Usage documentation
- **`_description.md`**: Feature overview

---

## Best Practices

### Code Organization

1. **Separate Concerns**: Keep API client logic separate from command handlers
```python
# Good structure
class ExampleClient:
    def get_data(self): pass

def get_data_command(client, args):
    return CommandResults(...)

def main():
    if command == 'example-get-data':
        return_results(get_data_command(client, args))
```

2. **Function Documentation**: Always include docstrings
```python
def get_data(self, query: str) -> dict:
    """
    Get data from API.

    Args:
        query: Search query string

    Returns:
        dict: API response data

    Raises:
        DemistoException: If API returns error
    """
```

3. **Type Hints**: Use Python type hints for clarity
```python
def process_items(items: list) -> dict:
    return createContext(items)
```

### Security

1. **Never Log Credentials**: Don't log API keys or passwords
```python
# Bad
demisto.debug(f'API Key: {api_key}')

# Good
demisto.debug('Authenticating with API')
```

2. **Use Encrypted Config**: Store sensitive data in type 4 fields
```yaml
- display: API Key
  name: api_key
  type: 4  # Encrypted
  required: true
```

3. **Validate Input**: Always validate user input
```python
def command_handler(args: dict):
    if not args.get('query'):
        raise ValueError('query argument is required')
    if len(args['query']) > 1000:
        raise ValueError('query exceeds maximum length')
```

### Testing

1. **Unit Tests**: Test command handlers and client methods
```python
def test_get_data_command(mocker):
    mock_client = mocker.Mock()
    mock_client.get_data.return_value = {'items': []}

    result = get_data_command(mock_client, {'query': 'test'})
    assert isinstance(result, CommandResults)
```

2. **Error Cases**: Test error handling
```python
def test_get_data_error(mocker):
    mock_client = mocker.Mock()
    mock_client.get_data.side_effect = DemistoException('API Error')

    with pytest.raises(Exception):
        get_data_command(mock_client, {'query': 'test'})
```

### Documentation

1. **Command Examples**: Provide example outputs for each command
2. **README**: Include setup instructions and usage examples
3. **Error Messages**: Provide helpful error messages for troubleshooting

### Performance

1. **Connection Pooling**: Reuse HTTP connections
```python
# BaseClient handles this automatically
```

2. **Pagination**: Implement for large result sets
3. **Caching**: Cache authentication tokens when appropriate

### Compatibility

1. **Version Support**: Test on supported XSOAR/XSIAM versions
2. **Python Version**: Ensure Python 3.8+ compatibility
3. **Dependencies**: Minimize external dependencies; prefer standard library

---

## Common Patterns Reference

### Import Pattern
```python
import demistomock as demisto
from CommonServerPython import *
from CommonServerUserPython import *
```

### Client Pattern
```python
class MyClient(BaseClient):
    def __init__(self, base_url: str, api_key: str, verify: bool = True):
        super().__init__(base_url=base_url, verify=verify)
        self.api_key = api_key
```

### Command Handler Pattern
```python
def my_command(client: MyClient, args: dict) -> CommandResults:
    result = client.api_call(args.get('param'))
    return CommandResults(
        outputs_prefix='Prefix.Object',
        outputs=result,
        readable_output=tableToMarkdown('Title', result)
    )
```

### Main Function Pattern
```python
def main():
    try:
        params = demisto.params()
        client = MyClient(params.get('url'), params.get('key'))

        if demisto.command() == 'test-module':
            return_results(test_module(client))
        elif demisto.command() == 'my-command':
            return_results(my_command(client, demisto.args()))
    except Exception as e:
        return_error(str(e))

if __name__ in ('__main__', '__builtin__', 'builtins'):
    main()
```
