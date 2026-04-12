# XQL Federated Search Reference

This file is loaded on-demand when a query targets external data sources
(S3, GCS, Azure Blob Storage).

## Overview

Federated Search provides unified access to distributed, non-ingested data
without pre-ingestion or centralization. Query data in place using XQL,
reducing ingestion complexity and costs.

**Not enabled by default** — contact Customer Support to enable for your tenant.

### Use Cases

- **Incident Investigation** — Query historical events not ingested into XSIAM
- **Compliance Audits** — Access historical data without ingestion overhead
- **Long-term Data Storage** — Retain data for years without full ingestion
- **Data Linking** — Join external datasets with ingested datasets for unified analysis

## Supported Configurations

| Property | Options |
|----------|---------|
| Storage | Amazon S3, Google Cloud Storage (GCS), Azure Blob Storage |
| Formats | CSV, Parquet, JSONL (Parquet recommended) |
| Partitioning | Hive format: `ds=yyyy-mm-dd` (e.g., `ds=2025-10-07`) |

### Supported Regions

- **AWS S3:** us-east-1, us-west-2, ap-northeast-2, ap-southeast-2, eu-west-1, eu-central-1
- **GCS:** Multiple regions (Africa, Asia, Australia, Europe, Middle East, North America, South America)
- **Azure Blob Storage:** eastus2 only

## Limitations

- Bucket must be in the same region as your XSIAM tenant (single-region) or same multi-region (multi-region tenants)
- **Unavailable in Federated Search:** correlations, widgets, dashboards, playbooks, API queries
- **Unavailable XQL stages:** `search`, `target`, `view` — only `dataset`, `filter`, `fields`, `join`, `limit`, `sort`, `comp`, `alter` stages work
- Query costs are calculated by timeframe, complexity, and cross-cloud egress
- Currently limited to ad-hoc queries via Query Builder only — no scheduled queries or API execution

## Dataset Naming

- Name must start with `external_` (e.g., `external_cloudtrail_logs`, `external_gcs_events`)
- Schema is auto-detected from the first 500 records on creation
- The `ds` field (Hive partition key) cannot be deleted
- After initial schema save, fields added during setup cannot be deleted — only new fields can be added, or existing fields edited

## Querying External Datasets

External datasets are queried with standard XQL syntax. Access them via
**Investigation & Response → Investigation → Query Builder** (XQL mode).

### Basic Query

```xql
dataset = external_cloudtrail_logs
| filter ds = "2025-10-07"
| fields event_time, user_identity_arn, event_name, source_ip_address
| limit 100
```

### Filtering by Partition (Date Range)

Always filter by `ds` to control query scope and cost — querying without a
partition filter scans all partitions.

```xql
dataset = external_cloudtrail_logs
| filter ds >= "2025-09-01" and ds <= "2025-09-30"
| filter event_name = "AssumeRole"
| fields event_time, user_identity_arn, recipient_account_id
```

### Joining External Data with Ingested Data

Join an external dataset with an ingested XSIAM dataset to correlate
historical data with live detections.

```xql
dataset = external_cloudtrail_logs
| filter ds = "2025-10-07"
| join (
    dataset = xdr_data
    | filter event_type = ENUM.STORY_EVENT_TYPE_NETWORK
  ) as ingested
  on external_cloudtrail_logs.source_ip_address = ingested.action_remote_ip
| fields
    external_cloudtrail_logs.event_time,
    external_cloudtrail_logs.user_identity_arn,
    external_cloudtrail_logs.event_name,
    ingested.actor_process_image_name,
    ingested.actor_process_command_line
| limit 200
```

### Counting Events per Day Across Partitions

```xql
dataset = external_cloudtrail_logs
| filter ds >= "2025-10-01" and ds <= "2025-10-07"
| comp count() as event_count by ds
| sort asc ds
```

### Filtering for Specific IAM Actions

```xql
dataset = external_cloudtrail_logs
| filter ds = "2025-10-07"
  and event_name in ("CreateUser", "DeleteUser", "AttachUserPolicy", "DetachUserPolicy")
| fields event_time, user_identity_arn, event_name, request_parameters
| sort desc event_time
```

## Key Query Rules

- Filter by `ds` (partition) early to limit scan scope and cost
- External dataset names always start with `external_` — autocomplete shows them in Query Builder
- Cannot use `search`, `target`, or `view` stages with external datasets
- JOIN syntax is standard XQL; alias the joined dataset to disambiguate field names
- Parquet files yield best performance; CSV and JSONL are supported but slower at scale

## Setup Summary

These steps are condensed for reference. Full click-by-click setup is done via
**Settings → Configurations → Data Management → Dataset Management → External Datasets**.

### Amazon S3

1. Create IAM policy granting `s3:ListBucket` and `s3:GetObject` on the target bucket
2. Create IAM role using Web Identity, Google identity provider, Audience=00000
3. In Federated Search wizard: enter Role ARN and region, then generate an identity string
4. Update the AWS trust relationship: replace `accounts.google.com:aud` with the generated identity; set max session duration to 12 hours
5. Configure dataset: name (`external_*`), S3 URI (`s3://bucket-name/table-name/`), format, then test → validate schema → create

### Google Cloud Storage

1. In Federated Search wizard: generate an identity string and specify region
2. In GCS project IAM, add the generated identity as a principal with the **Storage Object Viewer** role
3. Configure dataset: name (`external_*`), GS URI (`gs://bucket-name/table-name/`), format, then test → validate schema → create

### Azure Blob Storage

1. Create an app registration in Azure (Accounts in this organizational directory only)
2. In Federated Search wizard: enter Directory ID, Application ID, and Object ID; select region (eastus2)
3. Generate identity → in Azure, add a Federated credential (Issuer: `https://accounts.google.com`, Type: Explicit subject identifier)
4. Assign the **Storage Blob Data Reader** role to the app on the target storage container
5. Configure dataset: name (`external_*`), Container URL, format, then test → validate schema → create

## Managing External Datasets

Navigate to **Settings → Configurations → Data Management → Dataset Management → External Datasets**.

| Action | How |
|--------|-----|
| Add | Click Add External Dataset |
| Check connection | Hover row → Check connection (also auto-checked weekly) |
| View schema | Hover row → eye icon |
| Edit | Opens setup wizard (change description and schema) |
| Delete | Hover row → Delete dataset (removes connection only; external storage unaffected) |
| Run query | Hover row → triangle icon (opens Query Builder with dataset pre-selected) |

Create, delete, and update actions are recorded in Management Audit Logs under the **External Datasets** type.

## Key Takeaways

- Dataset names must start with `external_` — this is required, not a convention
- Always filter by `ds` to scope partitions and control cost
- Parquet is the recommended format for best query performance
- Schema auto-detected from first 500 records; `ds` partition field cannot be deleted
- XQL stages `search`, `target`, and `view` are not available for external datasets
- JOIN with ingested datasets works with standard XQL join syntax
- Queries are ad-hoc only (Query Builder); no API execution or scheduling
- Bucket/container must be co-located in the same region as your XSIAM tenant
