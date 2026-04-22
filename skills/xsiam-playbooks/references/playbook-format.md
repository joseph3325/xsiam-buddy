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

---

## 5. Task Type Examples

### 5.1 Start Task

Every playbook has exactly one start task with ID `"0"`. The start task has an empty name and no script.

```yaml
"0":
  id: "0"
  taskid: 7c123a77-9e7e-412d-8292-c2a58536723c
  type: start
  task:
    id: 7c123a77-9e7e-412d-8292-c2a58536723c
    version: -1
    name: ""
    iscommand: false
    brand: ""
    playbooktaskmissingcomponent: null
    istaskmissingcomponenterrordismissed: false
  nexttasks:
    '#none#':
    - "1"
  separatecontext: false
  continueonerrortype: ""
  view: |-
    {
      "position": {
        "x": 450,
        "y": 220
      }
    }
  note: false
  timertriggers: []
  ignoreworker: false
  skipunavailable: false
  quietmode: 0
  isoversize: false
  isautoswitchedtoquietmode: false
```

### 5.2 Regular Task — Integration Command

Executes an integration command. Uses `script` field with `'Brand|||command-name'` format.

```yaml
"2":
  id: "2"
  taskid: 402b5c0f-1b77-4624-8622-69eaee26aaf1
  type: regular
  task:
    id: 402b5c0f-1b77-4624-8622-69eaee26aaf1
    version: -1
    name: Call to the content bundle endpoint
    description: Download files from Core server.
    script: Core REST API|||core-api-download
    type: regular
    iscommand: true
    brand: Core REST API
    playbooktaskmissingcomponent: null
    istaskmissingcomponenterrordismissed: false
  nexttasks:
    '#none#':
    - "3"
  scriptarguments:
    retry-count:
      simple: "2"
    uri:
      simple: /xsoar/content/bundle
  separatecontext: false
  continueonerrortype: ""
  view: |-
    {
      "position": {
        "x": 450,
        "y": 540
      }
    }
  note: false
  timertriggers: []
  ignoreworker: false
  skipunavailable: false
  quietmode: 0
  isoversize: false
  isautoswitchedtoquietmode: false
```

### 5.3 Regular Task — Integration Command with Error Handling

Same as 5.2 but with error branch enabled. Adds `continueonerror: true` and `continueonerrortype: errorPath`. The `nexttasks` includes `'#error#'` pointing to the error handler task.

```yaml
"2":
  id: "2"
  taskid: 402b5c0f-1b77-4624-8622-69eaee26aaf1
  type: regular
  task:
    id: 402b5c0f-1b77-4624-8622-69eaee26aaf1
    version: -1
    name: Call to the content bundle endpoint
    description: Download files from Core server.
    script: Core REST API|||core-api-download
    type: regular
    iscommand: true
    brand: Core REST API
    playbooktaskmissingcomponent: null
    istaskmissingcomponenterrordismissed: false
  nexttasks:
    '#error#':
    - "8"
    '#none#':
    - "11"
  scriptarguments:
    retry-count:
      simple: "2"
    uri:
      simple: /xsoar/content/bundle
  separatecontext: false
  continueonerror: true
  continueonerrortype: errorPath
  view: |-
    {
      "position": {
        "x": 450,
        "y": 540
      }
    }
  note: false
  timertriggers: []
  ignoreworker: false
  skipunavailable: false
  quietmode: 0
  isoversize: false
  isautoswitchedtoquietmode: false
```

### 5.4 Regular Task — Automation Script

Executes a built-in or custom automation script. Uses `scriptName` field (not `script`). Sets `iscommand: false`.

```yaml
"4":
  id: "4"
  taskid: 697554ee-fba4-4764-8253-fdda1260d45e
  type: regular
  task:
    id: 697554ee-fba4-4764-8253-fdda1260d45e
    version: -1
    name: Get the server URL
    description: Get the Server URL.
    scriptName: GetServerURL
    type: regular
    iscommand: false
    brand: ""
    playbooktaskmissingcomponent: null
    istaskmissingcomponenterrordismissed: false
  nexttasks:
    '#none#':
    - "2"
  separatecontext: false
  continueonerrortype: ""
  view: |-
    {
      "position": {
        "x": 450,
        "y": 360
      }
    }
  note: false
  timertriggers: []
  ignoreworker: false
  skipunavailable: false
  quietmode: 0
  isoversize: false
  isautoswitchedtoquietmode: false
```

### 5.5 Condition Task — Inline Conditions

Branching logic using inline `conditions` array. Routes flow based on evaluated conditions. Must include `#default#` as fallback branch in `nexttasks`.

```yaml
"5":
  id: "5"
  taskid: 0a076dd9-b990-4ed0-86df-b8b93b731dee
  type: condition
  task:
    id: 0a076dd9-b990-4ed0-86df-b8b93b731dee
    version: -1
    name: Is Splunk Enabled?
    description: Check if Splunk instance is enabled.
    type: condition
    iscommand: false
    brand: ""
    playbooktaskmissingcomponent: null
    istaskmissingcomponenterrordismissed: false
  nexttasks:
    '#default#':
    - "7"
    "Yes":
    - "6"
  conditions:
  - label: "Yes"
    condition:
    - - operator: isEqualString
        left:
          value:
            simple: ${modules.brand}
          iscontext: true
        right:
          value:
            simple: SplunkPy
  separatecontext: false
  continueonerrortype: ""
  view: |-
    {
      "position": {
        "x": 450,
        "y": 700
      }
    }
  note: false
  timertriggers: []
  ignoreworker: false
  skipunavailable: false
  quietmode: 0
  isoversize: false
  isautoswitchedtoquietmode: false
```

**Condition structure explained:**
- `conditions` is an array of labeled condition groups
- Each group has a `label` (must match a key in `nexttasks`) and a `condition` array
- `condition` is an array of arrays — outer array is AND logic, inner arrays are OR logic
- Each condition entry has `operator`, `left` (value + `iscontext`), and `right` (value)

### 5.6 Condition Task — Script-Based

Uses a script to evaluate the condition instead of inline `conditions`. The script returns `yes`/`no` and nexttasks routes accordingly.

```yaml
"3":
  id: "3"
  taskid: 903caf3b-9015-4b10-80bd-de7ba4ed2c8d
  type: condition
  task:
    id: 903caf3b-9015-4b10-80bd-de7ba4ed2c8d
    version: -1
    name: Does the IP have critical exposures?
    description: Checks if one number is bigger than the other.
    scriptName: IsGreaterThan
    type: condition
    iscommand: false
    brand: ""
    playbooktaskmissingcomponent: null
    istaskmissingcomponenterrordismissed: false
  nexttasks:
    "no":
    - "4"
    "yes":
    - "5"
  scriptarguments:
    first:
      complex:
        root: Expanse
        accessor: IP.SeverityCounts.CRITICAL
    second:
      simple: "0"
  separatecontext: false
  continueonerrortype: ""
  view: |-
    {
      "position": {
        "x": 450,
        "y": 530
      }
    }
  note: false
  timertriggers: []
  ignoreworker: false
  skipunavailable: false
  quietmode: 0
  isoversize: false
  isautoswitchedtoquietmode: false
```

### 5.7 Title Task

Section header for visual organization. Non-functional — used to label phases of the playbook (e.g., "Enrichment Phase", "Done").

```yaml
"12":
  id: "12"
  taskid: 6e650c7e-2aaa-4169-8b2c-dc9c6e1977ad
  type: title
  task:
    id: 6e650c7e-2aaa-4169-8b2c-dc9c6e1977ad
    version: -1
    name: Done
    type: title
    iscommand: false
    brand: ""
    playbooktaskmissingcomponent: null
    istaskmissingcomponenterrordismissed: false
  separatecontext: false
  continueonerrortype: ""
  view: |-
    {
      "position": {
        "x": 450,
        "y": 1280
      }
    }
  note: false
  timertriggers: []
  ignoreworker: false
  skipunavailable: false
  quietmode: 0
  isoversize: false
  isautoswitchedtoquietmode: false
```

Title tasks are commonly used as terminal tasks (no `nexttasks`). They are also used as section headers between phases, in which case they have `nexttasks` pointing to the first task of the next phase.

### 5.8 Playbook Task (Sub-Playbook)

Calls another playbook. Must set `separatecontext: true` to prevent context pollution. Passes inputs via `scriptarguments`. Optionally configures `loop` for iteration.

```yaml
"6":
  id: "6"
  taskid: b3a1f9c2-47d8-4e6a-9c15-2d8e7f3a9b01
  type: playbook
  task:
    id: b3a1f9c2-47d8-4e6a-9c15-2d8e7f3a9b01
    version: -1
    name: Block Indicators - Generic v3
    playbookName: Block Indicators - Generic v3
    description: Block malicious indicators across multiple platforms.
    type: playbook
    iscommand: false
    brand: ""
    playbooktaskmissingcomponent: null
    istaskmissingcomponenterrordismissed: false
  nexttasks:
    '#none#':
    - "7"
  scriptarguments:
    IP:
      simple: ${incident.sourceIP}
    Domain:
      simple: ${incident.domain}
  separatecontext: true
  continueonerrortype: ""
  view: |-
    {
      "position": {
        "x": 450,
        "y": 800
      }
    }
  note: false
  timertriggers: []
  ignoreworker: false
  skipunavailable: false
  quietmode: 0
  isoversize: false
  isautoswitchedtoquietmode: false
```

**Optional `loop` block** — for iterating over values:

```yaml
  loop:
    iscommand: false
    exitCondition: ""
    wait: 1
    max: 100
```

### 5.9 Collection Task (User Input)

Pauses playbook execution to collect input from an analyst. Presents a message with options.

```yaml
"8":
  id: "8"
  taskid: d4e2a7b1-83f5-4c9d-a612-5f7c8e9d0b34
  type: collection
  task:
    id: d4e2a7b1-83f5-4c9d-a612-5f7c8e9d0b34
    version: -1
    name: Collect Escalation Approval
    description: Ask analyst for approval before escalating.
    type: collection
    iscommand: false
    brand: ""
    playbooktaskmissingcomponent: null
    istaskmissingcomponenterrordismissed: false
  nexttasks:
    '#none#':
    - "9"
  scriptarguments:
    message:
      simple: |
        Potential security breach detected.
        Approve escalation to Security Operations Center?
    options:
    - Approve
    - Deny
    task_key_field: ApprovalDecision
  separatecontext: false
  continueonerrortype: ""
  view: |-
    {
      "position": {
        "x": 450,
        "y": 1000
      }
    }
  note: false
  timertriggers: []
  ignoreworker: false
  skipunavailable: false
  quietmode: 0
  isoversize: false
  isautoswitchedtoquietmode: false
```

---

## 6. nexttasks Branch Labels

| Label | Meaning | Used By |
|---|---|---|
| `'#none#'` | Default/only path forward | All task types |
| `'#default#'` | Fallback when no condition matches | Condition tasks |
| `'#error#'` | Error handling path | Tasks with `continueonerror: true` |
| `"yes"` / `"no"` | Common condition labels | Condition tasks |
| Custom labels | Any string matching a `conditions` label | Condition tasks |
