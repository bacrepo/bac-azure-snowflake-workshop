---
title: "Lesson 04 — Data engineering I"
---

# Lesson 04 — Data engineering I

**Goal:** the core build loop — create tables, load files, transform with
scripting and **dynamic tables**, and recover with **time travel**.

[← Back to Home](index.md)

---

## 1. Databases, schemas & tables

Objects live in a three-level tree: `DATABASE.SCHEMA.TABLE`.

```sql
CREATE DATABASE IF NOT EXISTS LEARN_DB;
CREATE SCHEMA   IF NOT EXISTS LEARN_DB.WORKSHOP;
USE SCHEMA LEARN_DB.WORKSHOP;

CREATE OR REPLACE TABLE sales (
  id          INT,
  region      STRING,
  sold_on     DATE,
  amount      NUMBER(12,2)
);
```

## 2. COPY INTO — load staged files

```sql
CREATE OR REPLACE FILE FORMAT csv_with_header
  TYPE = 'CSV'
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';

COPY INTO sales
  FROM @my_stage
  FILE_FORMAT = (FORMAT_NAME = 'csv_with_header')
  ON_ERROR = 'CONTINUE';   -- skip bad rows while learning

SELECT * FROM sales LIMIT 10;
```

## 3. Scripting — logic in SQL

Snowflake Scripting adds variables, loops, and branching. Run a block with
`EXECUTE IMMEDIATE`, or wrap it in a stored procedure.

```sql
EXECUTE IMMEDIATE $$
BEGIN
  CREATE OR REPLACE TABLE sales_clean AS
    SELECT * FROM sales WHERE amount IS NOT NULL;
  RETURN 'loaded ' || (SELECT COUNT(*) FROM sales_clean) || ' rows';
END;
$$;
```

## 4. Dynamic tables — declarative pipelines

A **dynamic table** defines its contents as a query; Snowflake keeps it fresh
automatically within your **target lag**. No manual refresh, no scheduling.

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

## 5. Time travel — undo for data

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

- Build loop: **table → `COPY INTO` → transform → serve**.
- **Scripting** adds procedural logic; **dynamic tables** keep derived data
  fresh declaratively (`TARGET_LAG`).
- **Time travel** (`AT` / `BEFORE` / `UNDROP`) is your safety net.

[← Back to Home](index.md)
