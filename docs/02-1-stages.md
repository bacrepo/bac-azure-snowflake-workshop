---
title: "Stages - Path to Snowflake Data"
---

# Stages - Path to Snowflake Data

**Goal:** understand where files live *before* they become tables — and why
**external stages** are the standard approach in real projects.

[← Back to Home](index.md)

---

## The big picture: land first, load second

A common mistake is piping data **directly** from source systems into Snowflake.
The recommended pattern is to **land files first** in cloud object storage, then
have Snowflake read from there via a **stage**.

```
  Source Systems
       │
       ▼
 ┌───────────────────────────────────────────────┐
 │  Cloud Object Storage  (the "landing zone")   │
 │                                                │
 │  Azure → ADLS Gen2 / Blob Storage             │
 │  AWS   → S3                                   │
 │  GCP   → GCS                                  │
 └───────────────┬───────────────────────────────┘
                 │  External Stage
                 ▼
 ┌───────────────────────────────────────────────┐
 │            Snowflake                           │
 │                                                │
 │  COPY INTO  /  Snowpipe  /  Snowpipe Streaming │
 │        ↓                                       │
 │  Bronze → Silver → Gold                        │
 └───────────────────────────────────────────────┘
```

### Why land first, load second?

| Benefit | Explanation |
|---------|-------------|
| **Decouple ingestion from transformation** | Source systems push files on their own schedule; Snowflake picks them up independently — no tight coupling |
| **Replay & reprocess** | Raw files stay in storage; if a load fails or logic changes, just re-ingest — no need to ask the source again |
| **Lower cost** | Object storage is far cheaper than Snowflake credits; keep the warehouse off until data is ready |
| **Platform-native tooling** | Use ADF (Azure), Glue (AWS), or Dataflow (GCP) for extraction — tools built for moving data at scale |
| **Security boundary** | Landing zone can sit behind its own network/IAM policies before Snowflake ever touches the data |

> 💡 **Rule of thumb:** let each platform do what it does best — cloud
> services move data, Snowflake transforms and serves it.

---

## What is a stage?

A **stage** is a pointer to a folder that Snowflake can read files from.
It is the **landing zone** between a raw file (CSV, JSON, Parquet, etc.) and a Snowflake table.

```
  raw file  ──▶  STAGE  ──▶  COPY INTO  ──▶  TABLE
```

There are two types:

| | Internal Stage | External Stage |
| --- | --- | --- |
| Where files live | Inside Snowflake (managed for you) | **Your own cloud storage** (Azure Blob, S3, GCS) |
| Setup effort | One line | URL + credentials |
| Best for | Quick demos, small ad-hoc uploads | **Production pipelines, data lakes, large loads** |
| **Used in practice** | Rarely | **Almost always** |

---

## 1 — Internal Stage

Snowflake stores the files for you. Simple, but limited — you can't share the
files with other tools, and storage is tied to Snowflake.

```sql
CREATE OR REPLACE STAGE my_internal_stage;

-- Upload via Snowsight UI:  open the stage → click "+ Files"
-- Upload via SnowSQL CLI:
PUT file:///path/to/data.csv @my_internal_stage;

LIST @my_internal_stage;   -- see what's staged
```

> Use internal stages for **workshop exercises** and **quick one-off loads**.
> For anything beyond that, use an external stage.

---

## 2 — External Stage  ⭐ Most Used

An **external stage** points Snowflake at storage **you own** — an Azure Blob
container, an AWS S3 bucket, or a GCP Cloud Storage bucket.
The data stays in your cloud; Snowflake just reads it.

**This is the standard pattern in production.**

### The easy setup

**Step 1 — Create a schema for raw data:**

```sql
CREATE DATABASE IF NOT EXISTS LEARN_DB;
CREATE SCHEMA IF NOT EXISTS LEARN_DB.RAW;
USE SCHEMA LEARN_DB.RAW;
```

**Step 2 — Create the external stage:**

**Azure (SAS token — recommended for Azure):**

```sql
CREATE OR REPLACE STAGE az_external_stage
  URL = 'azure://<storage_account>.blob.core.windows.net/<container>/<path>/'
  CREDENTIALS = ( AZURE_SAS_TOKEN = '<your-sas-token>' );

LIST @az_external_stage;   -- verify Snowflake can see your files
```

**AWS (key/secret):**

```sql
CREATE OR REPLACE STAGE s3_external_stage
  URL = 's3://<bucket>/<path>/'
  CREDENTIALS = ( AWS_KEY_ID = '<your-access-key>' AWS_SECRET_KEY = '<your-secret-key>' );

LIST @s3_external_stage;
```

> **Why SAS tokens?** A SAS (Shared Access Signature) token is the simplest way
> to grant Snowflake read access to Azure Blob storage. You generate it in the
> Azure Portal with a specific expiry and permission scope — no service principal
> or Entra ID setup required. **This is the recommended approach for workshops
> and getting started quickly.**

### For production — Storage Integration

In production, replace the inline SAS token with a **Storage Integration**.
This keeps credentials out of SQL and is centrally governed by an admin.

```sql
CREATE STORAGE INTEGRATION az_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'AZURE'
  ENABLED = TRUE
  AZURE_TENANT_ID = '<tenant-id>'
  STORAGE_ALLOWED_LOCATIONS = ('azure://<account>.blob.core.windows.net/<container>/');

-- Then create the stage using the integration (no inline credentials):
CREATE OR REPLACE STAGE az_external_stage
  URL = 'azure://<account>.blob.core.windows.net/<container>/<path>/'
  STORAGE_INTEGRATION = az_integration;
```

---

## 3 — The real-world pattern: push files to cloud, read via external stage

In practice, you **don't** upload files to Snowflake directly. Instead, you push
files to cloud storage with a CLI tool, and Snowflake reads them through an
external stage.

```
  your file  ──▶  cloud storage  ──▶  external stage  ──▶  COPY INTO  ──▶  TABLE
```

### Azure (this workshop)

```bash
azcopy copy local-data.csv "https://<account>.blob.core.windows.net/<container>/<path>/local-data.csv?<sas-token>"
```

### AWS

```bash
aws s3 cp local-data.csv s3://<bucket>/<path>/local-data.csv
```

### GCP

```bash
gsutil cp local-data.csv gs://<bucket>/<path>/local-data.csv
```

Then in Snowflake, the external stage picks it up:

```sql
LIST @az_external_stage;                       -- see the file
COPY INTO my_table FROM @az_external_stage;    -- load it
```

> **Note:** Snowflake also has its own CLI — **`snow`** (Snowflake CLI) — which
> can stage files and run queries. It is an alternative to SnowSQL and the
> cloud-specific CLIs above.

---

## 4 — Medallion Architecture: organize data in layers

Once files land via a stage, data flows through **progressive layers**, getting
cleaner and more reliable at each step. This is the **Medallion Architecture**.

```
 Cloud Storage          Snowflake
 ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
 │  FILES   │───▶│  BRONZE  │───▶│  SILVER  │───▶│   GOLD   │
 │ (landing │    │ (as-is   │    │ (cleaned,│    │ (business│
 │  zone)   │    │  copy)   │    │  joined) │    │  ready)  │
 └──────────┘    └──────────┘    └──────────┘    └──────────┘
   ADLS/S3/GCS     COPY INTO       transforms      aggregates,
                   Snowpipe        & standards      models, KPIs
```

### The layers

| Layer | What happens | Data quality |
|-------|-------------|--------------|
| **Files in cloud storage** | Source systems land raw files (CSV, Parquet, JSON) in ADLS / S3 / GCS | Untouched — exactly as the source sent it |
| **Bronze** | `COPY INTO` or Snowpipe loads files as-is into Snowflake tables | Raw copy — schema matches the source, no transformation yet |
| **Silver** | Cleansing, deduplication, type casting, standardizing column names, joining reference data | Clean & consistent — ready for analysts to explore |
| **Gold** | Aggregations, business logic, KPIs, dimensional models, reporting views | Business-ready — trusted numbers for dashboards and decisions |

### Many names, same idea

| Layer | Medallion | Also called | Also called | Also called |
|-------|-----------|-------------|-------------|-------------|
| Files from source | **Stage/External** | RAW / STAGE | LANDING | SOURCE |
| After COPY INTO Snowflake | **Bronze** | STAGING | RAW | INGESTED |
| After cleansing / standardize | **Silver** | CURATED | PROCESSED | CONFORMED |
| Ready to use | **Gold** | ANALYTICS | CURATED | PRESENTATION |

> Don't get hung up on naming. Pick one convention for your project and stick
> with it. What matters is the **progression**: raw → clean → ready.

### Example: Snowflake schemas as layers

A common pattern is to use **one schema per layer** inside the same database:

```sql
-- One schema per medallion layer
CREATE SCHEMA IF NOT EXISTS bronze;   -- raw loads land here
CREATE SCHEMA IF NOT EXISTS silver;   -- cleaned & standardized
CREATE SCHEMA IF NOT EXISTS gold;     -- business-ready

-- Bronze: raw copy from landing zone
COPY INTO bronze.sales
  FROM @landing_stage/sales/
  FILE_FORMAT = (TYPE = PARQUET)
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

-- Silver: clean and standardize
CREATE OR REPLACE TABLE silver.sales AS
SELECT
    sale_id,
    TRIM(customer_name)              AS customer_name,
    sale_amount::NUMBER(12,2)        AS sale_amount,
    TO_DATE(sale_date, 'YYYY-MM-DD') AS sale_date
FROM bronze.sales
WHERE sale_id IS NOT NULL;

-- Gold: aggregate for reporting
CREATE OR REPLACE TABLE gold.daily_sales AS
SELECT
    sale_date,
    COUNT(*)        AS total_orders,
    SUM(sale_amount) AS total_revenue
FROM silver.sales
GROUP BY sale_date;
```

> 💡 **Why separate schemas?** Each layer can have its own access roles —
> data engineers own Bronze, analysts read Silver, dashboards query Gold.
> This keeps governance clean and prevents accidental writes to curated data.

---

## Recap

| Concept | Key point |
| --- | --- |
| **Land first, load second** | Push files to cloud storage, then let Snowflake read via a stage |
| **Stage** | A pointer to a folder Snowflake reads files from before `COPY INTO` |
| **Internal stage** | Snowflake-managed — quick and easy, but rarely used in production |
| **External stage** | Points at **your** cloud storage — **the standard approach** |
| **SAS token** | Simplest way to authenticate an Azure external stage (recommended to start) |
| **Storage Integration** | Production-grade credential management — no secrets in SQL |
| **The pattern** | `azcopy` / `aws s3 cp` / `gsutil cp` → cloud storage → external stage → `COPY INTO` |
| **Medallion Architecture** | Organize data in layers — Bronze (raw) → Silver (clean) → Gold (ready) |

[← Back to Home](index.md) | [Next: Stages demo →](02-2-stages-demo.md)
