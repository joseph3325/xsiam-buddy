# Playbook Docs — Google Docs Layout Fixes

**Date:** 2026-04-29
**Skill:** `xsiam-docs-playbooks` (bumped 0.3.1 → 0.3.4)

## Problem

When the HTML output of the `xsiam-docs-playbooks` skill is pasted into Google Docs, two layout defects appear:

1. **Flow-diagram task nodes are left-aligned** inside the diagram wrapper instead of centered. The original template centered via `margin: 0 auto` on the inner task table, but Google Docs strips that style. `align="center"` on `<table>` does *not* center the table — Google Docs treats it as text alignment for cell contents instead. `text-align: center` on the wrapper `<td>` also fails because block-level tables don't honor it.
2. **The document body has no right margin** in Google Docs — content runs flush against the right edge of the page while a normal indent appears on the left. The original body used `padding: 20px`, and Google Docs has been observed to honor `padding-left` while ignoring `padding-right`, producing the asymmetric margins.

Both render correctly in a browser; both break only after the Google Docs paste.

## Goals

- Task nodes (start, command, condition, sub-playbook, section, manual) appear as narrow centered boxes inside the flow-diagram wrapper in Google Docs.
- Pasted documents have symmetric left/right margins inside Google Docs.
- No regression in browser rendering.
- No change to the skill's content sections, semantic structure, or section order.

## Non-Goals

- No restyling of colors, typography, or spacing.
- No change to the markdown-vs-HTML rationale or any other skill in the plugin.

## Iteration history

This work shipped in three iterations because Google Docs' table-layout algorithm has unintuitive interactions with nested tables and fixed-width ancestors:

- **v0.3.2** — Wrapped body in a centered 700px `<table align="center" width="700">` and added `align="center"` + `width=` attributes to every task-node table. Symptom in Google Docs: task nodes still rendered full-width inside the wrapper (text centered, not box centered), and — worse — the fixed-width ancestor caused nested-table columns to collapse to near-zero width in parallel-branch flow nodes and argument tables (text wrapped one letter per line).
- **v0.3.3** — Reverted the wrapper table; switched body to `max-width: 750px; margin: 0 auto` (browser only) with no padding. This fixed the column-collapse problem and produced symmetric Google Docs margins, but task nodes were still not centered — Google Docs interprets `align="center"` on `<table>` as text-align for cell contents.
- **v0.3.4** — Replaced each single-task node template with a 3-column spacer wrapper (15% empty / 70% task / 15% empty). Google Docs honors percent column widths, producing visually centered task boxes. Parallel-branch nodes keep their 50/50 split (no spacer needed).

## Final Approach

### Fix 1 — Bare body shell (symmetric margins)

`references/html-styling-guide.md` Document Shell:

```html
<body style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; max-width: 750px; margin: 0 auto; color: #1a2533; line-height: 1.6;">

  <!-- content here -->

</body>
```

Notes:
- `max-width: 750px; margin: 0 auto` are browser-only (Google Docs strips body styles).
- No body `padding` — Google Docs has been observed to apply `padding-left` while ignoring `padding-right`, producing asymmetric margins after paste.
- **Do not** wrap content in an outer container table with a fixed pixel width. A fixed-width ancestor causes Google Docs to collapse nested-table columns in parallel-branch flow nodes and argument tables.

### Fix 2 — 3-column spacer wrapper for task nodes

Single-task flow nodes (start, command, sub-playbook, section, manual, condition) wrap in a 3-column row: empty 15% cells flank a 70% middle cell containing the task table.

```html
<tr>
  <td style="padding: 4px 0;">
    <table style="width: 100%; border-collapse: collapse;">
      <tr>
        <td style="width: 15%;"></td>
        <td style="width: 70%;">
          <table style="width: 100%; border-collapse: collapse;">
            <tr>
              <td style="width: 6px; background-color: #1976D2; border-radius: 4px 0 0 4px;"></td>
              <td style="padding: 10px 16px; background-color: #E3F2FD; border: 1px solid #BBDEFB; border-left: none; border-radius: 0 4px 4px 0;">
                <span style="font-weight: bold; color: #1565C0; font-size: 14px;">Task 2: Enrich IP Address</span><br>
                <span style="color: #7a8898; font-size: 12px;">ip — CrowdStrike</span>
              </td>
            </tr>
          </table>
        </td>
        <td style="width: 15%;"></td>
      </tr>
    </table>
  </td>
</tr>
```

Parallel-branch splits keep their 50/50 layout (no spacer cells needed). Condition nodes use the same 3-column spacer pattern as regular task nodes; the branch-arrows row uses 50/50 to align YES/NO labels under the condition box.

### SKILL.md changes

- `version: 0.3.1` → `0.3.4`.
- Added a paragraph in workflow step 4 explaining: do not use a fixed-width wrapper; use the 3-column spacer pattern to center task nodes; do not put `padding` on `<body>`.

## Files Touched

| File | Change |
|---|---|
| `skills/xsiam-docs-playbooks/references/html-styling-guide.md` | Bare body shell; 3-column spacer wrapper applied to Task Node and Condition Node templates; explanatory notes covering the failure modes of fixed-width wrappers, body padding, and `align="center"` on `<table>`. |
| `skills/xsiam-docs-playbooks/SKILL.md` | Version bump; workflow-step-4 paragraph documenting the constraints. |

## Validation

1. Generate documentation for a representative playbook (sequential + parallel + condition branches) using the updated skill.
2. Open the resulting `.html` in a browser — verify task nodes are visibly narrower than the page and centered, and the body still looks correct.
3. Select All, Copy, paste into a fresh Google Doc — verify:
   - Single-task flow nodes (Task 0: Start, sub-playbook, section, etc.) render as narrow centered boxes.
   - Parallel-branch nodes (e.g., Task 8 + Task 9) split 50/50 with no column-collapse artifacts.
   - Document content has symmetric left and right page margins.
   - Argument tables in the Detailed Task Reference render with normal column widths (no one-letter-per-line collapse).

## Risk

Low. Changes are confined to two files in one skill, are reference-data only (no runtime code), and use widely-supported HTML table layout primitives. The iteration history above documents the failure modes that were ruled out.

## Final Commit Trail

- `4c651b2` v0.3.2 — wrapper-table approach (broken — columns collapsed)
- `33418c2`, `4677cdc`, `ddab3fe` — `align="center"` on task tables (didn't center in Google Docs)
- `14cd9cb` — v0.3.2 version bump
- `d17baa5` — v0.3.3 — reverted wrapper, dropped body padding
- `9527315` — v0.3.4 — 3-column spacer wrapper (final working solution)
