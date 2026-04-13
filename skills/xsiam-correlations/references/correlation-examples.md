# Correlation Rule Examples

Complete, production-ready `.yml` correlation rule examples. Each example demonstrates
a different execution mode and severity pattern. Inline `# ANNOTATION:` comments explain
design choices.

---

## Example 1: Real-Time Single Event (Canary Alert)

**Use case:** Thinkst Canary honeypot trigger -- any interaction with a canary device
is inherently suspicious and warrants an immediate high-severity alert.

**Key patterns:**
- `execution_mode: REAL_TIME` -- every matching event fires an alert
- Static severity -- canary alerts are always high-priority
- Simple XQL -- filter and extract, no aggregation needed

```yaml
# ANNOTATION: Fresh UUID v4 for this rule
global_rule_id: "a1b2c3d4-5e6f-4a7b-8c9d-0e1f2a3b4c5d"

# ANNOTATION: snake_case rule identifier following vendor_detection convention
rule_id: "canary_honeypot_alert"

rule_name: "Canary - Honeypot Interaction Detected"

rule_labels:
  - "honeypot"
  - "canary"
  - "deception"

rule_source: "xsiam-buddy"

# ANNOTATION: name and alert_name must be identical
name: "Canary - Honeypot Interaction Detected"
alert_name: "Canary - Honeypot Interaction Detected"

description: >
  Detects any interaction with a Thinkst Canary honeypot device. Canary triggers
  indicate unauthorized network reconnaissance or lateral movement, as legitimate
  users and systems should never interact with honeypot devices. Investigate the
  source IP and user immediately.

# ANNOTATION: Always "alerts" for correlation rules
dataset: alerts

# ANNOTATION: Always "ALERTS" for correlation rules
action: ALERTS

alert_domain: NETWORK
alert_category: "Reconnaissance"

# ANNOTATION: REAL_TIME because every canary interaction is independently alertable
execution_mode: REAL_TIME

# ANNOTATION: Short search window for real-time; just enough for event processing
search_window: "1m"

# ANNOTATION: No crontab for REAL_TIME rules

# ANNOTATION: Detection logic only -- alert metadata lives in the YAML wrapper
xql_query: |
  dataset = thinkst_canary_generic_alert_raw
  | filter summary not in ("Multiple Canaries Disconnected")
  | alter source_ip = _alert_data -> sourceip,
         canary_name = _alert_data -> devicename,
         canary_port = _alert_data -> dst_port,
         event_description = _alert_data -> description
  | fields _time, summary, source_ip, canary_name, canary_port, event_description

# ANNOTATION: Static HIGH -- any canary trigger is suspicious by definition
severity: SEV_020_HIGH

suppression_enabled: true

# ANNOTATION: 1h suppression -- same canary triggering repeatedly is the same incident
suppression_duration: "1h"

# ANNOTATION: Suppress by alert_name so repeated triggers from the same canary
# device within the window are treated as one incident
suppression_fields:
  - "alert_name"

# ANNOTATION: Canary interactions map to Reconnaissance (attacker scanning the network)
mitre_defs:
  "T1595":
    - "TA0043"
```

---

## Example 2: Scheduled Threshold (Failed Auth Spike)

**Use case:** Detect password spray or brute force attacks by identifying a single
source IP failing authentication against 5 or more distinct user accounts within
a 15-minute window.

**Key patterns:**
- `execution_mode: SCHEDULED` with crontab and search_window
- Aggregation with `comp count_distinct` and threshold `filter`
- search_window (15m) is 3x the cron interval (5m) to avoid gaps

```yaml
global_rule_id: "b2c3d4e5-6f7a-4b8c-9d0e-1f2a3b4c5d6e"

rule_id: "custom_failed_auth_spike"

rule_name: "Authentication - Failed Auth Spike from Single Source"

rule_labels:
  - "brute_force"
  - "credential_access"
  - "authentication"

rule_source: "xsiam-buddy"

name: "Authentication - Failed Auth Spike from Single Source"
alert_name: "Authentication - Failed Auth Spike from Single Source"

description: >
  Detects 5 or more distinct user accounts failing authentication from the same
  source IP within a 15-minute window. This pattern indicates password spraying
  or brute force attacks. Investigate the source IP for compromise indicators
  and consider blocking at the firewall.

dataset: alerts
action: ALERTS

alert_domain: IDENTITY
alert_category: "Credential Access"

# ANNOTATION: SCHEDULED because this detection requires aggregation across multiple events
execution_mode: SCHEDULED

# ANNOTATION: 15m lookback covers the detection window
search_window: "15m"

# ANNOTATION: Run every 5 minutes; search_window (15m) is 3x interval to avoid gaps
crontab: "*/5 * * * *"

# ANNOTATION: Aggregation-based detection: count distinct users per source IP,
# then filter to sources exceeding the threshold
xql_query: |
  dataset = xdr_data
  | filter event_type = ENUM.EVENT_LOG
    and action_evtlog_event_id = 4625
    and agent_hostname != null
  | alter target_user = arrayindex(
      regextract(action_evtlog_message, "Account Name:.*?(\w.*)\r\n"), 0
    ),
    source_ip = arrayindex(
      regextract(action_evtlog_message, "Source Network Address:.*?(\S+)\r\n"), 0
    )
  | filter target_user != null
    and target_user not contains "$"
    and source_ip != null
    and source_ip != "-"
  | comp count_distinct(target_user) as distinct_users,
        count() as total_failures
    by source_ip
  | filter distinct_users >= 5
  | sort desc distinct_users
  | fields source_ip, distinct_users, total_failures

# ANNOTATION: MEDIUM because threshold-based detections can have false positives
# (e.g., misconfigured service accounts); triage within SLA
severity: SEV_030_MEDIUM

suppression_enabled: true

# ANNOTATION: 30m suppression -- same source IP spraying = same incident
suppression_duration: "30m"

# ANNOTATION: Suppress by source_ip since the same attacker IP is one incident
suppression_fields:
  - "source_ip"

# ANNOTATION: T1110 = Brute Force, T1110.003 = Password Spraying
# TA0006 = Credential Access
mitre_defs:
  "T1110":
    - "TA0006"
  "T1110.003":
    - "TA0006"
```

---

## Example 3: Scheduled with Dynamic Severity (Third-Party Score)

**Use case:** Darktrace model breach alerts with severity derived from the numeric
`percentscore` field. Darktrace does not emit a discrete severity -- the score IS
the severity signal, so dynamic computation via `if()` in XQL is appropriate.

**Key patterns:**
- `execution_mode: SCHEDULED` with hourly runs
- Dynamic `xdm.alert.severity` via `if()` chain in XQL
- Static fallback severity in YAML (`SEV_040_LOW`)
- `datamodel dataset` with raw field aliasing

```yaml
global_rule_id: "c3d4e5f6-7a8b-4c9d-0e1f-2a3b4c5d6e7f"

rule_id: "darktrace_critical_model_breach"

rule_name: "Darktrace - Critical Model Breach"

rule_labels:
  - "darktrace"
  - "model_breach"
  - "third_party"

rule_source: "xsiam-buddy"

name: "Darktrace - Critical Model Breach"
alert_name: "Darktrace - Critical Model Breach"

description: >
  Surfaces Darktrace model breaches with a percent score of 87 or higher.
  Severity is dynamically computed from the score: 100 = CRITICAL,
  92-99 = HIGH, 87-91 = MEDIUM. Excludes known noisy Antigena rules.
  Investigate the source device and triggered components for threat context.

dataset: alerts
action: ALERTS

alert_domain: NETWORK
alert_category: "Anomaly Detection"

# ANNOTATION: SCHEDULED because we deduplicate by event ID and want batch processing
execution_mode: SCHEDULED

# ANNOTATION: 1h lookback with hourly runs
search_window: "1h"

# ANNOTATION: Run every hour
crontab: "0 * * * *"

# ANNOTATION: Dynamic severity via if() is appropriate here because Darktrace
# provides a numeric percentscore but no discrete severity field.
# The score IS the severity signal.
xql_query: |
  datamodel dataset = darktrace_darktrace_raw
  | fields *, darktrace_darktrace_raw.percentscore as percentscore,
            darktrace_darktrace_raw.triggeredComponents as triggeredComponents
  | sort desc _time
  | dedup xdm.event.id
  | filter percentscore >= 87
  | filter xdm.alert.original_threat_name not in (
      "Antigena::Network::Significant Anomaly::Antigena Significant Anomaly from Client Block",
      "Antigena::Network::Insider Threat::Antigena Large Data Volume Outbound Block"
    )
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
  | alter connection_hostname = if(array_length(connection_hostname) = 0, null, connection_hostname)
  | alter xdm.target.domain = coalesce(connection_hostname, dns_lookup)
  | alter xdm.alert.severity = if(
      percentscore >= 87 and percentscore <= 91, "MEDIUM",
      percentscore >= 92 and percentscore <= 99, "HIGH",
      percentscore = 100, "CRITICAL",
      "LOW"
    )
  | fields _time, xdm.alert.severity, xdm.alert.original_threat_name,
           xdm.alert.description, xdm.event.id, xdm.source.host.hostname,
           xdm.source.host.ipv4_addresses, xdm.target.domain, percentscore

# ANNOTATION: Static fallback severity -- if the if() chain produces no match,
# the alert still gets a severity. LOW is appropriate as a safe default.
severity: SEV_040_LOW

suppression_enabled: true

# ANNOTATION: 2h suppression by threat name -- same model breach type from
# different devices should still be investigated, but the same threat
# name repeating is likely the same underlying issue
suppression_duration: "2h"

suppression_fields:
  - "xdm.alert.original_threat_name"

# ANNOTATION: Darktrace model breaches are anomaly-based; map to the most
# relevant technique based on the alert category. Generic mapping here;
# refine per specific model breach type if needed.
mitre_defs:
  "T1071":
    - "TA0011"
```

---

## Example 4: Real-Time with Suppression (Firewall Deny)

**Use case:** Detect repeated firewall deny events targeting sensitive ports from
the same external source IP. Useful for identifying port scanning, exploitation
attempts, or persistent unauthorized access attempts.

**Key patterns:**
- `execution_mode: REAL_TIME` -- each blocked connection is alertable
- Suppression by `source_ip` + `destination_port` to collapse repeated blocks
- `incidr() = false` to filter to external sources only
- Low severity because firewall is already blocking the traffic

```yaml
global_rule_id: "d4e5f6a7-8b9c-4d0e-1f2a-3b4c5d6e7f8a"

rule_id: "firewall_repeated_deny_sensitive_port"

rule_name: "Firewall - Repeated Deny on Sensitive Port"

rule_labels:
  - "firewall"
  - "network_security"
  - "port_scan"

rule_source: "xsiam-buddy"

name: "Firewall - Repeated Deny on Sensitive Port"
alert_name: "Firewall - Repeated Deny on Sensitive Port"

description: >
  Detects firewall deny events targeting sensitive ports (SSH 22, RDP 3389,
  SMB 445, WinRM 5985/5986) from external source IPs. While the firewall
  is blocking the traffic, repeated attempts indicate reconnaissance or
  brute force targeting exposed services. Review the source IP for threat
  intelligence and consider permanent block listing.

dataset: alerts
action: ALERTS

alert_domain: NETWORK
alert_category: "Network Security"

# ANNOTATION: REAL_TIME because each denied connection to a sensitive port
# is independently noteworthy for threat intelligence
execution_mode: REAL_TIME

# ANNOTATION: Short search window for real-time processing
search_window: "1m"

# ANNOTATION: No crontab for REAL_TIME rules

# ANNOTATION: Filter to external sources hitting sensitive ports with deny action
xql_query: |
  dataset = xdr_data
  | filter event_type = ENUM.NETWORK_CONNECTION
    and action_type = "block"
    and action_remote_port in (22, 445, 3389, 5985, 5986)
    and action_remote_ip != null
    and incidr(action_remote_ip, "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16") = false
  | alter src_country = iploc(action_remote_ip)
  | fields _time, agent_hostname, action_remote_ip, action_remote_port,
           action_local_ip, action_local_port, src_country

# ANNOTATION: LOW because the firewall is already blocking the traffic;
# this alert is for visibility and threat intelligence, not immediate response
severity: SEV_040_LOW

suppression_enabled: true

# ANNOTATION: 1h suppression -- same source hitting the same port repeatedly
# is a single reconnaissance/attack attempt
suppression_duration: "1h"

# ANNOTATION: Suppress by source IP AND destination port so different ports
# from the same source still generate separate alerts (different attack vectors)
suppression_fields:
  - "action_remote_ip"
  - "action_remote_port"

# ANNOTATION: T1046 = Network Service Discovery (port scanning)
# T1205 = Traffic Signaling (probing specific ports)
# TA0043 = Reconnaissance, TA0005 = Defense Evasion
mitre_defs:
  "T1046":
    - "TA0043"
  "T1205":
    - "TA0005"
```
