---
name: xsiam-playbooks
description: >
  This skill should be used when the user asks to "create a playbook",
  "build a playbook", "design a playbook", "XSIAM playbook", "XSOAR playbook",
  "incident response workflow", "automation workflow", "playbook YAML",
  or needs to generate playbook definitions with documentation for
  Cortex XSIAM or XSOAR.
version: 0.1.0
---

# XSIAM Playbook Development

Generate XSOAR/XSIAM playbook YAML definitions with companion Markdown documentation.

## Before Starting

Read the reference file to understand the playbook YAML format:
- `references/playbook-format.md` — Complete playbook YAML schema, task types, conditions, and patterns

## Workflow

### 1. Understand the Use Case

Determine the playbook's purpose:
- **Incident response** — automated triage and remediation for a specific alert type
- **Enrichment** — gather context about indicators (IPs, hashes, domains, users)
- **Remediation** — take containment/remediation actions (block, isolate, disable)
- **Utility** — reusable logic (data transformation, notification, ticketing)

Ask the user:
- What triggers this playbook? (alert type, manual, sub-playbook call)
- What integrations/commands does it use?
- What decisions need to be made? (severity thresholds, verdicts)
- What's the expected outcome? (incident closed, indicator blocked, report generated)

### 2. Design the Flow

Map out the playbook logic before generating YAML:

1. **Start** → always task "0"
2. **Enrichment phase** — gather data (can be parallel tasks)
3. **Decision point** — condition task based on enrichment results
4. **Action phase** — remediation or escalation based on decision
5. **Closure** — close incident or hand off

Present the flow to the user as a numbered list before generating YAML.

### 3. Generate the YAML

Build the playbook YAML following these rules:

**Task numbering**: Sequential string IDs starting from "0". Task "0" is always `type: start`.

**Task structure**: Each task must have:
- `id` and `taskid` (use UUID v4 for taskid)
- `type` (start, regular, condition, playbook, title, collection)
- `task` block with name, description, script (for commands), iscommand
- `nexttasks` mapping next task IDs
- `separatecontext` (false for most, true for sub-playbooks)
- `view` with position coordinates

**Condition tasks**: Define `conditions` array with label, operator, left/right values. Use `#default#` for the else branch.

**Sub-playbook tasks**: Set `type: playbook`, reference by `playbookName`, set `separatecontext: true`, pass inputs via `scriptarguments`.

**Command tasks**: Set `iscommand: true`, reference command as `script: '|||command-name'` or `script: 'IntegrationName|||command-name'`.

### 4. Generate Companion Documentation

Create a Markdown file documenting the playbook:

```markdown
# Playbook: [Name]

## Description
What this playbook does and when it runs.

## Trigger
What alert type, incident type, or condition triggers this playbook.

## Dependencies
- Integrations: [list]
- Sub-playbooks: [list]
- Scripts: [list]

## Inputs
| Name | Description | Required | Default |
|------|-------------|----------|---------|
| ... | ... | Yes/No | ... |

## Flow
1. [Step description]
2. [Step description]
   - If [condition]: [action]
   - Else: [action]
3. [Step description]

## Outputs
| Path | Description | Type |
|------|-------------|------|
| ... | ... | ... |

## Error Handling
How the playbook handles failures at each stage.
```

### 5. Output Both Files

Deliver two files:
- `PlaybookName.yml` — the YAML playbook definition
- `PlaybookName.md` — the companion documentation

### 6. Validation Checklist

Before delivering, verify:
- [ ] Task "0" exists and is type "start"
- [ ] All tasks have unique IDs and valid UUIDs for taskid
- [ ] All nexttasks reference valid task IDs
- [ ] No orphaned tasks (every task is reachable from start)
- [ ] Condition tasks have both a labeled branch and `#default#`
- [ ] Sub-playbooks have `separatecontext: true`
- [ ] Command tasks have `iscommand: true` and valid script references
- [ ] Playbook inputs match what tasks actually reference
- [ ] Outputs are populated by at least one task
- [ ] View positions don't overlap (increment y by ~200 per task)

## Common Playbook Patterns

**Enrichment → Triage → Remediation → Close**:
Start → Enrich IP/Hash/Domain → Check severity → If malicious: isolate + block → Close incident

**Parallel Enrichment**:
Start → Fork to [IP enrichment, Domain enrichment, File enrichment] → Merge → Decision

**Polling Loop**:
Start → Trigger scan → Sub-playbook with loop (check status every 30s, exit on complete) → Process results

**Conditional Vendor**:
Start → Check which integrations are enabled → Branch to vendor-specific sub-playbook → Merge results
