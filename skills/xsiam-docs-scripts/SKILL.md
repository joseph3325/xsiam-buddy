---
name: xsiam-docs-scripts
description: >
  This skill should be used when the user asks to "document a script",
  "create script documentation", "script README", "script docs",
  "describe this script", "explain this script", "script reference doc",
  "script design doc", "document this automation", or needs to generate
  professional HTML documentation for a Cortex XSIAM or XSOAR automation
  script. Use this whenever the user has a script YAML file and needs
  documentation, or when they describe a script and want a formatted
  reference document. This skill produces Google Docs-ready HTML with
  visual diagrams — not plain markdown.
---

# XSIAM Script Documentation

Generate professional, visually rich HTML documentation for XSIAM/XSOAR automation scripts. The output is designed to paste directly into Google Docs with full formatting preserved.

## Before Starting

Read these reference files:
- `references/script-doc-spec.md` — Section specifications, what to extract from script YAML, and content guidelines
- `../xsiam-docs-playbooks/references/html-styling-guide.md` — HTML/CSS patterns that survive Google Docs paste, component templates

## Why HTML Instead of Markdown

Google Docs strips markdown formatting on paste. HTML with inline styles pastes cleanly — tables, colors, backgrounds, and typography all transfer. Every style must be inline (no `<style>` blocks or CSS classes) because Google Docs ignores them.

## Workflow

### 1. Get the Script Source

The user will provide one of:
- **A script YAML file path** — read it and parse the structure
- **A pasted script YAML** — parse directly from the conversation
- **A description of a script** — work from the description (less detail, but still useful)

If a YAML file is provided, read and parse it to extract:
- Script name, comment (description), tags
- All arguments (name, type, required, defaults, description)
- All outputs (contextPath, description, type)
- The embedded Python code (`script: |-` block)
- Docker image, execution target, permissions level
- Script type based on tags (standard, transformer, filter, dynamic-section, field-change-triggered, widget)

### 2. Analyze the Python Code

This is the most important step for script documentation. Read through the embedded Python carefully and understand:
- **What it does** — the core logic, not just what the `comment` field says
- **How it processes arguments** — which arg-parsing helpers it uses, validation logic
- **What external calls it makes** — `demisto.executeCommand()` calls to other integrations/scripts
- **How it returns data** — `CommandResults`, raw values (transformers), booleans (filters)
- **Error handling** — what the `try/except` catches, how failures surface
- **Context data access** — `demisto.alert()`, `demisto.context()`, `demisto.incident()`

Build a mental model of the data flow: input args enter, get processed, external calls may happen, results get formatted and returned.

### 3. Build the Document Structure

The HTML document follows this section order (see `references/script-doc-spec.md` for detail on each):

1. **Title Banner** — Script name with colored header bar
2. **Overview Card** — Metadata table: type, tags, execution target, permissions, docker image
3. **Description** — What this script does, when to use it, what problem it solves
4. **Data Flow Diagram** — Visual representation of input-to-output flow
5. **Arguments Reference** — Table of all arguments with types, defaults, and descriptions
6. **Outputs Reference** — Table of all context outputs with paths and types
7. **Logic Walkthrough** — Step-by-step explanation of what the Python code does
8. **Usage Examples** — How to invoke the script (war room, playbook task, transformer/filter context)
9. **Integration Dependencies** — What integrations it chains to via `executeCommand()`
10. **Error Handling** — How the script handles failures and edge cases
11. **Related Content** — Links to related scripts, integrations, or playbooks

Skip sections that don't apply (e.g., no Integration Dependencies section if no `executeCommand()` calls exist, no Data Flow Diagram for trivial scripts).

### 4. Build the Data Flow Diagram

The data flow diagram communicates the script's processing pipeline at a glance. Unlike a playbook flow (which maps task-to-task connections), a script flow maps the journey from input arguments through processing logic to output.

**Use a vertical table-based layout** with these components:
- **Input nodes** — Green left-border nodes showing each argument entering the script
- **Processing nodes** — Blue left-border nodes for each major logic step
- **External call nodes** — Purple left-border nodes for `executeCommand()` calls
- **Decision nodes** — Orange left-border nodes for conditional branches in the logic
- **Output nodes** — Navy left-border nodes for the final return values
- **Arrow connectors** — Use `▼` characters between sequential steps

The diagram should read top-to-bottom, showing the data transformation pipeline. Group the input arguments into a single "Inputs" phase, show the processing steps, and end with the outputs.

For simple scripts (parse args, process, return), keep the diagram compact. For complex scripts with branching logic or multiple `executeCommand()` calls, show each path.

See `../xsiam-docs-playbooks/references/html-styling-guide.md` for the exact HTML patterns for nodes and arrows — adapt the playbook flow diagram patterns for script data flows.

### 5. Apply HTML Styling

All styling must use inline CSS and follow the **Palo Alto Networks brand palette** defined in `../xsiam-docs-playbooks/references/html-styling-guide.md`. The key brand colors are:
- **Navy `#003366`** — title banners, section headers, table headers
- **PAN Orange `#FA582D`** — accent bar in title, decision highlights, warning callouts
- **Light blue-gray `#f7f9fc`** — overview card labels, alternate table rows, diagram background

Follow the patterns in the HTML styling guide for:
- **Title banner** — Single navy block with "PALO ALTO NETWORKS | CORTEX XSIAM" label, thin orange accent line, then script name and description
- **Callout boxes** — Light background with colored left border (info=blue, warning=amber, tip=green)
- **Tables** — Colored header row, alternating row backgrounds, clean borders
- **Type badges** — Small inline spans with rounded corners and background colors
- **Code blocks** — Monospace font with dark background for Python snippets
- **Section headers** — Consistent sizing with bottom borders

### 6. Script-Type-Specific Sections

The script's tags determine its type, and each type has unique documentation needs:

| Type | Extra documentation to include |
|------|-------------------------------|
| **Standard** | Full args/outputs tables, usage in playbook tasks and war room |
| **Transformer** | How the `value` argument is populated automatically, where it appears in playbook input mappings |
| **Filter** | How the boolean return drives playbook condition branching |
| **Dynamic Section** | Where it appears in layouts, what context data it reads, that it re-executes on tab open |
| **Field-Change Triggered** | What field triggers it, the payload structure (`new`, `old`, `cliName`), how to assign it in layout editor |
| **Widget** | The expected return schema for the widget type, dashboard refresh behavior |

Add a "Script Type" callout near the top of the document (after Description) that explains what the type means for how this script is used.

### 7. Output

Deliver the documentation as a single `.html` file named after the script:
- `ScriptName_Documentation.html`

The file should be a complete HTML document with `<html>`, `<head>`, and `<body>` tags. Include a minimal `<style>` block only for print media queries — all visual styling must be inline.

Tell the user: "Open this HTML file in your browser, then Select All (Cmd+A) and Copy (Cmd+C), then paste into Google Docs. All formatting will transfer."

### 8. Quality Checklist

Before delivering, verify:
- [ ] Every argument from the YAML appears in the Arguments Reference table
- [ ] Every output from the YAML appears in the Outputs Reference table
- [ ] The Logic Walkthrough accurately reflects what the Python code does
- [ ] The Description explains the "why" — not just the "what"
- [ ] Script type is correctly identified from tags and documented with type-specific guidance
- [ ] Data flow diagram accurately represents the processing pipeline
- [ ] Usage examples match the script's actual arguments and invocation pattern
- [ ] All `executeCommand()` calls are documented in Integration Dependencies
- [ ] Error handling behavior is documented (what happens on failure)
- [ ] Code snippets in the doc use dark background code blocks for readability
- [ ] All inline styles are present (no reliance on CSS classes)
- [ ] HTML renders correctly in a browser
- [ ] Tables are not excessively wide (stay within ~750px for Google Docs)
- [ ] No placeholder text remains
