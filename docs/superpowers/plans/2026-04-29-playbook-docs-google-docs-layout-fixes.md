# Playbook Docs Google Docs Layout Fixes Implementation Plan

> **Status:** Implemented and verified. Final version `0.3.4` committed at `9527315`. This plan has been retroactively updated to reflect the working solution (the original v0.3.2 plan described a wrapper-table approach that broke nested-table columns in Google Docs and was abandoned).

**Goal:** Center flow-diagram task nodes in Google Docs and restore symmetric page margins after paste.

**Architecture:** Reference-only edits to two files in one skill. Use a 3-column spacer wrapper (15% / 70% / 15%) around each single-task flow node so Google Docs renders it as a narrow centered box. Use a bare `<body>` with no padding so Google Docs' own page margins (which are symmetric) provide the document spacing.

**Tech Stack:** Markdown reference files containing inline-styled HTML snippets. No runtime code, no test suite — validation is manual paste-into-Google-Docs.

**Spec:** `docs/superpowers/specs/2026-04-29-playbook-docs-google-docs-layout-fixes-design.md`

---

## File Structure

| File | Responsibility | Change Type |
|---|---|---|
| `skills/xsiam-docs-playbooks/references/html-styling-guide.md` | Canonical HTML/CSS templates the skill follows when emitting playbook docs | Modify (Document Shell + Task Node + Parallel Branch Split + Condition Node templates) |
| `skills/xsiam-docs-playbooks/SKILL.md` | Skill workflow + version metadata | Modify (version bump 0.3.1 → 0.3.4 + workflow-step-4 note) |

---

## Why these specific shapes (failure modes ruled out)

Three patterns that look reasonable but break in Google Docs were tried and rejected during implementation. Documented here so future maintainers don't reintroduce them:

1. **Wrapping content in `<table align="center" width="700">` to constrain document width.** Google Docs honors the `width` attribute, but the fixed-width ancestor causes nested-table columns to collapse to near-zero — argument-table headers like "Argument" wrap one letter per line, parallel-branch task boxes become slivers.
2. **Adding `align="center" width="420"` to a task-node `<table>`.** Google Docs interprets `align="center"` on `<table>` as text-align for cell contents, not as table positioning. Result: full-width task box with text centered inside.
3. **Putting `padding: 20px` on `<body>`.** Google Docs honors `padding-left` while ignoring `padding-right`, producing asymmetric document margins after paste.

The 3-column spacer wrapper centers tasks because Google Docs honors percent column widths. The bare body produces symmetric margins because Google Docs falls back to its own page margins (which are symmetric by default).

---

### Task 1: Replace Document Shell with bare body

**Files:**
- Modify: `skills/xsiam-docs-playbooks/references/html-styling-guide.md` (Document Shell section)

- [x] Find the Document Shell snippet and replace the `<body>` line with the bare-body version (no wrapper table, no body padding):

```html
<body style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; max-width: 750px; margin: 0 auto; color: #1a2533; line-height: 1.6;">

  <!-- content here -->

</body>
```

- [x] Replace the trailing explanatory paragraph with text that documents the constraint: no fixed-width wrapper; no body padding; rely on Google Docs' own page margins for symmetry.

- [x] Commit.

---

### Task 2: Apply 3-column spacer wrapper to Task Node template

**Files:**
- Modify: `skills/xsiam-docs-playbooks/references/html-styling-guide.md` Task Node (Regular/Command) section

- [x] Replace the inner `<table>` snippet with the 3-column spacer pattern. The outer wrapper row's `<td>` no longer needs `text-align: center`. The middle 70% cell holds the existing 2-cell border + content table.

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

- [x] Commit.

---

### Task 3: Strip `align="center"` and `width=` from Parallel Branch Split tables

**Files:**
- Modify: `skills/xsiam-docs-playbooks/references/html-styling-guide.md` Parallel Branch Split section

Parallel branches naturally fill the wrapper width with a 50/50 split. The legacy `align="center"` and `width="100%"` attributes from v0.3.2 don't help (and now confuse readers) — drop them.

- [x] Replace the outer split table opening tag and both branch inner-table opening tags. Remove `align="center"` and `width="..."`. Remove `text-align: center` from the wrapper `<td>`.

```html
<tr>
  <td style="padding: 4px 0;">
    <table style="width: 100%; border-collapse: collapse;">
      <tr>
        <td style="width: 50%; vertical-align: top; padding: 0 8px;">
          <table style="width: 100%; border-collapse: collapse;">
            ...
          </table>
        </td>
        <td style="width: 50%; vertical-align: top; padding: 0 8px;">
          <table style="width: 100%; border-collapse: collapse;">
            ...
          </table>
        </td>
      </tr>
    </table>
  </td>
</tr>
```

- [x] Commit.

---

### Task 4: Apply 3-column spacer wrapper to Condition Node

**Files:**
- Modify: `skills/xsiam-docs-playbooks/references/html-styling-guide.md` Condition Node with Branch Labels section

The condition node uses the same 3-column spacer pattern as Task 2. The branch-arrows row underneath keeps its 50/50 split for YES/NO label alignment but loses the `align="center"` and `width="100%"` attributes.

- [x] Replace the condition node table with the 3-column spacer wrapper (border color `#FA582D`, background `#FFF3E0`, text `#c0392b`).

- [x] Strip `align="center"` and `width="100%"` from the branch-arrows table. Remove `text-align: center` from the wrapper `<td>`.

- [x] Commit.

---

### Task 5: Update SKILL.md — version bump and step-4 note

**Files:**
- Modify: `skills/xsiam-docs-playbooks/SKILL.md`

- [x] Bump `version: 0.3.1` → `version: 0.3.4`.

- [x] In workflow step 4, add a paragraph above the brand-palette intro that documents the three constraints: no fixed-width wrapper table; use the 3-column spacer pattern to center task nodes; no body `padding`.

- [x] Commit.

---

### Task 6: Manual validation in browser and Google Docs

This task is manual — there is no automated test for HTML paste fidelity into Google Docs.

- [x] Generate documentation for a representative playbook (sequential + parallel branches). Test playbook used: `SC - Okta - Unapproved User Creation`.
- [x] Open in a browser — verify task nodes are visibly narrower than the page and centered.
- [x] Paste into a fresh Google Doc — verify single-task flow nodes are centered narrow boxes, parallel-branch nodes split 50/50 cleanly with no column-collapse, document has symmetric page margins, and argument tables render with normal column widths.
- [x] User confirmed the v0.3.4 output is correct.

---

## Self-Review

**Spec coverage:**
- Spec Fix 1 (bare body shell, no padding) → Task 1.
- Spec Fix 2 (3-column spacer for single tasks) → Task 2 (Task Node), Task 4 (Condition Node).
- Parallel-branch attribute cleanup → Task 3.
- SKILL.md version bump + step-4 note → Task 5.
- Spec "Validation" → Task 6.
All spec sections covered.

**Iteration record:** v0.3.2 (wrapper table, broken) and v0.3.3 (reverted wrapper, still not centered) are documented in commit history (`4c651b2` … `d17baa5`). The final working state is v0.3.4 (`9527315`). Future maintainers can read the spec's "Iteration history" section to understand what was tried and why each prior attempt failed.
