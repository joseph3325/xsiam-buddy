# xsiam-widgets Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a new `xsiam-widgets` skill that generates XQL widget queries with `| view graph` visualization for Cortex XSIAM dashboards.

**Architecture:** A standalone skill directory (`skills/xsiam-widgets/`) with a `SKILL.md` workflow file and a `references/widget-view-graph-spec.md` reference file. Reuses shared XQL references via tiered loading. Registered in `marketplace.json` and documented in `CLAUDE.md`.

**Tech Stack:** Markdown (YAML front-matter), no runtime code.

---

### Task 1: Create the widget view graph reference file

**Files:**
- Create: `skills/xsiam-widgets/references/widget-view-graph-spec.md`

- [ ] **Step 1: Create the references directory**

```bash
mkdir -p skills/xsiam-widgets/references
```

- [ ] **Step 2: Write `widget-view-graph-spec.md`**

Create `skills/xsiam-widgets/references/widget-view-graph-spec.md` with the following content:

```markdown
# Widget View Graph Reference

This reference documents the XQL `| view graph` stage used to configure widget visualizations in Cortex XSIAM dashboards.

## Syntax

```
| view graph type = <type> [subtype = <subtype>] [header = "<title>"] xaxis = <field> yaxis = <field>[,<field2>] [series = <field>]
```

The `| view graph` stage must always be the **final stage** in the query. It configures visual presentation only and does not affect the underlying data.

## Parameters

| Parameter | Required | Type | Description |
|-----------|----------|------|-------------|
| `type` | Yes | keyword | Graph visualization type (see Graph Types below) |
| `subtype` | No | keyword | Layout variant for the graph type |
| `header` | No | quoted string | Display title shown above the widget |
| `xaxis` | Yes | field name | Field for x-axis (must contain string/categorical values, including `_time`) |
| `yaxis` | Yes | field name(s) | Field(s) for y-axis (must contain numeric values); comma-separated for multiple |
| `series` | No | field name | Field for grouping data into multiple series |

## Graph Types

### area
Time series with filled area beneath the line.
- **Use when:** Showing volume or magnitude over time where the cumulative area matters.
- **X-axis:** `_time` or categorical field.
- **Y-axis:** Numeric aggregation (count, sum, avg).

### bubble
Three-dimensional scatter with variable point size.
- **Use when:** Comparing three numeric dimensions per entity.
- **X-axis:** Numeric or categorical field.
- **Y-axis:** Numeric field(s) — comma-separated yaxis values encode y-position and bubble size.

### column
Vertical bar chart for categorical comparison.
- **Subtypes:** `grouped` (side-by-side bars per category), `stacked` (bars stacked within category).
- **Use when:** Comparing counts or values across discrete categories.
- **X-axis:** Categorical field (e.g., `action_type`, `_vendor`, `_time`).
- **Y-axis:** Numeric aggregation.

### funnel
Sequential stage visualization showing drop-off.
- **Use when:** Showing progression through ordered stages (e.g., alert triage pipeline).
- **X-axis:** Stage label field.
- **Y-axis:** Count or value at each stage.

### gauge
Single value displayed against a range/threshold.
- **Use when:** Showing a metric relative to a target or threshold.
- **X-axis:** Label field.
- **Y-axis:** Numeric value.

### line
Line chart for time series or sequential trends.
- **Use when:** Showing trends, rates, or changes over time.
- **X-axis:** `_time` or ordered categorical field.
- **Y-axis:** Numeric aggregation.
- **Series:** Use `series` to split into multiple lines by a grouping field.

### map
Geographic distribution visualization.
- **Use when:** Showing data distributed by geographic location.
- **X-axis:** Location/country field.
- **Y-axis:** Numeric aggregation.

### pie
Proportional breakdown of a whole.
- **Use when:** Showing how categories contribute to a total. Best with 2-7 slices.
- **X-axis:** Category field.
- **Y-axis:** Numeric aggregation (count, sum).

### scatter
Two-dimensional point plot for correlation analysis.
- **Use when:** Exploring relationships between two numeric fields.
- **X-axis:** Numeric or categorical field.
- **Y-axis:** Numeric field.

### single
Single prominent numeric value (KPI).
- **Subtypes:** `standard`.
- **Use when:** Displaying one key metric (total count, unique entities, success rate).
- **X-axis:** Label field (often the entity being measured).
- **Y-axis:** Single numeric aggregation.

### wordcloud
Frequency-based text visualization.
- **Use when:** Showing relative frequency of text values (e.g., top domains, user agents).
- **X-axis:** Text field.
- **Y-axis:** Frequency count.

## Chart Type Selection Guide

| Data Shape | Recommended Type | Why |
|------------|-----------------|-----|
| Single numeric KPI | `single` | One number, prominent display |
| Values over time | `line` or `area` | Trend visibility |
| Category comparison | `column` | Side-by-side bar comparison |
| Part-of-whole (2-7 categories) | `pie` | Proportion at a glance |
| Top-N ranking | `column` (sorted desc) | Ordered magnitude comparison |
| Two numeric dimensions | `scatter` | Correlation analysis |
| Three numeric dimensions | `bubble` | X, Y, and size encoding |
| Sequential stages | `funnel` | Drop-off visibility |
| Geographic distribution | `map` | Spatial patterns |
| Text frequency | `wordcloud` | Term prominence |
| Single value vs range/threshold | `gauge` | Progress/threshold display |
| Multi-series trend over time | `line` with `series` | Compare trends across groups |
| Stacked category breakdown | `column` with `subtype = stacked` | Part-of-whole per category |

## Aggregation Patterns Per Chart Type

The data query must produce appropriate aggregated output before the `| view graph` stage. Use `comp` to create numeric y-axis values.

| Chart Type | Typical Aggregation | Example |
|------------|-------------------|---------|
| column / line / area | `comp count() / sum() / avg() by <category>` | `comp count(_id) as event_count by action_type` |
| pie | `comp count() by <category>` | `comp count(_id) as total by _vendor` |
| single | `comp count_distinct() / sum()` (no group-by) | `comp count_distinct(agent_id) as unique_agents` |
| scatter / bubble | `comp avg() as y, sum() as x by <entity>` | `comp avg(duration) as avg_dur, sum(bytes) as total_bytes by host` |
| line with series | `comp count() by <time_bucket>, <group>` | `comp count(_id) as events by _time, _vendor` |
| funnel | `comp count() by <stage_field>` | `comp count(_id) as stage_count by triage_status` |
| wordcloud | `comp count() by <text_field>` | `comp count(_id) as freq by domain_name` |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Missing aggregation before `| view graph` | Add a `comp` stage to produce numeric y-axis values |
| Numeric field on x-axis without categorical meaning | Use `alter` to cast to string, or use a categorical field |
| `| view graph` not as the final stage | Move it to the end — no stages after `| view graph` |
| Missing `header` parameter | Add `header = "Descriptive Title"` for dashboard readability |
| Too many data points (>100 categories on x-axis) | Add `limit` before `| view graph` or increase aggregation granularity |
| Using `series` without a multi-value grouping dimension | Remove `series` or add a second group-by field in `comp` |
| Pie chart with too many slices (>7) | Use `column` instead, or add `limit 7` and group remaining as "Other" |
| Single value widget with `group-by` in comp | Remove the `by` clause to produce one row |
```

- [ ] **Step 3: Verify the file was created**

```bash
ls -la skills/xsiam-widgets/references/widget-view-graph-spec.md
```

Expected: file exists with non-zero size.

- [ ] **Step 4: Commit**

```bash
git add skills/xsiam-widgets/references/widget-view-graph-spec.md
git commit -m "feat(widgets): add view graph reference spec for widget skill"
```

---

### Task 2: Create the widget SKILL.md

**Files:**
- Create: `skills/xsiam-widgets/SKILL.md`

- [ ] **Step 1: Write `SKILL.md`**

Create `skills/xsiam-widgets/SKILL.md` with the following content:

````markdown
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
````

- [ ] **Step 2: Verify the file was created**

```bash
cat -n skills/xsiam-widgets/SKILL.md | head -5
```

Expected: YAML front-matter starting with `---`.

- [ ] **Step 3: Commit**

```bash
git add skills/xsiam-widgets/SKILL.md
git commit -m "feat(widgets): add SKILL.md workflow for widget query generation"
```

---

### Task 3: Register the skill in marketplace.json

**Files:**
- Modify: `/.claude-plugin/marketplace.json:16-27` (skills array)

- [ ] **Step 1: Add the widget skill to the skills array**

In `.claude-plugin/marketplace.json`, add `"./skills/xsiam-widgets"` to the `skills` array. The updated array should be:

```json
"skills": [
  "./skills/xsiam-scripts",
  "./skills/xsiam-integrations",
  "./skills/xsiam-event-collectors",
  "./skills/xsiam-xql",
  "./skills/xsiam-correlations",
  "./skills/xsiam-splunk-to-xql",
  "./skills/xsiam-playbooks",
  "./skills/xsiam-docs-playbooks",
  "./skills/xsiam-docs-scripts",
  "./skills/xsiam-widgets",
  "./skills/xsiam-shared"
]
```

Place `"./skills/xsiam-widgets"` before `"./skills/xsiam-shared"` (shared is always last since it's not a user-facing skill).

- [ ] **Step 2: Verify the JSON is valid**

```bash
python3 -c "import json; json.load(open('.claude-plugin/marketplace.json')); print('valid')"
```

Expected: `valid`

- [ ] **Step 3: Commit**

```bash
git add .claude-plugin/marketplace.json
git commit -m "chore: register xsiam-widgets skill in marketplace.json"
```

---

### Task 4: Update CLAUDE.md with widget skill documentation

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add xsiam-widgets to the directory tree**

In the `Plugin Architecture` section's directory tree, add the widget skill entry after `xsiam-xql/`:

```
  xsiam-widgets/      # Dashboard widget queries with | view graph
    SKILL.md
    references/widget-view-graph-spec.md
```

- [ ] **Step 2: Add widget skill to the XQL Skill Family table**

In the `XQL Skill Family` section, add a row to the table:

```markdown
| xsiam-widgets | Dashboard widget visualization queries | XQL with `| view graph` in code block with header comments |
```

The full table becomes:

```markdown
| Skill | Purpose | Output |
|---|---|---|
| xsiam-xql | Standalone queries (hunting, investigation, analytics) | XQL in code block with header comments |
| xsiam-correlations | Detection rules (correlation rule engineering) | Complete `.json` with embedded XQL |
| xsiam-splunk-to-xql | SPL translation (Splunk migration) | XQL in code block with "Translated from SPL" note |
| xsiam-widgets | Dashboard widget visualization queries | XQL with `| view graph` in code block with header comments |
```

- [ ] **Step 3: Verify CLAUDE.md is coherent**

Read through the modified sections to confirm the new entries are consistent with existing style and don't introduce contradictions.

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add xsiam-widgets skill to CLAUDE.md architecture docs"
```

---

### Task 5: Reinstall plugin and verify skill loads

**Files:** None (verification only)

- [ ] **Step 1: Reinstall the plugin locally**

```bash
claude plugin install xsiam-buddy@xsiam-buddy
```

- [ ] **Step 2: Verify the widget skill appears in the skill list**

Confirm that `xsiam-widgets` appears when listing available skills. The skill description should trigger on widget-related keywords.

- [ ] **Step 3: Smoke test the skill**

Test the skill by asking it to generate a simple widget query (e.g., "create a column chart widget showing event counts by vendor from xdr_data"). Verify:
- The skill loads and reads reference files
- The output is a fenced XQL code block with header comments
- The query ends with `| view graph type = column ...`
- The header comments include Widget, Chart Type, Dataset, Description

- [ ] **Step 4: Final commit (if any adjustments needed)**

If smoke testing reveals issues, fix them and commit:

```bash
git add -A
git commit -m "fix(widgets): adjustments from smoke testing"
```
