# Design: XQL Skill Robustness Update

**Date:** 2026-03-01
**Status:** Approved

## Problem

Six new production XQL examples were added to `joseph3325/xsiam-dev/examples/xql`. Comparing these against the current skill revealed gaps in all three reference files:

- `xql-examples.md` — missing 6 examples
- `xql-syntax-reference.md` — missing `array_any()`, `fields *` with raw aliasing, `incidr() = false`, chained joins, and incomplete correlation rule severity guidance
- `xql-datasets.md` — missing 7 datasets and the `ad_users` preset

## Approach

Update all three reference files (Option B). Explicit documentation reduces hallucination risk on dataset names and ensures undocumented functions are available even when the relevant example is not in active context.

---

## Section 1: `xql-examples.md` — 6 New Examples

Each example cleaned to ISO date format and standard header style.

| Example | Dataset | Key Patterns Demonstrated |
|---|---|---|
| Abnormal Account Takeover Sign In | `abnormal_security_generic_alert_raw` | `if()` for conditional severity elevation |
| Canary Alert Notification | `thinkst_canary_generic_alert_raw` | Multi-join with subquery preprocessing, `preset = ad_users`, `_alert_data ->`, `dedup _id` |
| Darktrace Critical Alert | `datamodel dataset = darktrace_darktrace_raw` | `fields *` with raw aliasing, `array_length()`, `coalesce()`, dynamic `xdm.alert.severity` via `if()` |
| Google Workspace User Suspended | `google_workspace_alerts_raw` | Simple `->` extraction from nested JSON blobs |
| Microsoft Azure PowerShell Login | `msft_azure_ad_raw` | `fields *`, inline `not in` dataset lookup, `incidr() = false` |
| Microsoft Windows Domain Admin Auth | `microsoft_windows_raw` + `pan_dss_raw` | `array_any()`, chained joins, `event_data ->`, dynamic severity from endpoint lookup |

---

## Section 2: `xql-syntax-reference.md` — Targeted Additions

| Item | Location | Change |
|---|---|---|
| `array_any(arr, condition)` | Advanced Array Functions table | New row with example |
| `fields *` with raw aliasing | `fields` stage section | Document `\| fields *, dataset_name.field as alias` pattern |
| `incidr() = false` | Network Operators section | Add negative pattern example |
| Chained joins | `join` stage section | Add note + example for two sequential `join` stages |
| Dynamic severity in correlation rules | Correlation Rules section | Update: severity in YAML by default; `alter xdm.alert.severity = if(...)` allowed when user explicitly requests dynamic score-based severity |

---

## Section 3: `xql-datasets.md` — 7 New Datasets + 1 Preset

| Dataset | Category | Notes |
|---|---|---|
| `abnormal_security_generic_alert_raw` | Third-party alerts | Abnormal Security |
| `thinkst_canary_generic_alert_raw` | Third-party alerts | Thinkst Canary |
| `darktrace_darktrace_raw` | Third-party alerts | Darktrace |
| `google_workspace_alerts_raw` | Identity/SaaS | Google Workspace |
| `msft_azure_ad_raw` | Identity/SaaS | Correct name — not `microsoft_azure_ad_raw` |
| `pan_dss_raw` | Identity/Directory | Palo Alto Cloud Identity Engine (CIE) |
| `endpoints` | Endpoint inventory | XSIAM built-in endpoint registry |

Also add `ad_users` to the presets list (a named preset over `pan_dss_raw`).
