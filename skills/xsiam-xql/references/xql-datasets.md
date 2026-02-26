# XSIAM Datasets and Fields Reference

## Overview

This document provides a comprehensive reference of commonly used datasets in XSIAM and their key fields. Datasets can be queried directly with `dataset = dataset_name` or through presets for broader analysis.

---

## Endpoint Datasets

### xdr_data

Primary endpoint detection and response dataset. Contains all endpoint events including process execution, file operations, registry modifications, and behavioral indicators.

**Usage:**
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

**Common Use Cases:**

- Detect malicious process execution
- Identify suspicious file operations
- Monitor registry modifications
- Track command-line execution patterns

**Example Query:**
```
dataset = xdr_data
| filter action_type = "process_launch"
| filter action_process_image_name ~= "powershell\.exe"
| fields agent_hostname, action_process_command_line, _time
| limit 100
```

### endpoints

Endpoint inventory dataset containing metadata about all managed endpoints.

**Usage:**
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

**Common Use Cases:**

- Find offline endpoints needing attention
- Identify endpoints missing recent agents
- Inventory management and compliance
- Locate endpoints by OS or domain

**Example Query:**
```
dataset = endpoints
| filter agent_status = "Offline"
| filter timestamp_diff(last_seen, current_time(), "DAY") > 30
| fields endpoint_name, last_seen, agent_version
| sort last_seen asc
```

---

## Network Datasets

### panw_ngfw_traffic_raw

Palo Alto Networks Next-Generation Firewall (NGFW) traffic logs containing network flow information.

**Usage:**
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

**Common Use Cases:**

- Analyze outbound connectivity patterns
- Identify data exfiltration attempts
- Monitor encrypted traffic volumes
- Detect command-and-control communications

**Example Query:**
```
dataset = panw_ngfw_traffic_raw
| filter proto = "tcp"
| filter dport in (443, 80, 8080)
| comp sum(bytes_sent) as total_out, sum(bytes_received) as total_in by src, dst
| filter total_out > 1000000000
| sort total_out desc
| limit 50
```

### panw_ngfw_threat_raw

NGFW threat logs containing detected threats, vulnerabilities, and malicious activities.

**Usage:**
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

**Common Use Cases:**

- Track blocked threats and malware detections
- Monitor vulnerability exploitation attempts
- Analyze threat trends by category
- Investigate suspicious file downloads

**Example Query:**
```
dataset = panw_ngfw_threat_raw
| filter severity = "critical" or severity = "high"
| comp count() as threat_count by threat_category, src
| sort threat_count desc
| limit 25
```

---

## Cloud Datasets

### cloud_audit_log

Unified cloud audit logs from multiple cloud providers (AWS, Azure, GCP).

**Usage:**
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
| `request_id` | string | Unique request ID |
| `response_elements` | string | Response data (JSON) |
| `request_parameters` | string | Request parameters (JSON) |
| `service_name` | string | Cloud service (EC2, IAM, S3, etc.) |

**Common Use Cases:**

- Detect unauthorized API calls
- Monitor privileged access changes
- Track resource creation/deletion patterns
- Identify unusual account activity

**Example Query:**
```
dataset = cloud_audit_log
| filter operation_status = "Failure"
| filter operation_name in ("AssumeRole", "CreateUser", "DeleteUser", "PutBucketPolicy")
| comp count() as failures by caller_identity_name, error_code
| filter failures > 5
```

### microsoft_azure_ad_raw

Azure Active Directory logs including user authentication and directory changes.

**Usage:**
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

**Common Use Cases:**

- Monitor failed authentication attempts
- Track administrative changes
- Detect anomalous login patterns
- Audit privileged operations

**Example Query:**
```
dataset = microsoft_azure_ad_raw
| filter operation_name = "Sign-in activity"
| filter result = "Failure"
| bin _time span = 5m
| comp count() as failed_attempts by user_name, ip_address, _time
| filter failed_attempts > 10
```

---

## Identity Datasets

### okta_raw

Okta identity and access management (IAM) logs.

**Usage:**
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

**Common Use Cases:**

- Monitor failed authentication attempts
- Track MFA usage and failures
- Detect unusual login locations/times
- Audit admin and privileged actions

**Example Query:**
```
dataset = okta_raw
| filter outcome_result = "FAILURE"
| filter authentication_type = "password"
| comp count_distinct(actor_display_name) as unique_users by client_ip
| filter unique_users > 1
| sort unique_users desc
```

### microsoft_azure_ad_sign_in_raw

Azure AD sign-in specific logs with conditional access and risk signals.

**Usage:**
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

**Common Use Cases:**

- Detect risky sign-ins and anomalies
- Monitor conditional access policies
- Track MFA adoption and usage
- Identify geographic anomalies

**Example Query:**
```
dataset = microsoft_azure_ad_sign_in_raw
| filter status = "failure"
| filter error_code = 50058
| comp count() as failed_signin_count by user_principal_name, ip_address
| filter failed_signin_count > 3
```

---

## Alert and Incident Datasets

### xdr_alerts

XDR-generated alerts from detection logic and correlation rules.

**Usage:**
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

**Common Use Cases:**

- Monitor and triage alerts
- Track alert volume by severity/category
- Investigate alert trends
- Measure MTTR (mean time to resolve)

**Example Query:**
```
dataset = xdr_alerts
| filter severity in ("high", "critical")
| filter action_status = "New"
| comp count() as open_alerts by category
| sort open_alerts desc
```

### xdr_incidents

Incident records aggregating related alerts and providing higher-level context.

**Usage:**
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

**Common Use Cases:**

- Track incident lifecycle and metrics
- Identify incident patterns
- Measure SLAs and MTTR
- Correlate related alerts

**Example Query:**
```
dataset = xdr_incidents
| filter status != "Closed"
| filter timestamp_diff(creation_time, current_time(), "DAY") < 30
| comp count() as open_incidents by severity
| sort open_incidents desc
```

---

## Email Datasets

### microsoft_office_365_raw

Microsoft Office 365 audit logs including email and collaboration events.

**Usage:**
```
dataset = microsoft_office_365_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Event timestamp (UTC) |
| `workload` | string | Office 365 workload (Exchange, SharePoint, Teams, etc.) |
| `operation` | string | Type of operation |
| `organization_id` | string | Organization identifier |
| `user_id` | string | User ID performing action |
| `user_name` | string | User display name |
| `ip_address` | string | Client IP address |
| `mailbox_owner_upn` | string | Mailbox owner UPN |
| `mailbox_audit_enable` | string | Mailbox audit status |
| `external_access` | string | External access indicator |
| `object_id` | string | Object being modified |
| `object_type` | string | Type of object |
| `parameters` | string | Operation parameters (JSON) |
| `user_agent` | string | User agent |
| `client_ip_address` | string | Client IP |
| `succeeded` | string | Operation success status |
| `result_status` | string | Result status |
| `resource_name` | string | Resource name |
| `custom_properties` | string | Custom properties (JSON) |

**Common Use Cases:**

- Monitor mailbox access patterns
- Track sensitive file sharing
- Detect data exfiltration via email
- Audit administrator actions

### email_gateway_raw

Email gateway logs from mail security appliances.

**Usage:**
```
dataset = email_gateway_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Event timestamp (UTC) |
| `sender` | string | Email sender address |
| `recipient` | string | Email recipient address |
| `subject` | string | Email subject |
| `message_id` | string | Unique message ID |
| `message_size` | integer | Size of message in bytes |
| `attachment_count` | integer | Number of attachments |
| `attachment_names` | array | Names of attachments |
| `attachment_types` | array | Types of files attached |
| `attachment_hashes` | array | Hashes of attachments |
| `action` | string | Gateway action (allow, quarantine, drop, etc.) |
| `threat_name` | string | Detected threat name |
| `threat_category` | string | Threat category (malware, phishing, etc.) |
| `severity` | string | Severity level |
| `source_ip` | string | Sender IP address |
| `source_hostname` | string | Sender hostname |
| `destination_ip` | string | Recipient server IP |
| `url_count` | integer | Number of URLs in email |
| `urls` | array | URLs found in email |
| `headers` | string | Email headers (JSON) |
| `body_text` | string | Email body (if available) |

**Common Use Cases:**

- Monitor inbound email threats
- Detect phishing campaigns
- Track malware distribution via email
- Investigate compromised sender accounts

---

## Presets

Presets are predefined logical groupings of datasets that combine related data sources for specific security scenarios.

### preset = xdr_event

All XDR endpoint events across all agents.

**Usage:**
```
preset = xdr_event
```

**Included Sources:**
- `xdr_data` — endpoint events
- Endpoint behavioral analytics
- Process creation events
- File operation events
- Registry modification events

**Example Query:**
```
preset = xdr_event
| filter severity >= "high"
| comp count() as total_events by category
```

### preset = network_story

Network-related events from multiple sources.

**Usage:**
```
preset = network_story
```

**Included Sources:**
- `panw_ngfw_traffic_raw` — firewall traffic
- `panw_ngfw_threat_raw` — firewall threats
- Network behavioral analysis
- DNS query logs
- Proxy logs (if available)

**Example Query:**
```
preset = network_story
| filter src incidr("10.0.0.0/8")
| comp count() as events by dst
```

### preset = cloud_story

Cloud activity and audit events.

**Usage:**
```
preset = cloud_story
```

**Included Sources:**
- `cloud_audit_log` — cloud API calls
- Cloud resource changes
- Cloud identity events
- Cloud data access events

**Example Query:**
```
preset = cloud_story
| filter operation_status = "Failure"
| filter cloud_provider = "aws"
```

### preset = identity_story

Identity and authentication events.

**Usage:**
```
preset = identity_story
```

**Included Sources:**
- `okta_raw` — Okta events
- `microsoft_azure_ad_sign_in_raw` — Azure AD sign-ins
- `microsoft_azure_ad_raw` — Azure AD directory changes
- VPN authentication logs (if available)
- Windows authentication logs

**Example Query:**
```
preset = identity_story
| filter outcome_result = "FAILURE"
| bin _time span = 10m
| comp count() as failures by actor_id, _time
```

---

## Dataset Selection Best Practices

### When to Use Specific Datasets vs. Presets

| Use Case | Recommended Source |
|----------|-------------------|
| General threat hunting | `preset = xdr_event` |
| Endpoint-specific investigation | `dataset = xdr_data` |
| Network anomalies | `preset = network_story` |
| Firewall traffic analysis | `dataset = panw_ngfw_traffic_raw` |
| Failed logins | `preset = identity_story` |
| Cloud API misuse | `dataset = cloud_audit_log` |
| Malware detection | `dataset = xdr_alerts` |
| Cross-domain investigation | `datamodel` |

### Performance Tips

1. **Use specific datasets** when you know exactly which data you need
2. **Use presets** when you need broader multi-source correlation
3. **Filter early** with specific time ranges and conditions
4. **Select fields** before aggregation to reduce data processed
5. **Join strategically** to avoid Cartesian products

### Field Naming Conventions

- **Timestamps:** Fields ending in `_time` are in UTC
- **IPs:** Fields containing `ip`, `src`, `dst`, `source`, `destination`
- **Hashes:** Fields containing `hash`, `sha256`, `md5`
- **User identifiers:** `user_name`, `user_id`, `user_principal_name`
- **Process data:** Fields starting with `action_process_` or `actor_process_`
- **File data:** Fields starting with `action_file_`

---

## Cross-Dataset Join Examples

### Joining Endpoint and Firewall Data

```
dataset = xdr_data
| filter action_type = "process_launch"
| join type=inner (dataset = panw_ngfw_traffic_raw) as fw
  xdr_data.agent_ip = fw.src
| fields agent_hostname, action_process_image_name, fw.dst, fw.dport
```

### Joining Alerts with Incident Data

```
dataset = xdr_alerts
| join type=left (dataset = xdr_incidents) as inc
  xdr_alerts.alert_id in inc.alert_ids
| fields alert_name, severity, incident_id, incident_status
```

### Enriching Cloud Logs with User Information

```
dataset = cloud_audit_log
| join type=left (dataset = endpoints | fields endpoint_name, ip) as ep
  cloud_audit_log.caller_ip = ep.ip
| fields operation_name, caller_identity_name, ep.endpoint_name
```
