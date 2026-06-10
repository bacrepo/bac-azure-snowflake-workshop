---
title: "Lesson 04 — Loading data"
---

# Lesson 04 — Loading data

**Goal:** get a file of data *into* a Snowflake table — the core ETL pattern.

[← Previous: Databases](04-databases.md) · [Home](index.md) · [Next: Querying →](06-querying.md)

---

## The three pieces

To load a file, Snowflake uses three concepts:

1. **Stage** — a place files live before loading (a folder, essentially).
2. **File format** — how to *read* the file (CSV? JSON? comma- or tab-separated?).
3. **`COPY INTO`** — the command that reads staged files into a table.

```
  your file ──▶ STAGE ──(FILE FORMAT tells COPY how to parse)──▶ COPY INTO ──▶ TABLE
```

## 1. Create a stage and a file format

```sql
USE SCHEMA LEARN_DB.WORKSHOP;

-- An internal stage: Snowflake-managed storage you can upload files to.
CREATE OR REPLACE STAGE my_stage;

-- Describe a standard CSV with a header row.
CREATE OR REPLACE FILE FORMAT csv_with_header
  TYPE = 'CSV'
  FIELD_DELIMITER = ','
  SKIP_HEADER = 1
  FIELD_OPTIONALLY_ENCLOSED_BY = '"';
```

## 2. Put a file in the stage

In **Snowsight**: open the stage (Data → Databases → … → Stages → `MY_STAGE`)
and click **+ Files** to upload a CSV from your laptop.

Or with the **SnowSQL** CLI:

```sql
PUT file:///path/to/students.csv @my_stage;
```

Check it landed:

```sql
LIST @my_stage;
```

## 3. Copy it into a table

```sql
COPY INTO students
  FROM @my_stage
  FILE_FORMAT = (FORMAT_NAME = 'csv_with_header')
  ON_ERROR = 'CONTINUE';   -- skip bad rows instead of failing the whole load

SELECT * FROM students;
```

`ON_ERROR = 'CONTINUE'` is handy while learning — one malformed row won't abort
the entire load. In production you'd often want `'ABORT_STATEMENT'` so problems
are loud.

---

## Recap

- Loading = **stage** the file, define a **file format**, then **`COPY INTO`**.
- `PUT` uploads to an internal stage; **+ Files** does the same in Snowsight.
- `LIST @stage` shows what's staged; `ON_ERROR` controls how strict the load is.

[← Previous: Databases](04-databases.md) · [Home](index.md) · [Next: Querying →](06-querying.md)
