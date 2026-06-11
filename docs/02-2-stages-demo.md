---
title: "DEMO 02 — Stages Demo (Hands-on)"
---

# DEMO 02 — Stages Demo (Hands-on)

**Goal:** load CSV and Parquet files into Snowflake using an **internal stage**,
then connect an **external stage** to Azure ADLS.

[← Back to Home](index.md)

---

## Step 0 — Download the sample files

Download these two files to your local machine:

- [Demo.csv](Demo.csv)
- [Demo.parquet](Demo.parquet)

`Demo.csv` contains:

```
Name,Product,SalesAmount
Somchai,Laptop,45000
Nattaya,Monitor,8900
Praphan,Keyboard,1200
Wanida,Mouse,650
Kittisak,Headphones,3400
Suchada,Webcam,2100
Anuwat,DockingStation,5600
Malee,USBHub,890
Thanat,ExternalSSD,4300
Pim,GraphicsTablet,7800
```

---

## Step 1 — Create schema & internal stage

```sql
USE ROLE BAC_PLATFORM_ROLE;
USE WAREHOUSE WORKSHOP_WH;
CREATE DATABASE IF NOT EXISTS USERXX;
GRANT OWNERSHIP ON DATABASE USERXX TO ROLE BAC_DE_ROLE COPY CURRENT GRANTS;

USE ROLE BAC_DE_ROLE;
CREATE SCHEMA IF NOT EXISTS USERXX.RAW;
USE SCHEMA USERXX.RAW;

CREATE OR REPLACE STAGE stage_manual;
```

---

## Step 2 — Create file formats

```sql
CREATE OR REPLACE FILE FORMAT format_csv_no_header
  TYPE = 'CSV'
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  SKIP_HEADER = 1;

CREATE OR REPLACE FILE FORMAT format_csv
  TYPE = 'CSV'
  FIELD_OPTIONALLY_ENCLOSED_BY = '"'
  PARSE_HEADER = TRUE;

CREATE OR REPLACE FILE FORMAT format_parquet
  TYPE = 'PARQUET';
```

---

## Step 3 — Upload files to the internal stage

In **Snowsight**: go to Data → Databases → USER00 → RAW → Stages →
`STAGE_MANUAL` → click **+ Files** → upload `Demo.csv` and `Demo.parquet`.

Verify:

```sql
LIST @stage_manual;
```

You should see both files listed.

---

## Step 4 — Load CSV into a table (define schema manually)

The traditional way — **you** define the columns:

```sql
CREATE OR REPLACE TABLE sales_csv_manual (
  Name           VARCHAR,
  Product        VARCHAR,
  SalesAmount    NUMBER
);

COPY INTO sales_csv_manual
  FROM @stage_manual/Demo.csv
  FILE_FORMAT = (FORMAT_NAME = format_csv_no_header);

SELECT * FROM sales_csv_manual;
```

---

## Step 5 — Load CSV into a table (schema inferred)

Snowflake can also **infer the schema** from CSV — it reads the header row for
column names and samples the data to detect types:

```sql
-- Check what columns Snowflake detects
SELECT *
  FROM TABLE(INFER_SCHEMA(
    LOCATION => '@stage_manual/Demo.csv',
    FILE_FORMAT => 'format_csv'
  ));

-- Create the table using the inferred schema
CREATE OR REPLACE TABLE sales_csv_infer
  USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
      FROM TABLE(INFER_SCHEMA(
        LOCATION => '@stage_manual/Demo.csv',
        FILE_FORMAT => 'format_csv'
      ))
  );

COPY INTO sales_csv_infer
  FROM @stage_manual/Demo.csv
  FILE_FORMAT = (FORMAT_NAME = format_csv)
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

SELECT * FROM sales_csv_infer;
```

---

## Step 6 — Load Parquet into a table (schema inferred)

Parquet files **carry their own schema** (column names + types), so Snowflake
figures it out for you — no manual `CREATE TABLE` needed:

```sql
-- Check what columns Snowflake detects
SELECT *
  FROM TABLE(INFER_SCHEMA(
    LOCATION => '@stage_manual/Demo.parquet',
    FILE_FORMAT => 'format_parquet'
  ));

-- Create the table using the inferred schema
CREATE OR REPLACE TABLE sales_parquet_infer
  USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
      FROM TABLE(INFER_SCHEMA(
        LOCATION => '@stage_manual/Demo.parquet',
        FILE_FORMAT => 'format_parquet'
      ))
  );

COPY INTO sales_parquet_infer
  FROM @stage_manual/Demo.parquet
  FILE_FORMAT = (FORMAT_NAME = format_parquet)
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

SELECT * FROM sales_parquet_infer;
```

> **Why Parquet is great:** the schema is embedded in the file itself, so
> inference is reliable. CSV inference is a best-guess from sampled data.

---

## Recap — what we built

```
                          ┌──────────────────────────────────┐
  Demo.csv ──▶ + Files ──▶│  @stage_manual  (internal stage) │──▶ COPY INTO ──▶ sales_csv_manual
  Demo.parquet ──────────▶│                                  │──▶ COPY INTO ──▶ sales_parquet_infer
                          └──────────────────────────────────┘
```

| What | SQL |
| --- | --- |
| Schema for raw data | `CREATE SCHEMA RAW` |
| Internal stage | `CREATE STAGE stage_manual` |
| CSV format | `CREATE FILE FORMAT format_csv TYPE = 'CSV' ...` |
| Parquet format | `CREATE FILE FORMAT format_parquet TYPE = 'PARQUET'` |
| Table from CSV (manual) | Define columns yourself, then `COPY INTO` |
| Table from CSV (inferred) | `INFER_SCHEMA` + `USING TEMPLATE` + `COPY INTO` |
| Table from Parquet (inferred) | `INFER_SCHEMA` + `USING TEMPLATE` — no manual schema |

[← Back to Home](index.md) | [Next: External stage →](02-3-external-stage.md)
