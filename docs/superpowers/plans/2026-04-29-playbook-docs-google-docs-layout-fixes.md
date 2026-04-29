# Playbook Docs Google Docs Layout Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix two Google Docs paste defects in the `xsiam-docs-playbooks` skill — center flow-diagram task nodes and restore symmetric page margins.

**Architecture:** Reference-only edits to two files in one skill. Replace the body shell with a centered 700px wrapper table; add `align="center"` and `width` HTML attributes to every task-node table so Google Docs honors centering even after stripping CSS `margin: 0 auto`.

**Tech Stack:** Markdown reference files containing inline-styled HTML snippets. No runtime code, no test suite — validation is manual paste-into-Google-Docs.

**Spec:** `docs/superpowers/specs/2026-04-29-playbook-docs-google-docs-layout-fixes-design.md`

---

## File Structure

| File | Responsibility | Change Type |
|---|---|---|
| `skills/xsiam-docs-playbooks/references/html-styling-guide.md` | Canonical HTML/CSS templates the skill follows when emitting playbook docs | Modify (6 snippet replacements + 1 explanatory note) |
| `skills/xsiam-docs-playbooks/SKILL.md` | Skill workflow + version metadata | Modify (version bump + one-line note in step 4) |

No new files. No tests (the skill emits HTML; correctness is verified by rendering output and pasting into Google Docs).

---

### Task 1: Replace Document Shell with centered container table

**Files:**
- Modify: `skills/xsiam-docs-playbooks/references/html-styling-guide.md` (Document Shell section, around line 26–46)

- [ ] **Step 1: Replace the Document Shell snippet**

In `skills/xsiam-docs-playbooks/references/html-styling-guide.md`, find the Document Shell code block:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Playbook: [Name]</title>
  <style>
    @media print {
      .page-break { page-break-before: always; }
    }
  </style>
</head>
<body style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; max-width: 750px; margin: 0 auto; padding: 20px; color: #1a2533; line-height: 1.6;">

  <!-- content here -->

</body>
</html>
```

Replace it with:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Playbook: [Name]</title>
  <style>
    @media print {
      .page-break { page-break-before: always; }
    }
  </style>
</head>
<body style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; color: #1a2533; line-height: 1.6; margin: 0; padding: 0;">
<table align="center" width="700" cellpadding="0" cellspacing="0" border="0" style="margin: 0 auto;">
  <tr>
    <td style="padding: 20px;">

      <!-- all content here -->

    </td>
  </tr>
</table>
</body>
</html>
```

- [ ] **Step 2: Replace the trailing explanatory line**

Immediately after the snippet, find:

```
Keep max-width at 750px so tables don't overflow in Google Docs.
```

Replace with:

```
The outer `<table align="center" width="700">` is what actually centers the document in Google Docs. Body styles are stripped on paste, but Google Docs honors the legacy `align` and `width` attributes on `<table>`. All inner content tables can keep `width: 100%` and they will fit inside the 700px container.
```

- [ ] **Step 3: Verify the file still parses as Markdown**

Run: `head -50 skills/xsiam-docs-playbooks/references/html-styling-guide.md`
Expected: Document Shell heading visible, fenced code block intact, no broken backticks.

- [ ] **Step 4: Commit**

```bash
git add skills/xsiam-docs-playbooks/references/html-styling-guide.md
git commit -m "docs(playbook-docs): wrap body in centered 700px container table

Google Docs strips body styles on paste, so max-width: 750px and
margin: 0 auto are ignored — content tables stretch to the full page
width on the right while a default first-line indent appears on the
left. A wrapper table with align=\"center\" and width=\"700\" gives
symmetric margins because Google Docs honors those legacy attributes."
```

---

### Task 2: Add align="center" + width to Regular/Command task node

**Files:**
- Modify: `skills/xsiam-docs-playbooks/references/html-styling-guide.md` line 120

- [ ] **Step 1: Update the Task Node (Regular/Command) snippet**

Find the line (inside the "### Task Node (Regular/Command)" section):

```html
    <table style="margin: 0 auto; border-collapse: collapse; width: 420px;">
```

Replace with:

```html
    <table align="center" width="420" style="margin: 0 auto; border-collapse: collapse;">
```

- [ ] **Step 2: Commit**

```bash
git add skills/xsiam-docs-playbooks/references/html-styling-guide.md
git commit -m "docs(playbook-docs): center Regular/Command task node in Google Docs"
```

---

### Task 3: Add align="center" + width to Parallel Branch Split tables

**Files:**
- Modify: `skills/xsiam-docs-playbooks/references/html-styling-guide.md` lines 176, 180, 192

- [ ] **Step 1: Update the outer parallel-split table (line 176)**

Find (the first `<table>` inside the "### Parallel Branch Split" section, immediately after the `<td style="text-align: center; padding: 4px 0;">` opening):

```html
    <table style="margin: 0 auto; border-collapse: collapse; width: 100%;">
```

This pattern appears multiple times in the file. To target only this occurrence, locate it as the first occurrence inside the Parallel Branch Split section (it is followed within a few lines by a comment `<!-- Branch 1 -->`).

Replace with:

```html
    <table align="center" width="100%" style="margin: 0 auto; border-collapse: collapse;">
```

- [ ] **Step 2: Update Branch 1 inner table (line 180)**

Inside the `<!-- Branch 1 -->` block, find:

```html
          <table style="margin: 0 auto; border-collapse: collapse; width: 100%;">
```

Replace with:

```html
          <table align="center" width="100%" style="margin: 0 auto; border-collapse: collapse;">
```

- [ ] **Step 3: Update Branch 2 inner table (line 192)**

Inside the `<!-- Branch 2 -->` block, find the same pattern:

```html
          <table style="margin: 0 auto; border-collapse: collapse; width: 100%;">
```

Replace with:

```html
          <table align="center" width="100%" style="margin: 0 auto; border-collapse: collapse;">
```

- [ ] **Step 4: Verify all three replacements landed**

Run:
```bash
grep -n 'align="center" width="100%"' skills/xsiam-docs-playbooks/references/html-styling-guide.md
```
Expected: at least three matching lines from the Parallel Branch Split section.

- [ ] **Step 5: Commit**

```bash
git add skills/xsiam-docs-playbooks/references/html-styling-guide.md
git commit -m "docs(playbook-docs): center parallel-branch split tables in Google Docs"
```

---

### Task 4: Add align="center" + width to Condition Node and its branch-arrows table

**Files:**
- Modify: `skills/xsiam-docs-playbooks/references/html-styling-guide.md` lines 214 and 228

- [ ] **Step 1: Update the Condition Node table (line 214)**

Inside the "### Condition Node with Branch Labels" section, find the Condition node's inner table:

```html
    <table style="margin: 0 auto; border-collapse: collapse; width: 420px;">
```

Replace with:

```html
    <table align="center" width="420" style="margin: 0 auto; border-collapse: collapse;">
```

- [ ] **Step 2: Update the Branch arrows table (line 228)**

A few lines below, inside the `<!-- Branch arrows -->` block, find:

```html
    <table style="margin: 0 auto; border-collapse: collapse; width: 100%;">
```

This is the only remaining occurrence of this exact line in the file after Task 3. Replace with:

```html
    <table align="center" width="100%" style="margin: 0 auto; border-collapse: collapse;">
```

- [ ] **Step 3: Verify no `margin: 0 auto; border-collapse: collapse; width:` lines remain**

Run:
```bash
grep -n 'margin: 0 auto; border-collapse: collapse; width:' skills/xsiam-docs-playbooks/references/html-styling-guide.md
```
Expected: no output (zero matches).

- [ ] **Step 4: Verify the new attribute pattern count**

Run:
```bash
grep -c 'align="center" width=' skills/xsiam-docs-playbooks/references/html-styling-guide.md
```
Expected: `7` (one in Document Shell wrapper + six task-node tables).

- [ ] **Step 5: Commit**

```bash
git add skills/xsiam-docs-playbooks/references/html-styling-guide.md
git commit -m "docs(playbook-docs): center condition node and branch arrows in Google Docs"
```

---

### Task 5: Update SKILL.md — version bump and wrapper note

**Files:**
- Modify: `skills/xsiam-docs-playbooks/SKILL.md` (front-matter line 13, plus workflow step 4)

- [ ] **Step 1: Bump the version**

Find the front-matter line:

```yaml
version: 0.3.1
```

Replace with:

```yaml
version: 0.3.2
```

- [ ] **Step 2: Add the wrapper-table note to step 4**

In the "### 4. Apply HTML Styling" section, find the line:

```
All styling must use inline CSS and follow the **Palo Alto Networks brand palette** defined in `references/html-styling-guide.md`. The key brand colors are:
```

Insert a new paragraph immediately above it:

```
The document body is wrapped in a centered 700px container table (`<table align="center" width="700">`). This is required for Google Docs paste to render with symmetric left/right margins — the body's CSS centering is stripped on paste. All content tables go inside this wrapper. See the Document Shell pattern in `references/html-styling-guide.md`.

```

(Keep a blank line between the new paragraph and the existing "All styling must use inline CSS..." line.)

- [ ] **Step 3: Verify the front-matter still parses**

Run:
```bash
head -15 skills/xsiam-docs-playbooks/SKILL.md
```
Expected: `version: 0.3.2` visible, three `---` fences intact, no broken YAML.

- [ ] **Step 4: Commit**

```bash
git add skills/xsiam-docs-playbooks/SKILL.md
git commit -m "feat(playbook-docs): bump skill to 0.3.2 with Google Docs layout fixes

Notes the new centered wrapper-table requirement in workflow step 4."
```

---

### Task 6: Manual validation in browser and Google Docs

This task is manual — there is no automated test for HTML paste fidelity into Google Docs.

- [ ] **Step 1: Pick a representative playbook YAML**

Choose a playbook that exercises sequential, parallel, and condition branches. If none is at hand, ask the user to point at one (or use any recent playbook the user has documented before).

- [ ] **Step 2: Generate documentation using the updated skill**

Invoke the `xsiam-docs-playbooks` skill on the chosen playbook YAML. Save the resulting `*_Documentation.html` to a temporary path.

- [ ] **Step 3: Open the HTML in a browser**

Verify visually:
- Task nodes (start, command, condition, sub-playbook) appear centered inside the flow-diagram wrapper.
- Document content sits centered on the page (700px column).
- All other sections (Overview Card, Task Inventory, Decision Logic, Inputs/Outputs) render unchanged from the previous version.

- [ ] **Step 4: Paste into Google Docs**

Select All (Cmd+A), Copy (Cmd+C), paste into a fresh Google Doc. Verify:
- Flow-diagram task nodes (e.g., "Task 0: Start", parallel children, condition nodes) are horizontally centered inside the gray wrapper — not flush against the left edge.
- Document content has symmetric left and right margins inside the Google Doc page (no flush-right edge).
- All other sections render with no regressions.

- [ ] **Step 5: Report results**

If validation passes, the work is done. If any issue surfaces, file a follow-up note describing exactly which template/snippet still misbehaves, then iterate on `html-styling-guide.md`.

---

## Self-Review

**Spec coverage:**
- Spec Fix 1 "Centered container table" → Task 1.
- Spec Fix 2 (a) Regular/Command node → Task 2.
- Spec Fix 2 (b) Parallel Branch Split tables → Task 3.
- Spec Fix 2 (c) Condition Node + branch-arrows table → Task 4.
- Spec "SKILL.md changes" (version bump + step-4 note) → Task 5.
- Spec "Validation" → Task 6.
All spec sections are covered.

**Placeholder scan:** Every step has the exact target string and exact replacement string. No "TBD", "TODO", "appropriate error handling", or unresolved placeholders.

**Type / pattern consistency:** Every task-node replacement uses the same attribute order: `align="center" width="<value>" style="margin: 0 auto; border-collapse: collapse;"`. Width values match the original (`420` for fixed-width nodes, `100%` for full-width split/branch tables). The wrapper table in Task 1 uses `width="700"` consistently with the spec.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-04-29-playbook-docs-google-docs-layout-fixes.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
