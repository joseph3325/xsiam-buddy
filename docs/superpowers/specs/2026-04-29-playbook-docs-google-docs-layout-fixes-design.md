# Playbook Docs — Google Docs Layout Fixes

**Date:** 2026-04-29
**Skill:** `xsiam-docs-playbooks` (bump 0.3.1 → 0.3.2)

## Problem

When the HTML output of the `xsiam-docs-playbooks` skill is pasted into Google Docs, two layout defects appear:

1. **Flow-diagram task nodes are left-aligned** inside the diagram wrapper instead of centered. The current template centers via `margin: 0 auto` on the inner task table, but Google Docs strips that style and falls back to default left alignment for block-level tables.
2. **The document body has no right margin** in Google Docs — content runs flush against the right edge of the page while a normal indent appears on the left. The body's `max-width: 750px; margin: 0 auto` is ignored by Google Docs because it discards `<body>` styles on paste, so inner tables with `width: 100%` stretch to the full page width.

Both render correctly in a browser; both break only after the Google Docs paste.

## Goals

- Task nodes (start, command, condition, sub-playbook, section, manual) center horizontally inside the flow-diagram wrapper in Google Docs.
- Pasted documents have symmetric left/right margins inside Google Docs.
- No regression in browser rendering.
- No change to the skill's content sections, semantic structure, or section order.

## Non-Goals

- No restyling of colors, typography, or spacing.
- No change to the markdown-vs-HTML rationale or any other skill in the plugin.

## Approach

### Fix 1 — Centered container table for the document body

Replace the current body shell. Body becomes a thin wrapper; a single centered container table holds all content. Google Docs respects the legacy `align="center"` HTML attribute and the `width` attribute on `<table>` elements.

**Before** (`references/html-styling-guide.md` Document Shell):

```html
<body style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; max-width: 750px; margin: 0 auto; padding: 20px; color: #1a2533; line-height: 1.6;">

  <!-- content here -->

</body>
```

**After:**

```html
<body style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; color: #1a2533; line-height: 1.6; margin: 0; padding: 0;">
<table align="center" width="700" cellpadding="0" cellspacing="0" border="0" style="margin: 0 auto;">
  <tr>
    <td style="padding: 20px;">

      <!-- all content here -->

    </td>
  </tr>
</table>
</body>
```

All existing inner tables keep `width: 100%` and naturally fit inside the 700px container.

### Fix 2 — `align="center"` on every task-node inner table

Every inner task-node table in the flow diagram receives the `align="center"` HTML attribute and a `width` attribute alongside the existing `style`. Affected snippets in `references/html-styling-guide.md`:

- Task Node (Regular/Command) template
- Condition Node template
- Parallel Branch Split inner tables (each branch's inner table)

**Before** (Regular/Command and Condition Node templates):

```html
<table style="margin: 0 auto; border-collapse: collapse; width: 420px;">
```

**After:**

```html
<table align="center" width="420" style="margin: 0 auto; border-collapse: collapse;">
```

For the Parallel Branch Split, the outer split table and each branch's inner table use `width: 100%` rather than a fixed pixel width. Apply the same pattern using `width="100%"`:

```html
<table align="center" width="100%" style="margin: 0 auto; border-collapse: collapse;">
```

The Task Node Color Variants table (start, condition, sub-playbook, section, manual) does not need duplicate updates because the variants share the same outer template — only the colors change.

### SKILL.md changes

- Bump `version` from `0.3.1` to `0.3.2`.
- Add a one-line note in workflow step 4 (Apply HTML Styling) clarifying that the document body is wrapped in a centered 700px container table.

## Files Touched

| File | Change |
|---|---|
| `skills/xsiam-docs-playbooks/references/html-styling-guide.md` | Replace Document Shell snippet; update Task Node, Condition Node, and Parallel Branch Split tables to add `align="center"` and `width` attributes; add a short note explaining why these legacy attributes are used. |
| `skills/xsiam-docs-playbooks/SKILL.md` | Version bump; one-line note about the wrapper table. |

## Validation

1. Generate documentation for a representative playbook (one with sequential, parallel, and condition branches) using the updated skill.
2. Open the resulting `.html` in a browser — verify task nodes are centered and the body still looks correct.
3. Select All, Copy, paste into a fresh Google Doc — verify:
   - Task nodes (Task 0: Start, parallel Task 1/Task 2, condition nodes) are centered inside the flow-diagram wrapper.
   - Left and right page margins are symmetric.
   - All other sections (overview card, task inventory, decision logic, inputs/outputs) render unchanged.

## Risk

Very low. Changes are confined to two files in one skill, are reference-data only (no runtime code), and use widely-supported legacy HTML table attributes.
