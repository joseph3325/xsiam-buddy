# XQL Query Cookbook

Production-ready query examples organized by use case. Each example includes
a header comment block, the query, and an annotation of the pattern demonstrated.

---

## Threat Hunting

### IOC Search (IP/Domain/Hash)

**Pattern:** Search multiple hash and network fields for a single IOC value using `$variable` substitution
**Datasets:** xdr_data

```xql
// Title: IOC Search (IP/Domain/Hash)
// Description: Searches process, file, network, and DNS fields for a given IOC across 30 days
// Author: xsiam-buddy
// Datasets: xdr_data
// Modified: 2026-04-12

config timeframe = 30d
| dataset = xdr_data
| filter action_process_image_sha256 = $ioc
       or action_file_sha256 = $ioc
       or action_module_sha256 = $ioc
       or dns_query contains $ioc
       or url contains $ioc
       or action_remote_ip = $ioc
| fields _time, agent_hostname, event_type, event_sub_type, actor_effective_username,
         action_process_image_name, action_process_image_path,
         action_process_image_sha256, action_file_path, dns_query, url,
         action_remote_ip
| sort desc _time
| limit 1000
```

**Annotation:** Uses `$variable` substitution so the same query works for hashes, IPs, and domains. The `config timeframe = 30d` preamble extends the search window beyond the default. Multiple `or` conditions cover all common IOC field locations in XDR endpoint data.

---

### File Hash Reputation Lookup

**Pattern:** Find all process executions matching a specific hash with `config timeframe` and `view column order`
**Datasets:** xdr_data

```xql
// Title: Process Executions of Hash in Last Month
// Description: Finds all executions matching a given hash across SHA256 and MD5 fields over 30 days
// Author: xsiam-buddy
// Datasets: xdr_data
// Modified: 2026-04-12

config timeframe = 30d
| dataset = xdr_data
| filter action_process_image_sha256 = $hash
       or action_process_image_md5 = $hash
       or action_file_sha256 = $hash
       or action_module_sha256 = $hash
| fields _time, agent_hostname, event_type, event_sub_type, actor_effective_username,
         action_process_image_path, action_process_image_sha256, action_process_image_md5,
         actor_process_image_name, action_file_path
| view column order = populated
```

**Annotation:** `config timeframe = 30d` sets a 30-day search window; note the pipe before `dataset` is required when `config` is present. `view column order = populated` auto-hides empty columns in the UI, making results easier to read when not all hash fields match.

---

### Lateral Movement Detection

**Pattern:** Detect a single user authenticating to many distinct hosts in a short window using `comp count_distinct`
**Datasets:** xdr_data

```xql
// Title: Lateral Movement Detection
// Description: Detects users authenticating to 5+ unique hosts within the query window — potential lateral movement
// Author: xsiam-buddy
// Datasets: xdr_data
// Modified: 2026-04-12

dataset = xdr_data
| filter event_type = ENUM.EVENT_LOG and action_evtlog_event_id = 4624
| alter target_user = arrayindex(regextract(action_evtlog_message, "Account Name:.*?(\w.*)\r\n"), 0),
       target_host = agent_hostname
| filter target_user != null and target_user not contains "$"
| comp count_distinct(target_host) as unique_hosts by target_user
| filter unique_hosts >= 5
| sort desc unique_hosts
```

**Annotation:** Windows event 4624 is a successful logon. Filtering out accounts ending in `$` excludes machine accounts. `count_distinct` on the host field surfaces users authenticating to an unusually high number of endpoints, a hallmark of lateral movement.

---

## Authentication

### Failed Login Spike Detection

**Pattern:** Time-bucket aggregation with `to_string` and `format_timestamp` for hourly spike analysis
**Datasets:** xdr_data

```xql
// Title: Failed Login Spike Detection
// Description: Aggregates failed logins into hourly buckets to identify authentication spikes
// Author: xsiam-buddy
// Datasets: xdr_data
// Modified: 2026-04-12

dataset = xdr_data
| filter event_type = ENUM.EVENT_LOG and action_evtlog_event_id = 4625
| alter hour_bucket = format_timestamp(_time, "%Y-%m-%d %H:00")
| comp count() as failed_logins by hour_bucket
| sort asc hour_bucket
```

**Annotation:** `format_timestamp` truncates each event to its hour, creating time buckets for aggregation. Sorting ascending by time produces a chronological view that makes spikes visually obvious in the XSIAM results table.

---

### Password Spray Detection

**Pattern:** `regextract` from raw event log messages with `count_distinct` aggregation
**Datasets:** xdr_data

```xql
// Title: Multiple Users Failing to Login from the Same Location
// Description: Detects potential password spraying — 5+ unique users failing to authenticate from the same host
// Author: xsiam-buddy
// Datasets: xdr_data
// Modified: 2026-04-12

dataset = xdr_data
// Windows event log 4625 = failed logon
| filter event_type = ENUM.EVENT_LOG and action_evtlog_event_id = 4625 and agent_hostname != null  // ENUM.EVENT_LOG is a typed constant, not a string — do not quote it
// Extract username and workstation from the event log message
| alter User_Name = arrayindex(regextract(action_evtlog_message, "Account For Which Logon Failed:\r\n.*\r\n.*Account Name:.*?(\w.*)\r\n"), 0),
       Host_Name = arrayindex(regextract(action_evtlog_message, "Workstation Name:.*?(\w.*)\r\n"), 0)
| comp count_distinct(User_Name) as Counter by Host_Name
| filter Counter >= 5 and Host_Name != null
| sort desc Counter
```

**Annotation:** `ENUM.EVENT_LOG` is a typed constant, not a string. `regextract()` with `arrayindex(..., 0)` pulls the first capture group match. Counting distinct usernames per source host identifies password spray patterns where an attacker tries many accounts from one location.

---

### Multi-Location Authentication

**Pattern:** `ENUM.STORY` events with inline field aliasing and `count_distinct`
**Datasets:** xdr_data

```xql
// Title: Top 10 Users Failing to Authenticate from Multiple Locations
// Description: Finds users with the most distinct source IPs for failed auth — useful for detecting account compromise
// Author: xsiam-buddy
// Datasets: xdr_data (STORY events — requires Azure, Okta, or Ping auth data)
// Modified: 2026-04-12

dataset = xdr_data
| filter event_type = ENUM.STORY and auth_identity_display_name != null and auth_outcome != "SUCCESS"
| fields auth_identity_display_name as user, auth_identity as email, auth_client as source
| comp count_distinct(source) as counter by user, email
| sort desc counter
| limit 10
```

**Annotation:** `ENUM.STORY` accesses correlated identity events from cloud identity providers (Azure AD, Okta, Ping). Inline `as` aliasing in `fields` works in the XSIAM XQL dialect. Counting distinct sources per user reveals compromised accounts being accessed from multiple locations.

---

## Cloud and Identity

### Azure AD Administrator Role Assigned

**Pattern:** `incidr() = false` as boolean negation; `datamodel dataset` with raw field aliasing
**Datasets:** msft_azure_ad_raw

```xql
// Title: Microsoft Azure - PowerShell Application Login Activity
// Description: Detects successful PowerShell application logins originating outside trusted IP ranges
// Author: xsiam-buddy
// Datasets: msft_azure_ad_raw (via datamodel)
// Modified: 2026-04-12

datamodel dataset = msft_azure_ad_raw
// Pull all XDM fields plus the raw ipAddress field not surfaced in the datamodel
| fields *, msft_azure_ad_raw.ipAddress as ipaddress
| filter xdm.source.application.name = "Azure Active Directory PowerShell"
  and xdm.event.outcome = "SUCCESS"
  and incidr(ipaddress, "104.204.240.0/22", "144.90.0.0/16") = false
| fields xdm.event.id, xdm.source.application.name, xdm.event.operation_sub_type,
         xdm.event.outcome, xdm.event.outcome_reason, xdm.session_context_id,
         xdm.source.user.identifier, xdm.source.user.username,
         xdm.source.user.sam_account_name, xdm.source.host.ipv4_addresses,
         xdm.target.resource.id, xdm.target.resource.name, xdm.alert.description,
         xdm.network.http.browser, xdm.source.host.os_family
```

**Annotation:** `fields *` with `dataset_name.field as alias` accesses raw fields not mapped to XDM within a `datamodel dataset` query. `incidr() = false` excludes IPs within trusted CIDR ranges — the function-call form is required for boolean comparison.

---

### Google Workspace Administrator Role Assigned

**Pattern:** Arrow notation (`->`), `arraymap`, `arrayfilter`, `arraydistinct`, `arraymerge` for nested JSON extraction
**Datasets:** google_workspace_admin_console_raw

```xql
// Title: Google Workspace - Administrator Role Assigned
// Description: Detects when a Google Administrator role is assigned to a user
// Author: xsiam-buddy
// Datasets: google_workspace_admin_console_raw
// Modified: 2026-04-12

dataset = google_workspace_admin_console_raw
// Filter on user role assignment events
| filter events ~= "\"name\":\"(ASSIGN_ROLE|GRANT_ADMIN_PRIVILEGE)\""
// JSON extractions via dynamic array looping
| alter actor_user = actor -> email,
       category = arraydistinct(arraymap(arrayfilter(events -> [], "@element" -> name in ("ASSIGN_ROLE", "GRANT_ADMIN_PRIVILEGE")), "@element" -> name)),
       category_sub_type = arraydistinct(arraymap(events -> [], "@element" -> type)),
       role_name = arraydistinct(arraymap(arrayfilter(arraymerge(arraymap(events -> [], "@element" -> parameters{})), "@element" -> name = "ROLE_NAME"), "@element" -> value)),
       target_user = to_string(arraydistinct(arraymap(arrayfilter(arraymerge(arraymap(events -> [], "@element" -> parameters{})), "@element" -> name = "USER_EMAIL"), "@element" -> value))),
       role_assignment_id = to_string(arraydistinct(arraymap(arrayfilter(arraymerge(arraymap(events -> [], "@element" -> parameters{})), "@element" -> name = "ROLE_ASSIGNMENT_ID"), "@element" -> value)))
// Filter to actual admin role assignments only
| filter role_name contains "admin"
| fields _time, category, category_sub_type, role_name, role_assignment_id, target_user, actor_user, ipAddress  // ipAddress is a raw camelCase field from the Google source — not an XDM mapping
```

**Annotation:** Demonstrates deeply nested JSON extraction using chained `arraymap` and `arrayfilter` with `@element` loop variable. `arraymerge` flattens nested arrays before filtering. The `->` arrow notation navigates into JSON objects and `-> []` accesses array elements.

---

### Google Workspace User Suspended

**Pattern:** Simple `->` arrow extraction from top-level JSON blob fields
**Datasets:** google_workspace_alerts_raw

```xql
// Title: Google Workspace - User Suspended
// Description: Surfaces user suspended alerts from the Google Workspace alerts dataset
// Author: xsiam-buddy
// Datasets: google_workspace_alerts_raw
// Modified: 2026-04-12

dataset = google_workspace_alerts_raw
| filter type = "User suspended"
// Extract email and severity from nested JSON blobs
| alter email    = data -> email,
       severity  = metadata -> severity
| fields _time, alertId, type, source, severity, email, securityInvestigationToolLink, data, metadata
```

**Annotation:** The `->` arrow notation extracts scalar values from top-level JSON fields (`data`, `metadata`). This is the simplest form of JSON traversal in XQL — no array operations needed when the JSON structure is flat.

---

### Abnormal Security Account Takeover

**Pattern:** Conditional field override with `if()` based on domain matching
**Datasets:** abnormal_security_generic_alert_raw

```xql
// Title: Abnormal - Account Takeover Sign In
// Description: Generates account takeover alerts from Abnormal Security; elevates severity for specific domains
// Author: xsiam-buddy
// Datasets: abnormal_security_generic_alert_raw
// Modified: 2026-04-12

dataset = abnormal_security_generic_alert_raw
| filter severity = "Account Takeover"
  and case_status = "Action Required"
| alter severity_level = if(affectedEmployee contains "@yourdomain.com", "MEDIUM", severity_level)
| fields caseId, case_status, severity, severity_level, analysis, genai_summary, affectedEmployee, firstObserved
```

**Annotation:** `if()` conditionally overwrites the raw `severity_level` field — appropriate here because it is a source field, not YAML metadata. The `contains` operator performs substring matching on the email domain.

---

### Okta MFA Bypass Detection

**Pattern:** Filtering STORY events for MFA bypass indicators from Okta
**Datasets:** xdr_data

```xql
// Title: Okta MFA Bypass Detection
// Description: Detects successful authentications where MFA was bypassed or not enforced in Okta
// Author: xsiam-buddy
// Datasets: xdr_data (STORY events — requires Okta data ingestion)
// Modified: 2026-04-12

dataset = xdr_data
| filter event_type = ENUM.STORY
  and auth_outcome = "SUCCESS"
  and auth_mfa_is_activated = "false"
  and auth_identity_provider contains "okta"
| fields _time, auth_identity_display_name, auth_identity, auth_client,
         auth_mfa_is_activated, auth_identity_provider, auth_app_name
| sort desc _time
| limit 100
```

**Annotation:** `ENUM.STORY` events aggregate identity data from cloud providers including Okta. Filtering on `auth_mfa_is_activated = "false"` with `auth_outcome = "SUCCESS"` catches successful logins that bypassed multi-factor authentication — a key indicator of compromised credentials or policy misconfiguration.

---

## Network

### Data Exfiltration Volume Analysis

**Pattern:** Aggregation with `comp sum()` and `sort` to surface top talkers by bytes sent
**Datasets:** xdr_data

```xql
// Title: Data Exfiltration Volume Analysis
// Description: Identifies endpoints sending the most data externally — potential data exfiltration
// Author: xsiam-buddy
// Datasets: xdr_data
// Modified: 2026-04-12

dataset = xdr_data
| filter event_type = ENUM.NETWORK_CONNECTION
  and action_remote_ip != null
  and incidr(action_remote_ip, "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16") = false
| comp sum(action_total_upload) as total_bytes_sent,
       count() as connection_count
  by agent_hostname, action_remote_ip
| filter total_bytes_sent > 104857600  // 100 MB threshold
| sort desc total_bytes_sent
| limit 50
```

**Annotation:** `ENUM.NETWORK_CONNECTION` targets network connection events. `incidr() = false` with RFC 1918 ranges filters to external-only destinations. `comp sum()` aggregates upload bytes per host-destination pair. The 100 MB threshold surfaces anomalous outbound data volumes.

---

### Firewall Block Analysis

**Pattern:** `iploc()` for geolocation enrichment combined with aggregation
**Datasets:** xdr_data

```xql
// Title: Firewall Block Analysis
// Description: Surfaces top blocked source IPs enriched with geolocation data
// Author: xsiam-buddy
// Datasets: xdr_data
// Modified: 2026-04-12

dataset = xdr_data
| filter event_type = ENUM.NETWORK_CONNECTION
  and action_type = "block"
| alter src_country = iploc(action_remote_ip)
| comp count() as block_count by action_remote_ip, src_country
| sort desc block_count
| limit 25
```

**Annotation:** `iploc()` enriches IP addresses with geographic location data. Aggregating blocked connections by source IP and country reveals the geographic distribution of blocked traffic and highlights persistent attackers.

---

## Enrichment

### Endpoint-to-Firewall Join

**Pattern:** `join type = left` to correlate endpoint data with network events
**Datasets:** xdr_data, endpoints

```xql
// Title: Endpoint-to-Firewall Join
// Description: Enriches network connection events with endpoint inventory metadata
// Author: xsiam-buddy
// Datasets: xdr_data, endpoints
// Modified: 2026-04-12

dataset = xdr_data
| filter event_type = ENUM.NETWORK_CONNECTION and action_remote_ip != null
| join type = left (
    dataset = endpoints
    | fields ip_address, endpoint_name, operating_system, endpoint_status, domain
  ) as ep agent_ip = ep.ip_address
| fields _time, agent_hostname, ep.endpoint_name, ep.operating_system,
         action_remote_ip, action_remote_port, action_total_upload, action_total_download
| sort desc _time
| limit 100
```

**Annotation:** A left join preserves all network events even when no endpoint match exists. The join subquery selects only needed fields from `endpoints` to reduce join overhead. The alias `ep` namespaces joined fields as `ep.field_name`.

---

### Domain Admin Authentication to Non-DC Assets

**Pattern:** Chained multi-join (three sequential `join` stages) with `pan_dss_raw` CIE data and `endpoints` inventory
**Datasets:** microsoft_windows_raw, pan_dss_raw, endpoints

```xql
// Title: Microsoft Windows - Domain Admin Authentication to Tier 1/2 Asset
// Description: Detects domain administrator credentials used on assets outside Tier 0 (non-Domain Controllers)
// Author: xsiam-buddy
// Datasets: microsoft_windows_raw, pan_dss_raw, endpoints
// Modified: 2026-04-12

dataset = microsoft_windows_raw
| fields _time, host_name, provider_name, event_id, message, event_data, _collector_type
| filter provider_name = "Microsoft-Windows-Security-Auditing" and event_id = 4624
// Extract auth details from the structured event_data JSON field
| alter source_username    = event_data -> TargetUserName,
       source_domain       = event_data -> TargetDomainName,
       session_logon_type  = event_data -> LogonType
// Exclude network and unlock logon types
| filter session_logon_type not in ("3", "7")
// Translate logon type number to human-readable label
| alter session_logon_type = if(
    session_logon_type = "2",  "Interactive",
    session_logon_type = "3",  "Network",
    session_logon_type = "4",  "Batch",
    session_logon_type = "5",  "Service",
    session_logon_type = "7",  "Unlock",
    session_logon_type = "8",  "Network Cleartext",
    session_logon_type = "9",  "New Credentials (RunAs)",
    session_logon_type = "10", "Remote Interactive",
    session_logon_type = "11", "Cached Interactive",
    session_logon_type
  )
// Join CIE — keep only users who are members of Domain Admins or Enterprise Admins
| join type = inner (
    dataset = pan_dss_raw
    | filter source = "ad" and type = "user"
      and array_any(security_groups, "@element" ~= "^cn=(Domain|Enterprise) Admins")
    | fields sam_account_name, netbios
  ) as ad_users source_username = ad_users.sam_account_name and source_domain = ad_users.netbios
// Join CIE computer records — used to exclude Domain Controllers via OU check
| join type = inner (
    dataset = pan_dss_raw
    | filter source = "ad" and type = "computer"
    | fields name, dns_host_name, dn, ou
  ) as ad_computers host_name = ad_computers.name or host_name = ad_computers.dns_host_name
| filter arrayindex(ou, 0) != "Domain Controllers"
| alter source_ip       = event_data -> IpAddress,
       source_port      = event_data -> IpPort,
       source_device    = event_data -> WorkstationName,
       session_logon_id = event_data -> TargetLogonId
// Join datamodel to add parent process from the same event
| join type = left (
    datamodel dataset = microsoft_windows_raw
    | fields _id, xdm.source.process.name as parent_process
  ) as process _id = process._id
| dedup _id
// Severity: MEDIUM if host has XDR agent present in endpoints dataset, else HIGH
| alter issue_severity = if(
    lowercase(host_name) in (
      dataset = endpoints
      | alter endpoint_name = lowercase(endpoint_name)
      | fields endpoint_name
    ),
    "MEDIUM", "HIGH"
  )
| fields _time, issue_severity, host_name, event_id, source_username, source_domain,
         source_ip, source_port, source_device, session_logon_type, session_logon_id,
         dn as target_dn, provider_name, parent_process, message, _collector_type
```

**Annotation:** Three sequential `join` stages: (1) inner join to `pan_dss_raw` user records to filter to Domain/Enterprise Admins via `array_any`; (2) inner join to `pan_dss_raw` computer records to resolve AD OU; (3) left join to the datamodel for parent process name. The inline dataset lookup inside `if()` dynamically checks the `endpoints` dataset for agent presence to set severity.

---

### CIE User/Group Enrichment (pan_dss_raw)

**Pattern:** `arrayexpand` with inline dataset lookup for AD group membership monitoring
**Datasets:** microsoft_windows_raw, pan_dss_raw (via ad_group_lookup)

```xql
// Title: Windows - AD Group Monitoring (On-Premise)
// Description: Monitors membership changes to specific VIP AD groups using a lookup table
// Author: xsiam-buddy
// Datasets: microsoft_windows_raw (via datamodel)
// Modified: 2026-04-12

datamodel dataset = microsoft_windows_raw
// Windows events: 4728 = added to global group, 4732 = added to local group, 4756 = added to universal group
| filter xdm.event.id in ("4728", "4732", "4756")
  and xdm.source.user.sam_account_name not contains "$"
// Expand groups array so each group can be compared individually
| arrayexpand xdm.target.user.groups
// Compare against lookup table (case-insensitive)
| filter lowercase(xdm.target.user.groups) in (dataset = ad_group_lookup | alter group_name = lowercase(group_name) | fields group_name)
// Extract target username from event description (not available in default XDM mappings)
| alter xdm.target.user.username = arrayindex(regextract(xdm.event.description, "[Cc][Nn]=([a-zA-Z0-9_-]+),"), 0)
| fields _time, xdm.event.id, xdm.event.description, xdm.event.outcome,
         xdm.source.host.hostname, xdm.source.user.domain, xdm.source.user.username,
         xdm.target.user.username, xdm.target.user.ou, xdm.target.user.groups
```

**Annotation:** `arrayexpand` converts a multi-valued field into separate rows so each value can be filtered individually. The inline dataset lookup (`in (dataset = ad_group_lookup | ...)`) dynamically fetches group names from a lookup table. `lowercase()` on both sides ensures case-insensitive comparison.

---

### Canary Alert with Endpoint and AD Enrichment

**Pattern:** `arrayexpand` + multi-join chain with `preset = ad_users` for CIE enrichment
**Datasets:** thinkst_canary_generic_alert_raw, endpoints, pan_dss_raw

```xql
// Title: Canary - Alert Notification
// Description: Raises alerts from Thinkst Canary, enriched with endpoint inventory and Active Directory user identity
// Author: xsiam-buddy
// Datasets: thinkst_canary_generic_alert_raw, endpoints, pan_dss_raw (via ad_users preset)
// Modified: 2026-04-12

dataset = thinkst_canary_generic_alert_raw
// Tune out noisy disconnection events
| filter summary not in ("Multiple Canaries Disconnected")
// Extract fields from the alert JSON blob
| alter ip_address = _alert_data -> sourceip,
       canary     = _alert_data -> devicename,
       _raw_json  = _alert_data -> raw_json
// Join endpoint dataset to enrich on IP match; extract short username from full UPN
| join type = left (
    dataset = endpoints
    | arrayexpand ip_address
    | alter user = lowercase(arrayindex(regextract(user, "\\(.+)$"), 0))
    | fields ip_address, endpoint_name, operating_system, mac_address, domain, user
  ) as jn ip_address = jn.ip_address
// Join CIE Active Directory to enrich user identity from the last-logged-in user on the endpoint
| join type = left (
    preset = ad_users
    | alter sam_account_name = lowercase(sam_account_name)
    | fields display_name, sam_account_name, distinguished_name, security_identifier,
             email, security_group_list, department
  ) as jn2 user = jn2.sam_account_name
// Dedup on XSIAM system event ID to remove duplicate ingestion
| dedup _id
| fields _alert_data, summary, description, ip_address, endpoint_name, operating_system,
         mac_address, domain, display_name, sam_account_name, distinguished_name,
         security_identifier, email, security_group_list, department, _raw_json
```

**Annotation:** Two sequential `join` stages: first to the `endpoints` dataset (with `arrayexpand` inside the subquery to handle multi-valued IP fields), then to `preset = ad_users` for Active Directory identity enrichment. `_alert_data ->` extracts fields from the raw alert JSON blob. `dedup _id` removes duplicate ingestion events.

---

## Advanced Patterns

### Window Functions (Session Ranking)

**Pattern:** `window` stage for time-series anomaly detection with sliding averages
**Datasets:** xdr_data

```xql
// Title: Anomalous Process Launch Volume
// Description: Uses a sliding 1-hour window average to detect endpoints with abnormal process launch rates
// Author: xsiam-buddy
// Datasets: xdr_data
// Modified: 2026-04-12

dataset = xdr_data
| filter event_type = ENUM.PROCESS and action_type = "process_launch"
| alter hour_bucket = format_timestamp(_time, "%Y-%m-%d %H:00")
| comp count() as launch_count by agent_hostname, hour_bucket
| window function=avg size=4h by agent_hostname
| alter deviation = launch_count - _window_avg
| filter deviation > 200
| sort desc deviation
| limit 50
```

**Annotation:** The `window` stage applies a sliding average (`function=avg`) over a 4-hour window grouped by hostname. Comparing the actual count against the window average surfaces endpoints with anomalous spikes. The `size` parameter sets the window span.

---

### arrayexpand + Multi-Join Chain (Canary to Endpoints to AD)

**Pattern:** Multi-join enrichment with `arrayexpand` preprocessing inside join subqueries
**Datasets:** thinkst_canary_generic_alert_raw, endpoints, pan_dss_raw

See [Canary Alert with Endpoint and AD Enrichment](#canary-alert-with-endpoint-and-ad-enrichment) above — that example demonstrates this pattern in full. The key techniques are:

- `arrayexpand` inside a join subquery to flatten multi-valued fields before joining
- Sequential left joins to chain enrichments (alert → endpoint → AD user)
- `preset = ad_users` as a shortcut for CIE Active Directory queries
- Preprocessing with `lowercase()` and `regextract()` inside join subqueries

---

### Transaction (User Session Grouping)

**Pattern:** `transaction` stage to group related events by user within a time window
**Datasets:** xdr_data

```xql
// Title: User Session Grouping
// Description: Groups user activity into sessions using transaction, then flags users with high event counts
// Author: xsiam-buddy
// Datasets: xdr_data (STORY events)
// Modified: 2026-04-12

dataset = xdr_data
| filter event_type = ENUM.STORY and auth_identity_display_name != null
| alter user_name = auth_identity_display_name
| transaction user_name maxspan = 2h
| comp count() as activity_count, first(_time) as first_activity, last(_time) as last_activity by user_name
| filter activity_count > 50
| sort desc activity_count
| limit 25
```

**Annotation:** `transaction` groups related events sharing the same `user_name` value within a 2-hour window into logical sessions. This enables session-level analysis — here, flagging users with more than 50 events in a single session. `first()` and `last()` capture the session time boundaries.

---

### Top Issues with Datamodel (config timeframe)

**Pattern:** `config timeframe` preamble with `issues` datamodel and `xdm.*` fields
**Datasets:** issues

```xql
// Title: Top Issue Sources Last 7 Days
// Description: Ranks alert detection methods by volume over the past 7 days
// Author: xsiam-buddy
// Datasets: issues
// Modified: 2026-04-12

config timeframe = 7d  // interval suffixes are case-insensitive (7d, 7D, 30d all valid)
| dataset = issues
| comp count() as IssueSourceCount by xdm.issue.detection.method
| sort desc IssueSourceCount
```

**Annotation:** `config timeframe = 7d` sets the query window as a preamble — when present, the `dataset` line must be preceded by a pipe. The `issues` dataset uses `xdm.*` field naming. This pattern is useful for operational dashboards monitoring alert volume by detection method.

---

### Process Execution Hash Lookup (config timeframe + view)

See [File Hash Reputation Lookup](#file-hash-reputation-lookup) above — that example demonstrates `config timeframe = 30d` combined with `view column order = populated` for hash-based process execution searches.

---

### Darktrace Critical Alert (Dynamic Severity + Nested Array Extraction)

**Pattern:** `fields *` with raw column aliasing, `array_length()` for empty array checks, `coalesce()`, dynamic `xdm.alert.severity` via `if()`
**Datasets:** darktrace_darktrace_raw

```xql
// Title: Darktrace - Critical Alert
// Description: Surfaces critical Darktrace model breaches with dynamic severity derived from risk score
// Author: xsiam-buddy
// Datasets: darktrace_darktrace_raw (via datamodel)
// Modified: 2026-04-12

datamodel dataset = darktrace_darktrace_raw
// Pull all XDM fields plus two raw dataset columns not in the datamodel
| fields *, darktrace_darktrace_raw.percentscore as percentscore,
          darktrace_darktrace_raw.triggeredComponents as triggeredComponents
| sort desc _time
// Keep only the most recent event per ID (because of desc sort)
| dedup xdm.event.id
| filter percentscore >= 87
| filter xdm.alert.original_threat_name not in (
    "Antigena::Network::Significant Anomaly::Antigena Significant Anomaly from Client Block",
    "Antigena::Network::Insider Threat::Antigena Large Data Volume Outbound Block"
  )
// Extract unique connection hostnames and DNS lookups from nested triggeredComponents array
| alter connection_hostname = arraydistinct(arraymap(
    arrayfilter(
      arraymerge(arraymap(triggeredComponents -> [], "@element" -> triggeredFilters{})),
      "@element" -> filterType = "Connection hostname"
    ),
    "@element" -> trigger.value
  )),
  dns_lookup = arraydistinct(arraymap(
    arrayfilter(
      arraymerge(arraymap(triggeredComponents -> [], "@element" -> triggeredFilters{})),
      "@element" -> filterType = "DNS host lookup"
    ),
    "@element" -> trigger.value
  ))
// Null out empty arrays so coalesce can pick the populated value
| alter connection_hostname = if(array_length(connection_hostname) = 0, null, connection_hostname)
| alter xdm.target.domain = coalesce(connection_hostname, dns_lookup)
// Dynamic severity derived from numeric risk score — appropriate here because the source
// does not emit a discrete severity field; the score IS the severity signal
| alter xdm.alert.severity = if(
    percentscore >= 87 and percentscore <= 91, "MEDIUM",
    percentscore >= 92 and percentscore <= 99, "HIGH",
    percentscore = 100, "CRITICAL",
    "LOW"
  )
| fields _time, xdm.alert.description, xdm.alert.mitre_tactics, xdm.alert.original_alert_id,
         xdm.alert.original_threat_id, xdm.alert.original_threat_name, xdm.alert.severity,
         xdm.event.description, xdm.event.id, xdm.event.log_level, xdm.event.type,
         xdm.network.rule, xdm.observer.product, xdm.observer.unique_identifier,
         xdm.observer.vendor, xdm.source.host.device_category, xdm.source.host.device_id,
         xdm.source.subnet, xdm.target.resource.parent_id, xdm.target.resource_before.id,
         xdm.target.resource_before.parent_id, xdm.observer.name, xdm.source.host.hostname,
         xdm.source.host.ipv4_addresses, xdm.source.host.mac_addresses,
         xdm.source.host.manufacturer, xdm.source.interface, xdm.source.ipv4,
         xdm.source.is_internal_ip, xdm.event.tags, xdm.target.domain, xdm.observer.action
```

**Annotation:** `fields *` with `dataset_name.field as alias` accesses raw columns not in the XDM datamodel. `array_length() = 0` detects empty arrays so they can be nulled out for `coalesce()` to pick the first non-null value. Dynamic `xdm.alert.severity` via `if()` is appropriate when the source has no discrete severity field and the numeric score is the severity signal.
