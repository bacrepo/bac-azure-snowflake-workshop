---
title: "Lesson 03 — Stages"
---

# Lesson 03 — Stages

**Goal:** understand where files live *before* they become tables — and the
difference between **internal** and **external** stages.

[← Back to Home](index.md)

---

## What is a stage?

A **stage** is a location Snowflake can read files from — essentially a folder.
It's the landing zone between a raw file (CSV, Excel, JSON) and a table.

```
  your file ──▶ STAGE ──▶ COPY INTO ──▶ TABLE
```

## Internal stages — Snowflake-managed

Snowflake stores the files for you. Best for ad-hoc uploads and workshop work.

```sql
USE SCHEMA LEARN_DB.WORKSHOP;   -- created in the next lesson

CREATE OR REPLACE STAGE my_stage;

-- Upload via Snowsight: open the stage and click + Files.
-- Or with the SnowSQL CLI:
PUT file:///path/to/data.csv @my_stage;

LIST @my_stage;   -- what's staged right now?
```

## External stages — point at Azure Blob

An **external** stage references storage *you* own — here, an **Azure Blob /
ADLS Gen2 container**. The data stays in Azure; Snowflake just reads it.

```sql
CREATE OR REPLACE STAGE az_stage
  URL = 'azure://<account>.blob.core.windows.net/<container>/<path>/'
  CREDENTIALS = ( AZURE_SAS_TOKEN = '<sas-token>' );
  -- production: use a STORAGE INTEGRATION instead of an inline token.

LIST @az_stage;
```

| | Internal stage | External stage |
| -- | -- | -- |
| Where files live | Snowflake-managed | Your Azure Blob container |
| Setup | one line | URL + credentials / integration |
| Best for | uploads, scratch | existing Azure data lakes, big loads |

> 💡 Prefer a **`STORAGE INTEGRATION`** over inline SAS tokens in production — it
> keeps credentials out of SQL and is centrally governed.

---

## Recap

- A **stage** is the folder files sit in before `COPY INTO`.
- **Internal** = Snowflake-managed; **external** = your **Azure Blob** container.
- `LIST @stage` shows what's there; `PUT` / **+ Files** upload to internal stages.

[← Back to Home](index.md)
