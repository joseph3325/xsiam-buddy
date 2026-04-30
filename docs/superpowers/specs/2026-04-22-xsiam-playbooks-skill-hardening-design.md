# xsiam-playbooks Skill Hardening Design

**Date**: 2026-04-22
**Status**: Draft
**Scope**: Full rewrite of `skills/xsiam-playbooks/SKILL.md` and `skills/xsiam-playbooks/references/playbook-format.md`

## Problem

The xsiam-playbooks skill (v0.1.0) has not been tested and its reference file does not match real XSIAM export format. Key gaps:

- Missing ~10 per-task boilerplate fields present in real exports (`note`, `timertriggers`, `ignoreworker`, `skipunavailable`, `quietmode`, `isoversize`, `isautoswitchedtoquietmode`, `continueonerrortype`, `playbooktaskmissingcomponent`, `istaskmissingcomponenterrordismissed`)
- Missing top-level fields (`vcShouldKeepItemLegacyProdMachine`, `view`, `inputSections`, `outputSections`, `dirtyInputs`, `adopted`)
- Includes CI-only fields (`fromversion`, `tests`) that should never appear
- No explicit field ordering at any level
- Conflates `script` (integration commands) with `scriptName` (automation scripts)
- SKILL.md workflow is thin compared to mature skills (integrations v2.0.0, scripts v1.1.0, event-collectors v1.1.0)
- Companion docs overlap with the dedicated `xsiam-docs-playbooks` skill

## Ground Truth

The canonical format reference is `jp-export-content-bundle.yml` — a real XSIAM export with 7 tasks covering start, regular (command), regular (script), title, and error-handling patterns. Secondary validation from the public Cortex content repo (`Packs/*/Playbooks/` in `palo-alto-networks/content`).

## Approach

Full rewrite of both files (Approach A), modeled on the proven integration/event-collector skill structure.

## Design

### File Structure

No new files. Rewrite in place:
- `skills/xsiam-playbooks/SKILL.md` — skill workflow
- `skills/xsiam-playbooks/references/playbook-format.md` — format spec

### playbook-format.md — Format Spec

#### Top-Level Field Ordering

Extracted from real export:

```
id
version
vcShouldKeepItemLegacyProdMachine
name
tags
starttaskid
tasks
view
inputs
inputSections
outputSections
outputs
```

Fields to never include: `fromversion`, `tests`, `marketplaces`, `timeout`.

Optional top-level fields (include when applicable): `description`, `dirtyInputs`, `adopted`.

#### Per-Task Field Ordering

Every task in the `tasks` dictionary follows this field order:

```
id
taskid
type
task
nexttasks
scriptarguments          # only on command/script/collection tasks
separatecontext
continueonerror          # only on tasks with error handling
continueonerrortype
view
note
timertriggers
ignoreworker
skipunavailable
quietmode
isoversize
isautoswitchedtoquietmode
```

#### Inner `task` Block Field Ordering

```
id
version
name
description
script                   # integration commands: 'Brand|||command-name'
scriptName               # automation scripts: ScriptName
type
iscommand
brand
playbooktaskmissingcomponent
istaskmissingcomponenterrordismissed
```

Additional fields for specific task types:
- Playbook tasks: `playbookName` (after `name`)
- Condition tasks: `conditions` array (after `nexttasks`)

#### Default Values for Boilerplate Fields

| Field | Default |
|---|---|
| `version` (root) | `-1` |
| `vcShouldKeepItemLegacyProdMachine` | `false` |
| `version` (task) | `-1` |
| `separatecontext` | `false` (except sub-playbooks: `true`) |
| `continueonerrortype` | `""` |
| `note` | `false` |
| `timertriggers` | `[]` |
| `ignoreworker` | `false` |
| `skipunavailable` | `false` |
| `quietmode` | `0` |
| `isoversize` | `false` |
| `isautoswitchedtoquietmode` | `false` |
| `playbooktaskmissingcomponent` | `null` |
| `istaskmissingcomponenterrordismissed` | `false` |

#### Task Types

Full examples for each type, all using real export structure with all boilerplate fields:

1. **Start** (`type: start`) — task `"0"`, empty name, `nexttasks.'#none#'` to first real task
2. **Regular/Command** (`type: regular`, `iscommand: true`) — integration command via `script: 'Brand|||command-name'`
3. **Regular/Script** (`type: regular`, `iscommand: false`) — automation via `scriptName: ScriptName`
4. **Condition** (`type: condition`) — branching with `conditions` array, labeled branches + `#default#`
5. **Title** (`type: title`) — section header, no functional behavior
6. **Playbook** (`type: playbook`) — sub-playbook call with `separatecontext: true`, `playbookName`, optional `loop`
7. **Collection** (`type: collection`) — user input with `message`, `options`, `task_key_field`

#### `script` vs `scriptName` Distinction

- Integration commands: `script: 'Brand|||command-name'` with `iscommand: true` and `brand: BrandName`
- Automation scripts: `scriptName: ScriptName` with `iscommand: false` and `brand: ""` (or `brand: Builtin`)

#### Error Handling Pattern

Tasks with error branches use:
- `continueonerror: true`
- `continueonerrortype: errorPath`
- `nexttasks.'#error#'` pointing to error handler task

Tasks without error handling omit `continueonerror` and set `continueonerrortype: ""`.

#### View Position Layout

- Default column: `x: 450`
- Vertical spacing: ~160-180px between sequential tasks
- Branch offsets: `x: 730` (or +280 from main column) for error/alternate paths
- Top-level `view` contains `linkLabelsPosition` and `paper.dimensions` computed from task positions

#### Inputs, InputSections, Outputs, OutputSections

Inputs follow the standard structure with `key`, `value` (simple or complex), `required`, `description`, `playbookInputQuery`.

`inputSections` wraps input keys into groups:
```yaml
inputSections:
- inputs:
  - InputKeyName
  name: General (Inputs group)
  description: Generic group for inputs
```

`outputSections` follows the same pattern:
```yaml
outputSections:
- outputs: []
  name: General (Outputs group)
  description: Generic group for outputs
```

#### Condition Operators

Retain the existing operator reference table (this is good content that doesn't need changes).

#### Script Argument Patterns

Retain the existing simple/complex argument documentation with filters and transformers.

### SKILL.md — Workflow

#### Frontmatter

Keep existing trigger descriptions. Bump version to `1.0.0`.

#### Before Starting

Single reference file load:
- `references/playbook-format.md`

#### Step 1: Gather Requirements

Category decision table:

| Category | Trigger | Typical Pattern |
|---|---|---|
| Incident Response | Alert/incident type | Enrichment → Triage → Remediation → Close |
| Enrichment | Sub-playbook call | Parallel lookups → Merge → Output |
| Remediation | Sub-playbook call | Validate → Act → Verify → Report |
| Utility | Manual or sub-playbook | Input → Transform → Output |

Conditional requirements checklist:
- Uses integration commands? → gather brand names, command names, argument mappings
- Has condition/branching logic? → gather decision criteria, branch labels
- Calls sub-playbooks? → gather playbook names, input/output mappings
- Needs user input? → gather collection task prompts, options
- Needs error handling? → identify which tasks need `#error#` branches
- Has parallel execution? → design fork/merge task topology

#### Step 2: Design the Flow

Present a numbered task list before generating YAML. Each entry shows:
- Task ID (string, sequential from "0")
- Task type
- Task name
- Nexttasks wiring (which task IDs follow)

Get user approval on the flow before proceeding to YAML.

#### Step 3: Generate the Unified YAML

6 ordered sub-steps:
1. Top-level metadata: `id`, `version: -1`, `vcShouldKeepItemLegacyProdMachine: false`, `name`, `tags`, `starttaskid: "0"`
2. Tasks dictionary: each task with all boilerplate fields in exact field order, real v4 UUIDs for `taskid` and inner `task.id` (identical for each task), view positions following layout rules
3. Top-level `view`: paper dimensions computed from task positions
4. `inputs` and `inputSections`
5. `outputSections` and `outputs`
6. Verify: no `fromversion`, no `tests`, no `marketplaces`, no `timeout`

#### Step 4: File Output

- Single `PlaybookName.yml` file
- Lightweight summary printed to conversation (not a separate file):
  - Description (1-2 sentences)
  - Trigger
  - Dependencies: integrations, sub-playbooks, scripts
  - Inputs table (name, description, required, default)
  - Flow outline (numbered steps with condition branches)
  - Outputs table (contextPath, description, type)

#### Step 5: Validation Checklist

~25 items:

**Structure:**
- [ ] Task `"0"` exists and is `type: start`
- [ ] All task IDs are unique strings
- [ ] All `taskid` values are valid v4 UUIDs
- [ ] `taskid` matches inner `task.id` for every task
- [ ] All `nexttasks` reference valid task IDs
- [ ] No orphaned tasks (every task reachable from `"0"`)
- [ ] No dead ends (every non-terminal task has `nexttasks`)

**Field Ordering:**
- [ ] Top-level fields follow specified order
- [ ] Per-task fields follow specified order
- [ ] Inner `task` block fields follow specified order

**Boilerplate:**
- [ ] All default fields present on every task (`note`, `timertriggers`, `ignoreworker`, `skipunavailable`, `quietmode`, `isoversize`, `isautoswitchedtoquietmode`)
- [ ] `continueonerrortype` present on every task (empty string default)
- [ ] `playbooktaskmissingcomponent` and `istaskmissingcomponenterrordismissed` present in every inner `task` block

**Conditions:**
- [ ] Every condition task has `#default#` in nexttasks
- [ ] Every condition label has a matching `conditions` entry
- [ ] Condition operators are valid (from operator reference table)

**Sub-Playbooks:**
- [ ] `separatecontext: true` on all playbook tasks
- [ ] `playbookName` set in inner `task` block
- [ ] Inputs passed via `scriptarguments`

**Commands:**
- [ ] `iscommand: true` on all integration command tasks
- [ ] `script` uses `'Brand|||command-name'` format
- [ ] `brand` field matches the integration name
- [ ] Automation scripts use `scriptName` (not `script`) with `iscommand: false`

**Inputs/Outputs:**
- [ ] All `inputs` referenced by tasks exist in `inputs` array
- [ ] `inputSections` lists all input keys
- [ ] `outputSections` present (even if empty)

**Omissions:**
- [ ] No `fromversion` field
- [ ] No `tests` field
- [ ] No `marketplaces` field
- [ ] No `timeout` field

### Common Playbook Patterns

Retain and expand the existing patterns section (Enrichment → Triage → Remediation → Close, Parallel Enrichment, Polling Loop, Conditional Vendor) but rewrite examples to use full task structure with boilerplate fields rather than abbreviated snippets.

## Out of Scope

- Companion Markdown documentation (handled by `xsiam-docs-playbooks` skill)
- Playbook API operations (get/insert/delete — vault has this but it's not content generation)
- Testing infrastructure (no runtime code in this plugin)

## Token Efficiency

The reference file will be larger than current due to full boilerplate examples, but this is a single-file load (no tiered loading needed). Playbooks don't have the variable complexity that XQL has — every invocation needs the full format spec.
