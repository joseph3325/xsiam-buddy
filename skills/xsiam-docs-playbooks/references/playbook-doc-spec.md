# Playbook Documentation Specification

What to extract from playbook YAML and how to present each section of the documentation.

---

## Parsing Playbook YAML

A playbook YAML has this top-level structure:

```yaml
id: unique-id
version: -1
name: "Display Name"
description: "Long-form description"
tags: [Category, Tags]
starttaskid: "0"
tasks:
  "0":
    id: "0"
    taskid: <uuid>
    type: start
    task:
      name: ""
      iscommand: false
    nexttasks:
      '#none#': ["1"]
  "1":
    id: "1"
    taskid: <uuid>
    type: regular
    task:
      name: "Task Name"
      description: "What it does"
      script: "IntegrationName|||command-name"
      iscommand: true
      brand: "IntegrationName"
    nexttasks:
      '#none#': ["2"]
    scriptarguments:
      arg_name:
        simple: "value"
    separatecontext: false
inputs: []
outputs: []
```

### Key Fields to Extract

| YAML Field | Documentation Use |
|---|---|
| `name` | Title banner, document title |
| `description` | Description section |
| `tags` | Overview card — category |
| `tasks` | Flow diagram, task inventory, detailed reference |
| `tasks.*.type` | Task type badges (start, regular, condition, playbook, title, collection) |
| `tasks.*.task.name` | Task display name |
| `tasks.*.task.description` | Task detail in inventory and reference |
| `tasks.*.task.script` | Integration and command identification |
| `tasks.*.task.brand` | Integration name |
| `tasks.*.nexttasks` | Flow connections — `#none#` = success, `#error#` = error path, named = condition branches |
| `tasks.*.conditions` | Decision logic — operator, left value, right value, branch label |
| `tasks.*.scriptarguments` | Task inputs in detailed reference |
| `tasks.*.separatecontext` | Sub-playbook isolation flag |
| `tasks.*.continueonerror` | Error handling behavior |
| `tasks.*.continueonerrortype` | `errorPath` means error branch exists |
| `tasks.*.skipunavailable` | Whether task silently skips if integration missing |
| `tasks.*.type: playbook` + `task.playbookName` | Sub-playbook reference |
| `inputs` | Playbook inputs table |
| `outputs` | Playbook outputs table |

### Identifying Task Types

| `type` value | What it is | Icon suggestion |
|---|---|---|
| `start` | Entry point (always task "0") | Green circle |
| `regular` | Command execution or script | Blue rectangle |
| `condition` | Decision node with branching | Orange diamond shape (rendered as rectangle with orange border) |
| `playbook` | Sub-playbook call | Purple rectangle |
| `title` / `section` | Visual grouping (no execution) | Gray rectangle |
| `collection` | Data collection from user/analyst | Teal rectangle |

### Tracing the Flow

To build the flow diagram, walk the task graph starting from `starttaskid`:

1. Start at task `starttaskid` (usually "0")
2. Follow `nexttasks['#none#']` for the success path
3. If `nexttasks` has multiple entries under `#none#`, those tasks run **in parallel**
4. If `nexttasks` has named keys (not `#none#` or `#error#`), those are **condition branches**
5. For `#error#` paths, note these for the Error Handling section
6. Continue until you reach tasks with no `nexttasks` (terminal nodes)

Detect convergence: if multiple paths lead to the same task ID, that task is a merge point.

---

## Section Specifications

### 1. Title Banner

**Content:** Playbook name from `name` field.
**Subtitle:** First sentence of `description`, or the full description if short.
**Visual:** Full-width colored bar (dark indigo background, white text).

### 2. Overview Card

A key-value metadata table. Extract:

| Field | Source |
|---|---|
| Category | First tag from `tags`, or infer from description (Incident Response, Enrichment, Remediation, Utility) |
| Trigger | Infer from description or ask user — what alert/incident type triggers this playbook |
| Integrations | Unique list of `brand` values from all command tasks |
| Sub-playbooks | List of `playbookName` values from playbook-type tasks |
| Total Tasks | Count of tasks (excluding start) |
| Error Handling | "Yes" if any task has `continueonerror: true`, otherwise "No" |

### 3. Description

**Content:** Full `description` field, expanded with context about what problem this playbook solves, when it runs, and what the expected outcome is.

If the description is terse (common in XSIAM exports), expand it based on the task flow. For example, if the playbook enriches IPs then isolates endpoints, write: "This playbook investigates suspicious IP addresses by enriching them through threat intelligence services, evaluating their risk score, and automatically isolating affected endpoints when a malicious verdict is returned."

### 4. Flow Diagram

**Content:** Visual representation of the entire task graph.

**Layout rules:**
- Vertical flow, top to bottom
- Each task is a colored node (color by type — see html-styling-guide.md)
- Sequential tasks connected by down arrows (▼)
- Parallel tasks shown side-by-side in the same row
- Condition tasks show branch labels (Yes/No or named branches) above their child paths
- Phase labels (Enrichment, Decision, Action, Closure) can group related tasks
- Terminal tasks (no nexttasks) get a "Done" indicator

**What to show in each node:**
- Task ID and name (bold)
- Integration name or "Sub-playbook" or "Condition" (smaller text below)

**Simplification for complex playbooks:** If a playbook has more than 15 tasks, group related sequential tasks under phase labels and show sub-playbooks as single nodes rather than expanding their internal flow.

### 5. Task Inventory

**Content:** A summary table listing every task.

**Columns:**
- **ID** — Task string ID
- **Task Name** — From `task.name`
- **Type** — Badge colored by type (command, condition, sub-playbook, manual, section)
- **Integration** — From `task.brand`, or "Built-in" for scripts, or "—" for non-command tasks
- **Description** — From `task.description`, truncated to one line

**Ordering:** Follow the flow order (BFS from start), not numeric ID order.

### 6. Detailed Task Reference

**Content:** Expanded detail for each non-trivial task (skip start and title/section tasks).

For each task, show:
- **Task name** as a sub-heading
- **Type** badge
- **Command**: The full `script` value (e.g., `CrowdStrike|||cs-falcon-search-device`)
- **Arguments**: Table of `scriptarguments` with name, value (simple or context path), and whether it uses a transformer
- **Next tasks**: Where flow goes on success
- **Error handling**: If `continueonerror` is set, describe behavior
- **Notes**: Any `skipunavailable`, `separatecontext`, or loop configuration

### 7. Decision Logic

**Content:** For each condition task, document what it evaluates and where each branch leads.

Extract from the `conditions` array:
- The field being evaluated (left side)
- The operator (equals, is greater than, contains, is not empty, etc.)
- The comparison value (right side)
- The branch label
- The next task(s) for that branch

Also show the `#default#` branch (the "else" path).

Present as a table with columns: Condition, Operator, Value, Branch, Next Task.

### 8. Inputs & Outputs

**Inputs table columns:** Name, Description, Required (Yes/No badge), Default Value
**Outputs table columns:** Path (context path), Description, Type

Extract from top-level `inputs` and `outputs` arrays.

### 9. Integration Dependencies

**Content:** Every integration this playbook needs to be configured, what commands it uses from each, and whether the task is skippable (`skipunavailable: true` means optional).

**Columns:** Integration, Commands Used, Required/Optional

Group commands under their integration. Mark as "Optional" if all tasks using that integration have `skipunavailable: true`.

### 10. Error Handling

**Content:** Document every task that has error handling configured.

Show:
- Which tasks have `continueonerror: true`
- Whether they use `continueonerrortype: errorPath` (routes to `#error#` branch) or just continue
- Where the error path leads
- Any notification or logging tasks on the error path

### 11. Related Content

**Content:** Cross-references to other content.

- Sub-playbooks called by this playbook
- Parent playbooks that call this one (if known)
- Scripts referenced
- Integration documentation links

---

## Content Quality Guidelines

### Writing Style
- Write in present tense ("This playbook enriches..." not "This playbook will enrich...")
- Use active voice ("The playbook isolates the endpoint" not "The endpoint is isolated by the playbook")
- Be specific about what each task does — avoid vague descriptions like "processes data"
- Use XSIAM terminology consistently (alert vs. incident, data lake vs. database)

### When Information is Missing
- If the YAML has empty descriptions, infer purpose from the command name and arguments
- If the trigger is unclear, use a callout box asking the user to fill it in
- If there are tasks with no clear purpose, document them honestly rather than guessing

### XQL Queries
If any task argument contains an XQL query, format it in a dark code block for readability.

### Handling Large Playbooks
For playbooks with 20+ tasks:
- Use phase labels to group the flow diagram
- Consider a "Summary" paragraph before the flow diagram explaining the high-level phases
- In the detailed task reference, group tasks by phase rather than listing flat
