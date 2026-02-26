# XSOAR/XSIAM Playbook YAML Format Reference

This document provides a comprehensive reference for developing XSOAR/XSIAM playbooks, including YAML structure, task types, common patterns, and best practices.

---

## 1. Top-Level Playbook YAML Structure

The root structure of a playbook defines metadata, tasks, inputs, outputs, and versioning information.

```yaml
id: incident-enrichment-playbook
version: -1
name: Incident Enrichment and Triage
description: >
  Enriches incident data with threat intelligence and performs automated
  triage to determine severity and required response actions.
starttaskid: "0"
tasks:
  "0":
    id: "0"
    taskid: 11111111-1111-1111-1111-111111111111
    type: start
    task:
      id: 11111111-1111-1111-1111-111111111111
      version: -1
      name: ""
      iscommand: false
      brand: ""
    nexttasks:
      '#none#':
      - "1"
    separatecontext: false
    view: |-
      {"position":{"x":50,"y":50}}
  "1":
    # Additional tasks follow...
inputs:
  - key: IndicatorValue
    value:
      complex:
        root: incident
        accessor: indicator_value
    required: true
    description: The indicator to investigate
outputs:
  - contextPath: Investigation.Verdict
    description: The verdict of the investigation
    type: string
  - contextPath: Investigation.Severity
    description: Assessed severity level
    type: number
fromversion: 6.5.0
tests:
  - No tests (auto formatted)
```

### Key Root-Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | Unique playbook identifier (lowercase, hyphens) |
| `version` | number | Yes | Schema version (typically `-1`) |
| `name` | string | Yes | Human-readable playbook name |
| `description` | string | No | Brief description of playbook purpose |
| `starttaskid` | string | Yes | ID of the start task (almost always "0") |
| `tasks` | object | Yes | Dictionary of task definitions by ID |
| `inputs` | array | No | Playbook input parameters |
| `outputs` | array | No | Playbook output values |
| `fromversion` | string | No | Minimum XSOAR version (e.g., "6.5.0") |
| `tests` | array | No | Test configurations |

---

## 2. Task Types

XSOAR/XSIAM supports the following task types:

### 2.1 Start Task (`start`)
Entry point for the playbook. Every playbook must have exactly one start task with ID "0".

```yaml
"0":
  id: "0"
  taskid: 11111111-1111-1111-1111-111111111111
  type: start
  task:
    id: 11111111-1111-1111-1111-111111111111
    version: -1
    name: ""
    iscommand: false
    brand: ""
  nexttasks:
    '#none#':
    - "1"
  separatecontext: false
  view: |-
    {"position":{"x":50,"y":50}}
```

### 2.2 Regular Task (`regular`)
Standard task that executes a command, script, or performs logic.

```yaml
"1":
  id: "1"
  taskid: 22222222-2222-2222-2222-222222222222
  type: regular
  task:
    id: 22222222-2222-2222-2222-222222222222
    version: -1
    name: Get Endpoint Details
    description: Retrieve endpoint information using XDR
    script: '|||xdr-get-endpoints'
    type: regular
    iscommand: true
    brand: Cortex XDR - IR
  nexttasks:
    '#none#':
    - "2"
  scriptarguments:
    endpoint_id_list:
      simple: ${incident.endpointids}
    filters:
      complex:
        root: inputs
        accessor: EndpointFilter
  separatecontext: false
  view: |-
    {"position":{"x":50,"y":200}}
```

**Regular Task Fields:**
- `script`: Format is `integration|||command` (e.g., `Cortex XDR - IR|||xdr-get-endpoints`)
- `iscommand`: `true` if calling integration command, `false` if running automation/script
- `scriptarguments`: Key-value map of command arguments

### 2.3 Condition Task (`condition`)
Branching logic based on context values or conditions. Routes flow to different tasks.

```yaml
"2":
  id: "2"
  taskid: 33333333-3333-3333-3333-333333333333
  type: condition
  task:
    id: 33333333-3333-3333-3333-333333333333
    version: -1
    name: Is Malicious Verdict?
    description: Check if threat verdict indicates malicious activity
    type: condition
    iscommand: false
    brand: ""
  nexttasks:
    '#default#':
    - "5"
    "yes":
    - "3"
    "no":
    - "4"
  conditions:
  - label: "yes"
    condition:
    - - operator: isEqualString
        left:
          value:
            simple: ${verdict}
          iscontext: true
        right:
          value:
            simple: malicious
  - label: "no"
    condition:
    - - operator: isEqualString
        left:
          value:
            simple: ${verdict}
          iscontext: true
        right:
          value:
            simple: benign
  separatecontext: false
  view: |-
    {"position":{"x":50,"y":400}}
```

**Condition Task Fields:**
- `nexttasks`: Maps condition labels to next task IDs. Must include `#default#` as fallback.
- `conditions`: Array of condition objects, each with:
  - `label`: Condition name (maps to nexttasks key)
  - `condition`: Array of condition groups (AND logic between groups, OR within group)

### 2.4 Title Task (`title`)
Section header for visual organization of the playbook. Non-functional, purely UI.

```yaml
"10":
  id: "10"
  taskid: 44444444-4444-4444-4444-444444444444
  type: title
  task:
    id: 44444444-4444-4444-4444-444444444444
    version: -1
    name: Enrichment Phase
    type: title
    iscommand: false
    brand: ""
  nexttasks:
    '#none#':
    - "11"
  separatecontext: false
  view: |-
    {"position":{"x":50,"y":600}}
```

### 2.5 Playbook Task (`playbook`)
Calls a sub-playbook, useful for code reuse and modularity.

```yaml
"3":
  id: "3"
  taskid: 55555555-5555-5555-5555-555555555555
  type: playbook
  task:
    id: 55555555-5555-5555-5555-555555555555
    version: -1
    name: Block Indicators - Generic v3
    playbookName: Block Indicators - Generic v3
    description: Call sub-playbook to block malicious indicators
    type: playbook
    iscommand: false
    brand: ""
  nexttasks:
    '#none#':
    - "5"
  scriptarguments:
    IP:
      complex:
        root: ip
        accessor: Address
        filters:
        - - operator: isNotEqualString
            left:
              value:
                simple: ${ip.Address}
              iscontext: true
            right:
              value:
                simple: 127.0.0.1
    URL:
      complex:
        root: url
        accessor: Data
  separatecontext: true
  loop:
    iscommand: false
    exitCondition: ""
    wait: 1
    max: 100
  view: |-
    {"position":{"x":50,"y":800}}
```

**Playbook Task Fields:**
- `playbookName`: Exact name of the sub-playbook to call
- `separatecontext`: Should be `true` to avoid context pollution
- `loop`: Optional configuration for iterating over values:
  - `iscommand`: Whether loop variable is from command output
  - `exitCondition`: Exit early if condition is true
  - `wait`: Delay in seconds between iterations
  - `max`: Maximum number of iterations

### 2.6 Collection Task (`collection`)
Collects user input during playbook execution (interactive task).

```yaml
"6":
  id: "6"
  taskid: 66666666-6666-6666-6666-666666666666
  type: collection
  task:
    id: 66666666-6666-6666-6666-666666666666
    version: -1
    name: Collect Escalation Approval
    description: Ask analyst for approval before escalating
    type: collection
    iscommand: false
    brand: ""
  nexttasks:
    '#none#':
    - "7"
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
  view: |-
    {"position":{"x":50,"y":1000}}
```

---

## 3. Complete Task Structure

### 3.1 Command Task (Regular Task with Integration)

```yaml
"2":
  id: "2"
  taskid: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
  type: regular
  task:
    id: aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa
    version: -1
    name: Get File Details
    description: Retrieve file hash and metadata from endpoint
    script: '|||xdr-get-file-details'
    type: regular
    iscommand: true
    brand: Cortex XDR - IR
  nexttasks:
    '#none#':
    - "3"
    on-error:
    - "error-handler"
  scriptarguments:
    file_hash:
      simple: ${incident.fileHash}
    endpoint_id:
      simple: ${incident.endpointId}
    timeout_seconds:
      simple: "30"
  separatecontext: false
  continueonerrortype: ""
  view: |-
    {"position":{"x":50,"y":250}}
```

### 3.2 Automation/Script Task

```yaml
"8":
  id: "8"
  taskid: bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb
  type: regular
  task:
    id: bbbbbbbb-bbbb-bbbb-bbbb-bbbbbbbbbbbb
    version: -1
    name: Parse JSON Response
    description: Extract and transform API response data
    script: ParseJSON
    type: regular
    iscommand: false
    brand: Builtin
  nexttasks:
    '#none#':
    - "9"
  scriptarguments:
    json_string:
      simple: ${ApiResponse}
    json_keys:
      simple: data,verdict,score
  separatecontext: false
  view: |-
    {"position":{"x":50,"y":1100}}
```

---

## 4. Condition Task Structure (Detailed)

### 4.1 Simple Condition

```yaml
"4":
  id: "4"
  taskid: cccccccc-cccc-cccc-cccc-cccccccccccc
  type: condition
  task:
    id: cccccccc-cccc-cccc-cccc-cccccccccccc
    version: -1
    name: Check Severity
    type: condition
    iscommand: false
    brand: ""
  nexttasks:
    '#default#':
    - "6"
    "critical":
    - "5"
    "high":
    - "5"
    "low":
    - "6"
  conditions:
  - label: "critical"
    condition:
    - - operator: greaterThanOrEqual
        left:
          value:
            simple: ${incident.severity}
          iscontext: true
        right:
          value:
            simple: "4"
  - label: "high"
    condition:
    - - operator: greaterThanOrEqual
        left:
          value:
            simple: ${incident.severity}
          iscontext: true
        right:
          value:
            simple: "3"
      - operator: lessThan
        left:
          value:
            simple: ${incident.severity}
          iscontext: true
        right:
          value:
            simple: "4"
  separatecontext: false
  view: |-
    {"position":{"x":50,"y":500}}
```

### 4.2 Complex Condition (AND/OR Logic)

```yaml
"5":
  id: "5"
  taskid: dddddddd-dddd-dddd-dddd-dddddddddddd
  type: condition
  task:
    id: dddddddd-dddd-dddd-dddd-dddddddddddd
    version: -1
    name: Should Block IP?
    type: condition
    iscommand: false
    brand: ""
  nexttasks:
    '#default#':
    - "7"
    "block":
    - "6"
  conditions:
  - label: "block"
    condition:
    - - operator: isEqualString
        left:
          value:
            simple: ${verdict}
          iscontext: true
        right:
          value:
            simple: malicious
      - operator: greaterThan
        left:
          value:
            simple: ${threat_score}
          iscontext: true
        right:
          value:
            simple: "80"
    - - operator: isInList
        left:
          value:
            simple: ${sourceIP}
          iscontext: true
        right:
          value:
            simple: ${known_malicious_ips}
  separatecontext: false
  view: |-
    {"position":{"x":50,"y":600}}
```

---

## 5. Sub-Playbook Task (Detailed)

```yaml
"6":
  id: "6"
  taskid: eeeeeeee-eeee-eeee-eeee-eeeeeeeeeeee
  type: playbook
  task:
    id: eeeeeeee-eeee-eeee-eeee-eeeeeeeeeeee
    version: -1
    name: Block Indicators - Generic v3
    playbookName: Block Indicators - Generic v3
    description: >
      Block malicious indicators across multiple platforms.
      Supports IP, Domain, URL, and File Hash blocking.
    type: playbook
    iscommand: false
    brand: ""
  nexttasks:
    '#none#':
    - "7"
    on-error:
    - "error-handler"
  scriptarguments:
    IP:
      complex:
        root: IOC
        accessor: IP.Address
        filters:
        - - operator: isNotEmpty
            left:
              value:
                simple: ${IOC.IP.Address}
              iscontext: true
    URL:
      complex:
        root: IOC
        accessor: URL
        filters:
        - - operator: startWith
            left:
              value:
                simple: ${IOC.URL}
              iscontext: true
            right:
              value:
                simple: "http"
    Domain:
      simple: ${incident.domain}
  separatecontext: true
  loop:
    iscommand: false
    exitCondition: ""
    wait: 1
    max: 100
  continueonerrortype: ""
  view: |-
    {"position":{"x":50,"y":800}}
```

---

## 6. Inputs and Outputs

### 6.1 Playbook Inputs

Inputs allow callers to pass parameters to the playbook.

```yaml
inputs:
- key: IndicatorValue
  value:
    complex:
      root: incident
      accessor: indicator_value
  required: true
  description: The indicator value to investigate (IP, domain, hash, etc.)
  playbookInputQuery:

- key: ThresholdScore
  value:
    simple: "75"
  required: false
  description: Threat score threshold for triggering automated response
  playbookInputQuery:

- key: BlockAction
  value:
    simple: block_and_alert
  required: false
  description: Action to take on malicious indicators (block_only, block_and_alert, quarantine)
  playbookInputQuery:

- key: IncidentFields
  value:
    complex:
      root: incident
      accessor: customFields
      filters:
      - - operator: isNotEmpty
          left:
            value:
              simple: ${incident.customFields}
            iscontext: true
  required: false
  description: Custom incident fields for enrichment
  playbookInputQuery:
```

**Input Fields:**
- `key`: Input parameter name (used in playbook as `${inputs.ParameterName}`)
- `value`: Default/expected value (complex or simple)
- `required`: Whether the input must be provided
- `description`: Human-readable description
- `playbookInputQuery`: Advanced query for filtering input values (usually empty)

### 6.2 Playbook Outputs

Outputs expose results from the playbook to callers.

```yaml
outputs:
- contextPath: Investigation.Verdict
  description: Final verdict on the indicator (benign, malicious, suspicious)
  type: string

- contextPath: Investigation.Score
  description: Threat score (0-100)
  type: number

- contextPath: Investigation.RelatedIOCs
  description: List of related indicators found during investigation
  type: array

- contextPath: Investigation.BlockedIndicators
  description: Indicators that were successfully blocked
  type: unknown

- contextPath: Investigation.ReportUrl
  description: URL to detailed investigation report
  type: string
```

**Output Fields:**
- `contextPath`: Key where value is stored in context
- `description`: Description of the output value
- `type`: Data type (string, number, array, unknown, object, etc.)

---

## 7. Common Condition Operators

Reference table of condition operators available in XSOAR/XSIAM:

| Operator | Description | Example |
|----------|-------------|---------|
| `isEqualString` | String equality | `${status}` == "active" |
| `isNotEqualString` | String inequality | `${status}` != "inactive" |
| `isExists` | Value exists (not null/empty) | `${result}` exists |
| `isNotExists` | Value doesn't exist | `${error}` doesn't exist |
| `containsGeneral` | String contains substring | `${message}` contains "error" |
| `doesNotContainGeneral` | String doesn't contain substring | `${message}` doesn't contain "success" |
| `containsGeneral` (case-insensitive) | Case-insensitive contains | `${category}` contains (case-insensitive) "THREAT" |
| `greaterThan` | Numeric greater than | `${score}` > 80 |
| `lessThan` | Numeric less than | `${count}` < 10 |
| `greaterThanOrEqual` | Numeric greater than or equal | `${severity}` >= 3 |
| `lessThanOrEqual` | Numeric less than or equal | `${age_days}` <= 30 |
| `startWith` | String starts with | `${url}` starts with "https://" |
| `endWith` | String ends with | `${filename}` ends with ".exe" |
| `isInList` | Value in comma-separated list | `${status}` in "active,pending,review" |
| `isNotInList` | Value not in list | `${priority}` not in "low,medium" |
| `IsEmpty` | String is empty | `${optional_field}` is empty |
| `IsNotEmpty` | String is not empty | `${required_field}` is not empty |
| `matchRegex` | Regex pattern matching | `${email}` matches "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$" |
| `stringHasLength` | String length equals | `${code}` length == 6 |
| `isCaseSensitiveEqual` | Case-sensitive equality | `${id}` case-sensitive equals "ABC123" |

---

## 8. Script Argument Sources

Script arguments can reference different sources using simple or complex paths.

### 8.1 Simple Arguments

```yaml
# Hardcoded value
scriptarguments:
  timeout_seconds:
    simple: "60"

# Context reference
scriptarguments:
  endpoint_id:
    simple: ${incident.endpointId}

# Playbook input
scriptarguments:
  investigation_type:
    simple: ${inputs.InvestigationType}

# Multiple context paths
scriptarguments:
  alert_id:
    simple: ${incident.id}
  source_ip:
    simple: ${incident.sourceIP}
```

### 8.2 Complex Arguments (Data Transformation)

```yaml
# Complex path with root and accessor
scriptarguments:
  file_hashes:
    complex:
      root: incident
      accessor: associatedFiles
      filters:
      - - operator: isNotEmpty
          left:
            value:
              simple: ${incident.associatedFiles}
            iscontext: true

# Multiple accessors (nested paths)
scriptarguments:
  suspicious_users:
    complex:
      root: investigation
      accessor: threats
      filters:
      - - operator: isEqualString
          left:
            value:
              simple: ${investigation.threats.type}
            iscontext: true
          right:
            value:
              simple: user_account

# Transforming arrays
scriptarguments:
  target_ips:
    complex:
      root: IndicatorData
      accessor: value
      transformers:
      - operator: uniq
      - operator: sort
```

---

## 9. Common Playbook Patterns

### 9.1 Enrichment → Triage → Remediation → Closure

Standard incident response flow:

```yaml
# Tasks: 0 (Start) → 1 (Enrich) → 2 (Triage) → 3 (Decide) → 4 (Remediate) → 5 (Close)

"1":
  name: Enrich Indicator
  # Get threat intel on indicators

"2":
  name: Analyze Threat Data
  # Use enrichment to assess threat level

"3":
  type: condition
  name: Is Malicious?
  nexttasks:
    "yes":
    - "4"
    '#default#':
    - "5"

"4":
  name: Block Indicator
  # Execute blocking actions

"5":
  name: Close Incident
  # Update incident status
```

### 9.2 Parallel Enrichment

Execute multiple enrichment tasks in parallel, converge at decision point:

```yaml
# Task 0 (Start) branches to:
# ├─ Task 1 (Get IP Reputation)
# ├─ Task 2 (Get Domain Info)
# └─ Task 3 (Get File Hash Info)
# All converge to Task 4 (Consolidate Results) then Task 5 (Decide)

"0":
  nexttasks:
    '#none#':
    - "1"
    - "2"
    - "3"

"1":
  name: Get IP Reputation
  nexttasks:
    '#none#':
    - "4"

"2":
  name: Get Domain Info
  nexttasks:
    '#none#':
    - "4"

"3":
  name: Get File Hash Info
  nexttasks:
    '#none#':
    - "4"

"4":
  name: Consolidate Enrichment Results
  nexttasks:
    '#none#':
    - "5"

"5":
  type: condition
  name: Decision Point
```

### 9.3 Loop with Exit Condition

Sub-playbook that iterates over values with early exit:

```yaml
"6":
  type: playbook
  task:
    playbookName: Remediate Single Endpoint
  loop:
    iscommand: false
    exitCondition: |
      ${EndpointRemediation.Status} == "failed"
    wait: 5
    max: 20
```

### 9.4 Conditional Vendor Check

Check if integration is enabled before calling vendor-specific sub-playbook:

```yaml
"7":
  type: condition
  name: Is Fortinet Available?
  conditions:
  - label: "yes"
    condition:
    - - operator: isEqualString
        left:
          value:
            simple: ${IsIntegrationAvailable.Fortinet}
          iscontext: true
        right:
          value:
            simple: "true"
  nexttasks:
    "yes":
    - "8"
    '#default#':
    - "9"

"8":
  type: playbook
  task:
    playbookName: Block IP - Fortinet

"9":
  type: playbook
  task:
    playbookName: Block IP - Generic
```

### 9.5 Timer-based Escalation

Collection task with timeout for user input, escalation on timeout:

```yaml
"10":
  type: collection
  task:
    name: Wait for Approval
  scriptarguments:
    message:
      simple: Approve this action?
    options:
    - Approve
    - Deny
    task_key_field: UserApproval
    task_key_field_timeout: "300"  # 5 minutes

"11":
  type: condition
  name: Was Approved?
  conditions:
  - label: "yes"
    condition:
    - - operator: isEqualString
        left:
          value:
            simple: ${UserApproval}
          iscontext: true
        right:
          value:
            simple: Approve
  nexttasks:
    "yes":
    - "12"
    '#default#':
    - "13"

"12":
  name: Execute Action
  # Perform approved action

"13":
  name: Escalate to Manager
  # Escalate for manual review
```

---

## 10. Best Practices

### 10.1 Playbook Design

1. **Always Start with Task "0"**
   - Every playbook must have exactly one start task with ID "0"
   - No other tasks should reference task "0"

2. **Use Descriptive Task Names**
   - Use verb + noun format: "Get Endpoint Details", "Check Severity", "Block IP Address"
   - Avoid generic names like "Task 1", "Step 2"

3. **Keep Playbooks Modular**
   - Break complex workflows into reusable sub-playbooks
   - Each playbook should have a single primary responsibility
   - Sub-playbooks enable code reuse and easier testing

4. **Separate Context for Sub-Playbooks**
   - Always set `separatecontext: true` when calling sub-playbooks
   - Prevents context pollution and unintended variable overwriting
   - Explicitly pass inputs needed by sub-playbook

5. **Use Playbook Inputs/Outputs Instead of Hardcoding**
   - Define playbook inputs for all external dependencies
   - Use outputs to expose results to callers
   - Enables flexibility and reusability across different incident types

6. **Include Error Handling**
   - Add `on-error` nexttasks for critical commands
   - Route errors to dedicated error-handling tasks
   - Log errors with context for debugging

7. **Meaningful Task Descriptions**
   - Add descriptions to complex or non-obvious tasks
   - Include expected inputs and outputs
   - Document why decisions are made in condition tasks

### 10.2 Context Management

1. **Context Variable Naming**
   - Use clear, namespace-like paths: `${incident.field}`, `${investigation.results}`
   - Avoid generic names like `${data}` or `${result}`

2. **Complex Path Filters**
   - Use filters to validate data before using it
   - Check for empty/null values before accessing nested fields
   - Example: Check `${evidence}` exists before accessing `${evidence.hash}`

3. **Context Cleanup**
   - Use `DeleteContext` when storing large temporary data
   - Prevents memory issues in long-running playbooks

### 10.3 Performance Optimization

1. **Minimize Command Calls**
   - Batch related queries in a single command call when possible
   - Use command arguments for filtering rather than post-processing

2. **Parallel Execution**
   - Design flows to execute independent enrichment tasks in parallel
   - Reduces overall playbook execution time

3. **Early Exits**
   - Use condition tasks to exit early if criteria are met
   - Example: If verdict is conclusively benign, skip malware analysis

### 10.4 Testing and Documentation

1. **Test Playbooks Thoroughly**
   - Test with various input types and edge cases
   - Test error conditions and timeouts
   - Document expected behaviors

2. **Add Usage Examples**
   - Include comments or documentation on how to call the playbook
   - Specify required inputs and expected outputs
   - Document any external dependencies (integrations required)

3. **Version Management**
   - Use semantic versioning in playbook ID or documentation
   - Document breaking changes in playbook descriptions
   - Maintain backward compatibility when possible

### 10.5 Security Considerations

1. **Sensitive Data Handling**
   - Don't log sensitive data (passwords, tokens, PII)
   - Use masking in error messages
   - Be cautious with playbook outputs containing sensitive data

2. **Access Control**
   - Design playbooks with least privilege in mind
   - Validate user inputs before using in commands
   - Document required permissions for each integration used

3. **Audit and Monitoring**
   - Log important decision points and actions taken
   - Include incident context in logs for auditability
   - Monitor for unexpected playbook failures

---

## 11. Common YAML Syntax Reference

### Context and Variables

```yaml
# Simple context reference
simple: ${incident.name}

# Nested context access
simple: ${incident.customFields.severity}

# Complex context with filters and transformers
complex:
  root: incident
  accessor: associatedAlerts
  filters:
  - - operator: isEqualString
      left:
        value:
          simple: ${incident.associatedAlerts.status}
        iscontext: true
      right:
        value:
          simple: open
  transformers:
  - operator: sort
  - operator: uniq
```

### Multi-line Strings

```yaml
# Pipe (|) preserves newlines
description: |
  This is a multi-line
  description that spans
  multiple lines.

# Folded (>) collapses newlines to spaces
description: >
  This is a long description that
  is written across multiple lines
  but will be collapsed into a single line.
```

### Lists and Dictionaries

```yaml
# List of items
nexttasks:
  '#none#':
  - "1"
  - "2"
  - "3"

# Dictionary/object
task:
  id: 12345
  name: Example
  version: -1
```

---

## 12. UUID Generation

Task IDs must be unique UUIDs (v4 format). Example valid UUIDs:
- `11111111-1111-1111-1111-111111111111`
- `a1b2c3d4-e5f6-7890-abcd-ef1234567890`

Generate UUIDs using:
- Online generators: https://www.uuidgenerator.net/
- Python: `python3 -c "import uuid; print(uuid.uuid4())"`
- Bash: `uuidgen` (on macOS/Linux)

---

## 13. Troubleshooting Common Issues

### Issue: Task Not Executing
**Cause**: Task ID not referenced in nexttasks of predecessor
**Solution**: Check that task ID is in predecessor's nexttasks, verify task ID spelling

### Issue: Variables Not Populating
**Cause**: Incorrect context path or variable not set by previous task
**Solution**: Verify context path syntax, check previous task outputs, use complex paths with filters

### Issue: Sub-Playbook Context Pollution
**Cause**: `separatecontext` set to `false`
**Solution**: Change `separatecontext` to `true`, explicitly pass needed inputs

### Issue: Condition Never Matches
**Cause**: Condition operator syntax error or incorrect context path
**Solution**: Verify operator name, check context path exists, use correct value format

### Issue: Infinite Loop
**Cause**: Loop max iterations too high or exit condition never true
**Solution**: Lower loop max, verify exit condition logic

---

## 14. Additional Resources

- XSOAR Documentation: https://xsoar.pan.dev/
- XSOAR Community: https://cortex.pan.dev/
- Integration Documentation: Search by integration name in Cortex XSOAR
- Playbook Examples: Built-in playbooks in XSOAR instance
