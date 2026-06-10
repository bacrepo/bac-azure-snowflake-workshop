---
title: "Lesson 04 — Data engineering I"
---

# Lesson 04 — Data engineering I

[← Back to Home](index.md)

---

## Role of a Data Engineer

> **Source → Land → Clean → Conform → Higher Value → Query-able**

A data engineer's job is to:

1. **Get data from sources** — files, APIs, databases
2. **Land** it into tables inside the warehouse (raw zone)
3. **Clean** — remove nulls, fix data types, deduplicate
4. **Conform** — standardise column names, formats, and business rules across sources
5. **Make higher value** — join, aggregate, enrich so it becomes useful
6. **Query-able** — serve it to analysts, dashboards, and applications

In Snowflake, the build loop looks like this:

```
Source files  ──►  External Stage  ──►  COPY INTO  ──►  Clean  ──►  Conform  ──►  Higher Value  ──►  Query-able
(Azure Blob)       (read files)        (land raw)      (nulls,      (standard     (join, enrich,     (serve to
                                                        types)       names/rules)   aggregate)         users)
```

---

## 1. External stage — connect to your source

An **external stage** points Snowflake at a cloud storage location (Azure Blob,
S3, GCS). The files stay where they are — Snowflake just reads them.

A single Azure container can hold many file types:

| File | Format |
|------|--------|
| `Demo.csv` | CSV |
| `Orders.parquet` / `Orders_INCR.parquet` | Parquet |
| `POC_EXCEL_DATA_3.xlsx` | Excel |
| `SAMPLE01.xlsx` / `SAMPLE02.xlsx` | Excel |
| `VBAK.csv` / `VBAK.parquet` / `VBAK_INCR.parquet` | CSV & Parquet |
| `VBEP.csv` / `VBEP.parquet` | CSV & Parquet |

### Create the stage

```sql
CREATE OR REPLACE STAGE stage_azure
  URL = 'azure://dlsbacpocsnowflake.blob.core.windows.net/rawbgc/'
  CREDENTIALS = (AZURE_SAS_TOKEN = '<your ADLS_SAS from .env>');
```

### List files in the stage

```sql
LIST @stage_azure;
```

This returns every file in the container — CSV, Parquet, Excel, etc.

---

## 2. External table — query Parquet without loading

If your source files are **Parquet**, you can skip `COPY INTO` entirely and
create an **external table**. It reads directly from the stage — no data is
copied into Snowflake storage.

This is useful when you want to:
- Query data **in place** without waiting for a load
- Keep the data in your cloud storage as the single source of truth
- Explore or profile files before deciding what to load

```sql
CREATE OR REPLACE EXTERNAL TABLE orders_ext
  WITH LOCATION = @stage_azure/
  FILE_FORMAT = (TYPE = 'PARQUET')
  PATTERN = '.*Orders.*[.]parquet';

-- Query it like a normal table:
SELECT * FROM orders_ext LIMIT 10;
```

> **Tip:** External tables work best for **read-heavy / low-frequency** access.
> For high-performance analytics, `COPY INTO` a native table is still faster.

---

## 3. Databases, schemas & tables

Before loading, create a landing zone. Objects live in a three-level tree:
`DATABASE.SCHEMA.TABLE`.

```sql
CREATE DATABASE IF NOT EXISTS USER00_DB;
CREATE SCHEMA   IF NOT EXISTS USER00_DB.RAW_AZURE;
USE SCHEMA USER00_DB.RAW_AZURE;

CREATE OR REPLACE TABLE sales (
  id          INT,
  region      STRING,
  sold_on     DATE,
  amount      NUMBER(12,2)
);
```

---

## 4. COPY INTO — land data into tables

`COPY INTO` is the workhorse that moves data **from a stage into a table**.

### Load CSV files

```sql
CREATE OR REPLACE FILE FORMAT csv_with_header
  TYPE = 'CSV'
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';

COPY INTO sales
  FROM @stage_azure/Demo.csv
  FILE_FORMAT = (FORMAT_NAME = 'csv_with_header')
  ON_ERROR = 'CONTINUE';   -- skip bad rows while learning
```

### Load Parquet files

```sql
COPY INTO sales
  FROM @stage_azure/Orders.parquet
  FILE_FORMAT = (TYPE = 'PARQUET')
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

### Verify the data landed

```sql
SELECT * FROM sales LIMIT 10;
```

---

## 5. Clean & Conform — transform the raw data

Once data is landed, **clean** and **conform** it so downstream users get
consistent, trustworthy data.

```sql
-- Clean: remove nulls, fix types
CREATE OR REPLACE TABLE sales_clean AS
  SELECT
    id,
    UPPER(TRIM(region))              AS region,       -- conform: standardise
    TRY_TO_DATE(sold_on)             AS sold_on,      -- clean: safe cast
    amount
  FROM sales
  WHERE amount IS NOT NULL;                            -- clean: drop bad rows
```

### Scripting — logic in SQL

Snowflake Scripting adds variables, loops, and branching for more complex
transforms:

```sql
EXECUTE IMMEDIATE $$
BEGIN
  CREATE OR REPLACE TABLE sales_clean AS
    SELECT * FROM sales WHERE amount IS NOT NULL;
  RETURN 'loaded ' || (SELECT COUNT(*) FROM sales_clean) || ' rows';
END;
$$;
```

---

## 6. Dynamic tables — make higher value, declaratively

A **dynamic table** defines its contents as a query; Snowflake keeps it fresh
automatically within your **target lag**. This is where you **join, aggregate,
and enrich** to create higher-value data — no manual refresh, no scheduling.

```sql
CREATE OR REPLACE DYNAMIC TABLE sales_by_region
  TARGET_LAG = '1 minute'
  WAREHOUSE  = LEARN_WH
AS
  SELECT region, SUM(amount) AS total_amount
  FROM sales_clean
  GROUP BY region;

SELECT * FROM sales_by_region;
```

Drop it when you're done:

```sql
DROP DYNAMIC TABLE sales_by_region;
```

---

## 7. Time travel — undo for data

Snowflake retains history, so you can query the past or recover from mistakes.

```sql
-- The table as it was 5 minutes ago:
SELECT * FROM sales AT(OFFSET => -60*5);

-- As it was just before a specific statement:
SELECT * FROM sales BEFORE(STATEMENT => '<query_id>');

-- Oops, dropped a table — bring it back:
DROP TABLE sales;
UNDROP TABLE sales;
```

---

## Recap

The data engineer's loop: **Source → Land → Clean → Conform → Higher Value → Query-able**.

| Step | Tool | What it does |
|------|------|-------------|
| Connect to source | **External stage** | Points at Azure Blob / S3 / GCS |
| Query in place (Parquet) | **External table** | Read files without loading |
| Land data | **`COPY INTO`** | Loads CSV, Parquet, etc. into tables |
| Clean & Conform | **SQL / Scripting** | Fix types, standardise, deduplicate |
| Higher value | **Dynamic tables** | Join, aggregate, enrich — auto-refreshed |
| Recover | **Time travel** | `AT` / `BEFORE` / `UNDROP` as a safety net |

[← Back to Home](index.md)
