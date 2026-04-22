# XSOAR/XSIAM Playbook YAML Format Reference

Complete format specification for playbook YAML files. All field orderings and structures are derived from real XSIAM exports and must be followed exactly for successful import.

---

## 1. Top-Level Field Ordering

Every playbook YAML follows this exact top-level field order:

| # | Field | Required | Description |
|---|-------|----------|-------------|
| 1 | `id` | Yes | Unique playbook identifier (lowercase with hyphens, or UUID-style) |
| 2 | `version` | Yes | Always `-1` for new playbooks |
| 3 | `vcShouldKeepItemLegacyProdMachine` | Yes | Always `false` |
| 4 | `name` | Yes | Human-readable playbook name |
| 5 | `description` | No | Brief description of playbook purpose |
| 6 | `tags` | No | List of category tags (e.g., `Management`, `Remediation`) |
| 7 | `starttaskid` | Yes | Always `"0"` |
| 8 | `tasks` | Yes | Dictionary of task definitions keyed by string ID |
| 9 | `view` | Yes | JSON string with `linkLabelsPosition` and `paper.dimensions` |
| 10 | `inputs` | Yes | Playbook input parameters (empty list `[]` if none) |
| 11 | `inputSections` | Yes | Groups inputs for the UI |
| 12 | `outputSections` | Yes | Groups outputs for the UI |
| 13 | `outputs` | Yes | Playbook output values (empty list `[]` if none) |

**Never include:** `fromversion`, `tests`, `marketplaces`, `timeout` — these are content-pack CI fields only.

**Example top-level skeleton:**

```yaml
id: my-playbook-id
version: -1
vcShouldKeepItemLegacyProdMachine: false
name: My Playbook Name
tags:
- Remediation
starttaskid: "0"
tasks:
  "0":
    # ... task definitions
view: |-
  {
    "linkLabelsPosition": {},
    "paper": {
      "dimensions": {
        "height": 1120,
        "width": 660,
        "x": 450,
        "y": 220
      }
    }
  }
inputs: []
inputSections:
- inputs: []
  name: General (Inputs group)
  description: Generic group for inputs
outputSections:
- outputs: []
  name: General (Outputs group)
  description: Generic group for outputs
outputs: []
```

---

## 2. Per-Task Field Ordering

Every task in the `tasks` dictionary follows this exact field order:

| # | Field | Required | Default | Notes |
|---|-------|----------|---------|-------|
| 1 | `id` | Yes | — | String task ID (matches the dictionary key) |
| 2 | `taskid` | Yes | — | UUID v4 (must match inner `task.id`) |
| 3 | `type` | Yes | — | `start`, `regular`, `condition`, `title`, `playbook`, `collection` |
| 4 | `task` | Yes | — | Inner task block (see Section 3) |
| 5 | `nexttasks` | Yes* | — | Maps branch labels to next task IDs (*terminal tasks may omit) |
| 6 | `scriptarguments` | Cond. | — | Only on command/script/collection tasks |
| 7 | `conditions` | Cond. | — | Only on condition tasks with inline conditions |
| 8 | `separatecontext` | Yes | `false` | `true` for sub-playbook tasks |
| 9 | `continueonerror` | Cond. | — | Only on tasks with error handling (`true`) |
| 10 | `continueonerrortype` | Yes | `""` | `errorPath` when `continueonerror: true` |
| 11 | `view` | Yes | — | JSON string with `position.x` and `position.y` |
| 12 | `note` | Yes | `false` | Whether task has a note |
| 13 | `timertriggers` | Yes | `[]` | Timer trigger configurations |
| 14 | `ignoreworker` | Yes | `false` | Whether to ignore worker |
| 15 | `skipunavailable` | Yes | `false` | Skip if integration unavailable |
| 16 | `quietmode` | Yes | `0` | Quiet mode level |
| 17 | `isoversize` | Yes | `false` | Whether task output is oversized |
| 18 | `isautoswitchedtoquietmode` | Yes | `false` | Auto-switch to quiet mode |

---

## 3. Inner `task` Block Field Ordering

The `task` block inside each task follows this field order:

| # | Field | Required | Notes |
|---|-------|----------|-------|
| 1 | `id` | Yes | UUID v4 — must be identical to outer `taskid` |
| 2 | `version` | Yes | Always `-1` |
| 3 | `name` | Yes | Task display name (empty string `""` for start task) |
| 4 | `playbookName` | Cond. | Only on playbook tasks — exact name of sub-playbook |
| 5 | `description` | No | Task description text |
| 6 | `script` | Cond. | Integration commands: `'Brand|||command-name'` |
| 7 | `scriptName` | Cond. | Automation scripts: `ScriptName` |
| 8 | `type` | Yes | Same value as outer `type` |
| 9 | `iscommand` | Yes | `true` for integration commands, `false` otherwise |
| 10 | `brand` | Yes | Integration brand name, or `""` |
| 11 | `playbooktaskmissingcomponent` | Yes | Always `null` |
| 12 | `istaskmissingcomponenterrordismissed` | Yes | Always `false` |

**`script` vs `scriptName` — which to use:**

| Scenario | Field | `iscommand` | `brand` | Example |
|---|---|---|---|---|
| Integration command | `script` | `true` | Integration name | `script: 'Core REST API|||core-api-download'` |
| Built-in command | `script` | `true` | `Builtin` | `script: Builtin|||closeInvestigation` |
| Automation script | `scriptName` | `false` | `""` | `scriptName: GetServerURL` |
| No script (start/title/manual) | Neither | `false` | `""` | — |

---

## 4. View Position Layout

Tasks are positioned on a visual canvas using `x,y` coordinates in the `view` JSON field.

**Layout rules:**
- Default main column: `x: 450`
- Starting y-position: `y: 220`
- Vertical spacing between sequential tasks: `~160-180px`
- Branch/error paths offset to the right: `x: 730` (or `+280` from main column)
- Additional parallel branches offset further: increments of `~280` on x-axis

**Per-task view format:**

```yaml
view: |-
  {
    "position": {
      "x": 450,
      "y": 220
    }
  }
```

**Top-level `view` field** — contains paper dimensions computed from task positions:

```yaml
view: |-
  {
    "linkLabelsPosition": {},
    "paper": {
      "dimensions": {
        "height": 1120,
        "width": 660,
        "x": 450,
        "y": 220
      }
    }
  }
```

`height` = (max task y) - (min task y) + 60. `width` = (max task x) - (min task x) + 380. `x` and `y` = position of the top-left-most task.
