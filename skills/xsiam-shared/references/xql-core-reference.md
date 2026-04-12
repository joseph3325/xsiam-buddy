# XQL Core Reference

XQL (XSIAM Query Language) is a pipe-delimited query language for security analytics and threat hunting in Palo Alto Networks Cortex XSIAM. Queries are constructed as a series of pipe-separated stages that transform and analyze security data.

## Query Structure

Every XQL query follows this pattern:

```
<data_source> | <stage1> | <stage2> | ... | <final_stage>
```

**Example:**
```
dataset = xdr_data
| filter action_type = "process_launch"
| fields agent_hostname, action_process_image_name, _time
| comp count() as event_count by agent_hostname
| sort event_count desc
| limit 10
```

### Comments

```
// single-line comment
/* multi-line
   comment */
```

---

## Data Selection

Data selection is the first mandatory stage in any XQL query.

### Dataset

Select a specific dataset directly.

```
dataset = dataset_name
```

**Example:**
```
dataset = xdr_data
```

### Data Model

Query the unified XSIAM data model for cross-source analysis.

```
datamodel
```

**Example:**
```
datamodel | filter _event_type = "authentication"
```

### Datamodel Dataset

Query a specific dataset through the unified XDM (Cross-Data Model) schema. Fields are prefixed with `xdm.*` instead of raw vendor field names.

```
datamodel dataset = dataset_name
```

**Example:**
```
datamodel dataset = microsoft_windows_raw
| filter xdm.event.id in ("4728", "4732", "4756")
| fields xdm.source.host.hostname, xdm.source.user.username, xdm.event.description
```

Use `datamodel dataset =` when the source has XDM mappings and you want normalized field names that work across vendors. Use plain `dataset =` for raw field access.

### Preset

Use a predefined logical grouping of datasets.

```
preset = preset_name
```

**Common presets:**
- `preset = xdr_event` -- all XDR endpoint events
- `preset = network_story` -- network-related events
- `preset = cloud_story` -- cloud activity events
- `preset = identity_story` -- identity/authentication events

**Example:**
```
preset = xdr_event
| filter severity >= "high"
```

### Config Timeframe Preamble

Set a query-level time window as the first line of the query. When used, `dataset` or `preset` must be prefixed with a pipe (`|`).

**Syntax:**
```
config timeframe = <interval>
| dataset = dataset_name
```

**Intervals:** case-insensitive -- `7d`, `30d`, `1h`, `24h`, etc.

**Example:**
```
config timeframe = 30d
| dataset = xdr_data
| filter action_process_image_sha256 = $hash
```

Note: `config timeframe` overrides the XSIAM UI time picker. Use it when the query needs a specific fixed lookback regardless of dashboard settings.

### Time Range via Filter

Time ranges can also be set dynamically in filter stages:

```
dataset = xdr_data
| filter _time > timestamp_sub(current_time(), "7d")
| filter _time < current_time()
```

---

## Operators

### Comparison Operators

| Operator | Description |
|----------|-------------|
| `=` | Equal to |
| `!=` | Not equal to (also accepts `<>`) |
| `<` | Less than |
| `>` | Greater than |
| `<=` | Less than or equal to |
| `>=` | Greater than or equal to |

### Boolean Operators

| Operator | Description |
|----------|-------------|
| `and` | Both conditions must be true |
| `or` | Either condition can be true |
| `not` | Negate a condition |

**Examples:**
```
| filter severity = "critical" and category = "malware"
| filter user_name = "admin" or user_name = "root"
| filter not action_type = "process_launch"
```

### String and Range Operators

| Operator | Description |
|----------|-------------|
| `~=` | Regex match (case-sensitive) |
| `!~=` | Regex negated match (case-sensitive) |
| `contains` | Substring match (case-insensitive) |
| `not contains` | Substring negated match (case-insensitive) |
| `in` | Value in list |
| `not in` | Value not in list |
| `incidr` | IPv4 address in CIDR range |
| `not incidr` | IPv4 address not in CIDR range |
| `incidr6` | IPv6 address in CIDR range |
| `not incidr6` | IPv6 address not in CIDR range |

**Examples:**
```
| filter action_file_name ~= ".*\.exe$"
| filter url contains "malware"
| filter domain !~= "^trusted-.*\.com$"
| filter user_name in ("admin", "root", "system")
| filter action_type not in ("enum_process", "enum_process_memory")
| filter destination_ip incidr("10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16")
```

### ENUM Types

Some fields use typed enumerated constants rather than plain strings. Reference them with the `ENUM.` prefix -- do not quote them as strings.

**Common values:**
- `ENUM.EVENT_LOG` -- Windows or Syslog event log entries
- `ENUM.STORY` -- correlated story events (cross-source auth/identity events)
- `ENUM.NETWORK_CONNECTION` -- network connection events
- `ENUM.FILE` -- file operation events
- `ENUM.PROCESS` -- process execution events

**Examples:**
```
| filter event_type = ENUM.EVENT_LOG and action_evtlog_event_id = 4625
```
```
| filter event_type = ENUM.STORY and auth_outcome != "SUCCESS"
```

### Null Checks

XQL distinguishes between two types of empty values:
- **null** -- field has no value assigned
- **empty string** (`""`) -- field contains a zero-length string

```
| filter field_name = null          // matches null values
| filter field_name != null         // excludes null values
| filter field_name = ""            // matches empty strings
| filter field_name != ""           // excludes empty strings
```

### String Quoting

**Single double quotes** -- literal strings. Wildcards (`*`) behave as XQL wildcards. No escape sequences processed.
```
| filter action_file_name = "malware*.exe"
```

**Triple double quotes** -- regex-style strings. Escape sequences are processed (`\n`, `\t`, `\\`). Use when you need regex matching or special characters.
```
| filter action_file_path ~= """C:\\Users\\.*\\Desktop"""
```

### Network Operators

`incidr(ip, cidr)` returns a boolean. Use `= false` to exclude IPs in a range:

```
| filter source_ip incidr("10.0.0.0/8")
| filter incidr(source_ip, "10.0.0.0/8") = false
```

Note: the function-call form `incidr(ip, cidr)` is required for `= false` comparison -- the infix operator form does not support it.

---

## Pipeline Stages

All 23 XQL stages. The typical ordering is: filter, fields, alter, comp, sort, limit, with other stages as needed.

### alter

Create new computed columns or modify existing fields using functions and expressions.

**Syntax:**
```
| alter <field> = <expression> [, <field> = <expression> ...]
```

**Key notes:**
- Supports all XQL functions in expressions
- Multiple assignments can be comma-separated in a single alter stage
- Commonly used for data transformation, enrichment, and conditional logic

**Examples:**
```
| alter full_path = concat(action_file_path, action_file_name)
```
```
| alter event_hour = format_timestamp(_time, "YYYY-MM-DD HH:00:00")
```
```
| alter is_internal = if(source_ip incidr("10.0.0.0/8"), "yes", "no")
```
```
| alter alert_priority = if(severity = "critical", 1, if(severity = "high", 2, 3))
```

### arrayexpand

Expand an array field into separate records (one record per array element).

**Syntax:**
```
| arrayexpand <array_field>
```

**Key notes:**
- Useful for flattening nested data before filtering or aggregation
- Each array element becomes its own row, with all other fields duplicated

**Examples:**
```
dataset = xdr_data
| alter tags_array = split(tags, ",")
| arrayexpand tags_array
```
```
| alter events = events -> []
| arrayexpand events
```

### bin

Bucket (group) time or numeric values into ranges. Commonly used for timeline analysis and histograms.

**Syntax:**
```
| bin <field> span = <interval>
```

**Key notes:**
- Time intervals: `1m`, `5m`, `10m`, `30m`, `1h`, `6h`, `12h`, `24h`, `1d`, `7d`, `30d`
- Time suffixes: `MS` (milliseconds), `S` (seconds), `M` (minutes), `H` (hours), `D` (days), `W` (weeks), `MO` (months), `Y` (years)
- Works with both time and numeric fields

> **Note:** `bin` and `config timeframe` intervals are case-insensitive (`1h` and `1H` both work). `timestamp_diff` units use quoted uppercase full names (`"HOUR"`, `"DAY"`, etc.).

**Examples:**
```
| bin _time span = 1h
| comp count() as events_per_hour by _time
```
```
| bin file_size span = 1000
| comp count() as count by file_size
```

### call

Reference a predefined query from the XSIAM Query Library.

**Syntax:**
```
| call <query_name>
```

**Key notes:**
- Enables query reuse and modular query design
- The referenced query must exist in the Query Library
- Useful for standardizing common detection patterns across teams

**Example:**
```
dataset = xdr_data
| call suspicious_process_query
```

### comp

Compute aggregation statistics over results.

**Syntax:**
```
| comp <aggregate_function>(<field>) [as <alias>] [by <group_field>]
| comp <agg1> as alias1, <agg2> as alias2 by group_field
```

**Key notes:**
- Must use `by` for grouping; without `by` aggregates over all rows
- Multiple aggregations can be computed in a single comp stage
- See the Core Aggregate Functions section for available functions

**Examples:**
```
| comp count() as total_events
```
```
| comp count() as event_count by agent_hostname
```
```
| comp count_distinct(user_name) as unique_users, sum(bytes_sent) as total_bytes by source_ip
```
```
| comp first(_time) as first_seen, last(_time) as last_seen by alert_id
```

### config

Configure query behavior settings.

**Syntax:**
```
| config <setting> = <value>
```

**Key notes:**
- `case_sensitive` -- enable/disable case-sensitive string comparisons (default: `true`)
- `timeframe` -- override the query time range (e.g., `timeframe = 30d`)
- `max_runtime_minutes` -- set maximum query execution time
- When `config timeframe` is the first line, `dataset`/`preset` must be pipe-prefixed

**Examples:**
```
config timeframe = 30d
| dataset = xdr_data
| filter action_process_image_sha256 = $hash
```
```
| config object_type = "user_list"
```

### dedup

Remove duplicate rows based on field values, keeping the first unique value.

**Syntax:**
```
| dedup <field> [, <field> ...]
```

**Key notes:**
- Works with numbers and strings only (not arrays or objects)
- Multiple fields create a composite dedup key

**Examples:**
```
| dedup agent_hostname
```
```
| dedup src, dst
```

### fields

Define which fields to return in results. Reduces output size and improves readability.

**Syntax:**
```
| fields <field> [, <field> ...]
```

**Key notes:**
- Supports wildcards: `| fields action_file_*`
- Supports exclusion with minus: `| fields - _raw_log`
- Use early in the pipeline to reduce data size before aggregations
- Wildcard with raw column aliasing for datamodel queries (see example)

**Examples:**
```
| fields agent_hostname, action_process_image_name, action_process_image_path, _time
```
```
| fields *, darktrace_darktrace_raw.percentscore as percentscore,
          darktrace_darktrace_raw.triggeredComponents as triggeredComponents
```

### filter

Narrow results using boolean expressions.

**Syntax:**
```
| filter <expression>
```

**Key notes:**
- Supports all comparison, boolean, string/range, and network operators
- Use early in the pipeline to reduce data volume
- Supports inline dataset lookups (subqueries)

**Examples:**
```
| filter action_type = "process_launch"
```
```
| filter severity >= "high" and category = "malware"
```
```
| filter action_file_name ~= ".*\.exe$"
```
```
| filter destination_ip incidr("10.0.0.0/8", "172.16.0.0/12")
```

**Inline dataset lookup:**
```
| filter lowercase(xdm.target.user.groups) in (
    dataset = ad_group_lookup
    | alter group_name = lowercase(group_name)
    | fields group_name
  )
```

### getrole

Enrich events with identity role information.

**Syntax:**
```
| getrole
```

**Key notes:**
- Requires Identity Threat Module license
- Adds role-based context to authentication and identity events
- No parameters -- operates on the current result set

**Example:**
```
preset = identity_story
| filter auth_outcome = "FAILURE"
| getrole
| fields user_name, role, auth_outcome, _time
```

### iploc

Associate IP addresses with geolocation attributes.

**Syntax:**
```
| iploc <ip_field>
```

**Key notes:**
- Adds country, region, city, latitude, longitude fields to results
- IPv4 only
- The enrichment fields are added as new columns

**Examples:**
```
dataset = panw_ngfw_traffic_raw
| iploc src_ip
| fields src_ip, src_ip_country, src_ip_city, _time
```
```
| iploc destination_ip
| filter destination_ip_country != "United States"
```

### join

Combine rows from two datasets based on matching field values.

**Syntax:**
```
| join type=<join_type> (<subquery>) as <alias> <field_1> = <alias>.<field_2>
```

**Key notes:**
- Join types: `inner`, `left`, `right`, `cross`
- Conflict strategies: `left` (keep left values), `right` (keep right values), `both` (keep both, prefixed)
- Subqueries can include their own filter, alter, arrayexpand, and fields stages
- Multiple sequential join stages can be chained

**Examples:**
```
dataset = xdr_data
| join type=inner (dataset = endpoints | fields endpoint_name, endpoint_id) as endpoints
  xdr_data.agent_hostname = endpoints.endpoint_name
```
```
dataset = panw_ngfw_traffic_raw
| join type=left (preset = cloud_story | fields identity_name) as cloud
  panw_ngfw_traffic_raw.src = cloud.source_ip
```

**Chained joins:**
```
dataset = microsoft_windows_raw
| join type = inner (
    dataset = pan_dss_raw
    | filter source = "ad" and type = "user"
    | fields sam_account_name, netbios
  ) as ad_users source_username = ad_users.sam_account_name and source_domain = ad_users.netbios
| join type = inner (
    dataset = pan_dss_raw
    | filter source = "ad" and type = "computer"
    | fields name, dns_host_name, ou
  ) as ad_computers host_name = ad_computers.name or host_name = ad_computers.dns_host_name
```

### limit

Restrict the maximum number of returned records.

**Syntax:**
```
| limit <number>
```

**Key notes:**
- Defaults: Basic queries (Query Builder): 1000; Correlation Rules / API queries: 1,000,000 (1M)
- Use to avoid returning excessive results and to improve query performance

**Examples:**
```
| limit 100
```
```
| limit 10
```

### replacenull

Replace null values with a specified text string.

**Syntax:**
```
| replacenull <field> = "<replacement_text>"
```

**Key notes:**
- Only replaces `null`, not empty strings (`""`)
- Useful for normalizing output and ensuring consistent display values

**Examples:**
```
| replacenull agent_hostname = "UNKNOWN"
```
```
| replacenull user_name = "N/A"
| replacenull source_ip = "0.0.0.0"
```

### search

Free-text search across all fields in a dataset.

**Syntax:**
```
search <search_term> in <dataset>
```

**Key notes:**
- Limited to 90 days of data
- Less precise than `filter` but useful for exploratory queries
- Searches across all fields, not just a specific one
- Prefer `filter` for production queries and correlation rules

**Example:**
```
search "mimikatz" in xdr_data
```

### sort

Order results by one or more fields.

**Syntax:**
```
| sort <field> <asc|desc> [, <field> <asc|desc> ...]
```

**Key notes:**
- Default direction varies by context; always specify explicitly
- Multiple fields are sorted in the order specified

**Examples:**
```
| sort event_count desc
```
```
| sort severity desc, _time asc
```

### tag

Add tags to the `_tag` system field on matching records.

**Syntax:**
```
| tag add "<tag_value>"
```

**Key notes:**
- Tags persist on records and can be filtered in subsequent queries
- Useful for marking records during investigation workflows
- Only the `add` operation is supported

**Example:**
```
dataset = xdr_data
| filter action_process_image_name contains "mimikatz"
| tag add "suspicious_tool"
```

### target

Save query results to a dataset or lookup table.

**Syntax:**
```
| target <target_type> = "<target_name>"
```

**Key notes:**
- Target types: `dataset`, `lookup`
- Size limits: Lookup via XQL: 50 MB; Lookup via Dataset Management: 30 MB
- Useful for creating enrichment tables and persisting investigation results

**Example:**
```
dataset = xdr_data
| filter action_process_image_sha256 != null
| comp count() as exec_count by action_process_image_sha256
| filter exec_count = 1
| target lookup = "rare_hashes"
```

### top

Return approximate top elements with count and percentage.

**Syntax:**
```
| top <number> <field>
```

**Key notes:**
- Returns `_count` and `_percentage` columns automatically
- Approximate results -- suitable for exploratory analysis
- Simpler alternative to `comp count() by field | sort desc | limit N`

**Example:**
```
dataset = xdr_data
| top 10 agent_hostname
```

### transaction

Group related events into transactions based on shared field values within a time window.

**Syntax:**
```
| transaction <field> [, <field> ...] [maxspan = <time_interval>]
```

**Key notes:**
- Maximum 50 fields per transaction
- Groups related events (e.g., session start/end) into single transaction records
- Useful for detecting multi-step attacks and correlating session activity

**Examples:**
```
| transaction user_name maxspan = 30m
| comp count() as step_count by user_name
```
```
| transaction session_id maxspan = 2h
| filter step_count > 5
```

### union

Combine results from multiple queries into a single result set (appends rows).

**Syntax:**
```
(<query1>) union (<query2>) [union (<query3>) ...]
```

**Key notes:**
- Unlike `join` (which merges columns), `union` stacks result sets vertically
- All queries should ideally return the same columns for clean output

**Examples:**
```
(dataset = xdr_data | filter severity = "critical")
union
(dataset = xdr_alerts | filter alert_source = "malware")
```
```
dataset = auth_logs
| union (dataset = login_logs)
```

### view

Configure how results are displayed in the Query Builder UI.

**Syntax:**
```
| view <option> = <value>
```

**Key notes:**
- Options include: highlight fields, graph type, column order
- Does not affect the data -- only the visual presentation
- `column order = populated` suppresses columns with no non-null values

**Examples:**
```
| fields _time, agent_hostname, action_process_image_path, action_file_path
| view column order = populated
```
```
| view graph_type = line
```

### windowcomp

Perform window functions across ordered result sets.

**Syntax:**
```
| windowcomp <function>(<field>) as <alias> [partition by <field>] [order by <field>]
```

**Key notes:**
- Supports window functions (see xql-advanced-functions.md for the full list) and standard aggregate functions as running calculations
- `partition by` defines grouping; `order by` defines row ordering within each partition

**Example:**
```
dataset = xdr_data
| comp count() as event_count by agent_hostname, _time
| windowcomp sum(event_count) as running_total partition by agent_hostname order by _time
```

---

## Common Functions

Core functions organized by category. Used within `alter`, `comp`, `filter`, and `windowcomp` stages.

### String Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `concat` | Concatenate two or more strings | `concat(string1, string2 [, ...])` |
| `format_string` | Format string with placeholders | `format_string("%s has %d items", str_val, int_val)` |
| `len` | Return string length | `len(string)` |
| `lowercase` | Convert string to lowercase | `lowercase(string)` |
| `uppercase` | Convert string to uppercase | `uppercase(string)` |
| `trim` | Remove leading and trailing whitespace | `trim(string)` |
| `ltrim` | Remove leading whitespace | `ltrim(string)` |
| `rtrim` | Remove trailing whitespace | `rtrim(string)` |
| `replace` | Replace substring occurrences | `replace(string, old_substr, new_substr)` |
| `replex` | Replace regex matches | `replex(string, regex_pattern, replacement)` |
| `split` | Split string by delimiter into array | `split(string, delimiter)` |
| `regextract` | Extract regex matches into array | `regextract(string, regex_pattern)` |
| `regexcapture` | Capture regex group matches into fields | `regexcapture(string, regex_pattern)` |
| `string_count` | Count occurrences of substring | `string_count(string, substring)` |
| `wildcard_match` | Test if string matches wildcard pattern | `wildcard_match(string, pattern)` |
| `sha256` | Compute SHA-256 hash of string | `sha256(string)` |
| `substring` | Extract portion of a string | `substring(string, start, length)` |

**regextract notes:** Returns an array of all matches. To get the first match as a scalar, wrap with `arrayindex(..., 0)`:
```
| alter username = arrayindex(regextract(action_evtlog_message, "Account Name:\s+(\S+)"), 0)
```

> `regexcapture` creates **named fields** from capture groups (e.g., `regexcapture(msg, "User=(?P<user>\w+)")` produces a field called `user`). Use `regextract` + `arrayindex` when you need a single unnamed scalar value instead.

### Math Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `add` | Add two numeric values | `add(value1, value2)` |
| `subtract` | Subtract second value from first | `subtract(value1, value2)` |
| `multiply` | Multiply two numeric values | `multiply(value1, value2)` |
| `divide` | Divide first value by second | `divide(value1, value2)` |
| `round` | Round to specified decimal places | `round(value [, decimal_places])` |
| `floor` | Round down to nearest integer | `floor(value)` |
| `pow` | Raise value to exponent | `pow(base, exponent)` |

### Type Conversion Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `to_string` | Convert value to string | `to_string(value)` |
| `to_number` | Convert value to number | `to_number(value)` |
| `to_integer` | Convert value to integer | `to_integer(value)` |
| `to_float` | Convert value to float | `to_float(value)` |
| `to_boolean` | Convert value to boolean | `to_boolean(value)` |
| `coalesce` | Return first non-null value | `coalesce(val1, val2 [, ...])` |
| `if` | Conditional expression | `if(condition, true_value, false_value)` |
| `convert_from_base_64` | Decode Base64 string | `convert_from_base_64(string)` |
| `base64_encode` | Encode string to Base64 | `base64_encode(string)` |
| `base64_decode` | Decode string from Base64 | `base64_decode(string)` |

### Time and Date Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `current_time` | Return current timestamp | `current_time()` |
| `format_timestamp` | Format timestamp as string | `format_timestamp(timestamp, "YYYY-MM-DD HH:mm:ss")` |
| `parse_timestamp` | Parse string to timestamp | `parse_timestamp("format", string)` |
| `timestamp_diff` | Difference between two timestamps | `timestamp_diff(ts1, ts2, "UNIT")` |
| `timestamp_sub` | Subtract interval from timestamp | `timestamp_sub(timestamp, "7d")` |
| `timestamp_add` | Add interval to timestamp | `timestamp_add(timestamp, "7d")` |
| `to_timestamp` | Convert value to timestamp | `to_timestamp(value, "format")` |
| `to_epoch` | Convert timestamp to epoch value | `to_epoch(timestamp, unit)` |
| `parse_epoch` | Parse epoch value to timestamp | `parse_epoch(epoch_value [, unit])` |
| `extract_time` | Extract time component (year, month, etc.) | `extract_time(timestamp, component)` |
| `date_floor` | Round timestamp down to time unit | `date_floor(timestamp, unit)` |
| `timestamp_seconds` | Convert timestamp to epoch seconds | `timestamp_seconds(timestamp)` |
| `time_frame_end` | Return end of current query timeframe | `time_frame_end()` |

**Time difference units:** `"SECOND"`, `"MINUTE"`, `"HOUR"`, `"DAY"`, `"WEEK"`, `"MONTH"`, `"YEAR"`

**Interval format for timestamp_add/sub:** `"7d"`, `"2h"`, `"30m"`, `"60s"`

**Format patterns:**
- `"YYYY-MM-DD"` -- 2024-01-15
- `"YYYY-MM-DD HH:mm:ss"` -- 2024-01-15 14:30:45
- `"MM/DD/YYYY"` -- 01/15/2024
- `"HH:mm:ss"` -- 14:30:45

**Examples:**
```
| filter _time > timestamp_sub(current_time(), "7d")
```
```
| alter event_date = format_timestamp(_time, "YYYY-MM-DD")
```
```
| alter days_old = timestamp_diff(_time, current_time(), "DAY")
```

### IP Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `incidr` | Test if IPv4 is in CIDR range | `incidr(ip, "cidr")` |
| `incidr6` | Test if IPv6 is in CIDR range | `incidr6(ip, "cidr")` |
| `is_ipv4` | Test if string is valid IPv4 | `is_ipv4(string)` |
| `is_ipv6` | Test if string is valid IPv6 | `is_ipv6(string)` |
| `is_known_private_ipv4` | Test if IPv4 is RFC 1918 private | `is_known_private_ipv4(ip)` |
| `is_known_private_ipv6` | Test if IPv6 is private/link-local | `is_known_private_ipv6(ip)` |
| `ip_to_int` | Convert IP address to integer | `ip_to_int(ip_string)` |
| `ip_in_int` | Convert integer to IP address | `ip_in_int(integer)` |

### Core Aggregate Functions

Used within the `comp` stage.

| Function | Description | Syntax |
|----------|-------------|--------|
| `count` | Count of records | `count(field)` or `count()` |
| `count_distinct` | Count of unique values | `count_distinct(field)` |
| `sum` | Sum of numeric values | `sum(field)` |
| `avg` | Average of values | `avg(field)` |
| `min` | Minimum value | `min(field)` |
| `max` | Maximum value | `max(field)` |
| `values` | Collect distinct values into array | `values(field)` |
| `first` | First value encountered | `first(field)` |
| `last` | Last value encountered | `last(field)` |
| `earliest` | Value from earliest timestamp | `earliest(field)` |
| `latest` | Value from latest timestamp | `latest(field)` |
| `median` | Median value | `median(field)` |
| `list` | Collect all values into array | `list(field)` |

### JSON Access

| Function | Description | Syntax |
|----------|-------------|--------|
| `json_extract` | Extract JSON value (preserves type) | `json_extract(field, "$.path")` |
| `json_extract_scalar` | Extract JSON value as string | `json_extract_scalar(field, "$.path")` |

JSON paths use dot-notation: `"$.key.subkey"`. Field names are case-sensitive.

---

## Arrow Notation (JSON Navigation)

Navigate into JSON objects and arrays embedded in string fields.

**Syntax:**
```
field -> key              -- extract scalar value from JSON object
field -> []               -- expand JSON array into an XQL array
field -> key{}            -- extract array of objects under a key
```

**Examples:**
```
| alter actor_email = actor -> email
```
```
| alter all_events = events -> []
```

**Nested paths:**
```
| alter city = location -> address -> city
```

**With arrayexpand for iteration:**
```
| alter items = data -> items -> []
| arrayexpand items
| alter item_name = items -> name
```

Extract scalar values from nested JSON returned by arrow notation:
```
| alter event_type = json_extract_scalar(raw_data, "$.event.type")
| alter source_ip = json_extract_scalar(raw_data, "$.event.source.ip")
```

---

## Common Query Patterns

### Threat Hunting by IOC

Find hosts or users associated with known indicators of compromise.

```
dataset = xdr_data
| filter action_file_sha256 = "4bac2d4b0efc47e68a09e250a51c0891eca686b7b4546f56995031eea8d426eb"
| fields agent_hostname, agent_ip, action_process_image_name, action_file_name, _time
| sort _time desc
```

### Timeline Analysis

Bucket events by time interval to identify activity patterns.

```
dataset = xdr_data
| filter severity >= "high"
| bin _time span = 1h
| comp count() as events_per_hour by _time
| sort _time asc
```

### Lateral Movement Detection

Join endpoint and network data to identify suspect connections.

```
dataset = panw_ngfw_traffic_raw
| join type=inner (dataset = xdr_data | fields agent_hostname, agent_ip) as xdr
  panw_ngfw_traffic_raw.src = xdr.agent_ip
| filter proto = "tcp" and dport in (445, 139, 3389)
| comp count() as connection_count by agent_hostname
```

### Data Exfiltration Analysis

Identify unusual outbound data transfers.

```
dataset = panw_ngfw_traffic_raw
| comp sum(bytes_sent) as total_bytes_out by src, dst
| filter total_bytes_out > 1000000000
| sort total_bytes_out desc
```

### User Behavior Analysis

Track user actions across authentication and endpoint events.

```
preset = identity_story
| transaction user_name maxspan = 2h
| comp count() as activity_count, first(_time) as first_activity by user_name
| filter activity_count > 50
```

### Failed Authentication Spike Detection

Identify potential brute-force attacks.

```
dataset = okta_raw
| filter outcome_result = "FAILURE"
| bin _time span = 5m
| comp count() as failed_attempts by actor_id, _time
| filter failed_attempts > 10
| sort failed_attempts desc
```

### File Hash Reputation Lookup

Track process execution by SHA256 hash.

```
dataset = xdr_data
| filter action_process_image_sha256 != null
| comp count_distinct(agent_hostname) as hosts_affected by action_process_image_sha256
| sort hosts_affected desc
| limit 20
```

---

## Best Practices

1. **Performance**: Use `fields` early to reduce data size before aggregations. Filter as early as possible.
2. **Readability**: Break complex queries into multiple lines for clarity. Use `//` comments to document intent.
3. **Regex**: Regex patterns are case-sensitive with `~=` and `!~=`. Use `contains` for case-insensitive substring matching.
4. **Time Zones**: Timestamps are in UTC unless otherwise configured.
5. **Limits**: Use `limit` to avoid returning excessive results. Default limits differ between Query Builder (1000) and API (1M).
6. **Aliases**: Use `as` keyword to rename output columns for clarity.
7. **Null Handling**: Use `coalesce()` to provide defaults for null values. Distinguish between `null` and `""` in filters.
8. **Testing**: Test queries on small time windows first before expanding scope. Use `config timeframe` for fixed lookback windows.
9. **Wide Datasets**: Use `view column order = populated` to suppress empty columns when investigating across datasets with many fields.
10. **Subqueries**: Inline dataset lookups in `filter` must produce a single column via a `fields` stage.
