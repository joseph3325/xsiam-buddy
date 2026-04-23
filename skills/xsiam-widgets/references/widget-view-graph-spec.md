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
