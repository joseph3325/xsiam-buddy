# XQL Datasets — Extended Reference

This file is loaded on-demand when a query targets uncommon datasets,
third-party alert sources, email data, identity deep-dives, or cold storage.

For core datasets (xdr_data, endpoints, panw_ngfw_traffic_raw, panw_ngfw_threat_raw,
cloud_audit_log, xdr_alerts, xdr_incidents, okta_raw, microsoft_azure_ad_sign_in_raw)
see `xql-datasets-core.md`.

---

## Third-Party Alert Datasets

Third-party integrations ingest vendor alert data into XSIAM as raw datasets.
Each vendor's dataset has its own field structure. Many embed complex data in
JSON blobs that require arrow notation (`->`) to access nested fields.

### abnormal_security_generic_alert_raw

Alert data from Abnormal Security's email security platform. Surfaces
account takeover, phishing, and BEC detections.

**Usage:**
```
dataset = abnormal_security_generic_alert_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `caseId` | string | Unique case identifier |
| `case_status` | string | Case status (e.g., "Action Required") |
| `severity` | string | Alert severity label (e.g., "Account Takeover") |
| `severity_level` | string | Severity level for alert mapping |
| `affectedEmployee` | string | Email address of the affected user |
| `firstObserved` | timestamp | Time the anomaly was first detected |
| `analysis` | string | Analyst-facing description |
| `genai_summary` | string | AI-generated summary of the alert |

**Example Query — Recent high-severity cases:**
```
dataset = abnormal_security_generic_alert_raw
| filter case_status = "Action Required"
| fields caseId, severity, affectedEmployee, analysis, genai_summary, firstObserved
| sort firstObserved desc
| limit 50
```

---

### thinkst_canary_generic_alert_raw

Alert data from Thinkst Canary honeypot tokens and devices. Alert details
are stored in the `_alert_data` JSON blob — use arrow notation to extract fields.

**Usage:**
```
dataset = thinkst_canary_generic_alert_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `summary` | string | Short alert description |
| `description` | string | Full alert description |
| `_alert_data` | JSON | Raw alert payload — use `->` notation to extract fields |
| `_id` | string | XSIAM system event ID — use for `dedup _id` |

**Arrow Notation Access Patterns:**
```
dataset = thinkst_canary_generic_alert_raw
| alter source_ip = _alert_data -> sourceip,
       canary_name = _alert_data -> canaryname,
       token_type = _alert_data -> token_type,
       memo = _alert_data -> memo
| fields summary, source_ip, canary_name, token_type, memo, _time
```

**Common Use Cases:**
- Canary token triggers (credential theft, file access, DNS lookups)
- Honeytoken-based insider threat detection
- Dedup by `_id` to avoid double-counting repeated triggers

**Example Query — Unique source IPs triggering canaries:**
```
dataset = thinkst_canary_generic_alert_raw
| dedup _id
| alter source_ip = _alert_data -> sourceip
| comp count() as trigger_count by source_ip, summary
| sort trigger_count desc
| limit 25
```

---

### darktrace_darktrace_raw

Model breach and alert data from Darktrace network AI. Must be queried via
`datamodel dataset =` to access XDM fields. Raw columns like `percentscore`
and `triggeredComponents` require explicit aliasing.

**Usage:**
```
datamodel dataset = darktrace_darktrace_raw
```

**Note:** Use `fields *, darktrace_darktrace_raw.percentscore as percentscore`
to pull raw columns alongside XDM fields.

**Key Fields (raw — access via aliasing):**

| Field | Type | Description |
|-------|------|-------------|
| `percentscore` | number | Risk score 0-100 |
| `triggeredComponents` | JSON array | Components that triggered the model breach |

**Arrow Notation for triggeredComponents:**

The `triggeredComponents` field is a JSON array. Use `arraymap` or `arrayfilter`
combined with arrow notation to extract individual component details:

```
datamodel dataset = darktrace_darktrace_raw
| fields *, darktrace_darktrace_raw.percentscore as percentscore,
         darktrace_darktrace_raw.triggeredComponents as triggeredComponents
| filter percentscore > 75
| alter component_names = arraymap(triggeredComponents, "@element" -> component)
| fields xdm.alert.name, percentscore, component_names, _time
| sort percentscore desc
| limit 50
```

---

### google_workspace_alerts_raw

Alert data from the Google Workspace Alert Center. Alert payloads and metadata
are stored in JSON fields that require arrow notation to extract.

**Usage:**
```
dataset = google_workspace_alerts_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `alertId` | string | Unique alert identifier |
| `type` | string | Alert type (e.g., "User suspended", "Government-backed attack") |
| `source` | string | Alert source |
| `data` | JSON | Alert payload — use `data -> email`, `data -> loginDetails` etc. |
| `metadata` | JSON | Alert metadata — use `metadata -> severity`, `metadata -> status` etc. |
| `securityInvestigationToolLink` | string | Link to Google Security Investigation Tool |

**Arrow Notation Access Patterns:**
```
dataset = google_workspace_alerts_raw
| alter alert_email = data -> email,
       alert_severity = metadata -> severity,
       alert_status = metadata -> status
| fields alertId, type, source, alert_email, alert_severity, alert_status, _time
| sort _time desc
```

**Example Query — Government-backed attack alerts:**
```
dataset = google_workspace_alerts_raw
| filter type = "Government-backed attack"
| alter affected_user = data -> email
| fields alertId, affected_user, source, _time
```

---

## Email Datasets

### microsoft_office_365_raw

Microsoft Office 365 audit logs covering Exchange, SharePoint, Teams, OneDrive,
and other O365 workloads. Use the `workload` field to narrow to a specific service.

**Usage:**
```
dataset = microsoft_office_365_raw
```

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `_time` | timestamp | Event timestamp (UTC) |
| `workload` | string | Office 365 workload (Exchange, SharePoint, Teams, OneDrive, AzureActiveDirectory) |
| `operation` | string | Type of operation (e.g., "MailboxLogin", "FileAccessed", "MemberAdded") |
| `organization_id` | string | Organization identifier |
| `user_id` | string | User ID performing action |
| `user_name` | string | User display name |
| `ip_address` | string | Client IP address |
| `mailbox_owner_upn` | string | Mailbox owner UPN (Exchange events) |
| `mailbox_audit_enable` | string | Mailbox audit status |
| `external_access` | string | External access indicator |
| `object_id` | string | Object being modified |
| `object_type` | string | Type of object |
| `parameters` | JSON | Operation parameters — use `parameters -> Name`, `parameters -> Value` for details |
| `user_agent` | string | User agent |
| `client_ip_address` | string | Client IP |
| `succeeded` | string | Operation success status |
| `result_status` | string | Result status |
| `resource_name` | string | Resource name |
| `custom_properties` | JSON | Custom properties |

**Common Use Cases:**
- Monitor mailbox access patterns and forwarding rule changes
- Track sensitive file sharing in SharePoint/OneDrive
- Detect data exfiltration via email (large attachment sends, external forwarding)
- Audit administrator actions across O365 workloads

**Example Query — Suspicious mailbox forwarding rules:**
```
dataset = microsoft_office_365_raw
| filter workload = "Exchange"
| filter operation in ("Set-Mailbox", "New-InboxRule", "Set-InboxRule")
| fields user_id, operation, parameters, mailbox_owner_upn, ip_address, _time
| sort _time desc
| limit 100
```

**Example Query — External file sharing in SharePoint:**
```
dataset = microsoft_office_365_raw
| filter workload = "SharePoint"
| filter operation = "SharingSet" or operation = "AnonymousLinkCreated"
| fields user_id, object_id, operation, _time
| sort _time desc
```

---

### email_gateway_raw

Email gateway logs from mail security appliances. Contains message metadata,
attachment details, threat verdicts, and URLs extracted from messages.

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
| `subject` | string | Email subject line |
| `message_id` | string | Unique message ID |
| `message_size` | integer | Size of message in bytes |
| `attachment_count` | integer | Number of attachments |
| `attachment_names` | array | Names of attachments |
| `attachment_types` | array | Types of files attached |
| `attachment_hashes` | array | Hashes of attachments |
| `action` | string | Gateway action (allow, quarantine, drop, block) |
| `threat_name` | string | Detected threat name |
| `threat_category` | string | Threat category (malware, phishing, spam, BEC) |
| `severity` | string | Severity level |
| `source_ip` | string | Sender IP address |
| `source_hostname` | string | Sender hostname |
| `destination_ip` | string | Recipient server IP |
| `url_count` | integer | Number of URLs in email |
| `urls` | array | URLs found in email |
| `headers` | JSON | Email headers — use `headers -> "X-Originating-IP"` etc. |
| `body_text` | string | Email body (if available) |

**Common Use Cases:**
- Monitor inbound email threats (phishing, malware attachments)
- Detect phishing campaigns by correlating sender/subject/URL patterns
- Track malware distribution via email attachments
- Investigate compromised sender accounts sending outbound spam

**Example Query — Blocked phishing with URLs:**
```
dataset = email_gateway_raw
| filter threat_category = "phishing"
| filter action in ("quarantine", "drop", "block")
| filter url_count > 0
| fields sender, recipient, subject, urls, threat_name, _time
| sort _time desc
| limit 100
```

**Example Query — Suspicious attachment types:**
```
dataset = email_gateway_raw
| filter attachment_count > 0
| arrayexpand attachment_types
| filter attachment_types in ("exe", "scr", "js", "vbs", "bat", "ps1", "hta", "iso", "img")
| fields sender, recipient, subject, attachment_names, attachment_types, action, _time
| sort _time desc
```

**Extracting Header Fields (JSON arrow notation):**
```
dataset = email_gateway_raw
| alter originating_ip = headers -> "X-Originating-IP",
       auth_results = headers -> "Authentication-Results"
| fields sender, recipient, originating_ip, auth_results
```

---

## Identity Deep-Dive

### pan_dss_raw (Cloud Identity Engine)

Raw identity data from the Palo Alto Cloud Identity Engine (CIE). Contains
Active Directory user and computer records synchronized from on-premise AD.
Refreshed daily.

**Usage:**
```
dataset = pan_dss_raw
```

#### User Records

Filter to user records with `source = "ad" and type = "user"`.

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `sam_account_name` | string | SAM account name (short username) |
| `netbios` | string | NetBIOS domain name |
| `display_name` | string | Full display name |
| `email` | string | User email address |
| `security_groups` | array | List of group CNs the user belongs to |
| `dn` | string | Distinguished name (full LDAP path) |
| `ou` | string | Organizational unit path |
| `department` | string | Department name |
| `security_identifier` | string | Windows SID |

**Example Query — List all Domain Admin users:**
```
dataset = pan_dss_raw
| filter source = "ad" and type = "user"
| filter array_any(security_groups, "@element" ~= "^cn=Domain Admins")
| fields sam_account_name, display_name, email, dn
```

**Example Query — Users in a specific OU:**
```
dataset = pan_dss_raw
| filter source = "ad" and type = "user"
| filter ou contains "OU=Finance"
| fields sam_account_name, display_name, department, security_groups
```

#### Computer Records

Filter to computer records with `source = "ad" and type = "computer"`.

**Key Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Computer short name |
| `dns_host_name` | string | Fully qualified DNS hostname |
| `dn` | string | Distinguished name |
| `ou` | array | Organizational unit path (index 0 = immediate OU) |

**Example Query — All computers excluding Domain Controllers:**
```
dataset = pan_dss_raw
| filter source = "ad" and type = "computer"
| filter arrayindex(ou, 0) != "Domain Controllers"
| fields name, dns_host_name, dn
```

#### Join Patterns for Enrichment

**Join to xdr_data by username — enrich endpoint events with AD group membership:**
```
dataset = xdr_data
| filter action_type = "process_launch"
| join type=left (
    dataset = pan_dss_raw
    | filter source = "ad" and type = "user"
    | fields sam_account_name, security_groups, department
  ) as identity
  user_name = identity.sam_account_name
| fields agent_hostname, user_name, action_process_image_name,
        identity.security_groups, identity.department
```

**Join to endpoints by hostname — enrich identity records with endpoint status:**
```
dataset = pan_dss_raw
| filter source = "ad" and type = "computer"
| join type=left (
    dataset = endpoints
    | fields endpoint_name, agent_status, last_seen
  ) as ep
  dns_host_name = ep.endpoint_name
| fields name, dns_host_name, ep.agent_status, ep.last_seen
```

**Join to microsoft_windows_raw — correlate auth events with AD group membership:**
```
dataset = microsoft_windows_raw
| filter event_id = 4624
| join type=left (
    dataset = pan_dss_raw
    | filter source = "ad" and type = "user"
    | fields sam_account_name, security_groups
  ) as ad
  target_user_name = ad.sam_account_name
| filter array_any(ad.security_groups, "@element" ~= "^cn=Domain Admins")
| fields target_user_name, ad.security_groups, src_ip, _time
```

#### Preset: ad_users

The `ad_users` preset is a named view over `pan_dss_raw` that surfaces
Active Directory user records from CIE.

```
preset = ad_users
```

**Key difference:** The preset uses `security_group_list` (not `security_groups`
as on `pan_dss_raw`). When filtering by group membership in the preset:

```
preset = ad_users
| filter array_any(security_group_list, "@element" contains "Domain Admins")
| fields sam_account_name, display_name, email, security_group_list
```

---

### msft_azure_ad_raw

Sign-in and audit log data from Microsoft Azure Active Directory. Query via
`datamodel dataset =` to access XDM-mapped fields. Use field aliasing to
surface raw columns not mapped to XDM.

**Usage:**
```
datamodel dataset = msft_azure_ad_raw
```

**Important:** The correct dataset name is `msft_azure_ad_raw` — not
`microsoft_azure_ad_raw`. The latter is a separate dataset for general
Azure AD audit/directory events.

Use `fields *, msft_azure_ad_raw.ipAddress as ipaddress` to surface the
raw `ipAddress` field alongside XDM fields.

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

**Example Query — Failed sign-ins with raw IP:**
```
datamodel dataset = msft_azure_ad_raw
| fields *, msft_azure_ad_raw.ipAddress as ipaddress
| filter xdm.event.outcome = "FAILURE"
| comp count() as failures by xdm.source.user.username, ipaddress
| filter failures > 5
| sort failures desc
```

---

## Custom-Ingested Data

### xdr_http_collector

Data posted to the XSIAM HTTP collector endpoint is queryable via XQL
in the `xdr_http_collector` dataset. This is the primary mechanism for
ingesting custom, arbitrary JSON data into XSIAM for querying and
correlation rule evaluation.

**Usage:**
```
dataset = xdr_http_collector
```

**How Data Gets There:**

The HTTP collector endpoint accepts JSON (or gzipped JSON) via POST:

```
POST https://api-<tenant>.xdr.<region>.paloaltonetworks.com/logs/v1/event
Authorization: <api-key>
Content-Type: application/json
```

The payload is newline-delimited JSON (NDJSON) — one JSON object per line.
Each object becomes a row in `xdr_http_collector`.

**Key Characteristics:**

- Fields are dynamic — whatever keys are in the posted JSON become queryable columns
- No predefined schema; field names come directly from the posted data
- Data can be posted from integrations, scripts, external tools, or any HTTP client
- Posted data can trigger correlation rules for automated alerting

**Example Query — Query custom-posted data:**
```
dataset = xdr_http_collector
| fields src_ip, action, severity, _time
| sort _time desc
| limit 100
```

**Example Query — Aggregate custom events by source:**
```
dataset = xdr_http_collector
| comp count() as event_count by action
| sort event_count desc
```

**Integration Pattern:**

The URL is constructed from the tenant context at runtime:

```python
url = demisto.demistoUrls()['server'].replace(
    'https://', 'https://api-'
) + '/logs/v1/event'
```

See the `JP-XQL HTTP Collector_v2` integration for a full implementation
example with JSON and gzip posting modes.

---

## Cold Storage

### Overview

XSIAM uses a two-tier storage model:

| Tier | Query Syntax | Performance | Use Case |
|------|-------------|-------------|----------|
| **Hot storage** | `dataset = <name>` | Fast, near real-time | Active investigation, alerting, dashboards |
| **Cold storage** | `cold_dataset = <name>` | Slower, requires compute units (CU) | Historical lookbacks, compliance, audits |

Hot storage is the default tier for all ingested data. Cold storage provides
long-term retention at lower cost, queryable on demand.

### Querying Cold Storage

Use the `cold_dataset` keyword instead of `dataset`:

```
cold_dataset = panw_ngfw_traffic_raw
| filter src = "10.0.1.50"
| fields src, dst, dport, action, _time
| sort _time desc
| limit 1000
```

### Combining Hot and Cold Storage

You can query both tiers in the same XQL statement to span the full
retention window. Hot storage covers recent data; cold storage covers
the historical tail:

```
dataset = panw_ngfw_traffic_raw
| union (cold_dataset = panw_ngfw_traffic_raw)
| filter src = "10.0.1.50"
| fields src, dst, dport, action, _time
| sort _time desc
```

### Archived Data (Historical Imports)

Cortex XSIAM can import historical data from third-party sources into cold
storage as archived datasets. Each imported data source becomes a cold storage
dataset named `archive_<dataset name>`.

**Import process:**
1. Extract data from third-party sources
2. Format data per XSIAM requirements (recommended format for successful querying)
3. Send files to XSIAM via HTTP collector
4. Data lands in cold storage as an archived dataset

**Querying archived data:**
```
cold_dataset = archive_legacy_siem_data
| fields src_ip, dst_ip, event_type, _time
| sort _time desc
```

**Requirements:**
- Period-Based Retention - Cold Storage add-on license
- View/Edit RBAC permission for Data Management

### Retention and Storage Tiers

| Aspect | Detail |
|--------|--------|
| **Hot storage retention** | Configurable per dataset; depends on license tier |
| **Cold storage retention** | Extended retention period; requires cold storage license add-on |
| **Additional hot storage** | Flexible add-on (minimum 1,000 GB); configurable priority (Low, Medium, High) |
| **Retention enforcement** | Applied to all log-type datasets except Host Inventory, Vulnerability Assessment, Metrics, Users |

**Checking retention status:**
- Dataset Management page shows HOT RANGE, COLD RANGE, and TOTAL DAYS STORED per dataset
- Settings > Cortex XSIAM License shows overall license details

### Performance Considerations

- Cold storage queries consume compute units (CU) — use filters and time ranges to minimize cost
- Always filter early with specific time ranges when querying cold storage
- Select only needed fields before aggregation
- Cold storage queries are significantly slower than hot storage — not suitable for dashboards or real-time alerting
- Use hot storage for active investigation; reserve cold storage for historical lookbacks and compliance

---

## Dataset Refresh Intervals

Understanding refresh timing is important when querying identity and inventory datasets:

| Dataset | Refresh Interval |
|---------|-----------------|
| Most system datasets | Near real-time |
| `endpoints` | Every hour |
| `pan_dss_raw` | Daily |
| Forensics datasets | Configurable (default: one-time snapshot; scheduled: minimum 12 hours) |

---

## Cross-Dataset Join Patterns (Extended)

### Enrich Abnormal Security alerts with AD identity

```
dataset = abnormal_security_generic_alert_raw
| filter case_status = "Action Required"
| join type=left (
    dataset = pan_dss_raw
    | filter source = "ad" and type = "user"
    | fields email, sam_account_name, security_groups, department
  ) as identity
  affectedEmployee = identity.email
| fields caseId, severity, affectedEmployee, identity.department,
        identity.security_groups, analysis
```

### Correlate email gateway threats with endpoint activity

```
dataset = email_gateway_raw
| filter threat_category = "phishing" and action = "allow"
| join type=inner (
    dataset = xdr_data
    | filter action_type = "url_access"
    | fields agent_hostname, user_name, url, _time
  ) as endpoint
  arraystring(urls, ",") contains endpoint.url
| fields sender, recipient, subject, endpoint.agent_hostname, endpoint.url, _time
```

### Enrich O365 audit events with endpoint metadata

```
dataset = microsoft_office_365_raw
| filter workload = "Exchange"
| join type=left (
    dataset = endpoints
    | fields endpoint_name, ip, agent_status
  ) as ep
  client_ip_address = ep.ip
| fields user_id, operation, client_ip_address, ep.endpoint_name, ep.agent_status, _time
```
