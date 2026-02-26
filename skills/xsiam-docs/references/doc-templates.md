# XSIAM Documentation Templates

Comprehensive templates for all XSIAM documentation types. Use these templates as starting points for creating consistent, high-quality documentation across integrations, runbooks, release notes, incident response procedures, and design documents.

---

## 1. Integration README Template

Use this template when documenting new or updated XSIAM integrations with vendor products and services.

```markdown
## IntegrationName

### Description
Brief description of what this integration does and which vendor/product it connects to.

### Prerequisites
- Required API permissions
- Authentication requirements
- Network access requirements

### Configuration
| Parameter | Description | Required | Default |
|-----------|-------------|----------|---------|
| Server URL | Base URL of the API | Yes | https://api.vendor.com |
| API Key | Authentication key | Yes | |
| Verify SSL | Verify SSL certificates | No | true |
| Proxy | Use system proxy | No | false |

### Commands
#### command-name
Description of what this command does.

##### Required Permissions
`permission.name`

##### Input
| Argument | Description | Required | Default |
|----------|-------------|----------|---------|
| arg_name | Description | Yes | |

##### Context Output
| Path | Type | Description |
|------|------|-------------|
| Vendor.Entity.Field | String | Description |

##### Command Example
`!command-name arg_name=value`

##### Human Readable Output
>### Results
>|Field1|Field2|
>|---|---|
>|value1|value2|

### Troubleshooting
Common issues and their resolutions.
```

### Integration README Guidelines

- **Description**: Clearly state the vendor/product and primary function. Include what problems it solves.
- **Prerequisites**: List all setup requirements before the integration can be configured.
- **Configuration**: Document every parameter with type, requirement level, and default values.
- **Commands**: For each command, include required permissions, inputs, outputs, examples, and expected human-readable results.
- **Troubleshooting**: Address common configuration errors, authentication issues, and API rate limiting.

---

## 2. Runbook / SOP Template

Use this template for Standard Operating Procedures (SOPs) and runbooks that guide analysts through complex processes.

```markdown
# Runbook: [Title]

## Purpose
What this runbook accomplishes and when to use it.

## Scope
Systems, teams, and scenarios this runbook covers.

## Prerequisites
- Access requirements
- Tool requirements
- Knowledge requirements

## Severity Classification
| Level | Criteria | Response Time |
|-------|----------|---------------|
| Critical | ... | Immediate |
| High | ... | 1 hour |
| Medium | ... | 4 hours |
| Low | ... | Next business day |

## Procedure

### Step 1: Initial Assessment
1. Action to take
2. What to look for
3. Expected outcome

### Step 2: Investigation
1. Queries to run
2. Data to collect
3. Indicators to check

### Step 3: Containment
1. Immediate actions
2. Scope assessment
3. Communication requirements

### Step 4: Remediation
1. Fix actions
2. Verification steps
3. Monitoring requirements

### Step 5: Post-Incident
1. Documentation requirements
2. Lessons learned
3. Process improvements

## Escalation Path
| Condition | Escalate To | Method |
|-----------|-------------|--------|
| Cannot contain | SOC Manager | Phone + Slack |
| Data breach suspected | CISO | Phone + Email |

## References
- Related playbooks
- Vendor documentation
- Internal knowledge base articles
```

### Runbook Guidelines

- **Purpose**: Define the runbook's objective and typical use cases.
- **Scope**: Specify which systems, teams, and scenarios the runbook covers.
- **Prerequisites**: List access rights, tools, and knowledge needed by the analyst.
- **Severity Classification**: Define severity levels and corresponding response times.
- **Procedure**: Break down the response into clear sequential steps with specific actions and expected outcomes.
- **Escalation Path**: Document when and how to escalate issues based on conditions encountered.
- **References**: Link to related playbooks, vendor docs, and knowledge base articles.

---

## 3. Release Notes Template

Use this template for documenting releases of packs, integrations, scripts, and playbooks.

```markdown
# [Pack Name] Release Notes

## [Version] (YYYY-MM-DD)

### Integrations
#### IntegrationName
- Added new command `command-name` for [purpose].
- Fixed an issue where [description of bug].
- Updated the Docker image to *demisto/python3:x.y.z*.

### Scripts
#### ScriptName
- Added support for [feature].
- Improved performance of [operation].

### Playbooks
#### PlaybookName
- New playbook for [use case].
- Updated to support [new feature/integration].

### Classifiers & Mappers
- Updated incoming mapper for [integration] to include [new fields].

### Breaking Changes
- [Description of any breaking changes and migration steps]
```

### Release Notes Guidelines

- **Version and Date**: Use semantic versioning (MAJOR.MINOR.PATCH) with ISO 8601 date format.
- **Organization**: Group changes by content type (Integrations, Scripts, Playbooks, etc.).
- **Detail Level**: Include feature additions, bug fixes, performance improvements, and dependency updates.
- **Breaking Changes**: Clearly identify any breaking changes and provide migration instructions.
- **Docker Images**: Specify exact Docker image versions for reproducibility.
- **Commands**: When referencing commands, use backticks for code formatting.

---

## 4. Incident Response Procedure Template

Use this template for documenting threat-specific incident response procedures aligned with MITRE ATT&CK.

```markdown
# IR Procedure: [Threat Type]

## Overview
- **Threat Category**: [Malware/Phishing/Unauthorized Access/etc.]
- **MITRE ATT&CK**: [Technique IDs]
- **Severity Range**: [Medium-Critical]
- **Associated Playbook**: [Playbook Name]

## Detection
### Indicators
- XQL queries for detection
- Alert sources and types
- Behavioral indicators

### Initial Triage Checklist
- [ ] Confirm alert is not false positive
- [ ] Identify affected assets
- [ ] Determine scope of impact
- [ ] Classify severity

## Investigation
### Data Collection
- Endpoint data to gather
- Network logs to review
- Identity/auth logs to check

### Analysis Steps
1. Timeline reconstruction
2. Root cause identification
3. Lateral movement assessment
4. Data exposure assessment

## Containment
### Immediate Actions
- Endpoint isolation steps
- Network blocking rules
- Account actions

### Evidence Preservation
- Forensic image requirements
- Log retention
- Chain of custody

## Remediation
- Malware removal steps
- Credential reset requirements
- System restoration procedures
- Patch/configuration changes

## Recovery
- Service restoration checklist
- Monitoring requirements
- User communication

## Post-Incident
- Report template reference
- Metrics to capture
- Process improvement items
```

### IR Procedure Guidelines

- **Overview**: Include threat category, MITRE ATT&CK techniques, severity range, and associated playbook.
- **Detection**: Provide XQL queries, alert indicators, and behavioral signs to watch for.
- **Triage Checklist**: Use checkboxes to guide initial assessment and severity classification.
- **Investigation**: Document data collection requirements and step-by-step analysis procedures.
- **Containment**: Detail immediate isolation actions and evidence preservation requirements.
- **Remediation**: Provide specific technical steps to remove threats and restore security.
- **Recovery**: Include verification steps and ongoing monitoring requirements.
- **Post-Incident**: Document reporting, metrics, and process improvements for future incidents.

---

## 5. Design Document Template

Use this template for documenting architectural designs, feature implementations, and integration plans.

```markdown
# Design: [Feature/Integration Name]

## Summary
Brief description of what is being built and why.

## Background
Context, motivation, and any relevant prior work.

## Requirements
### Functional
- FR-1: [Requirement]
- FR-2: [Requirement]

### Non-Functional
- NFR-1: Performance: [Requirement]
- NFR-2: Security: [Requirement]

## Architecture
### Components
Description of components and their interactions.

### Data Flow
How data moves through the system.

### Dependencies
External services, integrations, and libraries.

## Implementation Plan
### Phase 1: [Name]
- Tasks and deliverables

### Phase 2: [Name]
- Tasks and deliverables

## Testing Strategy
- Unit test coverage
- Integration test plan
- Manual test scenarios

## Risks and Mitigations
| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| ... | High | Medium | ... |

## Open Questions
- [Question 1]
- [Question 2]
```

### Design Document Guidelines

- **Summary**: Provide a concise overview of what is being designed and the business drivers.
- **Background**: Explain the context, previous attempts, and why this solution is needed now.
- **Requirements**: Separate functional requirements (what it does) from non-functional requirements (performance, security, scalability).
- **Architecture**: Document system components, their relationships, data flows, and external dependencies.
- **Implementation Plan**: Break the implementation into phases with clear deliverables and timeline.
- **Testing Strategy**: Define unit, integration, and manual test approaches to ensure quality.
- **Risks and Mitigations**: Identify potential risks and document strategies to address them.
- **Open Questions**: Document unresolved issues that need discussion or research.

---

## Best Practices for All Templates

### General Documentation Standards

1. **Consistency**: Use consistent formatting, terminology, and structure across all documents.
2. **Clarity**: Write for the intended audience (analysts, developers, SOC managers).
3. **Completeness**: Ensure all sections are filled with relevant information; remove N/A sections.
4. **Actionability**: Provide step-by-step instructions that can be followed without additional research.
5. **Examples**: Include real-world examples and code samples where applicable.
6. **Accessibility**: Use clear language, avoid jargon, and define technical terms on first use.
7. **Maintenance**: Keep documentation up-to-date as processes and integrations evolve.

### Formatting Conventions

- **Headings**: Use markdown heading hierarchy (# for main, ## for sections, ### for subsections).
- **Tables**: Use markdown tables for structured data, configuration parameters, and decision matrices.
- **Code**: Use backticks for inline code and triple backticks with language specifiers for code blocks.
- **Emphasis**: Use *italics* for file names and versions, **bold** for important terms.
- **Lists**: Use ordered lists for procedures and unordered lists for options.
- **Links**: Include references to related documentation, vendor resources, and internal knowledge bases.

### Content Guidelines

- **Audience Focus**: Consider who will use the document (analysts, administrators, developers, managers).
- **Detail Level**: Balance comprehensiveness with readability; use sub-sections for detailed information.
- **Procedure Steps**: Number steps clearly and provide expected outcomes for verification.
- **Error Handling**: Document common errors, troubleshooting steps, and escalation paths.
- **Security**: Never include sensitive credentials, API keys, or internal IP addresses in documentation.
- **Version Control**: Include dates, version numbers, and change logs for tracking updates.

### Maintenance Checklist

- Review documentation quarterly or after major changes
- Update links to ensure they point to current resources
- Incorporate feedback from analysts and end users
- Archive outdated versions while maintaining version history
- Test procedures periodically to ensure they remain accurate
- Update MITRE ATT&CK references as tactics and techniques evolve

---

## Template Customization Tips

### When to Customize

- **Organization-Specific Details**: Adapt severity classifications, escalation paths, and approval workflows to your organization.
- **Product-Specific Information**: Include vendor-specific configuration options, API endpoints, and authentication methods.
- **Team Processes**: Modify procedures to match your SOC's incident response workflow and tools.
- **Compliance Requirements**: Add sections for audit trails, compliance documentation, and regulatory requirements.

### Sections You Can Expand

- Add **Metrics** sections to track performance and compliance.
- Add **Automation** sections to document playbook configurations.
- Add **Training** sections to specify skills required before using the procedure.
- Add **FAQ** sections to address common questions from analysts.
- Add **Related Documents** sections to provide cross-references across documentation sets.

---

## Document Storage and Organization

### File Naming Conventions

```
integration-[vendor-name]-readme.md
runbook-[process-name].md
release-notes-[pack-name]-[version].md
ir-procedure-[threat-type].md
design-[feature-name].md
```

### Folder Structure

```
/documentation/
├── integrations/
│   ├── integration-vendor-name-readme.md
│   └── ...
├── runbooks/
│   ├── runbook-incident-response.md
│   └── ...
├── release-notes/
│   ├── release-notes-pack-v1.0.0.md
│   └── ...
├── ir-procedures/
│   ├── ir-procedure-malware.md
│   └── ...
└── design-documents/
    ├── design-feature-name.md
    └── ...
```

---

## Version Control and Change Management

- Maintain change logs with dates and descriptions of updates
- Include version numbers (MAJOR.MINOR.PATCH format) in document headers
- Use git or similar version control for tracking documentation changes
- Reference documentation versions in playbooks and procedures for traceability
- Archive outdated versions for compliance and historical reference

---

## Review and Approval Process

1. **Draft**: Create document from appropriate template
2. **Review**: Subject matter expert review for accuracy and completeness
3. **Testing**: Test procedures in lab environment to ensure accuracy
4. **Approval**: Get sign-off from team lead or manager
5. **Publication**: Publish to documentation repository with version control
6. **Notification**: Notify relevant teams of new or updated documentation
7. **Feedback**: Establish feedback mechanism for continuous improvement

