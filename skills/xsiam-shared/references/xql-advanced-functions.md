# XQL Advanced Functions Reference

This file is loaded on-demand when a query requires advanced array/JSON manipulation,
window functions, approximate aggregates, URL parsing, or advanced IP operations.

For core functions (string, math, type conversion, time/date, basic JSON, basic IP,
core aggregates), see `xql-core-reference.md`.

---

## Advanced Array Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `arraycreate` | Create array from listed values | `arraycreate(val1, val2 [, ...])` |
| `arrayconcat` | Concatenate two or more arrays | `arrayconcat(array1, array2 [, ...])` |
| `arraydistinct` | Remove duplicate elements from array | `arraydistinct(array)` |
| `arrayfilter` | Filter array elements by condition | `arrayfilter(array, "condition")` |
| `arrayindex` | Get element at specified index (0-based) | `arrayindex(array, index)` |
| `arrayindexof` | Get index of first matching element | `arrayindexof(array, value)` |
| `array_length` | Return number of elements in array | `array_length(array)` |
| `arraymap` | Apply expression to each array element | `arraymap(array, "expression")` |
| `arraymerge` | Merge multiple arrays into one | `arraymerge(array1, array2 [, ...])` |
| `arrayrange` | Create array of integers in range | `arrayrange(start, end [, step])` |
| `arraystring` | Join array elements into a delimited string | `arraystring(array, delimiter)` |
| `array_all` | Test if all elements meet condition | `array_all(array, "condition")` |
| `array_any` | Test if any element meets condition | `array_any(array, "condition")` |

### The @element Iterator Pattern

`arraymap`, `arrayfilter`, `array_any`, and `array_all` accept a string-based expression
where `@element` refers to the current array element being evaluated. The expression is
evaluated once per element.

**arraymap -- transform each element:**
```
| alter uppercased = arraymap(user_list, uppercase("@element"))
```

**arrayfilter -- keep elements matching a condition:**
```
| alter admins = arrayfilter(security_groups, "@element" contains "Admin")
```

**array_any -- boolean test (true if any element matches):**
```
| filter array_any(security_groups, "@element" ~= "^cn=(Domain|Enterprise) Admins")
```

**array_all -- boolean test (true if all elements match):**
```
| filter array_all(scores, "@element" > 80)
```

When the array contains JSON objects (from arrow expansion), use `@element` with arrow
notation to navigate into each object:

```
| alter role_names = arraymap(
    arrayfilter(params, "@element" -> name = "ROLE_NAME"),
    "@element" -> value
  )
```

### Chained Array Operations

Complex nested JSON often requires chaining multiple array functions. The typical pattern
for extracting a specific field from nested JSON arrays is:

1. Expand the outer array with `-> []`
2. Extract inner arrays with `-> key{}`
3. Merge with `arraymerge`
4. Filter to the desired key with `arrayfilter`
5. Map to the target value with `arraymap`
6. Deduplicate with `arraydistinct`

**Example -- extract ROLE_NAME values from nested Google Workspace event parameters:**
```
| alter role_names = arraydistinct(
    arraymap(
      arrayfilter(
        arraymerge(arraymap(events -> [], "@element" -> parameters{})),
        "@element" -> name = "ROLE_NAME"
      ),
      "@element" -> value
    )
  )
```

### Other Array Usage Notes

**arraycreate -- build an array from discrete fields:**
```
| alter ip_list = arraycreate(src_ip, dst_ip, nat_src_ip)
```

**arrayconcat -- combine two existing arrays:**
```
| alter all_groups = arrayconcat(source_groups, target_groups)
```

**arrayindex -- extract a scalar from a single-element array:**
```
| alter first_match = arrayindex(regextract(message, "User:\s+(\S+)"), 0)
```

**arrayindexof -- find position of a value:**
```
| alter pos = arrayindexof(status_codes, "403")
```

**arrayrange -- generate a sequence:**
```
| alter indices = arrayrange(0, 10, 2)
// produces [0, 2, 4, 6, 8]
```

**arraystring -- join array into a readable string:**
```
| alter group_list = arraystring(security_groups, ", ")
```

**arraymerge -- flatten nested arrays into one:**
```
| alter all_params = arraymerge(arraymap(events -> [], "@element" -> parameters{}))
```

**array_length -- check for empty arrays:**
```
| alter has_items = if(array_length(items) > 0, "yes", "no")
```

---

## Advanced JSON Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `json_extract_array` | Extract a JSON array | `json_extract_array(field, "$.path")` |
| `json_extract_scalar_array` | Extract a JSON array of strings | `json_extract_scalar_array(field, "$.path")` |
| `json_path_extract` | Extract using JSONPath expression | `json_path_extract(field, "$.path")` |
| `object_create` | Create JSON object from key-value pairs | `object_create("key1", val1, "key2", val2)` |
| `object_merge` | Merge two or more JSON objects | `object_merge(obj1, obj2 [, ...])` |
| `to_json_string` | Convert value to JSON string | `to_json_string(value)` |

Note: `json_extract` and `json_extract_scalar` are in the core reference. The functions
here handle array extraction, JSONPath, and object construction.

### JSON Path Notation

All JSON functions use `$.path` dot-notation. Field names within the JSON are
**case-sensitive** -- `"$.userName"` and `"$.username"` are different paths.

**json_extract_array -- get an array from JSON:**
```
| alter items = json_extract_array(raw_data, "$.results.items")
| arrayexpand items
```

**json_extract_scalar_array -- get an array of scalar strings:**
```
| alter emails = json_extract_scalar_array(user_data, "$.contacts.emails")
```

**json_path_extract -- JSONPath with deeper navigation:**
```
| alter deep_value = json_path_extract(payload, "$.data.nested.field")
```

**object_create -- build a JSON object from fields:**
```
| alter summary = object_create("user", user_name, "action", action_type, "time", _time)
```

**object_merge -- combine JSON objects:**
```
| alter enriched = object_merge(alert_data, geo_data, user_context)
```

**to_json_string -- serialize a value for storage or output:**
```
| alter payload_str = to_json_string(arraycreate(src_ip, dst_ip, user_name))
```

### Arrow Notation vs json_extract

Both access JSON data, but they serve different purposes:

- **Arrow notation** (`field -> key`) -- use when navigating JSON fields inline,
  especially with array expansion (`-> []`) and object extraction (`-> key{}`).
  Works naturally with `arraymap`/`arrayfilter` and `@element`.
- **json_extract / json_extract_scalar** -- use when you have a `$.path` string,
  need type preservation, or are extracting from deeply nested structures where
  arrow chaining becomes unwieldy.

---

## Window Functions (windowcomp)

Window functions operate within the `windowcomp` stage, applying calculations across
ordered, optionally partitioned result sets without collapsing rows (unlike `comp`).

| Function | Description | Syntax |
|----------|-------------|--------|
| `rank` | Assign rank within partition (ties get same rank, gaps after ties) | `rank()` |
| `row_number` | Assign sequential number within partition (no gaps) | `row_number()` |
| `first_value` | First value in window frame | `first_value(field)` |
| `last_value` | Last value in window frame | `last_value(field)` |
| `lag` | Value from a previous row in partition | `lag(field [, offset [, default]])` |

Standard aggregate functions (`sum`, `avg`, `count`, `min`, `max`, etc.) can also be
used within `windowcomp` as running calculations over the window.

### windowcomp Syntax

```
| windowcomp <function>(<field>) as <alias> [partition by <field>] [order by <field> [asc|desc]]
```

- **partition by** -- divides results into groups (like `GROUP BY` but without collapsing rows)
- **order by** -- defines the row ordering within each partition
- Both clauses are optional but almost always used in practice

### rank and row_number

`rank()` assigns the same rank to tied values and leaves gaps (1, 1, 3). `row_number()`
assigns strictly sequential numbers with no gaps (1, 2, 3).

**Example -- rank users by event count within each department:**
```
dataset = xdr_data
| comp count() as event_count by user_name, department
| windowcomp rank() as user_rank partition by department order by event_count desc
| filter user_rank <= 5
```

**Example -- number events sequentially per host:**
```
dataset = xdr_data
| windowcomp row_number() as seq partition by agent_hostname order by _time asc
```

### first_value and last_value

Retrieve the first or last value in the window frame for a given field.

**Example -- get each user's first and last login time per day:**
```
dataset = xdr_data
| filter event_type = ENUM.STORY and auth_outcome = "SUCCESS"
| alter event_day = format_timestamp(_time, "YYYY-MM-DD")
| windowcomp first_value(_time) as first_login partition by user_name, event_day order by _time asc
| windowcomp last_value(_time) as last_login partition by user_name, event_day order by _time asc
```

### lag

Access a value from a previous row. Useful for calculating deltas and detecting gaps.

```
lag(field)                    -- previous row's value (offset = 1)
lag(field, offset)            -- value from N rows back
lag(field, offset, default)   -- value from N rows back, with default if no prior row
```

**Example -- calculate time between consecutive events per host:**
```
dataset = xdr_data
| windowcomp lag(_time, 1) as prev_time partition by agent_hostname order by _time asc
| alter time_gap_seconds = timestamp_diff(_time, prev_time, "SECOND")
| filter time_gap_seconds > 3600
```

**Example -- detect login from a new IP (different from previous):**
```
dataset = xdr_data
| filter event_type = ENUM.STORY and auth_outcome = "SUCCESS"
| windowcomp lag(source_ip, 1, "") as prev_ip partition by user_name order by _time asc
| filter source_ip != prev_ip and prev_ip != ""
```

### Running Aggregates in windowcomp

Standard aggregate functions become running (cumulative) calculations within `windowcomp`.

**Example -- running total of bytes transferred per source IP:**
```
dataset = panw_ngfw_traffic_raw
| windowcomp sum(bytes_sent) as running_bytes partition by src order by _time asc
```

**Example -- running average of event counts:**
```
dataset = xdr_data
| comp count() as event_count by agent_hostname, _time
| windowcomp avg(event_count) as running_avg partition by agent_hostname order by _time asc
```

---

## Approximate Aggregate Functions

Used within the `comp` stage. These trade precision for performance on very large datasets.

| Function | Description | Syntax |
|----------|-------------|--------|
| `approx_count` | Approximate distinct count (HyperLogLog) | `approx_count(field)` |
| `approx_quantiles` | Approximate quantile values | `approx_quantiles(field, num_quantiles)` |
| `approx_top` | Approximate top frequent values | `approx_top(field, count)` |

### Performance Trade-offs

- **approx_count** uses HyperLogLog, providing ~2% error for cardinality estimation.
  Use instead of `count_distinct` when exact counts are unnecessary and the dataset
  is large (millions of rows). Significantly faster on high-cardinality fields.

- **approx_quantiles** divides the distribution into N equal-sized buckets. Useful for
  finding percentiles (e.g., P50, P95, P99) without sorting the entire dataset.

- **approx_top** returns the approximately most frequent values. Faster than
  `comp count() by field | sort desc | limit N` on very large datasets, but results
  are approximate.

**Example -- approximate unique user count per department:**
```
dataset = xdr_data
| comp approx_count(user_name) as approx_users by department
| sort approx_users desc
```

**Example -- find P50 and P99 response times:**
```
dataset = panw_ngfw_traffic_raw
| comp approx_quantiles(response_time_ms, 100) as percentiles by app_name
```

**Example -- approximate top 10 most-seen processes:**
```
dataset = xdr_data
| comp approx_top(action_process_image_name, 10) as top_processes
```

---

## URL Functions

| Function | Description | Syntax |
|----------|-------------|--------|
| `extract_url_host` | Extract hostname from URL | `extract_url_host(url)` |
| `extract_url_pub_suffix` | Extract public suffix (TLD) from URL | `extract_url_pub_suffix(url)` |
| `extract_url_registered_domain` | Extract registered domain from URL | `extract_url_registered_domain(url)` |

### Usage Notes

These functions parse URLs and return the relevant component. They handle common URL
formats including those with ports, paths, and query parameters.

| Input URL | `extract_url_host` | `extract_url_pub_suffix` | `extract_url_registered_domain` |
|-----------|--------------------|--------------------------|---------------------------------|
| `https://mail.google.com/inbox?a=1` | `mail.google.com` | `com` | `google.com` |
| `https://app.example.co.uk:8443/api` | `app.example.co.uk` | `co.uk` | `example.co.uk` |
| `http://192.168.1.1/admin` | `192.168.1.1` | (empty) | (empty) |

**Example -- group traffic by registered domain:**
```
dataset = panw_ngfw_url_raw
| alter domain = extract_url_registered_domain(url)
| comp sum(bytes_sent) as total_bytes by domain
| sort total_bytes desc
| limit 20
```

**Example -- detect traffic to unusual TLDs:**
```
dataset = panw_ngfw_url_raw
| alter tld = extract_url_pub_suffix(url)
| filter tld in ("tk", "ml", "ga", "cf", "gq", "xyz", "top", "buzz")
| comp count() as hits by tld, extract_url_registered_domain(url)
| sort hits desc
```

**Example -- extract hostnames for threat hunting:**
```
dataset = panw_ngfw_url_raw
| alter host = extract_url_host(url)
| filter host contains $ioc_domain
| fields _time, src, dst, url, host
```

---

## Advanced IP Functions

The core reference covers `incidr`, `is_ipv4`, `is_ipv6`, and `is_known_private_ipv4`.
These functions extend IP handling to IPv6 CIDR matching and integer conversion.

| Function | Description | Syntax |
|----------|-------------|--------|
| `incidr6` | Test if IPv6 address is in CIDR range | `incidr6(ip, "cidr")` |
| `ip_to_int` | Convert IP address string to integer | `ip_to_int(ip_string)` |
| `ip_in_int` | Convert integer to IP address string | `ip_in_int(integer)` |
| `is_known_private_ipv6` | Test if IPv6 is private or link-local | `is_known_private_ipv6(ip)` |

### Usage Notes

**incidr6 -- IPv6 CIDR matching:**
```
| filter incidr6(source_ipv6, "2001:db8::/32")
| filter incidr6(source_ipv6, "fe80::/10") = false
```

**ip_to_int / ip_in_int -- integer conversion for range comparisons:**
```
| alter ip_int = ip_to_int(source_ip)
| filter ip_int >= ip_to_int("10.0.0.1") and ip_int <= ip_to_int("10.0.0.254")
```

```
| alter ip_addr = ip_in_int(ip_integer_field)
```

**is_known_private_ipv6 -- filter out private IPv6 addresses:**
```
| filter is_known_private_ipv6(source_ipv6) = false
```

Integer conversion is useful when you need to check if an IP falls within an arbitrary
range (not a CIDR block), or when storing IPs in a compact numeric format.

---

## Regex Deep-Dive

The core reference covers basic `regextract` and `regexcapture` syntax. This section
covers advanced patterns and practical techniques.

### regexcapture -- Named Group Extraction

`regexcapture` uses `(?P<name>pattern)` syntax to produce named fields directly in the
result set. Each named group becomes a new column.

```
| alter regexcapture(syslog_message, "user=(?P<user>\w+)\s+action=(?P<action>\w+)\s+src=(?P<src_ip>\d+\.\d+\.\d+\.\d+)")
```

This produces three new fields: `user`, `action`, and `src_ip`.

**When to use regexcapture vs regextract:**
- `regexcapture` -- when you need multiple named fields from a single string (e.g., parsing syslog, CEF, key-value pairs)
- `regextract` + `arrayindex` -- when you need a single scalar value from one capture group

### regextract -- Scalar Extraction Pattern

`regextract` returns an array. The standard pattern to get a single scalar value is:

```
| alter value = arrayindex(regextract(field, "pattern_with_(capture_group)"), 0)
```

**Example -- extract username from Windows event log:**
```
| alter user = arrayindex(regextract(action_evtlog_message, "Account Name:\s+(\S+)"), 0)
```

**Example -- extract CN from distinguished name:**
```
| alter cn = arrayindex(regextract(xdm.event.description, "[Cc][Nn]=([a-zA-Z0-9_-]+),"), 0)
```

### replex -- Regex-Based Replacement

`replex` replaces all matches of a regex pattern with a replacement string. Unlike
`replace` (which matches literal substrings), `replex` supports full regex syntax.

```
replex(string, regex_pattern, replacement)
```

**Example -- mask credit card numbers:**
```
| alter masked = replex(message, "\d{12}(\d{4})", "************$1")
```

**Example -- normalize whitespace:**
```
| alter cleaned = replex(raw_text, "\s+", " ")
```

### wildcard_match -- Simple Pattern Matching

`wildcard_match` tests if a string matches a glob-style pattern using `*` (any characters)
and `?` (single character). Simpler and more readable than regex for basic patterns.

```
wildcard_match(string, pattern)
```

**Example:**
```
| filter wildcard_match(action_file_path, "C:\\Users\\*\\AppData\\*\\*.exe")
```

**Example:**
```
| filter wildcard_match(domain, "*.evil-domain.com")
```

### String Quoting for Regex

- **Double quotes** (`"pattern"`) -- literal strings; wildcards (`*`) behave as XQL wildcards, not regex
- **Triple double quotes** (`"""pattern"""`) -- regex-style strings; escape sequences are processed (`\n`, `\t`, `\\`)

Use triple quotes when your regex pattern contains backslashes that would otherwise need double-escaping:

```
// Without triple quotes -- must double-escape backslashes
| filter path ~= "C:\\\\Users\\\\.*\\\\Desktop"

// With triple quotes -- single backslash works
| filter path ~= """C:\Users\.*\Desktop"""
```

### Operator Behavior Reminder

- `~=` is **case-sensitive** regex match
- `!~=` is **case-sensitive** regex negated match
- `contains` is **case-insensitive** substring match
- `not contains` is **case-insensitive** substring negated match

Choose `contains` for simple case-insensitive substring checks. Use `~=` only when you
need regex features (anchors, character classes, alternation, quantifiers).
