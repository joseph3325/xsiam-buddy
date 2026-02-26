# XSIAM Content Development Plugin — Design

**Date:** 2026-02-24
**Status:** Approved

## Purpose

A Claude plugin that helps security professionals develop XSIAM content: automation scripts, XQL queries, playbooks, and documentation. Targets a mixed audience of SOC engineers, XSIAM admins, and security consultants.

## Architecture

Four focused skills sharing common reference knowledge files, bundled as one installable plugin.

### Plugin Structure

```
xsiam-content-dev/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── xsiam-automations/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── automation-yaml-spec.md
│   │       └── script-patterns.md
│   ├── xsiam-xql/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── xql-syntax-reference.md
│   │       └── xql-datasets.md
│   ├── xsiam-playbooks/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── playbook-format.md
│   └── xsiam-docs/
│       ├── SKILL.md
│       └── references/
│           └── doc-templates.md
└── README.md
```

## Skills

### 1. xsiam-automations
- Generates Python automation scripts (XSOAR conventions)
- Generates YAML metadata files
- Follows directory structure: `ScriptName/ScriptName.py`, `ScriptName.yml`, `ScriptName_test.py`
- Includes test stubs

### 2. xsiam-xql
- Natural language → XQL query generation
- Pipe-separated stage syntax
- Common dataset awareness
- Time range and field reference handling

### 3. xsiam-playbooks
- YAML playbook definitions (XSOAR-compatible)
- Companion Markdown documentation
- Common patterns: enrichment → decision → remediation → closure
- Sub-playbook references and conditional branching

### 4. xsiam-docs
- Integration READMEs, runbooks/SOPs, release notes, IR procedures, design docs
- Structured section templates for each doc type

## Knowledge Files

Shared reference material bundled with the plugin:
- XQL syntax (stages, operators, functions)
- XQL datasets and schemas
- Automation YAML spec (all required fields)
- Script patterns (CommonServerPython, error handling)
- Playbook YAML schema
- Content pack directory conventions
- Documentation templates

## Decisions

- No MCP servers needed (all generation is local)
- No hooks needed (skills trigger on demand)
- No commands needed (skills provide the right level of abstraction)
- No agents needed (skills are sufficient for the generation tasks)
- Plugin is for internal use (no `~~` placeholders needed)
