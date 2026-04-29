# HTML Styling Guide for Playbook Documentation

Google Docs-compatible HTML patterns. Every style is inline because Google Docs ignores `<style>` blocks and CSS classes.

## Brand Colors — Palo Alto Networks

All documentation uses the Palo Alto Networks / Cortex XSIAM brand palette:

| Role | Color | Hex | Usage |
|------|-------|-----|-------|
| Primary Navy | Dark navy | `#003366` | Title banners, section headers, table headers |
| Accent Orange | PAN Orange | `#FA582D` | Accent bar, warning callouts, condition highlights |
| Secondary Navy | Medium navy | `#004080` | Sub-section headers, secondary text |
| Light Background | Pale blue-gray | `#f7f9fc` | Overview card backgrounds, alternate rows |
| Body Text | Dark gray | `#1a2533` | Primary body text |
| Label Text | Muted blue-gray | `#7a8898` | Metadata labels, secondary descriptions |
| Borders | Light gray | `#e8ecf0` | Table borders, dividers |
| Page Background | Off-white | `#f0f2f5` | Document wrapper (browser only) |

Task type node colors remain functional (not brand-colored) so they communicate meaning at a glance.

---

## Document Shell

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
<body style="font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif; max-width: 750px; margin: 0 auto; color: #1a2533; line-height: 1.6;">

  <!-- content here -->

</body>
</html>
```

The body has `max-width: 750px; margin: 0 auto` for browser rendering only — Google Docs strips body styles on paste and uses the document's own page margins to space content (which are symmetric by default).

Do **not** wrap content in an outer container table with a fixed pixel width. Google Docs' table-layout algorithm interacts badly with deeply nested tables under a fixed-width ancestor: nested-table column widths collapse to near-zero, especially in argument tables and parallel-branch flow-diagram nodes. Let inner content tables (`width: 100%`) fill Google Docs' own page width, and use the 3-column spacer wrapper pattern (see Task Node section below) to center individual flow-diagram task nodes.

Avoid `padding` on `<body>`. Google Docs has been observed to honor `padding-left` while ignoring `padding-right`, producing asymmetric margins after paste.

---

## Playbook Title

The playbook name is a standard `<h4>` — the top-level heading in the document. No banner, no branding label, no background block.

```html
<h4 style="color: #003366; border-bottom: 2px solid #FA582D; padding-bottom: 8px; margin-top: 0; margin-bottom: 16px; font-size: 18px;">
  [Playbook Name]
</h4>
```

---

## Overview Card

A metadata summary table. Uses the light blue-gray background for label cells.

```html
<table style="width: 100%; border-collapse: collapse; margin-bottom: 24px; border: 1px solid #e8ecf0;">
  <tr>
    <td style="padding: 10px 16px; font-weight: 600; color: #7a8898; text-transform: uppercase; letter-spacing: 0.6px; font-size: 11px; width: 160px; border-bottom: 1px solid #e8ecf0; background-color: #f7f9fc;">Category</td>
    <td style="padding: 10px 16px; border-bottom: 1px solid #e8ecf0; color: #1a2533; font-weight: 500;">Incident Response</td>
  </tr>
  <tr>
    <td style="padding: 10px 16px; font-weight: 600; color: #7a8898; text-transform: uppercase; letter-spacing: 0.6px; font-size: 11px; width: 160px; border-bottom: 1px solid #e8ecf0; background-color: #f7f9fc;">Trigger</td>
    <td style="padding: 10px 16px; border-bottom: 1px solid #e8ecf0; color: #1a2533;">Alert type: Malware Detection</td>
  </tr>
  <tr>
    <td style="padding: 10px 16px; font-weight: 600; color: #7a8898; text-transform: uppercase; letter-spacing: 0.6px; font-size: 11px; width: 160px; border-bottom: 1px solid #e8ecf0; background-color: #f7f9fc;">Integrations</td>
    <td style="padding: 10px 16px; border-bottom: 1px solid #e8ecf0;">
      <span style="display: inline-block; background-color: #e6f0f7; color: #003366; padding: 2px 8px; border-radius: 3px; font-size: 12px; margin-right: 4px; font-weight: 500;">CrowdStrike</span>
      <span style="display: inline-block; background-color: #e6f0f7; color: #003366; padding: 2px 8px; border-radius: 3px; font-size: 12px; margin-right: 4px; font-weight: 500;">VirusTotal</span>
    </td>
  </tr>
  <tr>
    <td style="padding: 10px 16px; font-weight: 600; color: #7a8898; text-transform: uppercase; letter-spacing: 0.6px; font-size: 11px; width: 160px; background-color: #f7f9fc;">Sub-playbooks</td>
    <td style="padding: 10px 16px; color: #1a2533;">IP Enrichment - Generic v2, Isolate Endpoint - Generic</td>
  </tr>
</table>
```

---

## Section Headers

The playbook name uses `<h4>` as the top-level heading. All section headers use `<h5>`, and sub-sections use `<h6>`, to maintain correct nesting beneath the title.

```html
<h5 style="color: #003366; border-bottom: 2px solid #FA582D; padding-bottom: 8px; margin-top: 32px; margin-bottom: 16px; font-size: 16px;">
  Section Title
</h5>
```

For sub-sections:
```html
<h6 style="color: #004080; margin-top: 24px; margin-bottom: 12px; font-size: 14px;">
  Sub-section Title
</h6>
```

---

## Flow Diagram

The flow diagram uses a centered table layout. Each row is either a **task node** or an **arrow connector**.

### Task Node (Regular/Command)

Task nodes are centered using a **3-column spacer wrapper**: empty cells on the left and right (each 15% wide) flank the task table in the middle (70% wide). This is the only reliable way to center a task table inside a Google Docs page — `align="center"` and `text-align: center` on a `<table>` do not center it (Google Docs treats `align="center"` as text alignment for cell contents, and tables don't honor `text-align`).

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

### Task Node Color Variants

These colors are functional — they communicate task type regardless of brand. Keep them as-is.

| Task Type | Left border color | Background | Text color |
|-----------|------------------|------------|------------|
| Start | `#388E3C` | `#E8F5E9` | `#2E7D32` |
| Regular/Command | `#1976D2` | `#E3F2FD` | `#1565C0` |
| Condition | `#FA582D` | `#FFF3E0` | `#c0392b` |
| Sub-playbook | `#7B1FA2` | `#F3E5F5` | `#6A1B9A` |
| Section/Title | `#7a8898` | `#f7f9fc` | `#1a2533` |
| Manual/Collection | `#00897B` | `#E0F2F1` | `#00695C` |

Note: Condition nodes use PAN Orange (`#FA582D`) for the left border — this ties the functional color to the brand accent.

### Arrow Connector (Sequential)

```html
<tr>
  <td style="text-align: center; padding: 2px 0; color: #9aa5b4; font-size: 18px; line-height: 1;">
    &#x25BC;
  </td>
</tr>
```

### Arrow with Branch Label

```html
<tr>
  <td style="text-align: center; padding: 2px 0;">
    <span style="color: #9aa5b4; font-size: 18px;">&#x25BC;</span>
    <span style="color: #FA582D; font-size: 11px; font-weight: bold; margin-left: 8px;">YES</span>
  </td>
</tr>
```

### Parallel Branch Split

When tasks execute concurrently, use a multi-column layout. Parallel branches naturally fill the wrapper width (no spacer columns) — they share 50/50.

```html
<tr>
  <td style="padding: 4px 0;">
    <table style="width: 100%; border-collapse: collapse;">
      <tr>
        <!-- Branch 1 -->
        <td style="width: 50%; vertical-align: top; padding: 0 8px;">
          <table style="width: 100%; border-collapse: collapse;">
            <tr>
              <td style="width: 6px; background-color: #1976D2; border-radius: 4px 0 0 4px;"></td>
              <td style="padding: 8px 12px; background-color: #E3F2FD; border: 1px solid #BBDEFB; border-left: none; border-radius: 0 4px 4px 0;">
                <span style="font-weight: bold; color: #1565C0; font-size: 13px;">IP Enrichment</span><br>
                <span style="color: #7a8898; font-size: 11px;">Sub-playbook</span>
              </td>
            </tr>
          </table>
        </td>
        <!-- Branch 2 -->
        <td style="width: 50%; vertical-align: top; padding: 0 8px;">
          <table style="width: 100%; border-collapse: collapse;">
            <tr>
              <td style="width: 6px; background-color: #1976D2; border-radius: 4px 0 0 4px;"></td>
              <td style="padding: 8px 12px; background-color: #E3F2FD; border: 1px solid #BBDEFB; border-left: none; border-radius: 0 4px 4px 0;">
                <span style="font-weight: bold; color: #1565C0; font-size: 13px;">Domain Enrichment</span><br>
                <span style="color: #7a8898; font-size: 11px;">Sub-playbook</span>
              </td>
            </tr>
          </table>
        </td>
      </tr>
    </table>
  </td>
</tr>
```

### Condition Node with Branch Labels

```html
<!-- Condition node — uses the same 3-column spacer wrapper as a regular task node -->
<tr>
  <td style="padding: 4px 0;">
    <table style="width: 100%; border-collapse: collapse;">
      <tr>
        <td style="width: 15%;"></td>
        <td style="width: 70%;">
          <table style="width: 100%; border-collapse: collapse;">
            <tr>
              <td style="width: 6px; background-color: #FA582D; border-radius: 4px 0 0 4px;"></td>
              <td style="padding: 10px 16px; background-color: #FFF3E0; border: 1px solid #FFE0B2; border-left: none; border-radius: 0 4px 4px 0;">
                <span style="font-weight: bold; color: #c0392b; font-size: 14px;">Is the file malicious?</span><br>
                <span style="color: #7a8898; font-size: 12px;">Condition: Check verdict from enrichment</span>
              </td>
            </tr>
          </table>
        </td>
        <td style="width: 15%;"></td>
      </tr>
    </table>
  </td>
</tr>
<!-- Branch arrows -->
<tr>
  <td style="padding: 4px 0;">
    <table style="width: 100%; border-collapse: collapse;">
      <tr>
        <td style="width: 50%; text-align: center; padding: 4px;">
          <span style="display: inline-block; background-color: #C8E6C9; color: #2E7D32; padding: 2px 10px; border-radius: 3px; font-size: 11px; font-weight: bold;">YES</span><br>
          <span style="color: #9aa5b4; font-size: 18px;">&#x25BC;</span>
        </td>
        <td style="width: 50%; text-align: center; padding: 4px;">
          <span style="display: inline-block; background-color: #FFCDD2; color: #C62828; padding: 2px 10px; border-radius: 3px; font-size: 11px; font-weight: bold;">NO</span><br>
          <span style="color: #9aa5b4; font-size: 18px;">&#x25BC;</span>
        </td>
      </tr>
    </table>
  </td>
</tr>
```

### Complete Flow Diagram Wrapper

```html
<table style="width: 100%; border-collapse: collapse; margin-bottom: 24px;">
  <tr>
    <td style="text-align: center; padding: 16px; background-color: #f7f9fc; border: 1px solid #e8ecf0; border-radius: 4px;">
      <table style="width: 100%; border-collapse: collapse;">
        <!-- Insert task nodes and arrows here -->
      </table>
    </td>
  </tr>
</table>
```

### Phase Label (Optional)

Use between groups of tasks to indicate playbook phases:

```html
<tr>
  <td style="text-align: center; padding: 8px 0;">
    <span style="display: inline-block; background-color: #003366; color: #FFFFFF; padding: 4px 16px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; letter-spacing: 1px;">Enrichment Phase</span>
  </td>
</tr>
```

---

## Task Inventory Table

```html
<table style="width: 100%; border-collapse: collapse; margin-bottom: 24px;">
  <tr>
    <th style="background-color: #003366; color: #FFFFFF; padding: 10px 12px; text-align: left; font-size: 13px; font-weight: bold;">ID</th>
    <th style="background-color: #003366; color: #FFFFFF; padding: 10px 12px; text-align: left; font-size: 13px; font-weight: bold;">Task Name</th>
    <th style="background-color: #003366; color: #FFFFFF; padding: 10px 12px; text-align: left; font-size: 13px; font-weight: bold;">Type</th>
    <th style="background-color: #003366; color: #FFFFFF; padding: 10px 12px; text-align: left; font-size: 13px; font-weight: bold;">Integration</th>
    <th style="background-color: #003366; color: #FFFFFF; padding: 10px 12px; text-align: left; font-size: 13px; font-weight: bold;">Description</th>
  </tr>
  <tr>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">0</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px; font-weight: bold;">Start</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">
      <span style="display: inline-block; background-color: #E8F5E9; color: #2E7D32; padding: 2px 8px; border-radius: 3px; font-size: 11px;">start</span>
    </td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px; color: #9aa5b4;">—</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">Entry point</td>
  </tr>
  <!-- Alternate row -->
  <tr style="background-color: #f7f9fc;">
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">1</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px; font-weight: bold;">Enrich IP</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">
      <span style="display: inline-block; background-color: #E3F2FD; color: #1565C0; padding: 2px 8px; border-radius: 3px; font-size: 11px;">command</span>
    </td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">CrowdStrike</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">Look up IP reputation</td>
  </tr>
</table>
```

---

## Callout Boxes

### Info Callout

```html
<table style="width: 100%; border-collapse: collapse; margin: 16px 0;">
  <tr>
    <td style="width: 4px; background-color: #003366;"></td>
    <td style="padding: 12px 16px; background-color: #e6f0f7; border: 1px solid #ccdce8; border-left: none; font-size: 13px; color: #003366;">
      <strong>Note:</strong> This playbook requires the CrowdStrike integration to be configured with API read permissions.
    </td>
  </tr>
</table>
```

### Warning Callout

```html
<table style="width: 100%; border-collapse: collapse; margin: 16px 0;">
  <tr>
    <td style="width: 4px; background-color: #FA582D;"></td>
    <td style="padding: 12px 16px; background-color: #fef4f0; border: 1px solid #fcd5c8; border-left: none; font-size: 13px; color: #c0392b;">
      <strong>Warning:</strong> The isolation action in Task 5 is irreversible without manual intervention.
    </td>
  </tr>
</table>
```

### Tip Callout

```html
<table style="width: 100%; border-collapse: collapse; margin: 16px 0;">
  <tr>
    <td style="width: 4px; background-color: #388E3C;"></td>
    <td style="padding: 12px 16px; background-color: #E8F5E9; border: 1px solid #C8E6C9; border-left: none; font-size: 13px; color: #2E7D32;">
      <strong>Tip:</strong> Set the severity threshold input to "High" to skip enrichment on low-severity alerts.
    </td>
  </tr>
</table>
```

---

## Code Blocks

For XQL queries, command examples, or context paths:

```html
<table style="width: 100%; border-collapse: collapse; margin: 8px 0;">
  <tr>
    <td style="padding: 12px 16px; background-color: #1a2533; color: #e8ecf0; font-family: 'Courier New', Courier, monospace; font-size: 13px; border-radius: 4px; white-space: pre-wrap; line-height: 1.5;">dataset = xdr_data
| filter event_type = PROCESS
| fields agent_hostname, action_process_name</td>
  </tr>
</table>
```

---

## Type Badges

Reusable inline badges for task types, severity levels, and statuses:

```html
<!-- Task type badges -->
<span style="display: inline-block; background-color: #E3F2FD; color: #1565C0; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: bold;">command</span>
<span style="display: inline-block; background-color: #FFF3E0; color: #c0392b; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: bold;">condition</span>
<span style="display: inline-block; background-color: #F3E5F5; color: #6A1B9A; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: bold;">sub-playbook</span>
<span style="display: inline-block; background-color: #E0F2F1; color: #00695C; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: bold;">manual</span>

<!-- Severity badges -->
<span style="display: inline-block; background-color: #FFCDD2; color: #C62828; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: bold;">Critical</span>
<span style="display: inline-block; background-color: #FFE0B2; color: #E65100; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: bold;">High</span>
<span style="display: inline-block; background-color: #FFF9C4; color: #F57F17; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: bold;">Medium</span>
<span style="display: inline-block; background-color: #E8F5E9; color: #2E7D32; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: bold;">Low</span>

<!-- Required/Optional -->
<span style="display: inline-block; background-color: #FFCDD2; color: #C62828; padding: 2px 8px; border-radius: 3px; font-size: 11px;">Required</span>
<span style="display: inline-block; background-color: #e8ecf0; color: #7a8898; padding: 2px 8px; border-radius: 3px; font-size: 11px;">Optional</span>
```

---

## Inputs/Outputs Table

```html
<table style="width: 100%; border-collapse: collapse; margin-bottom: 16px;">
  <tr>
    <th style="background-color: #003366; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Name</th>
    <th style="background-color: #003366; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Description</th>
    <th style="background-color: #003366; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Required</th>
    <th style="background-color: #003366; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Default</th>
  </tr>
  <tr>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px; font-family: 'Courier New', monospace;">IP</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">IP address to investigate</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">
      <span style="display: inline-block; background-color: #FFCDD2; color: #C62828; padding: 2px 8px; border-radius: 3px; font-size: 11px;">Required</span>
    </td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px; color: #9aa5b4;">—</td>
  </tr>
</table>
```

---

## Decision Logic Table

For documenting condition task branch criteria:

```html
<table style="width: 100%; border-collapse: collapse; margin-bottom: 16px;">
  <tr>
    <th style="background-color: #FA582D; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Condition</th>
    <th style="background-color: #FA582D; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Operator</th>
    <th style="background-color: #FA582D; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Value</th>
    <th style="background-color: #FA582D; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Branch</th>
    <th style="background-color: #FA582D; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Next Task</th>
  </tr>
  <tr>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px; font-family: 'Courier New', monospace;">DBotScore.Score</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">is greater than</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">2</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">
      <span style="display: inline-block; background-color: #C8E6C9; color: #2E7D32; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: bold;">Malicious</span>
    </td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">Task 4: Isolate Endpoint</td>
  </tr>
  <tr style="background-color: #f7f9fc;">
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px; color: #9aa5b4;" colspan="3">—</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">
      <span style="display: inline-block; background-color: #e8ecf0; color: #7a8898; padding: 2px 8px; border-radius: 3px; font-size: 11px; font-weight: bold;">#default#</span>
    </td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">Task 6: Close Alert</td>
  </tr>
</table>
```

---

## Integration Dependencies Card

```html
<table style="width: 100%; border-collapse: collapse; margin-bottom: 16px;">
  <tr>
    <th style="background-color: #003366; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Integration</th>
    <th style="background-color: #003366; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Commands Used</th>
    <th style="background-color: #003366; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Required</th>
  </tr>
  <tr>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px; font-weight: bold;">CrowdStrike Falcon</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">
      <code style="background-color: #f7f9fc; padding: 2px 6px; border-radius: 3px; font-size: 12px; border: 1px solid #e8ecf0;">cs-falcon-search-device</code>,
      <code style="background-color: #f7f9fc; padding: 2px 6px; border-radius: 3px; font-size: 12px; border: 1px solid #e8ecf0;">cs-falcon-contain-host</code>
    </td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">
      <span style="display: inline-block; background-color: #FFCDD2; color: #C62828; padding: 2px 8px; border-radius: 3px; font-size: 11px;">Required</span>
    </td>
  </tr>
</table>
```

---

## Error Handling Section

```html
<table style="width: 100%; border-collapse: collapse; margin-bottom: 16px;">
  <tr>
    <th style="background-color: #c0392b; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Task</th>
    <th style="background-color: #c0392b; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">On Error</th>
    <th style="background-color: #c0392b; color: #FFFFFF; padding: 8px 12px; text-align: left; font-size: 13px;">Error Path</th>
  </tr>
  <tr>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">Task 3: Isolate Endpoint</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">Continue on error</td>
    <td style="padding: 8px 12px; border-bottom: 1px solid #e8ecf0; font-size: 13px;">&#8594; Task 7: Send Failure Notification</td>
  </tr>
</table>
```

---

## Document Footer

Add a footer at the bottom of every document:

Do NOT include any "Generated by" footer or branding line in the output.

---

## Google Docs Compatibility Notes

Things that work:
- Inline styles on all elements
- Tables with borders, padding, background colors
- Font sizes, weights, colors
- `font-family` (stick to Helvetica Neue, Helvetica, Arial, sans-serif for body; Courier New for code)
- `border-radius` (renders in browser, ignored in Docs but doesn't break anything)
- `colspan` and `rowspan`

Things that don't work (avoid):
- `<style>` blocks (ignored entirely)
- CSS classes (ignored)
- Flexbox or Grid layout
- `position: absolute/relative`
- `box-shadow`
- `opacity`
- External fonts or images
- SVG elements
- `display: flex` or `display: grid`

Workarounds:
- Use tables for all layout (columns, centering, side-by-side elements)
- Use `background-color` on table cells instead of `box-shadow` for visual depth
- Use Unicode characters for icons: &#x25BC; (arrow), &#x2714; (check), &#x26A0; (warning), &#x2716; (cross)
