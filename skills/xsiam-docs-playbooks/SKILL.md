---
name: xsiam-docs-playbooks
description: >
  This skill should be used when the user asks to "document a playbook",
  "create playbook documentation", "playbook README", "playbook docs",
  "describe this playbook", "explain this playbook", "playbook flow diagram",
  "playbook reference doc", "playbook design doc", or needs to generate
  professional HTML documentation for a Cortex XSIAM or XSOAR playbook.
  Use this whenever the user has a playbook YAML file and needs documentation,
  or when they describe a playbook and want a formatted reference document.
  This skill produces Google Docs-ready HTML with visual flow diagrams —
  not plain markdown.
version: 0.3.3
---

# XSIAM Playbook Documentation

Generate professional, visually rich HTML documentation for XSIAM/XSOAR playbooks. The output is designed to paste directly into Google Docs with full formatting preserved.

## Before Starting

Read these reference files:
- `references/playbook-doc-spec.md` — Section specifications, what to extract from playbook YAML, and content guidelines
- `references/html-styling-guide.md` — HTML/CSS patterns that survive Google Docs paste, component templates

## Why HTML Instead of Markdown

Google Docs strips markdown formatting on paste. HTML with inline styles pastes cleanly — tables, colors, backgrounds, and typography all transfer. Every style must be inline (no `<style>` blocks or CSS classes) because Google Docs ignores them.

## Header Hierarchy

The playbook name is the top-level heading, rendered as a standard `<h4>`. All content sections (Description, Flow Diagram, Task Inventory, etc.) use `<h5>`, and sub-sections within those use `<h6>`. Never use `<h1>`, `<h2>`, or `<h3>`.

## Workflow

### 1. Get the Playbook Source

The user will provide one of:
- **A playbook YAML file path** — read it and parse the structure
- **A pasted playbook YAML** — parse directly from the conversation
- **A description of a playbook** — work from the description (less detail, but still useful)

If a YAML file is provided, read and parse it to extract:
- Playbook name, description, ID
- All tasks (type, name, description, script/command, arguments)
- Task flow (nexttasks connections)
- Conditions and their branch logic
- Inputs and outputs
- Sub-playbook references
- Error handling paths

### 2. Build the Document Structure

The HTML document follows this section order (see `references/playbook-doc-spec.md` for detail on each):

1. **Playbook Title** — Playbook name as a standard `<h4>` heading
2. **Overview Card** — Metadata table: trigger, category, integrations, severity
3. **Description** — What this playbook does, when it runs, what it accomplishes
4. **Flow Diagram** — Visual representation of the task flow using styled HTML tables
5. **Task Inventory** — Summary table of all tasks with type badges, integration, and purpose
6. **Detailed Task Reference** — Each task expanded with inputs, outputs, conditions
7. **Decision Logic** — Condition tasks with their branch criteria and paths
8. **Inputs & Outputs** — Playbook-level parameters and return values
9. **Integration Dependencies** — What integrations must be configured and which commands are used
10. **Error Handling** — How the playbook handles failures, error paths, retry logic
11. **Related Content** — Links to sub-playbooks, scripts, or related playbooks

Skip sections that don't apply (e.g., no Error Handling section if no error paths exist).

### 3. Build the Flow Diagram

The flow diagram is the most important visual element. It communicates the playbook's logic at a glance.

**Use a vertical table-based layout** with these components:
- **Task nodes** — Table cells with colored left borders indicating task type:
  - Blue (`#1976D2`) for regular/command tasks
  - Orange (`#F57C00`) for condition/decision tasks
  - Purple (`#7B1FA2`) for sub-playbook tasks
  - Green (`#388E3C`) for the start task
  - Gray (`#616161`) for section/title tasks
- **Arrow connectors** — Use `↓` characters between sequential tasks, styled in gray
- **Branch labels** — Show condition outcomes (Yes/No, or named branches) next to arrows
- **Parallel paths** — Use side-by-side table cells for tasks that execute concurrently
- **Convergence points** — Show where parallel paths merge back

The diagram should read top-to-bottom. For complex playbooks with many parallel branches, consider grouping phases (Enrichment, Decision, Action, Closure) with section labels.

See `references/html-styling-guide.md` for the exact HTML patterns for flow diagram nodes, arrows, and branches.

### 4. Apply HTML Styling

Do **not** wrap the document body in a fixed-width container table — Google Docs' table-layout algorithm collapses nested-table columns when there is a fixed-width ancestor, which destroys the parallel-branch flow nodes and the argument tables. Let inner tables fill Google Docs' own page width, and use `align="center"` on each flow-diagram task node to center them. Symmetric page margins come from Google Docs' own page settings, not from body padding (which Google Docs has been observed to apply asymmetrically). See the Document Shell pattern in `references/html-styling-guide.md`.

All styling must use inline CSS and follow the **Palo Alto Networks brand palette** defined in `references/html-styling-guide.md`. The key brand colors are:
- **Navy `#003366`** — playbook title, section headers, table headers
- **PAN Orange `#FA582D`** — accent bar in title, condition highlights, decision logic table headers, warning callouts
- **Light blue-gray `#f7f9fc`** — overview card labels, alternate table rows, flow diagram background

Follow the patterns in `references/html-styling-guide.md` for:

- **Playbook title** — Standard `<h4>` heading with navy color and orange bottom border (no banner, no branding label)
- **Callout boxes** — Light background with colored left border (info=blue, warning=amber, tip=green)
- **Tables** — Colored header row, alternating row backgrounds, clean borders
- **Type badges** — Small inline spans with rounded corners and background colors
- **Code blocks** — Monospace font with light gray background
- **Section headers** — Consistent sizing with bottom borders

### 5. Output

Deliver the documentation as a single `.html` file named after the playbook:
- `PlaybookName_Documentation.html`

The file should be a complete HTML document with `<html>`, `<head>`, and `<body>` tags. Include a minimal `<style>` block only for print media queries — all visual styling must be inline.

Do NOT include any "Generated by" footer or branding line.

Tell the user: "Open this HTML file in your browser, then Select All (Cmd+A) and Copy (Cmd+C), then paste into Google Docs. All formatting will transfer."

### 6. Quality Checklist

Before delivering, verify:
- [ ] Every task from the YAML appears in both the flow diagram and task inventory
- [ ] All condition branches are documented with their criteria
- [ ] Flow diagram accurately represents the nexttasks connections
- [ ] No orphaned tasks (every task is reachable from start)
- [ ] Parallel execution paths are visually distinguished from sequential ones
- [ ] Integration dependencies list all integrations referenced by command tasks
- [ ] Inputs/outputs match the playbook YAML definitions
- [ ] All inline styles are present (no reliance on CSS classes)
- [ ] HTML renders correctly in a browser
- [ ] Tables are not excessively wide (stay within ~700px for Google Docs)
- [ ] No placeholder text remains
