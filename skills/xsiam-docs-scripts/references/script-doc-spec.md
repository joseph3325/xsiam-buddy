# Script Documentation Specification

What to extract from script YAML and how to present each section of the documentation.

---

## Parsing Script YAML

A script YAML has this top-level structure:

```yaml
commonfields:
  id: ScriptName
  version: -1

vcShouldKeepItemLegacyProdMachine: false

name: ScriptName
script: |-
  register_module_line('ScriptName', 'start', __line__())
  # ... Python code (platform auto-injects CommonServerPython at runtime) ...
  register_module_line('ScriptName', 'end', __line__())

type: python
tags:
  - Utility
comment: |
  Description of what this script does.
enabled: true

args:
  - supportedModules: []
    name: input_data
    required: true
    description: The data to process
    defaultValue: ''
    isArray: false

outputs:
  - contextPath: ScriptName.Result
    description: The processed result
    type: string

scripttarget: 0
subtype: python3
pswd: ""
runonce: false
dockerimage: demisto/python3:3.12.12.6947692
runas: DBotWeakRole
engineinfo: {}
mainengineinfo: {}
```

### Key Fields to Extract

| YAML Field | Documentation Use |
|---|---|
| `name` | Title banner, document title |
| `comment` | Description section (expand if terse) |
| `tags` | Script type identification, overview card |
| `script` | Logic walkthrough, data flow diagram, code analysis |
| `args` | Arguments reference table |
| `args.*.name` | Argument name |
| `args.*.required` | Required/Optional badge |
| `args.*.description` | Argument description |
| `args.*.defaultValue` | Default value column |
| `args.*.isArray` | Whether it accepts comma-separated lists |
| `args.*.predefined` | Allowed values (dropdown options) |
| `args.*.secret` | Whether value is masked |
| `outputs` | Outputs reference table |
| `outputs.*.contextPath` | Context path (e.g., `ScriptName.Result`) |
| `outputs.*.description` | Output description |
| `outputs.*.type` | Data type (string, number, boolean, etc.) |
| `scripttarget` | Execution target: 0 = server-side, 1 = endpoint-side (XDR agent) |
| `runas` | Permission level: DBotWeakRole (default) or DBotRole (elevated) |
| `dockerimage` | Docker image and Python version |
| `runonce` | Whether the script runs only once per execution |

### Identifying Script Type from Tags

| Tag | Type | How it changes behavior |
|---|---|---|
| _(none functional)_ | Standard | General-purpose automation, called from playbooks or war room |
| `transformer` | Transformer | Appears in playbook input mappings, receives `value` arg automatically |
| `filter` | Filter | Appears in playbook conditions, returns boolean |
| `dynamic-section` | Dynamic Section | Renders display content in layout tabs, re-runs on tab open |
| `field-change-triggered` | Field-Change Triggered | Fires when a layout field value changes, receives change payload |
| `widget` | Widget | Powers dashboard/report widgets, returns widget-specific schema |

Informational tags (`Utility`, `Data Processing`, `Enrichment`, etc.) are organizational only — document them in the overview card but they don't change the script's behavior.

### Analyzing the Python Code

Walk through the embedded Python (`script: |-` block) to extract:

1. **Imports** — What modules beyond `CommonServerPython` / `CommonServerUserPython`? Third-party imports indicate special capabilities (e.g., `ipaddress`, `json`, `re`, `datetime`).

2. **Argument parsing** — How does `main()` parse args? Which helpers are used (`argToList`, `argToBoolean`, `arg_to_number`, `arg_to_datetime`)? Any validation logic?

3. **Core logic** — What does the script actually compute? Follow the data from parsed args through processing to output. Identify the key functions and what they do.

4. **External calls** — Any `demisto.executeCommand()` or `execute_command()` calls? These are integration dependencies. Note the command name and what args are passed.

5. **Context access** — Does it read `demisto.alert()`, `demisto.context()`, or `demisto.incident()`? This tells you whether the script operates on alert/incident data.

6. **Return mechanism** — How does it return data?
   - `return_results(CommandResults(...))` — standard, with context output
   - `return_results(value)` — transformer, raw value return
   - `return_results(True/False)` — filter, boolean return
   - `json.dumps(...)` — widget, schema-specific return

7. **Error handling** — What does the `except` block do? Does it use `return_error()`, `DemistoException`, or custom error messages?

---

## Section Specifications

### 1. Title Banner

**Content:** Script name from `name` field.
**Subtitle:** First sentence of `comment`, or the full comment if short.
**Visual:** Full-width colored bar (dark navy background, white text), matching the playbook doc banner pattern.

### 2. Overview Card

A key-value metadata table. Extract:

| Field | Source |
|---|---|
| Script Type | Determine from `tags` — Standard, Transformer, Filter, Dynamic Section, Field-Change Triggered, or Widget |
| Tags | All tags from `tags` array |
| Execution Target | From `scripttarget` — "Server-side" (0) or "Endpoint (XDR Agent)" (1) |
| Permissions | From `runas` — "Standard (DBotWeakRole)" or "Elevated (DBotRole)" |
| Docker Image | From `dockerimage` — show the Python version (e.g., "Python 3.12") |
| Run Once | From `runonce` — "Yes" or "No" |
| Arguments | Count of `args` entries |
| Outputs | Count of `outputs` entries |

### 3. Description

**Content:** Full `comment` field, expanded with context about what problem this script solves, when to use it, and what the expected outcome is.

If the comment is terse (common in XSIAM scripts), expand it based on the Python code analysis. For example, if the script parses alert data, calls an enrichment command, and formats results, write: "This script extracts IP addresses from the current alert's network artifacts, enriches them through threat intelligence lookups, and returns a consolidated risk assessment with verdict and confidence score."

Include a sentence about **when to use** this script: in a playbook task, from the war room, as a transformer in input mappings, etc. — based on the script type.

### 4. Script Type Callout

If the script is not a standard type, add a callout box immediately after the description explaining what the type means:

- **Transformer**: "This script is a transformer — it appears as an option in playbook task input mappings. The `value` argument is populated automatically with the field being transformed."
- **Filter**: "This script is a filter — it appears as a condition option in playbook task conditions. It returns true or false to drive branch logic."
- **Dynamic Section**: "This script is a dynamic section — it renders display content inside a case/issue layout tab. It re-executes each time the tab is opened."
- **Field-Change Triggered**: "This script is field-change triggered — it fires automatically when a specific field value changes on a case/issue layout. It receives the old and new values as arguments."
- **Widget**: "This script is a widget — it powers a dashboard or report widget. It returns data in the schema expected by the widget renderer."

Use the info callout box style (blue left border).

### 5. Data Flow Diagram

**Content:** Visual representation of the script's processing pipeline from inputs to outputs.

**What to show:**

For **standard scripts**, show the argument-to-output pipeline:
```
[Arguments] → [Parse & Validate] → [Core Logic / Processing] → [Format Output] → [Context Outputs]
```

For **scripts with executeCommand() calls**, expand to show external calls:
```
[Arguments] → [Parse] → [executeCommand: ip] → [Process Results] → [Outputs]
```

For **transformer/filter scripts**, show the value transformation:
```
[value (auto)] + [config args] → [Transform/Evaluate] → [Transformed Value / Boolean]
```

For **dynamic section scripts**, show context reading:
```
[demisto.context() / demisto.alert()] → [Process Data] → [Markdown/HTML Display]
```

**Node colors (reuse from playbook flow diagram):**
- Green (`#388E3C` / `#E8F5E9`) — Input nodes (arguments entering)
- Blue (`#1976D2` / `#E3F2FD`) — Processing steps
- Purple (`#7B1FA2` / `#F3E5F5`) — External calls (`executeCommand`)
- Orange (`#FA582D` / `#FFF3E0`) — Decision/conditional logic
- Navy (`#003366` / `#e6f0f7`) — Output nodes (context paths returned)

**Simplification:** For trivial scripts (parse 1-2 args, process, return), the diagram can be a simple 3-node vertical flow. Don't over-diagram simple logic.

### 6. Arguments Reference

**Content:** Table of all script arguments.

**Columns:**
- **Name** — Argument name in monospace font
- **Type** — Infer from parsing helper used (`argToList` = list, `argToBoolean` = boolean, `arg_to_number` = number, `arg_to_datetime` = datetime, default = string)
- **Required** — Required/Optional badge
- **Default** — From `defaultValue`, or "—" if none
- **Description** — From `description` field

**Additional details per argument:**
- If `isArray: true`, note "Accepts comma-separated list" in description
- If `predefined` values exist, list them as allowed values
- If `secret: true`, note "Value is masked"
- For transformer/filter scripts, mark the `value` argument with a note: "Auto-populated when used as transformer/filter"

### 7. Outputs Reference

**Content:** Table of all context outputs.

**Columns:**
- **Context Path** — Full path in monospace (e.g., `ScriptName.Result`)
- **Description** — From `description` field
- **Type** — From `type` field

If the script has no `outputs` (common for transformers, filters, dynamic sections), show an info callout: "This script does not write to context. [Transformers return the transformed value directly / Filters return a boolean / Dynamic sections render display content only]."

### 8. Logic Walkthrough

**Content:** A numbered, step-by-step explanation of what the Python code does. This is the core value-add of script documentation — making the code accessible to non-developers.

Write each step in plain language, not code. Reference the function names for context but explain what they do:

**Example:**
> 1. **Parse arguments** — Reads the `ip_address` argument (required) and `threshold` argument (defaults to 70).
> 2. **Look up IP reputation** — Calls the `ip` command via `executeCommand()` to get the IP's DBot score from configured threat intel feeds.
> 3. **Evaluate risk** — Compares the DBot score against the threshold. Scores above the threshold are classified as "High Risk."
> 4. **Build result** — Creates a structured result with the IP, score, risk level, and vendor verdicts.
> 5. **Return output** — Returns a `CommandResults` object with a markdown table and context output under `IPRisk`.

For complex logic, use sub-steps. For very simple scripts (under 20 lines of logic), keep it to 3-4 steps.

### 9. Usage Examples

**Content:** Concrete examples of how to invoke the script in different contexts.

**War room / CLI:**
Show the `!ScriptName` command with example arguments:
```
!ScriptName input_data="192.168.1.1" limit="10"
```

**Playbook task:**
Describe how to add the script as an "Execute Script" task, which arguments to map, and what context output to reference in subsequent tasks.

**Transformer/Filter context (if applicable):**
Show where the script appears in the UI and how the `value` argument is populated automatically.

Format examples using the dark-background code block style from the HTML styling guide.

### 10. Integration Dependencies

**Content:** Every integration this script calls via `executeCommand()`.

**Columns:** Integration/Command, What It Does, Required/Optional

Only include this section if the script uses `executeCommand()` or `execute_command()`. If the script is self-contained (no external calls), omit this section entirely.

### 11. Error Handling

**Content:** How the script handles failures.

Document:
- What the `try/except` block catches
- What error message surfaces to the user (from `return_error()`)
- Whether partial failures are handled (e.g., looping over multiple items and continuing on individual failure)
- For `executeCommand()` calls, whether `isError()` checks are performed

### 12. Related Content

**Content:** Cross-references to related content.

- Integrations that this script chains to
- Playbooks that commonly use this script
- Other scripts that complement this one
- Related documentation

---

## Content Quality Guidelines

### Writing Style
- Write in present tense ("This script enriches..." not "This script will enrich...")
- Use active voice ("The script calls the ip command" not "The ip command is called by the script")
- Be specific about what the code does — avoid vague descriptions like "processes data"
- Use XSIAM terminology consistently (alert vs. incident, data lake vs. database)
- Explain technical concepts for a mixed audience — an SOC analyst should understand the description and usage, while a content developer should find the logic walkthrough and code references useful

### When Information is Missing
- If the YAML has an empty `comment`, infer purpose from the script name and Python code
- If argument descriptions are missing, infer from the argument name and how it's used in the code
- If there are unclear code patterns, document them honestly and add an info callout suggesting review

### Handling Complex Scripts
For scripts with 100+ lines of Python:
- Group the Logic Walkthrough by phases (Initialization, Processing, Output)
- In the Data Flow Diagram, show phases rather than individual operations
- Consider a "Summary" paragraph before the Logic Walkthrough explaining the high-level approach
