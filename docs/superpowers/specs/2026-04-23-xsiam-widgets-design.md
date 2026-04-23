# xsiam-widgets Skill Design

**Date:** 2026-04-23
**Status:** Draft
**Scope:** XQL widget queries with `| view graph` visualization stage

## Overview

A new skill for the xsiam-buddy plugin that generates dashboard widget queries for Cortex XSIAM. The skill produces XQL queries ending with a `| view graph` stage, ready to paste into the XSIAM Widget Builder.

This is scoped to XQL-based widgets only. Script-based widgets (Python widget classes) and Widget API JSON are explicitly out of scope for v1.

## Directory Structure

```
skills/
  xsiam-widgets/
    SKILL.md
    references/
      widget-view-graph-spec.md
```

The skill reuses shared XQL references from `xsiam-shared/references/` via tiered loading (same pattern as xsiam-xql and xsiam-correlations). The local reference file covers the visualization layer only.

## Workflow (5 Steps)

### Step 1: Understand Visualization Goal

Determine what the user wants to visualize:
- Data source (dataset)
- Metric(s) to display
- Grouping/breakdown dimension
- Chart type (if not specified, recommend based on data shape)

### Step 2: Select Dataset & Build Data Query

Load shared XQL references (tiered). Build the data pipeline:
- `dataset` selection
- `filter` for scoping
- `comp` / aggregation to produce numeric y-axis values
- `alter` if field transformation is needed
- `sort` and `limit` for result shaping

The query must produce data suitable for visualization: aggregated, with clear string/categorical x-axis fields and numeric y-axis fields.

### Step 3: Apply View Graph Stage

Load `widget-view-graph-spec.md`. Append `| view graph` as the final stage with:
- `type` (required) -- one of: area, bubble, column, funnel, gauge, line, map, pie, scatter, single, wordcloud
- `subtype` (optional) -- layout variant (e.g., `grouped`, `standard`)
- `header` (optional) -- quoted display title
- `xaxis` (required) -- string/categorical field for x-axis
- `yaxis` (required) -- numeric field(s) for y-axis, comma-separated for multiple
- `series` (optional) -- field for grouping into data series

Validate axis data types: x-axis must be string/categorical (including `_time`), y-axis must be numeric.

### Step 4: Validate Query

Quality checklist:
1. Correct dataset for the data source
2. Valid aggregation producing numeric y-axis values
3. X-axis field is string/categorical
4. Y-axis field is numeric (output of comp/aggregation)
5. `| view graph` is the final stage
6. Header is descriptive and meaningful
7. Chart type matches the data shape
8. No unnecessary or redundant stages
9. `series` field used when multi-series grouping is needed
10. `limit` applied when dataset is large

### Step 5: Format Output

Deliver as a fenced XQL code block with structured header comments:

```xql
/*
 * Widget: <descriptive title>
 * Chart Type: <type> (<subtype if applicable>)
 * Dataset: <dataset name>
 * Description: <what this widget shows>
 */
dataset = <dataset>
| <data pipeline stages>
| view graph type = <type> subtype = <subtype> header = "<title>" xaxis = <field> yaxis = <field> series = <field>
```

## Reference File: `widget-view-graph-spec.md`

Contents:

### Graph Type Reference

All 11 supported types with subtypes:
- **area** -- time series with filled area
- **bubble** -- three-dimensional data points (x, y, size)
- **column** -- categorical comparison; subtypes: `grouped`, `stacked`
- **funnel** -- sequential stage drop-off
- **gauge** -- single value against a range
- **line** -- time series or trend
- **map** -- geographic data
- **pie** -- part-of-whole proportion
- **scatter** -- correlation between two numeric fields
- **single** (single value) -- single KPI; subtypes: `standard`
- **wordcloud** -- frequency-based text visualization

### Parameter Reference

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `type` | Yes | keyword | Graph type (see list above) |
| `subtype` | No | keyword | Layout variant |
| `header` | No | quoted string | Display title |
| `xaxis` | Yes | field name | X-axis field (string/categorical) |
| `yaxis` | Yes | field name(s) | Y-axis field(s), comma-separated for multiple |
| `series` | No | field name | Grouping field for multi-series |

### Chart Type Selection Guide

| Data Shape | Recommended Type | Why |
|------------|-----------------|-----|
| Single numeric KPI | `single` | One number, prominent display |
| Values over time | `line` or `area` | Trend visibility |
| Category comparison | `column` | Side-by-side bar comparison |
| Part-of-whole | `pie` | Proportion at a glance |
| Top-N ranking | `column` (sorted desc) | Ordered magnitude comparison |
| Two numeric dimensions | `scatter` | Correlation analysis |
| Three numeric dimensions | `bubble` | X, Y, and size encoding |
| Sequential stages | `funnel` | Drop-off visibility |
| Geographic distribution | `map` | Spatial patterns |
| Text frequency | `wordcloud` | Term prominence |
| Single value vs range | `gauge` | Progress/threshold display |

### Aggregation Patterns Per Chart Type

| Chart Type | Typical Aggregation | Example |
|------------|-------------------|---------|
| column/line/area | `comp count() / sum() / avg() by <category>` | `comp count(_id) as event_count by action_type` |
| pie | `comp count() by <category>` | `comp count(_id) as total by _vendor` |
| single | `comp count_distinct() / sum()` (no group-by) | `comp count_distinct(agent_id) as unique_agents` |
| scatter/bubble | `comp avg() as y, sum() as x by <entity>` | `comp avg(duration) as avg_dur, sum(bytes) as total_bytes by host` |

### Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing aggregation before `| view graph` | Add `comp` stage to produce numeric y-axis |
| Numeric field on x-axis | Use `alter` to cast or use a categorical field |
| `| view graph` not as final stage | Move it to the end of the query |
| Missing `header` | Add a descriptive quoted header |
| Too many data points | Add `limit` before `| view graph` |
| Using `series` without multi-value data | Remove `series` or add group-by dimension |

## Shared Reference Loading (Tiered)

Same pattern as xsiam-xql:
- **Always load:** `xql-core-reference.md`, `xql-datasets-core.md`
- **On-demand:** `xql-advanced-functions.md`, `xql-datasets-extended.md`, `xql-federated-search.md`

## Skill Triggering

The skill description should trigger on phrases like:
- "create a widget", "build a widget", "dashboard widget"
- "visualize", "chart", "graph"
- "view graph", "XSIAM dashboard"
- "show me a pie chart of", "line chart for"

## Out of Scope (v1)

- Script-based widgets (Python widget classes)
- Widget API JSON format (`/public_api/v1/widgets/insert`)
- Dashboard layout and composition
- Drilldown configuration
- Parameter/filter configuration
- Report templates

## Plugin Registration

Add to `marketplace.json` skills array:
```json
"./skills/xsiam-widgets"
```

Update CLAUDE.md table to include the new skill.
