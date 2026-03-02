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
