---
name: xsiam-docs
description: >
  This skill should be used when the user asks to "write documentation",
  "create a README", "build a runbook", "write an SOP", "release notes",
  "incident response procedure", "integration documentation",
  "XSIAM documentation", "design doc", or needs to generate any type of
  documentation for Cortex XSIAM content, integrations, or SOC operations.
version: 0.1.0
---

# XSIAM Documentation

Generate documentation for XSIAM content: integration READMEs, runbooks, SOPs, release notes, IR procedures, and design docs.

## Before Starting

Read the reference file for documentation templates:
- `references/doc-templates.md` — Complete templates for all documentation types

## Workflow

### 1. Determine Documentation Type

Ask the user which type they need:
- **Integration README** — setup guide and command reference for an integration
- **Runbook / SOP** — step-by-step operational procedure for SOC teams
- **Release Notes** — changelog for a content pack version
- **IR Procedure** — incident response procedure for a specific threat type
- **Design Document** — architecture and implementation plan for new content

### 2. Gather Content

For each doc type, collect the necessary information:

**Integration README**:
- Integration name and vendor
- Authentication method and configuration parameters
- List of commands with their arguments and outputs
- Setup prerequisites

**Runbook / SOP**:
- Procedure name and purpose
- Trigger conditions (when to use this runbook)
- Step-by-step actions with expected outcomes
- Escalation criteria and contacts
- Related playbooks and XQL queries

**Release Notes**:
- Pack name and version number
- Changes by content type (integrations, scripts, playbooks)
- Bug fixes, new features, breaking changes
- Docker image updates

**IR Procedure**:
- Threat category and MITRE ATT&CK techniques
- Detection indicators and alert sources
- Investigation and containment steps
- Related playbooks and XQL queries

**Design Document**:
- Feature/integration being built
- Requirements (functional and non-functional)
- Architecture decisions and trade-offs
- Implementation phases

### 3. Generate the Document

Follow the appropriate template from `references/doc-templates.md`. Adapt sections to the specific content — skip sections that don't apply, expand sections that need detail.

**Writing standards**:
- Use clear, direct language. Avoid jargon unless it's standard XSIAM terminology.
- Write procedures as numbered steps with expected outcomes.
- Include specific command examples, XQL queries, or configuration values where relevant.
- Use tables for structured data (parameters, commands, fields).
- Include troubleshooting sections for integration docs and runbooks.

### 4. Cross-Reference Other Content

Where appropriate, link to related XSIAM content:
- Reference playbook names that implement the procedure
- Include XQL queries for investigation steps in runbooks
- Reference integration commands in IR procedures
- Link design docs to the content they describe

### 5. Output Format

Deliver documentation as Markdown files with descriptive filenames:
- `IntegrationName_README.md`
- `Runbook_ThreatType.md`
- `ReleaseNotes_PackName_Version.md`
- `IR_Procedure_ThreatType.md`
- `Design_FeatureName.md`

### 6. Quality Checklist

Before delivering, verify:
- [ ] All sections from the template are addressed (or explicitly noted as N/A)
- [ ] Command examples use correct syntax
- [ ] XQL queries are syntactically valid
- [ ] Configuration parameters match the actual integration YAML
- [ ] Escalation paths include specific contacts/channels
- [ ] Procedure steps are actionable (not vague)
- [ ] Tables are properly formatted
- [ ] No placeholder text remains
