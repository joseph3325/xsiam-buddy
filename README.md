# xsiam-buddy

A Claude Code plugin for building [Cortex XSIAM](https://www.paloaltonetworks.com/cortex/cortex-xsiam) and XSOAR content — automation scripts, integrations, XQL queries, playbooks, and documentation — using natural language.

## Installation

```bash
claude plugin marketplace add joseph3325/xsiam-buddy
claude plugin install xsiam-buddy@xsiam-buddy
```

## Skills

### `xsiam-scripts`
Generate standalone Python automation scripts embedded in importable YAML files. Produces unified `.yml` files ready for direct import into XSIAM, with Python embedded in the `script: |-` field, proper field ordering, and `demisto.alert()` normalization for XSIAM context.

**Example triggers:** "create a script", "write an XSIAM script", "build an automation", "XSOAR script"

---

### `xsiam-integrations`
Generate multi-command Python integrations with `BaseClient` and a corresponding YAML metadata file. Produces unified `.yml` files with Python embedded in `script.script: |-`, authentication configuration, and full command/argument definitions.

**Example triggers:** "build an integration", "create an integration for CrowdStrike", "write an XSIAM integration", "connect to an API"

---

### `xsiam-event-collectors`
Generate event collector integrations that ingest vendor events directly into the XSIAM data lake via `send_events_to_xsiam()`. Produces unified `.yml` files with Python embedded in `script.script: |-`, `isfetchevents: true`, proper `_time` normalization, and deduplication handling. Unlike regular integrations that create incidents, event collector data lands in the data lake and is queryable via XQL.

**Example triggers:** "create an event collector", "build an event collector", "ingest events into XSIAM", "fetch events"

---

### `xsiam-xql`
Generate XQL queries from natural language descriptions. Covers threat hunting, investigation, and analytics across all common XSIAM datasets. Uses tiered reference loading — core XQL references always load while advanced functions, extended datasets, and federated search references load on-demand based on query requirements.

**Example triggers:** "write an XQL query", "hunt for threats", "search XSIAM data", "query dataset"

---

### `xsiam-correlations`
Generate correlation rule JSON files matching the XSIAM export/import format. Produces complete `.json` files with embedded XQL, scheduling configuration, severity mapping, and MITRE ATT&CK tagging. Shares the tiered XQL reference layer with `xsiam-xql`.

**Example triggers:** "create a correlation rule", "build a detection rule", "detection engineering", "XSIAM alert rule"

---

### `xsiam-splunk-to-xql`
Translate existing Splunk SPL queries into equivalent XQL. Maps SPL commands, functions, and syntax to their XQL counterparts using a dedicated translation reference. Shares the tiered XQL reference layer with `xsiam-xql`.

**Example triggers:** "translate SPL", "convert Splunk to XQL", "migrate Splunk query", "SPL to XQL"

---

### `xsiam-playbooks`
Generate playbook YAML definitions (XSOAR-compatible) with companion Markdown documentation. Supports enrichment, triage, remediation, and utility patterns with proper task structure, condition branching, and sub-playbook references.

**Example triggers:** "create a playbook", "design a workflow", "incident response playbook"

---

### `xsiam-docs-playbooks`
Generate professional HTML documentation for XSIAM/XSOAR playbooks. Produces Google Docs-ready HTML with visual flow diagrams, task inventories, decision logic tables, and integration dependency maps — all styled with the Palo Alto Networks brand palette.

**Example triggers:** "document a playbook", "create playbook documentation", "playbook reference doc", "explain this playbook"

---

### `xsiam-docs-scripts`
Generate professional HTML documentation for XSIAM/XSOAR automation scripts. Produces Google Docs-ready HTML with data flow diagrams, argument/output reference tables, logic walkthroughs, and script-type-specific guidance (standard, transformer, filter, dynamic-section, field-change-triggered, widget).

**Example triggers:** "document a script", "create script documentation", "script reference doc", "explain this script"

## Bundled Knowledge

Each skill draws from reference files included in the plugin:

| Reference | Used By | Contents |
|---|---|---|
| XQL core reference | xsiam-xql, xsiam-correlations, xsiam-splunk-to-xql | Stages, operators, functions, and time syntax (always loaded) |
| XQL datasets core | xsiam-xql, xsiam-correlations, xsiam-splunk-to-xql | Common dataset names, presets, and joins (always loaded) |
| XQL advanced functions | xsiam-xql, xsiam-correlations, xsiam-splunk-to-xql | Array, JSON, and window functions (on-demand) |
| XQL datasets extended | xsiam-xql, xsiam-correlations, xsiam-splunk-to-xql | Third-party, email, and CIE datasets (on-demand) |
| XQL federated search | xsiam-xql, xsiam-correlations, xsiam-splunk-to-xql | External S3/GCS/Azure querying (on-demand) |
| Event collector spec | xsiam-event-collectors | `send_events_to_xsiam()`, `_time` handling, fetch-events flow |
| Correlation rule spec | xsiam-correlations | JSON export format, scheduling, severity, MITRE mapping |
| Correlation examples | xsiam-correlations | Complete example correlation rule JSON files |
| SPL-to-XQL mapping | xsiam-splunk-to-xql | SPL command and function translation reference |
| Script YAML spec | xsiam-scripts | Required field ordering and complete examples |
| Script types & patterns | xsiam-scripts | Specialized script types: enrichment, remediation, polling, etc. |
| Integration YAML spec | xsiam-integrations | Integration structure, configuration, and command schema |
| Integration patterns | xsiam-integrations | BaseClient, pagination, error handling patterns |
| Common patterns | xsiam-scripts, xsiam-integrations | Shared Python patterns: CommandResults, indicators, logging |
| Playbook YAML schema | xsiam-playbooks | Task types, conditions, and sub-playbook references |
| Playbook doc spec | xsiam-docs-playbooks | Section specs, YAML extraction, content guidelines |
| Script doc spec | xsiam-docs-scripts | Section specs, Python analysis, type-specific docs |
| HTML styling guide | xsiam-docs-playbooks, xsiam-docs-scripts | PAN brand palette, Google Docs-compatible HTML patterns |

## Usage

Describe what you want to build in natural language. Examples:

```
Create an integration for CrowdStrike Falcon that can get detections and contain hosts
```
```
Create an event collector for Okta that ingests system log events into the data lake
```
```
Write a standalone script that enriches an IP address using VirusTotal
```
```
Write an XQL query to find all failed logins from outside the US in the last 24 hours
```
```
Create a correlation rule to detect brute force attacks — more than 10 failed logins in 5 minutes
```
```
Translate this Splunk query to XQL: index=main sourcetype=syslog | stats count by src_ip
```
```
Build a phishing response playbook that enriches URLs, checks reputation, and blocks malicious indicators
```
```
Write a runbook for responding to ransomware alerts
```

## Repository Structure

```
xsiam-buddy/
├── .claude-plugin/
│   ├── plugin.json             # Plugin metadata
│   └── marketplace.json        # Marketplace manifest
├── skills/
│   ├── xsiam-scripts/          # Standalone script generation
│   ├── xsiam-integrations/     # Multi-command integration generation
│   ├── xsiam-event-collectors/ # Event collector integration generation
│   ├── xsiam-xql/              # XQL query generation
│   ├── xsiam-correlations/     # Correlation rule JSON generation
│   ├── xsiam-splunk-to-xql/    # SPL to XQL translation
│   ├── xsiam-playbooks/        # Playbook generation
│   ├── xsiam-docs-playbooks/   # Playbook documentation (HTML)
│   ├── xsiam-docs-scripts/     # Script documentation (HTML)
│   └── xsiam-shared/           # Shared XQL references and Python patterns
└── docs/
    └── plans/                  # Design documents
```

## Plugin Metadata

| Field | Value |
|---|---|
| Name | xsiam-buddy |
| Version | 0.8.0 |
| Author | joseph3325 |
| Keywords | xsiam, xsoar, cortex, xql, correlation, splunk, playbook, automation, security |
