# XQL Examples

Real-world production XQL queries. Use these to model syntax, field names, and
structural patterns when generating new queries.

---

## Google Workspace - Administrator Role Assigned

```xql
// Title: Google Workspace - Administrator Role Assigned
// Description: Detects when a Google Administrator role is assigned to a user
// Author: xsiam-buddy
// Dataset(s): google_workspace_admin_console_raw
// Query last modified: 2026-03-01
// Vendor Reference: https://developers.google.com/workspace/admin/reports/v1/appendix/activity/admin-delegated-admin-settings

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

**Demonstrates:** Arrow notation (`->`), `arraymap`, `arrayfilter`, `arraydistinct`, `arraymerge`, chained array operations.

---

## Multiple Users Failing to Login from the Same Location

```xql
// Title: Multiple Users Failing to Login from the Same Location
// Description: Detects potential password spraying — 5+ unique users failing to authenticate from the same host
// Author: xsiam-buddy
// Dataset(s): xdr_data
// Query last modified: 2026-03-01
// Vendor Reference: N/A

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

**Demonstrates:** `ENUM.EVENT_LOG`, `regextract()`, `arrayindex()`, extracting fields from raw event log messages.

---

## Top Issue Sources Last 7 Days

```xql
// Title: Top Issue Sources Last 7 Days
// Description: Ranks alert detection methods by volume over the past 7 days
// Author: xsiam-buddy
// Dataset(s): issues
// Query last modified: 2026-03-01
// Vendor Reference: N/A

config timeframe = 7d  // interval suffixes are case-insensitive (7d, 7D, 30d all valid)
| dataset = issues
| comp count() as IssueSourceCount by xdm.issue.detection.method
| sort desc IssueSourceCount
```

**Demonstrates:** `config timeframe` preamble (pipe before `dataset`), `xdm.*` field naming convention.

---

## Windows On-Prem AD Group Monitoring

```xql
// Title: Windows - AD Group Monitoring (On-Premise)
// Description: Monitors membership changes to specific VIP AD groups using a lookup table
// Author: xsiam-buddy
// Dataset(s): microsoft_windows_raw (via datamodel)
// Query last modified: 2026-03-01
// Vendor Reference: N/A

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

**Demonstrates:** `datamodel dataset =`, `arrayexpand`, inline dataset lookup (`in (dataset = ...)`), `xdm.*` field naming, `regextract` + `arrayindex` for extracting from descriptions.

---

## Process Executions of Hash in Last Month

```xql
// Title: Process Executions of Hash in Last Month
// Description: Finds all executions matching a given hash across SHA256 and MD5 fields over 30 days
// Author: xsiam-buddy
// Dataset(s): xdr_data
// Query last modified: 2026-03-01
// Vendor Reference: N/A

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

**Demonstrates:** `config timeframe` preamble, `$variable` substitution, `view column order = populated`.

---

## Top 10 Users Failing to Authenticate from Multiple Locations

```xql
// Title: Top 10 Users Failing to Authenticate from Multiple Locations
// Description: Finds users with the most distinct source IPs for failed auth — useful for detecting account compromise
// Author: xsiam-buddy
// Dataset(s): xdr_data (STORY events — requires Azure, Okta, or Ping auth data)
// Query last modified: 2026-03-01
// Vendor Reference: N/A

dataset = xdr_data
| filter event_type = ENUM.STORY and auth_identity_display_name != null and auth_outcome != "SUCCESS"
| fields auth_identity_display_name as user, auth_identity as email, auth_client as source
| comp count_distinct(source) as counter by user, email
| sort desc counter
| limit 10
```

**Demonstrates:** `ENUM.STORY`, inline field aliasing with `as` in `fields` stage (XSIAM XQL dialect — use `alter` for renaming in standard XQL), auth fields from story events.

---

## Abnormal Security - Account Takeover Sign In

```xql
// Title: Abnormal - Account Takeover Sign In
// Description: Generates account takeover alerts from Abnormal Security; elevates severity for specific domains
// Author: xsiam-buddy
// Dataset(s): abnormal_security_generic_alert_raw
// Query last modified: 2026-03-01
// Vendor Reference: N/A

dataset = abnormal_security_generic_alert_raw
| filter severity = "Account Takeover"
  and case_status = "Action Required"
| alter severity_level = if(affectedEmployee contains "@yourdomain.com", "MEDIUM", severity_level)
| fields caseId, case_status, severity, severity_level, analysis, genai_summary, affectedEmployee, firstObserved
```

**Demonstrates:** `if()` for conditional severity elevation, `contains` operator on a user-supplied domain substring.

---

## Canary Alert Notification

```xql
// Title: Canary - Alert Notification
// Description: Raises alerts from Thinkst Canary, enriched with endpoint inventory and Active Directory user identity
// Author: xsiam-buddy
// Dataset(s): thinkst_canary_generic_alert_raw, endpoints, pan_dss_raw (via ad_users preset)
// Query last modified: 2026-03-01
// Vendor Reference: N/A

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

**Demonstrates:** `_alert_data ->` arrow extraction from alert JSON blob; multi-join (two sequential `join` stages); `preset = ad_users`; preprocessing inside a join subquery (`arrayexpand`, `regextract`); `dedup _id` using the system event ID.

---

## Darktrace - Critical Alert

```xql
// Title: Darktrace - Critical Alert
// Description: Surfaces critical Darktrace model breaches with dynamic severity derived from risk score
// Author: xsiam-buddy
// Dataset(s): darktrace_darktrace_raw (via datamodel)
// Query last modified: 2026-03-01
// Vendor Reference: N/A

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

**Demonstrates:** `fields *` with raw dataset column aliasing (`dataset_name.field as alias`); `array_length()` to check for empty array; `coalesce()` to pick first non-null; dynamic `xdm.alert.severity` via `if()` (appropriate when the source has no discrete severity — score is the signal).

---

## Google Workspace - User Suspended

```xql
// Title: Google Workspace - User Suspended
// Description: Surfaces user suspended alerts from the Google Workspace alerts dataset
// Author: xsiam-buddy
// Dataset(s): google_workspace_alerts_raw
// Query last modified: 2026-03-01
// Vendor Reference: N/A

dataset = google_workspace_alerts_raw
| filter type = "User suspended"
// Extract email and severity from nested JSON blobs
| alter email    = data -> email,
       severity  = metadata -> severity
| fields _time, alertId, type, source, severity, email, securityInvestigationToolLink, data, metadata
```

**Demonstrates:** `->` arrow extraction from top-level JSON blob fields (`data`, `metadata`); Google Workspace alerts dataset.

---

## Microsoft Azure - PowerShell Application Login Activity

```xql
// Title: Microsoft Azure - PowerShell Application Login Activity
// Description: Detects successful PowerShell application logins originating outside trusted IP ranges
// Author: xsiam-buddy
// Dataset(s): msft_azure_ad_raw (via datamodel), ad_admin_accounts (lookup table)
// Query last modified: 2026-03-01
// Vendor Reference: N/A

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

**Demonstrates:** `fields *` with raw field aliasing in a `datamodel dataset =` query; `incidr() = false` as a boolean negation filter; `msft_azure_ad_raw` as the correct dataset name (not `microsoft_azure_ad_raw`).

---

## Microsoft Windows - Domain Admin Authentication to Tier 1/2 Asset

```xql
// Title: Microsoft Windows - Domain Admin Authentication to Tier 1/2 Asset
// Description: Detects domain administrator credentials used on assets outside Tier 0 (non-Domain Controllers)
// Author: xsiam-buddy
// Dataset(s): microsoft_windows_raw, pan_dss_raw, endpoints
// Query last modified: 2026-03-01
// Vendor Reference: https://learn.microsoft.com/en-us/windows-server/identity/securing-privileged-access/reference-tools-logon-types

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

**Demonstrates:** `array_any(arr, condition)` for filtering by array membership; chained joins (three sequential `join` stages); `event_data ->` for structured JSON field extraction; `pan_dss_raw` Cloud Identity Engine dataset; `endpoints` dataset; inline dataset lookup inside `if()` for dynamic severity; `dn as target_dn` field aliasing in `fields` stage.

---
