# xsiam-buddy

A Claude Code plugin for building [Cortex XSIAM](https://www.paloaltonetworks.com/cortex/cortex-xsiam) and XSOAR content — automation scripts, XQL queries, playbooks, and documentation — using natural language.

## Installation

```bash
claude plugin marketplace add joseph3325/xsiam-buddy
claude plugin install xsiam-buddy@xsiam-buddy
```

## Skills

### `xsiam-automations`
Generate Python automation scripts and YAML metadata files following XSOAR conventions. Supports both standalone scripts and multi-command integrations. Produces the full directory structure (`ScriptName/ScriptName.py`, `ScriptName.yml`, `ScriptName_test.py`) with test stubs included.

**Example triggers:** "create an automation", "write an XSIAM script", "build an integration for CrowdStrike"

---

### `xsiam-xql`
Generate XQL queries from natural language descriptions. Covers threat hunting, investigation, analytics, and correlation rules across all common XSIAM datasets. Handles pipe-separated stage syntax, time ranges, and field references.

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

| Reference | Contents |
|---|---|
| XQL syntax reference | Stages, operators, functions, and time syntax |
| XQL datasets | Common dataset names and field schemas |
| Automation YAML spec | All required and optional metadata fields |
| Script patterns | CommonServerPython, BaseClient, CommandResults, error handling |
| Playbook YAML schema | Task types, conditions, and sub-playbook references |
| Documentation templates | Section templates for each doc type |

## Usage

Describe what you want to build in natural language. Examples:

```
Create an integration for CrowdStrike Falcon that can get detections and contain hosts
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
│   ├── xsiam-automations/      # Automation script generation
│   ├── xsiam-xql/              # XQL query generation
│   ├── xsiam-playbooks/        # Playbook generation
│   └── xsiam-docs/             # Documentation generation
└── docs/
    └── plans/                  # Design documents
```

## Plugin Metadata

| Field | Value |
|---|---|
| Name | xsiam-buddy |
| Version | 0.1.2 |
| Author | joseph3325 |
| Keywords | xsiam, xsoar, cortex, xql, playbook, automation, security |
