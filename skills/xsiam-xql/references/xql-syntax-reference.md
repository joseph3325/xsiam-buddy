# XQL (XSIAM Query Language) Syntax Reference

## Overview

XQL is a query language designed for security analytics and threat hunting in Palo Alto Networks XSIAM. Queries are constructed as a series of pipe-separated stages that transform and analyze security data. All queries begin with a data source selection and chain operations with the pipe (`|`) operator.

## Query Structure

### Basic Syntax

Every XQL query follows this pattern:

```
<data_source> | <stage1> | <stage2> | <stage3> ... | <final_stage>
```

### Example Query Flow

```
dataset = xdr_data
| filter action_type = "process_launch"
| fields agent_hostname, action_process_image_name, _time
| comp count() as event_count by agent_hostname
| sort event_count desc
| limit 10
```

---

## Data Selection

Data selection is the first mandatory stage in any XQL query. Choose one of the following:

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

### Preset

Use a predefined logical grouping of datasets.

```
preset = preset_name
```

**Common presets:**
- `preset = xdr_event` — all XDR endpoint events
- `preset = network_story` — network-related events
- `preset = cloud_story` — cloud activity events
- `preset = identity_story` — identity/authentication events

**Example:**
```
preset = xdr_event
| filter severity >= "high"
```

### Time Range

Time ranges are typically specified in the XSIAM UI, but can also be set in filter stages:

```
dataset = xdr_data
| filter _time > timestamp_sub(current_time(), "7d")
| filter _time < current_time()
```

---

## Core Stages

Stages are executed in sequence, transforming data at each step. The typical order is:

1. **filter** — reduce rows by condition
2. **fields** — select columns
3. **alter** — create computed columns
4. **comp** — aggregate
5. **sort** — order results
6. **limit** — restrict count
7. Other operations as needed

### filter

Reduce dataset rows based on conditional logic.

**Syntax:**
```
| filter <condition>
```

**Comparison Operators:**
- `=` — equal to
- `!=` — not equal to
- `<` — less than
- `>` — greater than
- `<=` — less than or equal to
- `>=` — greater than or equal to

**Pattern Matching Operators:**
- `~=` — regex match
- `!~=` — regex not match
- `contains` — substring match (case-insensitive)
- `not contains` — substring not match
- `in` — value in list
- `not in` — value not in list
- `incidr` — IP address in CIDR range

**Logical Operators:**
- `and` — both conditions true
- `or` — either condition true
- `not` — negate condition

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

```
| filter user_name in ("admin", "root", "system")
```

### fields

Select specific columns to return. Reduces output size and improves readability.

**Syntax:**
```
| fields column1, column2, column3, ...
```

**Examples:**

```
| fields agent_hostname, action_process_image_name, action_process_image_path, _time
```

```
| fields src, dst, sport, dport, bytes_sent, bytes_received
```

### alter

Create new computed columns using functions and expressions. Commonly used for data transformation, enrichment, and conditional logic.

**Syntax:**
```
| alter column_name = expression
```

**String Functions:**
- `to_string(value)` — convert to string
- `uppercase(string)` — convert to uppercase
- `lowercase(string)` — convert to lowercase
- `trim(string)` — remove leading/trailing whitespace
- `length(string)` — return string length
- `concat(str1, str2, ...)` — concatenate strings
- `substring(string, start, length)` — extract substring
- `split(string, delimiter)` — split into array
- `replace(string, search, replacement)` — replace substring
- `regex_capture(string, pattern)` — extract regex match

**Numeric Functions:**
- `to_number(value)` — convert to number
- `multiply(a, b)` — multiply
- `add(a, b)` — addition
- `subtract(a, b)` — subtraction
- `divide(a, b)` — division

**Timestamp Functions:**
- `to_timestamp(value)` — convert to timestamp
- `format_timestamp(timestamp, format)` — format timestamp. Format: "YYYY-MM-DD HH:mm:ss"
- `parse_timestamp(string, format)` — parse string to timestamp
- `timestamp_diff(ts1, ts2, unit)` — difference between timestamps. Units: "SECOND", "MINUTE", "HOUR", "DAY"
- `timestamp_sub(timestamp, interval)` — subtract interval. Format: "7d", "2h", "30m"

**Conditional Functions:**
- `if(condition, true_value, false_value)` — conditional logic
- `coalesce(value1, value2, ...)` — return first non-null value

**JSON Functions:**
- `json_extract(json_string, path)` — extract value from JSON
- `json_extract_scalar(json_string, path)` — extract scalar value from JSON

**Array Functions:**
- `array_create(value1, value2, ...)` — create array
- `array_length(array)` — get array length
- `array_distinct(array)` — get distinct array values

**Network Functions:**
- `incidr(ip, cidr)` — check if IP in CIDR range

**Encoding Functions:**
- `base64_encode(string)` — encode to Base64
- `base64_decode(string)` — decode from Base64

**Examples:**

```
| alter full_path = concat(action_file_path, action_file_name)
```

```
| alter process_name_lower = lowercase(action_process_image_name)
```

```
| alter event_hour = format_timestamp(_time, "YYYY-MM-DD HH:00:00")
```

```
| alter is_internal = if(source_ip incidr("10.0.0.0/8"), "yes", "no")
```

```
| alter bytes_total = add(bytes_sent, bytes_received)
```

```
| alter alert_priority = if(severity = "critical", 1, if(severity = "high", 2, 3))
```

### comp

Aggregate data using functions. Must be paired with `by` for grouping.

**Aggregation Functions:**
- `count()` — count rows
- `count_distinct(field)` — count unique values
- `sum(field)` — sum numeric values
- `avg(field)` — calculate average
- `min(field)` — find minimum
- `max(field)` — find maximum
- `values(field)` — collect all values
- `first(field)` — get first value
- `last(field)` — get last value
- `array_distinct(field)` — get distinct array values

**Syntax:**
```
| comp <aggregation> as alias by grouping_field
| comp <agg1> as alias1, <agg2> as alias2 by grouping_field
```

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

### sort

Order results in ascending or descending order.

**Syntax:**
```
| sort field asc
| sort field desc
| sort field1 desc, field2 asc
```

**Examples:**

```
| sort event_count desc
```

```
| sort severity desc, _time asc
```

### limit

Restrict the number of results returned.

**Syntax:**
```
| limit number
```

**Examples:**

```
| limit 100
```

```
| limit 10
```

### dedup

Remove duplicate rows based on field values.

**Syntax:**
```
| dedup field
| dedup field1, field2, ...
```

**Examples:**

```
| dedup agent_hostname
```

```
| dedup src, dst
```

### join

Combine rows from two datasets based on matching field values. Supports multiple join types.

**Syntax:**
```
| join type=<inner|left|right|cross> (subquery) as <alias1> <alias1.field> = <alias2.field>
```

**Join Types:**
- `inner` — only matching rows
- `left` — all rows from left dataset, matching rows from right
- `right` — all rows from right dataset, matching rows from left
- `cross` — all combinations (Cartesian product)

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

### union

Combine results from multiple queries into a single result set.

**Syntax:**
```
(query1) union (query2) union (query3)
```

**Examples:**

```
(dataset = xdr_data | filter severity = "critical")
union
(dataset = xdr_alerts | filter alert_source = "malware")
```

### bin

Bucket (group) time or numeric values into ranges. Commonly used for timeline analysis.

**Syntax:**
```
| bin field span = <interval>
```

**Time Intervals:**
- `1m`, `5m`, `10m`, `30m` — minutes
- `1h`, `6h`, `12h`, `24h` — hours
- `1d`, `7d`, `30d` — days

**Examples:**

```
| bin _time span = 1h
| comp count() as events_per_hour by _time
```

```
| bin file_size span = 1000
| comp count() as count by file_size
```

### config

Query XSIAM configuration tables and reference data.

**Syntax:**
```
| config object_type = "configuration_type"
```

**Examples:**

```
| config object_type = "user_list"
```

```
| config object_type = "ip_list"
```

### arrayexpand

Expand array fields into individual rows, one per array element.

**Syntax:**
```
| arrayexpand array_field
```

**Examples:**

```
dataset = xdr_data
| alter tags_array = split(tags, ",")
| arrayexpand tags_array
```

### transaction

Group related events by shared field values within a time window. Useful for detecting multi-step attacks.

**Syntax:**
```
| transaction field maxspan = time_interval
```

**Examples:**

```
| transaction user_name maxspan = 30m
| comp count() as step_count by user_name
```

### iploc

Enrich IP addresses with geographic location data.

**Syntax:**
```
| alter geo_location = iploc(ip_field)
```

**Examples:**

```
| alter src_country = iploc(source_ip)
```

### window

Apply sliding window functions for time-series analysis and anomaly detection.

**Syntax:**
```
| window function=<function> size=<interval> by grouping_field
```

**Examples:**

```
| window function=avg size=1h by agent_hostname
```

---

## Operators and Expressions

### Comparison Operators

Used in filter and conditional statements:

- `=` — equal to
- `!=` — not equal to (also accepts `<>`)
- `<` — less than
- `>` — greater than
- `<=` — less than or equal to
- `>=` — greater than or equal to

### Pattern Matching Operators

- `~=` — regex match. Case-sensitive.
- `!~=` — regex not match. Case-sensitive.
- `contains` — substring match. Case-insensitive.
- `not contains` — substring not match. Case-insensitive.

**Examples:**

```
| filter action_file_name ~= ".*\.exe$"
| filter url contains "malware"
| filter domain !~= "^trusted-.*\.com$"
```

### Logical Operators

- `and` — both conditions must be true
- `or` — either condition can be true
- `not` — negate a condition

**Examples:**

```
| filter severity = "critical" and category = "malware"
| filter user_name = "admin" or user_name = "root"
| filter not action_type = "process_launch"
```

### List Operators

- `in` — value exists in list
- `not in` — value does not exist in list

**Examples:**

```
| filter user_name in ("admin", "root", "system")
| filter action_type not in ("enum_process", "enum_process_memory")
```

### Null Checks

- `field = null` — field is null
- `field != null` — field is not null

**Examples:**

```
| filter action_process_image_path != null
| filter user_name = null
```

### Network Operators

- `incidr(ip, cidr_range)` — check if IP is in CIDR range
- `incidr(ip, cidr1, cidr2, ...)` — check if IP is in any CIDR range

**Examples:**

```
| filter source_ip incidr("10.0.0.0/8")
| filter destination_ip incidr("10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16")
```

---

## Time Functions and Relative Time

### Current Time Reference

- `current_time()` — returns current system time

### Time Arithmetic

- `timestamp_add(timestamp, interval)` — add interval to timestamp
- `timestamp_sub(timestamp, interval)` — subtract interval from timestamp

**Interval Format:**
- `"7d"` — 7 days
- `"2h"` — 2 hours
- `"30m"` — 30 minutes
- `"60s"` — 60 seconds

### Time Difference

- `timestamp_diff(ts1, ts2, unit)` — calculate difference between timestamps

**Units:**
- `"SECOND"`, `"MINUTE"`, `"HOUR"`, `"DAY"`, `"WEEK"`, `"MONTH"`, `"YEAR"`

### Time Formatting

- `format_timestamp(timestamp, format)` — format timestamp for display
- `parse_timestamp(string, format)` — parse string to timestamp

**Format Patterns:**
- `"YYYY-MM-DD"` — 2024-01-15
- `"YYYY-MM-DD HH:mm:ss"` — 2024-01-15 14:30:45
- `"MM/DD/YYYY"` — 01/15/2024
- `"HH:mm:ss"` — 14:30:45

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

## Correlation Rule Syntax

XQL correlation rules are built on the same foundation as standard queries but generate alerts based on matching conditions.

### Basic Correlation Rule Structure

```
dataset = xdr_data
| filter <detection_condition>
| comp <aggregations> by <grouping_fields>
| alter severity_level = <severity_mapping>
| alter mitre_technique = <att&ck_mapping>
```

### Example: Suspicious Process Creation

```
dataset = xdr_data
| filter action_type = "process_launch" and action_process_image_name ~= "(powershell|cmd|rundll32)\.exe"
| alter severity_level = if(action_process_image_name = "powershell.exe", "high", "medium")
| comp count() as event_count by agent_hostname
| filter event_count > 5
```

### Alert Field Enhancement

```
| alter alert_name = concat("Suspicious Execution: ", action_process_image_name)
| alter alert_description = concat("Host ", agent_hostname, " executed ", action_process_image_name)
| alter mitre_technique = "T1086"
| alter mitre_tactic = "Execution"
```

### Severity Mapping

```
| alter severity = if(event_count > 100, "critical",
         if(event_count > 50, "high",
         if(event_count > 10, "medium", "low")))
```

---

## Notes and Best Practices

1. **Performance**: Use `fields` early to reduce data size before aggregations.
2. **Readability**: Break complex queries into multiple lines for clarity.
3. **Regex**: Regex patterns are case-sensitive with `~=` and `!~=`.
4. **Time Zones**: Timestamps are in UTC unless otherwise configured.
5. **Limits**: Use `limit` to avoid returning excessive results.
6. **Aliases**: Use `as` keyword to rename output columns for clarity.
7. **Null Handling**: Use `coalesce()` to provide defaults for null values.
8. **Testing**: Test queries on small time windows first before expanding scope.
