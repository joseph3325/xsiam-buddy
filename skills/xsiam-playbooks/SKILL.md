---
name: xsiam-playbooks
description: >
  This skill should be used when the user asks to "create a playbook",
  "build a playbook", "design a playbook", "XSIAM playbook", "XSOAR playbook",
  "incident response workflow", "automation workflow", "playbook YAML",
  or needs to generate playbook definitions for Cortex XSIAM or XSOAR.
version: 1.0.0
---

# XSIAM Playbook Development

Generate importable unified YAML files for Cortex XSIAM/XSOAR playbooks. Playbooks define automated workflows with tasks, conditions, and sub-playbook calls. The YAML must match the exact structure and field ordering of a real XSIAM export for successful import.

## Before Starting

Read the reference file to understand the playbook YAML format:
- `references/playbook-format.md` — Complete playbook YAML schema: top-level structure, per-task field ordering, task type examples with all boilerplate fields, condition operators, script argument patterns, common flow patterns

## What is a Playbook?

Playbooks are automated workflows that orchestrate tasks, integrations, scripts, and sub-playbooks to handle security incidents. They define the sequence of actions, decision points, and data flow for incident response.

**When to use which skill:**
- Automated incident workflows → **playbook** (this skill)
- Connecting to external APIs → **integration** (xsiam-integrations skill)
- Ingesting events into data lake → **event collector** (xsiam-event-collectors skill)
- Standalone data processing → **script** (xsiam-scripts skill)

## Workflow

### 1. Gather Requirements

Determine the playbook category:

| Category | Trigger | Typical Pattern |
|---|---|---|
| Incident Response | Alert/incident type | Enrichment → Triage → Remediation → Close |
| Enrichment | Sub-playbook call | Parallel lookups → Merge → Output |
| Remediation | Sub-playbook call | Validate → Act → Verify → Report |
| Utility | Manual or sub-playbook | Input → Transform → Output |

Then gather conditional requirements:

- **Uses integration commands?** → gather brand names, command names, argument mappings
- **Has condition/branching logic?** → gather decision criteria, branch labels, inline conditions vs script-based
- **Calls sub-playbooks?** → gather playbook names, input/output mappings
- **Needs user input?** → gather collection task prompts and options
- **Needs error handling?** → identify which tasks need `#error#` branches
- **Has parallel execution?** → design fork/merge task topology
- **What inputs does it accept?** → define `key`, default `value`, `required`, `description`
- **What outputs does it produce?** → define `contextPath`, `description`, `type`

### 2. Design the Flow

Map out the task list before generating YAML. Present it as a numbered list for user approval:

```
"0" (start) → "1"
"1" (title: Enrichment Phase) → "2"
"2" (command: xdr-get-endpoints) → "3"
"3" (condition: Is Malicious?) → yes: "4", #default#: "5"
"4" (command: block-indicator) → "6"
"5" (command: close-incident, close as benign) → "6"
"6" (title: Done)
```

Each entry shows: task ID, task type, task name, and nexttasks wiring.

Get user approval on the flow before proceeding to YAML generation.

### 3. Generate the Unified YAML

Build a single `.yml` file following these ordered sub-steps. The YAML structure must match real XSIAM export format for successful import.

1. **Top-level metadata** — `id` (lowercase with hyphens), `version: -1`, `vcShouldKeepItemLegacyProdMachine: false`, `name`, `tags` (if applicable), `starttaskid: "0"`
2. **Tasks dictionary** — each task with ALL boilerplate fields in exact field order per the format spec. Generate real v4 UUIDs for `taskid` and inner `task.id` (both must be identical for each task). Position tasks following view layout rules (`x: 450` main column, `y` increments of ~160-180px, branch offsets at `x: 730`).
3. **Top-level `view`** — JSON string with `linkLabelsPosition: {}` and `paper.dimensions` computed from task positions
4. **`inputs` and `inputSections`** — define all playbook inputs with `key`, `value`, `required`, `description`, `playbookInputQuery`. Wrap in `inputSections` with all keys listed.
5. **`outputSections` and `outputs`** — define all playbook outputs with `contextPath`, `description`, `type`. Wrap in `outputSections`.
6. **Verify omissions** — confirm no `fromversion`, no `tests`, no `marketplaces`, no `timeout`

### 4. File Output

Generate a single file:
- `PlaybookName.yml` — the unified YAML ready for import into XSIAM

After delivering the file, print a lightweight summary to the conversation:
- **Description** — 1-2 sentences on what the playbook does
- **Trigger** — what starts this playbook
- **Dependencies** — integrations, sub-playbooks, and scripts used
- **Inputs** — table with name, description, required, default value
- **Flow** — numbered steps with condition branches noted
- **Outputs** — table with contextPath, description, type

This summary is conversation-only — not a separate file. For full playbook documentation, use the `xsiam-docs-playbooks` skill.

### 5. Validation Checklist

Before delivering, verify:

**Structure:**
- [ ] Task `"0"` exists and is `type: start`
- [ ] All task IDs are unique strings
- [ ] All `taskid` values are valid v4 UUIDs
- [ ] `taskid` matches inner `task.id` for every task
- [ ] All `nexttasks` reference valid task IDs that exist in the `tasks` dictionary
- [ ] No orphaned tasks — every task is reachable from `"0"`
- [ ] No dead ends — every non-terminal task has `nexttasks`

**Field Ordering:**
- [ ] Top-level fields follow specified order: `id → version → vcShouldKeepItemLegacyProdMachine → name → tags → starttaskid → tasks → view → inputs → inputSections → outputSections → outputs`
- [ ] Per-task fields follow specified order: `id → taskid → type → task → nexttasks → scriptarguments → conditions → separatecontext → continueonerror → continueonerrortype → view → note → timertriggers → ignoreworker → skipunavailable → quietmode → isoversize → isautoswitchedtoquietmode`
- [ ] Inner `task` block fields follow specified order: `id → version → name → playbookName → description → script/scriptName → type → iscommand → brand → playbooktaskmissingcomponent → istaskmissingcomponenterrordismissed`

**Boilerplate:**
- [ ] All tasks have: `note: false`, `timertriggers: []`, `ignoreworker: false`, `skipunavailable: false`, `quietmode: 0`, `isoversize: false`, `isautoswitchedtoquietmode: false`
- [ ] All tasks have `continueonerrortype` (empty string `""` default, or `errorPath` for error-handling tasks)
- [ ] All inner `task` blocks have `playbooktaskmissingcomponent: null` and `istaskmissingcomponenterrordismissed: false`

**Conditions:**
- [ ] Every condition task has `#default#` in `nexttasks`
- [ ] Every condition label in `nexttasks` has a matching entry in `conditions` (for inline conditions)
- [ ] Condition operators are valid (from operator reference table)

**Sub-Playbooks:**
- [ ] `separatecontext: true` on all playbook-type tasks
- [ ] `playbookName` set in inner `task` block
- [ ] Inputs passed via `scriptarguments`

**Commands:**
- [ ] Integration command tasks have `iscommand: true`, `script: 'Brand|||command-name'`, and `brand` matching the integration
- [ ] Automation script tasks use `scriptName` (not `script`) with `iscommand: false`

**Inputs/Outputs:**
- [ ] All `${inputs.X}` references in tasks correspond to entries in the `inputs` array
- [ ] `inputSections` lists all input keys
- [ ] `outputSections` present (even if outputs list is empty)

**Omissions:**
- [ ] No `fromversion` field
- [ ] No `tests` field
- [ ] No `marketplaces` field
- [ ] No `timeout` field

## Key Conventions

- Playbook ID: lowercase with hyphens (e.g., `incident-enrichment-playbook`)
- Playbook name: human-readable with spaces (e.g., `Incident Enrichment Playbook`)
- Task names: verb + noun format (e.g., `Get Endpoint Details`, `Block IP Address`)
- UUIDs: real v4 format (e.g., `7c123a77-9e7e-412d-8292-c2a58536723c`) — not placeholder patterns
- View positions: main column at `x: 450`, branches offset at `x: 730`
- `vcShouldKeepItemLegacyProdMachine: false` — always present after `version`
- `separatecontext: true` for all sub-playbook tasks — prevents context pollution
- Error handling: `continueonerror: true` + `continueonerrortype: errorPath` + `nexttasks.'#error#'`
