---
name: xsiam-widgets
description: >
  This skill should be used when the user asks to "create a widget",
  "build a widget", "dashboard widget", "XSIAM widget", "visualize",
  "chart", "graph", "view graph", "pie chart", "line chart", "column chart",
  "show me a chart of", "build a dashboard visualization", or needs to
  generate XQL widget queries with `| view graph` visualization for
  Cortex XSIAM dashboards.
version: 1.0.0
---

# Widget Query Generation

## Scope

Generate XQL widget queries for Cortex XSIAM dashboards. Each query ends with a
`| view graph` visualization stage, ready to paste into the XSIAM Widget Builder.

**This skill handles:**
- XQL queries with `| view graph` for any of the 11 chart types
- Chart type selection based on data shape
- Aggregation patterns for visualization-ready output
- Multi-series and time-bucketed widget queries

**This skill does NOT handle:**
- Standalone XQL queries without visualization → use `xsiam-xql`
- Correlation rules with JSON wrappers → use `xsiam-correlations`
- Script-based widgets (Python widget classes) → out of scope for v1
- Widget API JSON format → out of scope for v1
- Dashboard layout, drilldowns, or parameter configuration → out of scope for v1

---

## Before Starting

### Always Read

Load these reference files before generating any widget query:

1. `../xsiam-shared/references/xql-core-reference.md` — XQL syntax, stage order, operators
2. `../xsiam-shared/references/xql-datasets-core.md` — Dataset names and field schemas
3. `references/widget-view-graph-spec.md` — Graph types, parameters, chart selection guide, aggregation patterns

### Read On-Demand (when the query needs these)

- `../xsiam-shared/references/xql-advanced-functions.md` — Complex functions (arraymap, arrayfilter, window functions)
- `../xsiam-shared/references/xql-datasets-extended.md` — Vendor-specific raw datasets
- `../xsiam-shared/references/xql-federated-search.md` — Cross-tenant federated queries

---

## Workflow

### Step 1: Understand Visualization Goal

Gather from the user:
- **Data source** — Which dataset contains the data to visualize?
- **Metric(s)** — What numeric value(s) to display (counts, sums, averages)?
- **Grouping** — How to break down the metric (by category, time, entity)?
- **Chart type** — What visualization type? If not specified, recommend one using the
  Chart Type Selection Guide in `widget-view-graph-spec.md`.

If the user provides only a vague request (e.g., "show me network traffic"), ask
clarifying questions to determine the metric, grouping, and time range.

### Step 2: Select Dataset & Build Data Query

Use the dataset selection guide in `xql-datasets-core.md` to pick the right data source.

Build the data pipeline following standard XQL stage order:

1. `config timeframe` (if non-default time window needed)
2. `dataset` / `preset` / `datamodel` — data source
3. `filter` — narrow to relevant events (filter early)
4. `alter` — transform fields if needed (e.g., extract, cast, compute)
5. `comp` — aggregate to produce numeric y-axis values with categorical x-axis grouping
6. `sort` — order results (e.g., `sort desc event_count`)
7. `limit` — cap results to a reasonable number for visualization

**Key requirement:** The query must produce data suitable for the chosen chart type:
- X-axis field must contain string/categorical values (including `_time`)
- Y-axis field must contain numeric values (output of `comp` aggregation)
- If using `series`, the grouping field must be included in the `comp by` clause

**Time bucketing for time-series charts (line, area):**
Use `bin _time span = 1h` (or appropriate interval) before `comp` to create time buckets,
then `comp count() by _time` or `comp sum(field) by _time` for the aggregation.

### Step 3: Apply View Graph Stage

Append `| view graph` as the final stage using the parameter reference from
`widget-view-graph-spec.md`:

```
| view graph type = <type> [subtype = <subtype>] header = "<title>" xaxis = <field> yaxis = <field> [series = <field>]
```

Rules:
- `type` is required — must be one of: area, bubble, column, funnel, gauge, line, map,
  pie, scatter, single, wordcloud
- `header` should be descriptive and meaningful for dashboard context
- `xaxis` must reference a field present in the query output
- `yaxis` must reference a numeric field present in the query output
- `series` is optional — use when the data has a multi-series grouping dimension
- `subtype` is optional — use `grouped` or `stacked` for column charts, `standard` for single value

### Step 4: Validate Query

Before delivering, verify ALL of the following:

1. Dataset is valid and appropriate for the data being visualized
2. `comp` stage produces numeric y-axis values with categorical grouping
3. X-axis field is string/categorical (not raw numeric)
4. Y-axis field is numeric (output of aggregation)
5. `| view graph` is the final stage — nothing after it
6. `header` is descriptive and meaningful
7. Chart type matches the data shape (see Chart Type Selection Guide)
8. No unnecessary or redundant stages
9. `series` field is used only when multi-series grouping is present in `comp by`
10. `limit` is applied when the dataset could produce too many categories (>50)
11. Field names in `xaxis`, `yaxis`, `series` match the aliased names from `comp`
12. Time-series charts use `bin _time span` for proper time bucketing

### Step 5: Format Output

Return the widget query in a fenced XQL code block with structured header comments:

```xql
/*
 * Widget: <descriptive title>
 * Chart Type: <type> (<subtype if applicable>)
 * Dataset: <dataset name>
 * Description: <what this widget shows>
 */
<query>
| view graph type = <type> subtype = <subtype> header = "<title>" xaxis = <field> yaxis = <field> series = <field>
```

Do not include prose description, expected output, or customization notes outside the
code block. The header comments carry that context.

---

## Quality Checklist

Before delivering the widget query, verify:

1. Header comments are complete (Widget, Chart Type, Dataset, Description)
2. Dataset is valid and appropriate for the stated visualization goal
3. Field names match the dataset schema (check field lists in dataset reference)
4. Correct operators for field types (ENUM unquoted, strings quoted, regex with `~=`)
5. `comp` aggregation produces numeric y-axis values
6. `comp by` clause includes both x-axis and series fields
7. Time range specified (via `config timeframe`, `filter _time`, or query context)
8. No placeholder text (no TODO, TBD, `<placeholder>`)
9. Stage order follows recommendation (filter early, comp for aggregation, view graph last)
10. `| view graph` is the absolute final stage
11. Chart type is appropriate for the data shape
12. `header` is descriptive and dashboard-ready
13. `limit` applied when categories could exceed 50

## Common Mistakes

| Mistake | Correct |
|---------|---------|
| Missing `comp` before `| view graph` | Add aggregation to produce numeric y-axis |
| `| view graph` followed by another stage | `| view graph` must be the last stage |
| `yaxis` references a non-numeric field | Use `comp` to create a numeric aggregation |
| `xaxis` references a field not in query output | Ensure `comp by` includes the x-axis field |
| Pie chart with 20+ slices | Use `limit 7` or switch to `column` |
| Time-series without `bin _time span` | Add time bucketing for proper x-axis grouping |
| `series` field not in `comp by` clause | Add the series field to `comp ... by x, series` |
| `"ENUM.VALUE"` (quoted) | `ENUM.VALUE` (unquoted) |
