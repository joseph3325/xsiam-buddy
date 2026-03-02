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

### `xsiam-xql`
Generate XQL queries from natural language descriptions. Covers threat hunting, investigation, analytics, and correlation rules across all common XSIAM datasets. Handles pipe-separated stage syntax, time ranges, joins, and field references.

**Example triggers:** "write an XQL query", "hunt for threats", "search XSIAM data", "build a correlation rule"

---

### `xsiam-playbooks`
Generate playbook YAML definitions (XSOAR-compatible) with companion Markdown documentation. Supports enrichment, triage, remediation, and utility patterns with proper task structure, condition branching, and sub-playbook references.

**Example triggers:** "create a playbook", "design a workflow", "incident response playbook"

---

### `xsiam-docs`
Generate all types of XSIAM documentation: integration READMEs, runbooks/SOPs, release notes, IR procedures, and design documents. Uses structured section templates for each doc type.

**Example triggers:** "write documentation", "create a runbook", "release notes", "IR procedure"

## Bundled Knowledge

Each skill draws from reference files included in the plugin:

| Reference | Used By | Contents |
|---|---|---|
| XQL syntax reference | xsiam-xql | Stages, operators, functions, and time syntax |
| XQL datasets | xsiam-xql | Common dataset names and field schemas |
| Script YAML spec | xsiam-scripts | Required field ordering and complete examples |
| Integration YAML spec | xsiam-integrations | Integration structure, configuration, and command schema |
| Integration patterns | xsiam-integrations | BaseClient, pagination, error handling patterns |
| Common patterns | xsiam-scripts, xsiam-integrations | Shared Python patterns: CommandResults, indicators, logging |
| Playbook YAML schema | xsiam-playbooks | Task types, conditions, and sub-playbook references |
| Documentation templates | xsiam-docs | Section templates for each doc type |

## Usage

Describe what you want to build in natural language. Examples:

```
Create an integration for CrowdStrike Falcon that can get detections and contain hosts
```
```
Write a standalone script that enriches an IP address using VirusTotal
```
```
Write an XQL query to find all failed logins from outside the US in the last 24 hours
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
│   ├── xsiam-xql/              # XQL query generation
│   ├── xsiam-playbooks/        # Playbook generation
│   ├── xsiam-docs/             # Documentation generation
│   └── xsiam-shared/           # Shared reference files (common patterns)
└── docs/
    └── plans/                  # Design documents
```

## Plugin Metadata

| Field | Value |
|---|---|
| Name | xsiam-buddy |
| Version | 0.1.5 |
| Author | joseph3325 |
| Keywords | xsiam, xsoar, cortex, xql, playbook, automation, security |
