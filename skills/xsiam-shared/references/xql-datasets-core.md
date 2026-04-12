# XQL Datasets — Core Reference

This document covers the top datasets, presets, field lists, and join patterns needed for standard XQL queries. For third-party alert datasets, email datasets, and cold storage deep-dives, see `xql-datasets-extended.md`.

---

## Dataset Selection Guide

| Use Case | Recommended Source |
|----------|-------------------|
| General threat hunting | `preset = xdr_event` |
| Endpoint-specific investigation | `dataset = xdr_data` |
| Network anomalies | `preset = network_story` |
| Firewall traffic analysis | `dataset = panw_ngfw_traffic_raw` |
| Firewall threat detection | `dataset = panw_ngfw_threat_raw` |
| Failed logins (multi-source) | `preset = identity_story` |
| Azure AD sign-ins specifically | `dataset = microsoft_azure_ad_sign_in_raw` |
| Okta authentication events | `dataset = okta_raw` |
| Cloud API misuse | `dataset = cloud_audit_log` |
| Alert triage and monitoring | `dataset = xdr_alerts` |
| Incident lifecycle and metrics | `dataset = xdr_incidents` |
| AD user/group enrichment | `preset = ad_users` |
| Cross-domain investigation | `datamodel` |

### When to Use Datasets vs. Presets

- **Datasets** — Use when you know exactly which data source you need. More efficient, predictable field names.
- **Presets** — Use when you need broader multi-source correlation or do not know which vendor-specific dataset contains the data. Presets combine related datasets under a single query target.
- **Datamodel** — Use when querying across multiple datasets with XDM-normalized fields. Access raw fields via `<dataset_name>.<field_name>`.

---

## Presets

Presets are predefined logical groupings that combine related datasets for specific security scenarios. Query with `preset = <name>`.

### xdr_event

All XDR endpoint events across all agents.

```
preset = xdr_event
```

**Included sources:**
- `xdr_data` — endpoint events
- Endpoint behavioral analytics
- Process creation, file operation, and registry modification events

**When to use:** General endpoint threat hunting when you want all event types.

### network_story

Network-related events from multiple sources.

```
preset = network_story
```

**Included sources:**
- `panw_ngfw_traffic_raw` — firewall traffic
- `panw_ngfw_threat_raw` — firewall threats
- Network behavioral analysis
- DNS query logs
- Proxy logs (if available)

**When to use:** Broad network investigation when you need traffic, threats, and DNS together.

### cloud_story

Cloud activity and audit events.

```
preset = cloud_story
```

**Included sources:**
- `cloud_audit_log` — cloud API calls
- Cloud resource changes
- Cloud identity events
- Cloud data access events

**When to use:** Cloud investigation spanning multiple providers or event types.

### identity_story

Identity and authentication events from all identity sources.

```
preset = identity_story
```

**Included sources:**
- `okta_raw` — Okta events
- `microsoft_azure_ad_sign_in_raw` — Azure AD sign-ins
- `microsoft_azure_ad_raw` — Azure AD directory changes
- VPN authentication logs (if available)
- Windows authentication logs

**When to use:** Multi-source authentication investigation (brute-force, credential stuffing, impossible travel).

### ad_users

A named preset over `pan_dss_raw` that surfaces Active Directory user records from the Palo Alto Cloud Identity Engine (CIE).

```
preset = ad_users
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `sam_account_name` | string | SAM account name (short username) |
| `display_name` | string | Full display name |
| `distinguished_name` | string | LDAP distinguished name |
| `security_identifier` | string | Windows SID |
| `email` | string | User email address |
| `security_group_list` | array | List of security group memberships |
| `department` | string | Department name |

**When to use:** Enrich endpoint or alert events with user identity. Filter by group membership.

> **Note:** The preset exposes `security_group_list`; the underlying `pan_dss_raw` dataset uses `security_groups`. Use `array_any(security_group_list, "@element" contains "Domain Admins")` to filter by group.

---

## Endpoint Datasets

### xdr_data

Primary endpoint detection and response dataset. Contains all endpoint events including process execution, file operations, registry modifications, and behavioral indicators.

```
dataset = xdr_data
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Event timestamp (UTC) |
| `agent_id` | string | Unique agent identifier |
| `agent_hostname` | string | Hostname of the endpoint |
| `agent_ip` | string | IP address of the endpoint |
| `agent_version` | string | Version of XDR agent |
| `os_type` | string | Operating system type (Windows, macOS, Linux) |
| `action_type` | string | Type of action (process_launch, file_operation, registry_modification, etc.) |
| `action_process_image_name` | string | Process executable name |
| `action_process_image_path` | string | Full path to process executable |
| `action_process_command_line` | string | Full command line of process |
| `action_process_image_sha256` | string | SHA256 hash of process executable |
| `action_process_parent_image_name` | string | Parent process name |
| `action_process_parent_image_path` | string | Parent process path |
| `action_process_id` | string | Process ID (PID) |
| `action_process_parent_id` | string | Parent Process ID (PPID) |
| `action_file_path` | string | Path to file operated on |
| `action_file_name` | string | Name of file operated on |
| `action_file_sha256` | string | SHA256 hash of file |
| `action_registry_path` | string | Registry path modified |
| `action_registry_value_name` | string | Registry value name |
| `action_registry_value_data` | string | Registry value data |
| `actor_process_image_name` | string | Initiating process name |
| `actor_process_image_path` | string | Initiating process path |
| `action_severity` | string | Severity level (low, medium, high, critical) |
| `action_category` | string | Event category (malware, exploit, evasion, etc.) |
| `cloud_correlation_id` | string | Cloud correlation ID for cross-product analysis |
| `dns_query` | string | DNS query (if applicable) |
| `url` | string | URL accessed (if applicable) |
| `user_name` | string | User executing the action |
| `user_sid` | string | User SID (Windows) |
| `domain` | string | Domain of endpoint |

**Common use cases:**
- Detect malicious process execution
- Identify suspicious file operations
- Monitor registry modifications
- Track command-line execution patterns

### endpoints

Endpoint inventory dataset containing metadata about all managed endpoints.

```
dataset = endpoints
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `endpoint_id` | string | Unique endpoint identifier |
| `endpoint_name` | string | Hostname of endpoint |
| `ip` | string | IP address |
| `ipv6` | string | IPv6 address |
| `os_type` | string | Operating system (Windows, macOS, Linux) |
| `os_version` | string | OS version/build number |
| `agent_version` | string | XDR agent version |
| `agent_status` | string | Agent status (Online, Offline, Stale) |
| `domain` | string | Domain name |
| `installation_date` | timestamp | Agent installation date |
| `last_seen` | timestamp | Last communication timestamp |
| `endpoint_status` | string | Endpoint status (Active, Inactive, Decommissioned) |
| `platform` | string | Platform identifier |
| `serial_number` | string | Device serial number |
| `mac_address` | string | MAC address |
| `users` | array | List of logged-in users |
| `installed_software` | array | List of installed applications |
| `tags` | array | Custom tags assigned to endpoint |
| `ip_address` | array | IP addresses assigned (use `arrayexpand` when joining on IP) |
| `user` | string | Last logged-in user (UPN format) |

**Common use cases:**
- Find offline endpoints needing attention
- Identify endpoints missing recent agents
- Inventory management and compliance
- Determine if a host has an XDR agent (for dynamic severity): `lowercase(host_name) in (dataset = endpoints | alter endpoint_name = lowercase(endpoint_name) | fields endpoint_name)`
- Join on `ip_address` to enrich alert events: `| arrayexpand ip_address` then join on `ip_address = jn.ip_address`

---

## Network Datasets

### panw_ngfw_traffic_raw

Palo Alto Networks Next-Generation Firewall (NGFW) traffic logs containing network flow information.

```
dataset = panw_ngfw_traffic_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Log timestamp (UTC) |
| `src` | string | Source IP address |
| `dst` | string | Destination IP address |
| `sport` | integer | Source port |
| `dport` | integer | Destination port |
| `proto` | string | Protocol (tcp, udp, icmp) |
| `action` | string | Action taken (allow, drop, deny, reset, etc.) |
| `rule_name` | string | Name of security rule matched |
| `rule_id` | string | Rule ID |
| `app` | string | Application identified (HTTP, SSH, FTP, etc.) |
| `category` | string | URL category |
| `bytes_sent` | integer | Bytes sent in flow |
| `bytes_received` | integer | Bytes received in flow |
| `packets_sent` | integer | Packets sent |
| `packets_received` | integer | Packets received |
| `session_duration` | integer | Duration of session in seconds |
| `from_zone` | string | Source zone |
| `to_zone` | string | Destination zone |
| `user_name` | string | Authenticated user (if available) |
| `device_name` | string | Firewall device name |
| `hostname_src` | string | Source hostname (if resolvable) |
| `hostname_dst` | string | Destination hostname (if resolvable) |
| `content_type` | string | Content type (if inspected) |
| `threat_name` | string | Threat identified (if any) |
| `severity` | string | Severity level |

**Common use cases:**
- Analyze outbound connectivity patterns
- Identify data exfiltration attempts
- Monitor encrypted traffic volumes
- Detect command-and-control communications

### panw_ngfw_threat_raw

NGFW threat logs containing detected threats, vulnerabilities, and malicious activities.

```
dataset = panw_ngfw_threat_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Detection timestamp (UTC) |
| `src` | string | Source IP address |
| `dst` | string | Destination IP address |
| `sport` | integer | Source port |
| `dport` | integer | Destination port |
| `proto` | string | Protocol |
| `threat_name` | string | Name of threat/vulnerability detected |
| `threat_id` | string | Threat/vulnerability ID |
| `threat_category` | string | Category (malware, spyware, virus, exploit, etc.) |
| `severity` | string | Severity (low, medium, high, critical) |
| `action` | string | Action taken (allow, drop, reset, block) |
| `file_name` | string | Name of file involved in threat |
| `file_type` | string | Type of file (executable, macro, script, etc.) |
| `file_hash` | string | Hash of file (MD5, SHA256) |
| `url` | string | URL associated with threat |
| `domain` | string | Domain associated with threat |
| `application_protocol` | string | Application protocol used |
| `user_name` | string | Associated user |
| `device_name` | string | Firewall device |
| `tunnel_type` | string | VPN tunnel type (if applicable) |
| `pcap_id` | string | PCAP capture ID for forensic analysis |

**Common use cases:**
- Track blocked threats and malware detections
- Monitor vulnerability exploitation attempts
- Analyze threat trends by category
- Investigate suspicious file downloads

---

## Cloud Datasets

### cloud_audit_log

Unified cloud audit logs from multiple cloud providers (AWS, Azure, GCP).

```
dataset = cloud_audit_log
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Event timestamp (UTC) |
| `cloud_provider` | string | Cloud provider (aws, azure, gcp) |
| `caller_ip` | string | IP address of caller |
| `caller_identity_type` | string | Type of identity (user, role, service) |
| `caller_identity_name` | string | Name of identity |
| `operation_name` | string | API operation performed |
| `operation_status` | string | Status (Success, Failure, Unauthorized) |
| `error_code` | string | Error code (if failed) |
| `error_message` | string | Error message (if failed) |
| `resource_type` | string | Type of resource (vm, bucket, database, etc.) |
| `resource_name` | string | Name of resource |
| `resource_id` | string | ID of resource |
| `action` | string | Action performed (Create, Delete, Modify, Read) |
| `project` | string | Cloud project ID |
| `region` | string | Geographic region |
| `user_agent` | string | User agent of caller |

**Common use cases:**
- Detect unauthorized API calls
- Monitor privileged access changes
- Track resource creation/deletion patterns
- Identify unusual account activity

### microsoft_azure_ad_raw

Azure Active Directory logs including user authentication and directory changes.

```
dataset = microsoft_azure_ad_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Event timestamp (UTC) |
| `operation_name` | string | Type of operation |
| `operation_status` | string | Success or failure |
| `user_id` | string | Azure AD user ID (GUID) |
| `user_name` | string | User principal name (UPN) |
| `user_display_name` | string | Display name of user |
| `ip_address` | string | Client IP address |
| `location` | string | Geographical location |
| `result` | string | Operation result (Success, Failure, Interrupted) |
| `error_code` | string | Error code (if applicable) |
| `error_description` | string | Error description |
| `actor_id` | string | ID of user performing action |
| `actor_name` | string | Name of user performing action |
| `target_id` | string | ID of target resource |
| `target_resource` | string | Name of target resource |
| `target_type` | string | Type of target (User, Group, Application) |
| `app_id` | string | Application ID |
| `app_display_name` | string | Application display name |
| `correlation_id` | string | Correlation ID for related events |
| `request_id` | string | Request ID |
| `properties` | string | Additional properties (JSON) |

**Common use cases:**
- Monitor failed authentication attempts
- Track administrative changes
- Detect anomalous login patterns
- Audit privileged operations

### microsoft_azure_ad_sign_in_raw

Azure AD sign-in specific logs with conditional access and risk signals.

```
dataset = microsoft_azure_ad_sign_in_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Sign-in timestamp (UTC) |
| `user_principal_name` | string | User UPN |
| `user_display_name` | string | User display name |
| `user_id` | string | User GUID |
| `ip_address` | string | Sign-in IP address |
| `ip_address_location` | string | Geolocation of IP |
| `location` | string | Inferred location |
| `country` | string | Country code |
| `status` | string | Sign-in status (success, failure, interrupt) |
| `error_code` | integer | Error code (if failed) |
| `error_details` | string | Error description |
| `app_id` | string | Application ID |
| `app_display_name` | string | Application display name |
| `resource_id` | string | Resource being accessed |
| `resource_display_name` | string | Resource display name |
| `client_app` | string | Client app type (Browser, MobileApps, etc.) |
| `device_detail_browser` | string | Browser information |
| `device_detail_device_id` | string | Device ID |
| `device_detail_operating_system` | string | Device OS |
| `device_detail_is_compliant` | string | Device compliance status |
| `mfa_detail_auth_method` | string | MFA method used |
| `mfa_detail_auth_detail` | string | MFA detail |
| `conditional_access_status` | string | CA policy status (notApplied, success, failure) |
| `conditional_access_policies` | array | Applied CA policies |
| `risk_detail` | string | Risk classification |
| `risk_level_aggregated` | string | Aggregated risk level |
| `risk_level_sign_in` | string | Sign-in risk level |
| `risk_state` | string | Risk state (atRisk, remediated, dismissed) |
| `original_request_id` | string | Original request ID |
| `correlation_id` | string | Correlation ID |

**Common use cases:**
- Detect risky sign-ins and anomalies
- Monitor conditional access policies
- Track MFA adoption and usage
- Identify geographic anomalies

### msft_azure_ad_raw (Datamodel Variant)

Sign-in and audit log data from Microsoft Azure AD, accessed via the XDM datamodel for normalized field access.

```
datamodel dataset = msft_azure_ad_raw
```

> **Important:** This is `msft_azure_ad_raw` (not `microsoft_azure_ad_raw`). Use `datamodel dataset =` syntax to access XDM fields.

Use `fields *, msft_azure_ad_raw.ipAddress as ipaddress` to surface the raw `ipAddress` field alongside XDM fields.

**Key XDM Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `xdm.source.user.username` | string | UPN of the authenticating user |
| `xdm.source.user.sam_account_name` | string | SAM account name |
| `xdm.source.application.name` | string | Application name |
| `xdm.event.outcome` | string | "SUCCESS" or "FAILURE" |
| `xdm.event.outcome_reason` | string | Reason for failure |
| `xdm.event.operation_sub_type` | string | Operation detail |
| `xdm.session_context_id` | string | Session context identifier |

**When to use:** When you need XDM-normalized fields for cross-dataset correlation or when writing datamodel queries.

---

## Identity Datasets

### okta_raw

Okta identity and access management (IAM) logs.

```
dataset = okta_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Event timestamp (UTC) |
| `event_type` | string | Type of Okta event |
| `actor_id` | string | Actor identifier |
| `actor_display_name` | string | Display name of actor |
| `actor_type` | string | Type of actor (User, System) |
| `target_id` | string | Target resource ID |
| `target_display_name` | string | Display name of target |
| `target_type` | string | Type of target (User, AppInstance, Group) |
| `client_ip` | string | IP address of client |
| `client_user_agent` | string | User agent string |
| `client_device_type` | string | Device type (Computer, Phone, etc.) |
| `transaction_id` | string | Transaction ID |
| `outcome_result` | string | Result (SUCCESS, FAILURE) |
| `outcome_reason` | string | Reason for outcome |
| `outcome_error_code` | string | Error code (if failed) |
| `outcome_error_summary` | string | Error summary |
| `authentication_type` | string | Type of authentication (password, mfa, etc.) |
| `authentication_provider` | string | Provider used for auth |
| `mfa_method` | string | MFA method (email, sms, okta, etc.) |
| `severity` | string | Severity level |
| `debug_context` | string | Debug information (JSON) |
| `request_uri` | string | Request URI |
| `http_method` | string | HTTP method (GET, POST, etc.) |
| `device_id` | string | Device identifier |
| `legacy_event_type` | string | Legacy event type for backwards compatibility |

**Common use cases:**
- Monitor failed authentication attempts
- Track MFA usage and failures
- Detect unusual login locations/times
- Audit admin and privileged actions

---

## Alert and Incident Datasets

### xdr_alerts

XDR-generated alerts from detection logic and correlation rules.

```
dataset = xdr_alerts
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Alert creation timestamp (UTC) |
| `alert_id` | string | Unique alert identifier |
| `alert_name` | string | Name/title of alert |
| `alert_description` | string | Description of alert |
| `alert_source` | string | Source of alert (WildFire, Cortex, etc.) |
| `severity` | string | Severity level (low, medium, high, critical) |
| `category` | string | Alert category (malware, exploit, evasion, etc.) |
| `action_status` | string | Status (New, In Progress, Resolved, Dismissed) |
| `host_name` | string | Affected hostname |
| `host_ip` | string | Affected host IP |
| `user_name` | string | Associated user |
| `user_id` | string | User ID |
| `source_ip` | string | Source IP address |
| `destination_ip` | string | Destination IP address |
| `file_name` | string | Associated file name |
| `file_hash` | string | File SHA256 hash |
| `process_name` | string | Associated process name |
| `process_command_line` | string | Process command line |
| `url` | string | Associated URL |
| `domain` | string | Associated domain |
| `source` | string | Data source (endpoint, network, cloud, etc.) |
| `sub_source` | string | Sub-source identifier |
| `mitre_attack_id` | string | MITRE ATT&CK technique ID |
| `mitre_attack_tactic` | string | MITRE ATT&CK tactic |
| `first_seen_at` | timestamp | First detection time |
| `last_seen_at` | timestamp | Last detection time |
| `event_count` | integer | Number of events in alert |
| `assigned_user` | string | User alert is assigned to |
| `notes` | string | Alert notes/comments |
| `tags` | array | Alert tags |
| `correlation_id` | string | Related correlation rule ID |

**Common use cases:**
- Monitor and triage alerts
- Track alert volume by severity/category
- Investigate alert trends
- Measure MTTR (mean time to resolve)

### xdr_incidents

Incident records aggregating related alerts and providing higher-level context.

```
dataset = xdr_incidents
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Incident creation timestamp (UTC) |
| `incident_id` | string | Unique incident identifier |
| `incident_name` | string | Incident title |
| `incident_description` | string | Description of incident |
| `severity` | string | Severity level (low, medium, high, critical) |
| `status` | string | Status (New, Investigating, Resolved, Closed) |
| `creation_time` | timestamp | When incident was created |
| `modification_time` | timestamp | Last modification time |
| `resolution_time` | timestamp | When incident was resolved |
| `assigned_user` | string | Assigned analyst |
| `assigned_team` | string | Assigned team |
| `alert_count` | integer | Number of related alerts |
| `affected_hosts` | array | List of affected hostnames |
| `affected_users` | array | List of affected users |
| `affected_assets` | array | List of affected resources |
| `primary_actor` | string | Primary threat actor or malware family |
| `mitre_attack_id` | array | MITRE ATT&CK technique IDs |
| `mitre_attack_tactic` | array | MITRE ATT&CK tactics |
| `ioc_list` | array | Indicators of compromise |
| `manual_severity` | string | Manually set severity |
| `resolution_comment` | string | Resolution notes |
| `labels` | array | Incident labels/tags |
| `xdr_url` | string | Link to incident in XDR UI |

**Common use cases:**
- Track incident lifecycle and metrics
- Identify incident patterns
- Measure SLAs and MTTR
- Correlate related alerts

---

## Hot and Cold Storage

XQL supports querying data in both hot (recent, fast) and cold (archived, slower) storage tiers.

- **Hot storage (default):** `dataset = <dataset_name>`
- **Cold storage:** `cold_dataset = <dataset_name>`
- **Both in same query:** You can reference both hot and cold datasets in a single query

```
// Query hot storage (default)
dataset = xdr_data
| filter agent_hostname = "web-server-01"

// Query cold storage
cold_dataset = xdr_data
| filter agent_hostname = "web-server-01"
```

> Cold storage queries are slower. Use hot storage for recent investigations and cold storage for historical lookbacks beyond the hot retention window.

---

## Result Limits

| Context | Default Limit |
|---------|---------------|
| Interactive (Query Builder) | 1,000 |
| Widgets, APIs, Scheduled queries, Correlation Rules | 1,000,000 |
| Legacy templates | 10,000 |

- Queries without an explicit `limit` stage return up to the default for their context
- Always add `| limit N` to control result size and improve performance
- Datamodel queries without a limit default to 1,000 results

---

## User Name Format

In `xdr_data`, user names are normalized to:

```
<company domain>\<username>
```

Use `normalized_user` fields for best results when filtering or joining on user identity.

---

## Field Naming Conventions

Understanding field naming patterns helps you find the right field without memorizing every dataset schema.

| Pattern | Convention | Examples |
|---------|-----------|----------|
| Timestamps | Fields ending in `_time` or `_at` | `_time`, `creation_time`, `first_seen_at`, `last_seen` |
| IP addresses | Fields containing `ip`, `src`, `dst` | `agent_ip`, `src`, `dst`, `caller_ip`, `host_ip`, `source_ip`, `destination_ip` |
| Hashes | Fields containing `hash`, `sha256`, `md5` | `action_process_image_sha256`, `action_file_sha256`, `file_hash` |
| User identifiers | Fields containing `user` | `user_name`, `user_id`, `user_principal_name`, `actor_display_name` |
| Process data | Fields starting with `action_process_` or `actor_process_` | `action_process_image_name`, `action_process_command_line`, `actor_process_image_path` |
| File data | Fields starting with `action_file_` | `action_file_path`, `action_file_name`, `action_file_sha256` |
| Registry data | Fields starting with `action_registry_` | `action_registry_path`, `action_registry_value_name`, `action_registry_value_data` |

---

## Cross-Dataset Join Patterns

### Endpoint + Firewall (correlate process activity with network flows)

```
dataset = xdr_data
| filter action_type = "process_launch"
| join type=inner (dataset = panw_ngfw_traffic_raw) as fw
  xdr_data.agent_ip = fw.src
| fields agent_hostname, action_process_image_name, fw.dst, fw.dport
```

### Alerts + Incidents (enrich alerts with incident context)

```
dataset = xdr_alerts
| join type=left (dataset = xdr_incidents) as inc
  xdr_alerts.alert_id in inc.alert_ids
| fields alert_name, severity, incident_id, incident_status
```

### Cloud + Endpoints (enrich cloud API calls with endpoint metadata)

```
dataset = cloud_audit_log
| join type=left (dataset = endpoints | fields endpoint_name, ip) as ep
  cloud_audit_log.caller_ip = ep.ip
| fields operation_name, caller_identity_name, ep.endpoint_name
```

---

## XDM Dataset Mappings

When using `datamodel dataset =` syntax, the following datasets have automatic XDM mappings:

- `xdr_data` — auto-mapped (with some exceptions)
- NGFW datasets: `panw_ngfw_traffic_raw`, `panw_ngfw_threat_raw`, `panw_ngfw_url_raw`, `panw_ngfw_filedata_raw`, `panw_ngfw_globalprotect_raw`, `panw_ngfw_hipmatch_raw`

Additional datasets gain XDM mappings via:
1. **Marketplace** — OOTB mappings via Data Model Rules in content packs
2. **Custom** — Create your own Data Model Rules

Access raw (unmapped) fields in a datamodel query with `<dataset_name>.<field_name>` syntax.
