# SPL to XQL Mapping Reference

This reference provides comprehensive mappings for translating Splunk SPL queries into Cortex XSIAM XQL. Use alongside `xql-core-reference.md` for full XQL syntax details.

## Command Mapping

Core SPL commands and their XQL stage equivalents.

| SPL Command | XQL Equivalent | Notes |
|---|---|---|
| `stats` | `comp` | Aggregation stage; syntax: `comp count() by field` |
| `eval` | `alter` | Field creation/modification; combine multiple evals into one alter |
| `table` | `fields` | Select output columns |
| `where` | `filter` | Row-level filtering; XQL `filter` supports same comparison operators |
| `search` | `filter` | SPL `search` and `where` both map to XQL `filter` |
| `sort` | `sort asc/desc` | XQL requires explicit `asc` or `desc`; no implicit ordering |
| `top` | `comp count() by field \| sort desc count \| limit N` | Or use `top` stage directly: `top field N` |
| `rare` | `comp count() by field \| sort asc count \| limit N` | No direct `rare` stage; use comp + sort + limit |
| `rex` | `alter field = arrayindex(regextract(source, "pattern"), 0)` | Wrap `regextract` with `arrayindex` for scalar result |
| `rex mode=sed` | `alter field = replex(field, "pattern", "replacement")` | `replex` for regex-based replacement |
| `mvexpand` | `arrayexpand` | Expand array field into multiple rows |
| `mvjoin` | `arraystring` | Join array elements into a string |
| `mvdedup` | `arraydistinct` | Remove duplicate array elements |
| `mvcombine` | `arraycreate` | Combine field values into an array; often paired with `comp` |
| `inputlookup` | `union (dataset = lookup_name)` | Lookup must be imported as XSIAM dataset first |
| `lookup` | `join type = left (...) as lk \| filter lk.field = field` | Join against lookup dataset; see join pattern below |
| `append` | `union` | Combine results from multiple queries |
| `appendpipe` | No direct equivalent | Rewrite as separate query + union |
| `iplocation` | `iploc` | IPv4 only; populates country/region/city fields |
| `fillnull` | `replacenull` | Replaces null only, not empty strings |
| `dedup` | `dedup` | Numbers and strings only in XQL; no array/object dedup |
| `rename X as Y` | `alter Y = X` | Field rename via alter assignment |
| `head N` | `limit N` | Limit output rows |
| `tail N` | `sort desc _time \| limit N` | No direct `tail`; reverse sort + limit |
| `eventstats` | `windowcomp` | Closest equivalent; supports partitioned aggregations |
| `streamstats` | `windowcomp` | Use window functions (running_sum, row_number, etc.) |
| `transaction` | `transaction` | Max 50 fields in XQL; timeout/maxpause syntax differs |
| `bin` / `bucket` | `bin` | Time bucketing: `bin _time span = 1h` |
| `timechart` | `bin _time span = ... \| comp ... by _time` | Decompose into bin + comp stages |
| `chart` | `comp ... by field` | Decompose into comp + optional pivot |
| `join` | `join` | XQL join types: inner, left, right, cross |
| `fields + X` | `fields X` | Include fields |
| `fields - X` | Not directly supported | Use `fields` to list only desired columns |
| `addtotals` | `alter total = add(field1, field2, ...)` | Manual sum via alter |
| `convert` | `alter` with casting functions | Use `to_number()`, `to_string()`, etc. |
| `makemv` | `split` | Split string into array |
| `format` | No direct equivalent | Build output string with `format_string()` in alter |
| `outputlookup` | `target lookup = "name"` | 50 MB limit; write results to lookup dataset |
| `return` | `fields` + `limit` | Combine fields selection with limit |
| `multisearch` | `union` | Union multiple dataset queries |
| `tstats` | `preset` or `datamodel` query | Depends on accelerated data model source |

## Function Mapping

SPL eval functions and their XQL equivalents for use within `alter` stages.

### String Functions

| SPL Function | XQL Function | Notes |
|---|---|---|
| `lower(x)` | `lowercase(x)` | |
| `upper(x)` | `uppercase(x)` | |
| `len(x)` | `len(x)` | Same name |
| `substr(x, start, len)` | `substring(x, start, len)` | 1-based indexing in both |
| `replace(x, regex, replacement)` | `replex(x, regex, replacement)` | `replex` for regex; `replace` for literal |
| `replace(x, literal, replacement)` | `replace(x, literal, replacement)` | Same for literal replacement |
| `trim(x)` | `trim(x)` | Same |
| `ltrim(x)` | `ltrim(x)` | Same |
| `rtrim(x)` | `rtrim(x)` | Same |
| `split(x, delim)` | `split(x, delim)` | Same; returns array |
| `urlencode(x)` | `url_encode(x)` | |
| `printf(fmt, ...)` | `format_string(fmt, ...)` | Format string syntax may differ |

### Numeric Functions

| SPL Function | XQL Function | Notes |
|---|---|---|
| `tonumber(x)` | `to_number(x)` | |
| `tostring(x)` | `to_string(x)` | |
| `round(x, precision)` | `round(x, precision)` | Same |
| `ceil(x)` / `ceiling(x)` | `ceil(x)` | Same |
| `floor(x)` | `floor(x)` | Same |
| `abs(x)` | `abs(x)` | Same |
| `pow(x, y)` | `power(x, y)` | |
| `log(x, base)` | `log(x, base)` | Same |
| `sqrt(x)` | `sqrt(x)` | Same |

### Multi-Value (Array) Functions

| SPL Function | XQL Function | Notes |
|---|---|---|
| `mvcount(x)` | `array_length(x)` | |
| `mvindex(x, i)` | `arrayindex(x, i)` | |
| `mvfilter(expr)` | `arrayfilter(x, "@element ...")` | Uses `@element` placeholder in XQL |
| `mvjoin(x, delim)` | `arraystring(x, delim)` | |
| `mvappend(x, y)` | `arrayconcat(x, y)` | |
| `mvdedup(x)` | `arraydistinct(x)` | |
| `mvsort(x)` | `arraysort(x)` | |
| `mvfind(x, regex)` | `arrayfilter(x, "@element ~= \"regex\"")` | Returns matching elements, not index |
| `mvzip(x, y, delim)` | No direct equivalent | Manual: `arraycreate` + `arraymap` workaround |
| `split(x, delim)` | `split(x, delim)` | Same |

### Time Functions

| SPL Function | XQL Function | Notes |
|---|---|---|
| `now()` | `current_time()` | |
| `time()` | `current_time()` | |
| `strftime(time, fmt)` | `format_timestamp(time, fmt)` | Format strings differ: SPL `%Y-%m-%d` -> XQL `%Y-%m-%d` (mostly compatible) |
| `strptime(str, fmt)` | `parse_timestamp(str, fmt)` | Same format string caveat |
| `relative_time(time, spec)` | `subtract(current_time(), Nd)` | Manual time arithmetic; no relative_time() |

### Conditional Functions

| SPL Function | XQL Function | Notes |
|---|---|---|
| `if(cond, true, false)` | `if(cond, true, false)` | Same syntax |
| `case(c1, v1, c2, v2, ...)` | `if(c1, v1, c2, v2, ...)` | Nested if or chained conditions |
| `coalesce(x, y, ...)` | `coalesce(x, y, ...)` | Same |
| `null()` | `null` | Literal null value |
| `isnull(x)` | `x = null` | Use equality check in filter |
| `isnotnull(x)` | `x != null` | Use inequality check in filter |
| `nullif(x, y)` | `if(x = y, null, x)` | No direct nullif; use if |
| `validate(c1, v1, ...)` | No direct equivalent | Use chained if() |

### Cryptographic and Network Functions

| SPL Function | XQL Function | Notes |
|---|---|---|
| `md5(x)` | No equivalent | Only SHA-256 available in XQL |
| `sha256(x)` | `sha256(x)` | Same |
| `sha1(x)` | No equivalent | Only SHA-256 available |
| `cidrmatch(cidr, ip)` | `incidr(ip, cidr)` | Argument order reversed |
| `urlencode(x)` | `url_encode(x)` | |

### JSON Functions

| SPL Function | XQL Function | Notes |
|---|---|---|
| `spath(json, path)` | `json_extract_scalar(json, "$.path")` | Or arrow notation: `json->path` |
| `json_extract(json, path)` | `json_extract_scalar(json, "$.path")` | Arrow notation preferred for simple paths |
| `json_array_to_mv` | `json_extract_array(json, "$.path")` | Returns array |
| `json_set` | `object_merge(json, object_create("key", value))` | Complex; use object_create + object_merge |

## Index/Sourcetype to Dataset Mapping

Splunk index and sourcetype references must be translated to XSIAM datasets. Do NOT carry over Splunk index names as dataset names.

### Endpoint and Agent Data

| Splunk Source | XSIAM Dataset | Notes |
|---|---|---|
| `index=main sourcetype=WinEventLog` | `dataset = xdr_data` | Filter by `event_type = ENUM.EVENT_LOG` |
| `index=main sourcetype=XmlWinEventLog` | `dataset = xdr_data` | Filter by `event_type = ENUM.EVENT_LOG` |
| `index=main sourcetype=syslog` | `dataset = xdr_data` | Filter by agent_os_type for Linux/Mac |
| `index=main sourcetype=linux_secure` | `dataset = xdr_data` | Filter by `event_type` and `agent_os_type` |
| `index=wineventlog` | `dataset = xdr_data` | Filter by `event_type = ENUM.EVENT_LOG` |
| `index=sysmon` | `dataset = xdr_data` | Filter by `event_type` and relevant action fields |

### Network and Firewall

| Splunk Source | XSIAM Dataset | Notes |
|---|---|---|
| `index=firewall` | `dataset = panw_ngfw_traffic_raw` | Or `panw_ngfw_threat_raw` for threat logs |
| `index=pan_logs sourcetype=pan:traffic` | `dataset = panw_ngfw_traffic_raw` | |
| `index=pan_logs sourcetype=pan:threat` | `dataset = panw_ngfw_threat_raw` | |
| `index=pan_logs sourcetype=pan:url` | `dataset = panw_ngfw_url_raw` | |
| `index=pan_logs sourcetype=pan:globalprotect` | `dataset = panw_ngfw_globalprotect_raw` | |

### Cloud and SaaS

| Splunk Source | XSIAM Dataset | Notes |
|---|---|---|
| `index=o365` / `sourcetype=o365:management:activity` | `dataset = microsoft_office_365_raw` | |
| `index=azure` / `sourcetype=azure:aad:audit` | `dataset = microsoft_azure_ad_raw` | Audit events |
| `index=azure` / `sourcetype=azure:aad:signin` | `dataset = microsoft_azure_ad_signin_raw` | Sign-in events |
| `index=okta` / `sourcetype=OktaIM2:log` | `dataset = okta_raw` | |
| `index=cloudtrail` / `sourcetype=aws:cloudtrail` | `dataset = cloud_audit_log` | Filter by `cloud_provider = AWS` |
| `index=gcp` / `sourcetype=google:gcp:pubsub:message` | `dataset = cloud_audit_log` | Filter by `cloud_provider = GCP` |
| `index=gsuite` / `sourcetype=gapps:report:activity` | `dataset = google_workspace_raw` | |

### Generic / Fallback

| Splunk Source | XSIAM Dataset | Notes |
|---|---|---|
| `index=main` (no sourcetype) | `dataset = xdr_data` | Broad; add filters to narrow |
| Custom index name | Ask user for XSIAM dataset name | No automatic mapping available |
| `sourcetype=_json` | Depends on ingestion | Check collector configuration |

## Known Untranslatable Constructs

SPL features with no direct XQL equivalent. Flag these in translations with `// UNTRANSLATED:` comments.

| SPL Feature | Status | Workaround |
|---|---|---|
| `eval mvzip(x, y, delim)` | No equivalent | Manual array construction: use `arraycreate` + `arraymap` to pair elements |
| Custom macros (`\`macro_name\``) | Requires expansion | Ask user for macro definition; expand inline before translating |
| `lookup` with CSV file | Requires import | CSV must be imported as XSIAM lookup dataset first; then use `join` |
| `outputlookup` | Partial | `target lookup = "name"` works but has 50 MB limit |
| `map` | No equivalent | Rewrite as explicit stages or multiple queries |
| `foreach` | No equivalent | Rewrite as explicit `alter` statements for each field |
| `cluster` | No equivalent | No clustering algorithm in XQL |
| `kmeans` | No equivalent | No ML clustering in XQL |
| `predict` | No equivalent | No ML prediction in XQL; use threshold-based logic |
| `anomalydetection` | No equivalent | Use threshold-based `comp` + `filter` for anomaly detection |
| `sendalert` / `sendemail` | No equivalent | Use playbook automation for alerting |
| `collect` | No equivalent | Use `target lookup` for persisting results |
| `mcollect` | No equivalent | No metric store equivalent |
| `rest` | No equivalent | Use integration/script for API calls |
| `inputcsv` / `outputcsv` | No equivalent | Use lookup datasets for CSV data |
| `typeahead` | No equivalent | UI-specific; no query equivalent |
| Subsearch `[search ...]` | Rewrite as `join` or `union` | SPL subsearches become XQL subqueries within join/union stages |

## Translation Examples

### Example 1: Simple Stats Query

**SPL:**
```spl
index=main sourcetype=WinEventLog EventCode=4625
| stats count by src_ip, user
| where count > 10
| sort - count
| table src_ip, user, count
```

**XQL:**
```xql
// Title: Failed login attempts by source IP and user
// Translated from SPL
// Description: Counts Windows failed logon events (4625) grouped by source IP and user, filtered to those with more than 10 attempts
// Datasets: xdr_data
// Modified: 2026-04-12

dataset = xdr_data
| filter event_type = ENUM.EVENT_LOG and action_evtlog_event_id = 4625
| comp count() as count by action_remote_ip, identity_name
| filter count > 10
| sort desc count
| fields action_remote_ip, identity_name, count
```

**Mapping annotations:**
- `index=main sourcetype=WinEventLog` -> `dataset = xdr_data` + `filter event_type = ENUM.EVENT_LOG`
- `EventCode=4625` -> `filter action_evtlog_event_id = 4625`
- `stats count by` -> `comp count() as count by`
- `where count > 10` -> `filter count > 10`
- `sort - count` -> `sort desc count` (SPL `-` prefix = descending)
- `table` -> `fields`

### Example 2: Multi-Value Expansion with Rex

**SPL:**
```spl
index=proxy sourcetype=squid
| rex field=url "https?://(?<domain>[^/]+)"
| mvexpand domain
| stats count by domain
| sort - count
| head 20
```

**XQL:**
```xql
// Title: Top 20 domains from proxy logs
// Translated from SPL
// Description: Extracts domains from proxy URLs using regex, expands multi-value results, and counts by domain
// Datasets: panw_ngfw_url_raw
// Modified: 2026-04-12
// Translation notes: Splunk proxy/squid index mapped to panw_ngfw_url_raw; adjust dataset if using a different proxy source

dataset = panw_ngfw_url_raw
| alter domain = arrayindex(regextract(url, "https?://([^/]+)"), 0)
| arrayexpand domain
| comp count() as count by domain
| sort desc count
| limit 20
```

**Mapping annotations:**
- `rex field=url "pattern"` -> `alter domain = arrayindex(regextract(url, "pattern"), 0)` (arrayindex extracts scalar from regextract array)
- `mvexpand` -> `arrayexpand`
- `stats count by` -> `comp count() as count by`
- `sort - count` -> `sort desc count`
- `head 20` -> `limit 20`

### Example 3: Lookup Join with Aggregation

**SPL:**
```spl
index=main sourcetype=WinEventLog EventCode=4624
| lookup user_department_lookup user OUTPUT department
| stats count by department
| where count > 100
| sort - count
```

**XQL:**
```xql
// Title: Successful logons by department
// Translated from SPL
// Description: Joins logon events with department lookup and counts by department
// Datasets: xdr_data, user_department_lookup
// Modified: 2026-04-12
// Translation notes: Lookup CSV must be imported as XSIAM lookup dataset named "user_department_lookup"

dataset = xdr_data
| filter event_type = ENUM.EVENT_LOG and action_evtlog_event_id = 4624
| join type = left (dataset = user_department_lookup) as lk
| filter lk.user = identity_name
| comp count() as count by lk.department
| filter count > 100
| sort desc count
```

**Mapping annotations:**
- `lookup user_department_lookup user OUTPUT department` -> `join type = left (dataset = user_department_lookup) as lk | filter lk.user = identity_name`
- SPL lookup auto-matches on field name; XQL requires explicit join condition
- Lookup CSV must be pre-imported as an XSIAM lookup dataset

### Example 4: Time-Bucketed Trend Analysis

**SPL:**
```spl
index=firewall sourcetype=pan:traffic action=blocked
| bin _time span=1h
| stats count by _time, app
| sort _time
```

**XQL:**
```xql
// Title: Hourly blocked traffic by application
// Translated from SPL
// Description: Buckets blocked firewall traffic into 1-hour intervals and counts by application
// Datasets: panw_ngfw_traffic_raw
// Modified: 2026-04-12

dataset = panw_ngfw_traffic_raw
| filter action = "blocked"
| bin _time span = 1h
| comp count() as count by _time, app
| sort asc _time
```

**Mapping annotations:**
- `index=firewall sourcetype=pan:traffic` -> `dataset = panw_ngfw_traffic_raw`
- `action=blocked` -> `filter action = "blocked"`
- `bin _time span=1h` -> `bin _time span = 1h` (same syntax)
- `stats count by _time, app` -> `comp count() as count by _time, app`
- `sort _time` -> `sort asc _time` (XQL requires explicit direction)

## Quick Reference: SPL to XQL Cheat Sheet

| Pattern | SPL | XQL |
|---|---|---|
| Count by group | `stats count by field` | `comp count() as count by field` |
| Unique count | `stats dc(field) by group` | `comp count_distinct(field) by group` |
| Create field | `eval new = old * 2` | `alter new = multiply(old, 2)` |
| Rename field | `rename old as new` | `alter new = old` |
| Filter rows | `where x > 5` | `filter x > 5` |
| Regex extract | `rex "(?<name>pattern)"` | `alter name = arrayindex(regextract(field, "(pattern)"), 0)` |
| Regex match | `where match(field, "pat")` | `filter field ~= "pat"` |
| Case-insensitive match | `where lower(x)="val"` | `alter x_lower = lowercase(x) \| filter x_lower = "val"` |
| Time bucket | `bin _time span=1h` | `bin _time span = 1h` |
| Top N | `top 10 field` | `comp count() by field \| sort desc count \| limit 10` |
| Dedup | `dedup field` | `dedup field` |
| Join datasets | `join type=left field [search idx]` | `join type = left (...) as alias \| filter alias.field = field` |
| Null handling | `fillnull value=0` | `replacenull(field, 0)` |
| IP in CIDR | `cidrmatch("10.0.0.0/8", ip)` | `incidr(ip, "10.0.0.0/8")` |
