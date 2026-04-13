# Correlation Rule Examples

Complete, production-ready correlation rule examples in JSON export format. Each example
demonstrates a different execution mode and severity pattern, matching the exact schema
XSIAM uses when exporting correlation rules via API.

All examples use the canonical 29-field order that XSIAM exports produce.

---

## Example 1: Real-Time Single Event (Canary Alert)

**Use case:** Thinkst Canary honeypot trigger -- any interaction with a canary device
is inherently suspicious and warrants an immediate high-severity alert.

**Key patterns:**
- `execution_mode: REAL_TIME` -- every matching event fires an alert
- Static severity -- canary alerts are always high-priority
- Simple XQL -- filter and extract, no aggregation (REAL_TIME constraint)
- Suppression by `alert_name` to collapse repeated triggers from the same canary

```json
[
  {
    "rule_id": 101,
    "name": "Canary - Honeypot Interaction Detected",
    "severity": "SEV_020_HIGH",
    "xql_query": "dataset = thinkst_canary_generic_alert_raw\n| filter summary not in (\"Multiple Canaries Disconnected\")\n| alter source_ip = _alert_data -> sourceip,\n       canary_name = _alert_data -> devicename,\n       canary_port = _alert_data -> dst_port,\n       event_description = _alert_data -> description\n| fields _time, summary, source_ip, canary_name, canary_port, event_description",
    "is_enabled": true,
    "description": "Detects any interaction with a Thinkst Canary honeypot device. Canary triggers indicate unauthorized network reconnaissance or lateral movement.",
    "alert_name": "Canary - Honeypot Interaction Detected",
    "alert_category": "OTHER",
    "alert_type": null,
    "alert_description": "Observed interaction with honeypot device indicating potential reconnaissance or lateral movement",
    "alert_domain": "DOMAIN_SECURITY",
    "alert_fields": {},
    "execution_mode": "REAL_TIME",
    "search_window": null,
    "simple_schedule": null,
    "timezone": null,
    "crontab": null,
    "suppression_enabled": true,
    "suppression_duration": "1 hours",
    "suppression_fields": ["alert_name"],
    "dataset": "alerts",
    "user_defined_severity": null,
    "user_defined_category": null,
    "mitre_defs": {
      "TA0043 - Reconnaissance": ["T1595 - Active Scanning"]
    },
    "investigation_query_link": "",
    "drilldown_query_timeframe": "ALERT",
    "mapping_strategy": "AUTO",
    "action": "ALERTS",
    "lookup_mapping": null
  }
]
```

### Design Decisions

- **Execution mode:** REAL_TIME because every canary interaction is independently alertable -- honeypots should never receive legitimate traffic, so any single event warrants an alert without needing aggregation.
- **Severity pattern:** Static SEV_020_HIGH because canary interactions are suspicious by definition. No need for dynamic computation -- the presence of the event is the signal.
- **Suppression:** 1 hour by `alert_name` so repeated triggers from the same canary device within the window are treated as one incident rather than flooding the queue.
- **MITRE mapping:** TA0043 Reconnaissance / T1595 Active Scanning because interacting with a honeypot indicates the attacker is probing the network to discover live hosts and services.

---

## Example 2: Scheduled Threshold (Failed Auth Spike)

**Use case:** Detect password spray or brute force attacks by identifying a single
source IP failing authentication against 5 or more distinct user accounts within
a 15-minute window.

**Key patterns:**
- `execution_mode: SCHEDULED` with crontab and search_window
- Aggregation with `comp count_distinct` and threshold `filter`
- search_window (15 minutes) is 3x the cron interval (5 minutes) to avoid gaps
- Suppression by `source_ip` to collapse repeated sprays from the same attacker

```json
[
  {
    "rule_id": 102,
    "name": "Authentication - Failed Auth Spike from Single Source",
    "severity": "SEV_030_MEDIUM",
    "xql_query": "dataset = xdr_data\n| filter event_type = ENUM.EVENT_LOG\n  and action_evtlog_event_id = 4625\n  and agent_hostname != null\n| alter target_user = arrayindex(\n    regextract(action_evtlog_message, \"Account Name:.*?(\\w.*)\\r\\n\"), 0\n  ),\n  source_ip = arrayindex(\n    regextract(action_evtlog_message, \"Source Network Address:.*?(\\S+)\\r\\n\"), 0\n  )\n| filter target_user != null\n  and target_user not contains \"$\"\n  and source_ip != null\n  and source_ip != \"-\"\n| comp count_distinct(target_user) as distinct_users,\n      count() as total_failures\n  by source_ip\n| filter distinct_users >= 5\n| sort desc distinct_users\n| fields source_ip, distinct_users, total_failures",
    "is_enabled": true,
    "description": "Detects 5 or more distinct user accounts failing authentication from the same source IP within a 15-minute window.",
    "alert_name": "Authentication - Failed Auth Spike from Single Source",
    "alert_category": "OTHER",
    "alert_type": null,
    "alert_description": null,
    "alert_domain": "DOMAIN_SECURITY",
    "alert_fields": {},
    "execution_mode": "SCHEDULED",
    "search_window": "15 minutes",
    "simple_schedule": "5 minutes",
    "timezone": "UTC",
    "crontab": "*/5 * * * *",
    "suppression_enabled": true,
    "suppression_duration": "30 minutes",
    "suppression_fields": ["source_ip"],
    "dataset": "alerts",
    "user_defined_severity": null,
    "user_defined_category": null,
    "mitre_defs": {
      "TA0006 - Credential Access": ["T1110 - Brute Force", "T1110.003 - Password Spraying"]
    },
    "investigation_query_link": "",
    "drilldown_query_timeframe": "ALERT",
    "mapping_strategy": "AUTO",
    "action": "ALERTS",
    "lookup_mapping": null
  }
]
```

### Design Decisions

- **Execution mode:** SCHEDULED because this detection requires aggregation (`comp count_distinct`) across multiple events within a time window. REAL_TIME cannot perform aggregation.
- **Severity pattern:** Static SEV_030_MEDIUM because threshold-based detections can produce false positives (e.g., misconfigured service accounts). Medium ensures triage within SLA without excessive escalation.
- **Suppression:** 30 minutes by `source_ip` since the same attacker IP spraying credentials is a single incident. The duration is 2x the search_window to prevent duplicate alerts across consecutive runs.
- **MITRE mapping:** TA0006 Credential Access with both T1110 Brute Force (general) and T1110.003 Password Spraying (specific sub-technique) because 5+ distinct users from one source is the hallmark spray pattern.

---

## Example 3: Real-Time with User-Defined Severity (Vendor Alert Forwarding)

**Use case:** Forward Microsoft Graph API security alerts into XSIAM with severity
and category inherited from the source alert fields. The vendor already classifies
alerts, so XSIAM should respect those classifications rather than overriding them.

**Key patterns:**
- `execution_mode: REAL_TIME` -- forward each vendor alert as it arrives
- `severity: "User Defined"` with `user_defined_severity` pointing to XDM field
- `alert_category: "User Defined"` with `user_defined_category` pointing to XDM field
- `alert_fields` mapping XDM fields to XSIAM alert columns for enrichment
- Dollar-sign field references (`$xdm.event.description`) for dynamic alert metadata

```json
[
  {
    "rule_id": 13,
    "name": "BSW - Microsoft Graph API Alerts",
    "severity": "User Defined",
    "xql_query": "datamodel dataset = msft_graph_security_alerts_raw\n| filter xdm.alert.description != null\n| alter xdm.source.host.hostname = xdm.source.host.fqdn\n| fields *",
    "is_enabled": true,
    "description": null,
    "alert_name": "$xdm.event.description",
    "alert_category": "User Defined",
    "alert_type": null,
    "alert_description": "$xdm.alert.description",
    "alert_domain": "DOMAIN_SECURITY",
    "alert_fields": {
      "email": "xdm.source.user.upn",
      "accountid": "xdm.source.user.upn",
      "fw_email_subject": "xdm.email.subject",
      "eventdescriptions": "xdm.event.description",
      "fw_email_recipient": "xdm.email.recipients"
    },
    "execution_mode": "REAL_TIME",
    "search_window": null,
    "simple_schedule": null,
    "timezone": null,
    "crontab": null,
    "suppression_enabled": false,
    "suppression_duration": null,
    "suppression_fields": null,
    "dataset": "alerts",
    "user_defined_severity": "xdm.alert.severity",
    "user_defined_category": "xdm.alert.subcategory",
    "mitre_defs": {},
    "investigation_query_link": "",
    "drilldown_query_timeframe": "ALERT",
    "mapping_strategy": "AUTO",
    "action": "ALERTS",
    "lookup_mapping": []
  }
]
```

### Design Decisions

- **Execution mode:** REAL_TIME because each Microsoft Graph API alert should be forwarded immediately as it arrives. No aggregation or batching needed.
- **Severity pattern:** User Defined because Microsoft already classifies alert severity. The `user_defined_severity` field points to `xdm.alert.severity` so XSIAM inherits the vendor classification rather than applying a static override.
- **Suppression:** Disabled because each vendor alert is a distinct event that should generate its own XSIAM alert. Suppression would risk hiding different security events that happen to arrive in quick succession.
- **MITRE mapping:** Empty because the vendor alerts span many different techniques. MITRE mapping is best applied per-alert by the vendor classification rather than blanket-assigned at the correlation rule level.

---

## Example 4: Scheduled with Dynamic Severity (Third-Party Score)

**Use case:** Darktrace model breach alerts with severity derived from the numeric
`percentscore` field. Darktrace does not emit a discrete severity -- the score IS
the severity signal, so dynamic computation via `if()` in XQL is appropriate.

**Key patterns:**
- `execution_mode: SCHEDULED` with hourly runs and dedup
- Dynamic `xdm.alert.severity` computed via `if()` chain in XQL (87-91 = MEDIUM, 92-99 = HIGH, 100 = CRITICAL)
- `severity: "User Defined"` with `user_defined_severity` pointing to the computed field
- `datamodel dataset` with raw field aliasing for nested Darktrace structures
- Complex array operations (`arraymap`, `arrayfilter`, `arraymerge`) to extract triggered components

```json
[
  {
    "rule_id": 103,
    "name": "Darktrace - Critical Model Breach",
    "severity": "User Defined",
    "xql_query": "datamodel dataset = darktrace_darktrace_raw\n| fields *, darktrace_darktrace_raw.percentscore as percentscore,\n          darktrace_darktrace_raw.triggeredComponents as triggeredComponents\n| sort desc _time\n| dedup xdm.event.id\n| filter percentscore >= 87\n| filter xdm.alert.original_threat_name not in (\n    \"Antigena::Network::Significant Anomaly::Antigena Significant Anomaly from Client Block\",\n    \"Antigena::Network::Insider Threat::Antigena Large Data Volume Outbound Block\"\n  )\n| alter connection_hostname = arraydistinct(arraymap(\n    arrayfilter(\n      arraymerge(arraymap(triggeredComponents -> [], \"@element\" -> triggeredFilters{})),\n      \"@element\" -> filterType = \"Connection hostname\"\n    ),\n    \"@element\" -> trigger.value\n  )),\n  dns_lookup = arraydistinct(arraymap(\n    arrayfilter(\n      arraymerge(arraymap(triggeredComponents -> [], \"@element\" -> triggeredFilters{})),\n      \"@element\" -> filterType = \"DNS host lookup\"\n    ),\n    \"@element\" -> trigger.value\n  ))\n| alter connection_hostname = if(array_length(connection_hostname) = 0, null, connection_hostname)\n| alter xdm.target.domain = coalesce(connection_hostname, dns_lookup)\n| alter xdm.alert.severity = if(\n    percentscore >= 87 and percentscore <= 91, \"MEDIUM\",\n    percentscore >= 92 and percentscore <= 99, \"HIGH\",\n    percentscore = 100, \"CRITICAL\",\n    \"LOW\"\n  )\n| fields _time, xdm.alert.severity, xdm.alert.original_threat_name,\n         xdm.alert.description, xdm.event.id, xdm.source.host.hostname,\n         xdm.source.host.ipv4_addresses, xdm.target.domain, percentscore",
    "is_enabled": true,
    "description": "Surfaces Darktrace model breaches with a percent score of 87 or higher. Severity is dynamically computed from the score.",
    "alert_name": "Darktrace - Critical Model Breach",
    "alert_category": "OTHER",
    "alert_type": null,
    "alert_description": null,
    "alert_domain": "DOMAIN_SECURITY",
    "alert_fields": {},
    "execution_mode": "SCHEDULED",
    "search_window": "1 hours",
    "simple_schedule": "1 hour",
    "timezone": "UTC",
    "crontab": "*/60 * * * *",
    "suppression_enabled": true,
    "suppression_duration": "2 hours",
    "suppression_fields": ["xdm.alert.original_threat_name"],
    "dataset": "alerts",
    "user_defined_severity": "xdm.alert.severity",
    "user_defined_category": null,
    "mitre_defs": {
      "TA0011 - Command and Control": ["T1071 - Application Layer Protocol"]
    },
    "investigation_query_link": "",
    "drilldown_query_timeframe": "ALERT",
    "mapping_strategy": "AUTO",
    "action": "ALERTS",
    "lookup_mapping": null
  }
]
```

### Design Decisions

- **Execution mode:** SCHEDULED because Darktrace events require deduplication (`dedup xdm.event.id`) and batch processing. Running hourly with a 1-hour search window ensures complete coverage without real-time overhead.
- **Severity pattern:** User Defined with `xdm.alert.severity` computed dynamically in the XQL via an `if()` chain. Darktrace provides a numeric `percentscore` but no discrete severity field, so the query maps score ranges to MEDIUM/HIGH/CRITICAL thresholds.
- **Suppression:** 2 hours by `xdm.alert.original_threat_name` because the same model breach type repeating is likely the same underlying issue. Different threat names from different devices still generate separate alerts.
- **MITRE mapping:** TA0011 Command and Control / T1071 Application Layer Protocol as a general mapping for Darktrace anomaly-based detections. In production, refine per specific model breach type.
