# Design: xsiam-scripts Alert Normalization Pattern

**Date:** 2026-02-27
**Scope:** `skills/xsiam-scripts/SKILL.md` only

## Problem

When a script calls `demisto.alert()`, the returned dict nests all custom fields under a `CustomFields` key. Scripts that reference those fields directly must use `issue['CustomFields']['field_name']`, which is verbose and error-prone.

## Solution

Enforce a normalization step immediately after every `demisto.alert()` call that flattens `CustomFields` into the top-level alert dict:

```python
issue = demisto.alert()
try:
    cf = issue.pop('CustomFields')
    issue.update(cf)
except: pass
```

After normalization, all fields (standard and custom) are accessible directly on `issue`.

## Changes

Two additions to `skills/xsiam-scripts/SKILL.md`:

1. **Python Code Conventions** — expand the existing `demisto.alert()` bullet to include the normalization pattern as a required code block.
2. **Validation Checklist** — add a checklist item verifying the normalization is present whenever `demisto.alert()` is called.

## Approach Selected

Approach B: Convention + checklist. The convention section drives proactive use during code generation; the checklist enforces it at validation time.
