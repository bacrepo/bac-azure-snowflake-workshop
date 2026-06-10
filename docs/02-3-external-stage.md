---
title: "Demo 03 — External Stage (Azure ADLS)"
---

# Demo 03 — External Stage (Azure ADLS)

**Goal:** connect Snowflake to Azure ADLS, explore the files on the stage,
and load data from the external stage into tables.

[← Back to Home](index.md)

---

## What's on the stage?

Your instructor has uploaded files to an Azure Data Lake Storage (ADLS)
container. Here's what's there:

| File | Format |
|------|--------|
| `Demo.csv` | CSV |
| `Demo.parquet` | Parquet |
| `Orders.parquet` | Parquet - SharePoint List |
| `Orders_INCR.parquet` | Parquet - SharePoint List 
| `POC_EXCEL_DATA_3.xlsx` | Excel - SharePoint File |
| `SAMPLE01.xlsx` | Excel |
| `SAMPLE02.xlsx` | Excel |
| `VBAK.csv` | CSV |
| `VBAK.parquet` | Parquet |
| `VBAK_INCR.parquet` | Parquet |
| `VBEP.csv` | CSV |
| `VBEP.parquet` | Parquet |

---

## Step 1 — Create the external stage

Connect Snowflake to your **Azure Data Lake Storage** so it can read files
directly from the cloud.

```sql
USE ROLE BAC_DE_ROLE;
USE WAREHOUSE WORKSHOP_WH;
USE SCHEMA USER00.RAW;

CREATE OR REPLACE STAGE stage_azure
  URL = 'azure://dlsbacpocsnowflake.blob.core.windows.net/rawbgc/'
  CREDENTIALS = ( AZURE_SAS_TOKEN = 'sv=2026-02-06&ss=bf&srt=sco&sp=rlx&se=2026-06-17T16:48:05Z&st=2026-06-10T08:33:05Z&spr=https&sig=tihpb7YnzyKqm7Va0bvjTD6WCnvNsMiRJi42vkEtTPY%3D' );
```

> **Note:** this SAS token expires on **2026-06-17**. If the workshop is after
> that date, generate a new one in the Azure Portal.

Verify:

```sql
LIST @stage_azure;
```

If you see the files listed above, the connection works.

---

## Step 2 — Load CSV with manual schema (`sales_csv_external`)

Define the columns yourself, then `COPY INTO` from the external stage — same
pattern as the internal stage, but reading from Azure:

```sql
CREATE OR REPLACE TABLE sales_csv_external (
  Name           VARCHAR,
  Product        VARCHAR,
  SalesAmount    NUMBER
);

COPY INTO sales_csv_external
  FROM @stage_azure/Demo.csv
  FILE_FORMAT = (FORMAT_NAME = format_csv_no_header);

SELECT * FROM sales_csv_external;
```

---

## Step 3 — Load CSV with inferred schema (`sales_csv_external_infer`)

Let Snowflake detect the schema from the CSV header — no manual `CREATE TABLE`
needed:

```sql
-- Check what columns Snowflake detects
SELECT *
  FROM TABLE(INFER_SCHEMA(
    LOCATION => '@stage_azure/Demo.csv',
    FILE_FORMAT => 'format_csv'
  ));

-- Create the table using the inferred schema
CREATE OR REPLACE TABLE sales_csv_external_infer
  USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
      FROM TABLE(INFER_SCHEMA(
        LOCATION => '@stage_azure/Demo.csv',
        FILE_FORMAT => 'format_csv'
      ))
  );

COPY INTO sales_csv_external_infer
  FROM @stage_azure/Demo.csv
  FILE_FORMAT = (FORMAT_NAME = format_csv)
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

SELECT * FROM sales_csv_external_infer;
```

---

## Recap — what we built

```
                          ┌──────────────────────────────────┐
  azcopy ──▶ Azure ADLS ──▶│  @stage_azure   (external stage) │
                          └──────────────┬───────────────────┘
                                         │
                     ┌───────────────────┼───────────────────┐
                     │                   ▼                   │
                     │  sales_csv_external       (manual)    │
                     │  sales_csv_external_infer (inferred)  │
                     └───────────────────────────────────────┘
```

| What | SQL |
| --- | --- |
| External stage (Azure) | `CREATE STAGE stage_azure URL = 'azure://...' CREDENTIALS = (...)` |
| Manual-schema load | `CREATE TABLE` + `COPY INTO` from `@stage_azure/Demo.csv` |
| Inferred-schema load | `INFER_SCHEMA` + `USING TEMPLATE` + `COPY INTO` from `@stage_azure/Demo.csv` |

[← Back to Home](index.md) | [Next: Data engineering →](03-0-data-engineering.md)
