---
title: "DEMO 06 — Incremental data"
---

# DEMO 06 — Incremental data

**Goal:** load only what changed — apply a delta file to an existing table
with the two production patterns: `DELETE` + `MERGE` and `DELETE` + `INSERT`,
both safe to re-run.

[← Back to Home](index.md)

---

## Full load vs. incremental load

So far we reloaded everything every time. That's a **full load** — simple, but
expensive once the source grows. Most days only a handful of rows actually
change.

An **incremental load** brings in *only what changed* since the last run, then
**merges** those changes into the table you already have.

```
Full load:         read ALL files  ──►  rebuild the whole table   (slow, every row)
Incremental load:  read CHANGED file ──►  MERGE into existing table (fast, deltas only)
```

In this lesson the source gives us two things:

| Source on the stage | What it is |
|---------------------|-----------|
| `VBAK/VBAK*.parquet` | The **full** history, split across many partitioned files in a folder |
| `VBAK_INCR.parquet` | The **delta** — new and changed rows since the last load |

We build a full bronze table from the partitions, a separate bronze table from
the delta, then **MERGE** the delta into the full table.

---

## Step 1 — External table over the partitioned full load (`ext.VBAK_PARTITION`)

The full history isn't one file — it's many Parquet files under a `VBAK/`
folder (e.g. `VBAK/VBAK_001.parquet`, `VBAK/VBAK_002.parquet`, …). A single
external table can read **all of them at once** with a folder `LOCATION` and a
`PATTERN`.

```sql
USE SCHEMA ext;

CREATE OR REPLACE EXTERNAL TABLE VBAK_PARTITION
  USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
    FROM TABLE(
      INFER_SCHEMA(
        LOCATION => '@raw_azure.stage_azure/VBAK/',
        FILE_FORMAT => 'raw.format_parquet',
        FILES => 'VBAK/VBAK*.parquet'
      )
    )
  )
  LOCATION = @raw_azure.stage_azure/VBAK/
  FILE_FORMAT = raw.format_parquet
  PATTERN = '.*VBAK.*[.]parquet';

SELECT * FROM ext.VBAK_PARTITION LIMIT 10;
```

> One external table, many files. Add more partition files to the folder and
> they show up here automatically — `PATTERN` matches them on read.

---

## Step 2 — External table over the incremental file (`ext.VBAK_INCR`)

The delta is a single file, `VBAK_INCR.parquet`.

```sql
CREATE OR REPLACE EXTERNAL TABLE VBAK_INCR
  USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
    FROM TABLE(
      INFER_SCHEMA(
        LOCATION => '@raw_azure.stage_azure/',
        FILE_FORMAT => 'raw.format_parquet',
        FILES => 'VBAK_INCR.parquet'
      )
    )
  )
  LOCATION = @raw_azure.stage_azure/
  FILE_FORMAT = raw.format_parquet
  PATTERN = 'VBAK_INCR.parquet';

SELECT * FROM ext.VBAK_INCR LIMIT 10;
```

---

## Step 3 — Bronze full load (`bronze.VBAK_FULL`)

Pick only the columns we need and give them business-friendly names. This is the
table we keep — the current full picture.

```sql
USE SCHEMA bronze;

CREATE OR REPLACE TABLE bronze.VBAK_FULL AS
SELECT
    VBELN,       -- Sales Document Number
    AUART,       -- Sales Document Type
    VBTYP,       -- SD Document Category
    AUDAT,       -- Document Date
    VKORG,       -- Sales Organization
    VTWEG,       -- Distribution Channel
    SPART,       -- Division
    VKBUR,       -- Sales Office
    VKGRP,       -- Sales Group
    KUNNR,       -- Sold-to Party
    NETWR        -- Net Value
FROM ext.VBAK_PARTITION;

SELECT COUNT(*) AS full_rows FROM bronze.VBAK_FULL;
```

---

## Step 4 — Bronze incremental (`bronze.VBAK_INCR`)

Same columns, same names — but sourced from the delta file. These are the rows
to apply.

```sql
CREATE OR REPLACE TABLE bronze.VBAK_INCR AS
SELECT
    VBELN,       -- Sales Document Number
    AUART,       -- Sales Document Type
    VBTYP,       -- SD Document Category
    AUDAT,       -- Document Date
    VKORG,       -- Sales Organization
    VTWEG,       -- Distribution Channel
    SPART,       -- Division
    VKBUR,       -- Sales Office
    VKGRP,       -- Sales Group
    KUNNR,       -- Sold-to Party
    NETWR        -- Net Value
FROM ext.VBAK_INCR;

SELECT COUNT(*) AS incr_rows FROM bronze.VBAK_INCR;
```

---

## Step 5 — Delete the overlap, then MERGE

A clean way to apply a delta: **first delete every row in the full table whose
key appears in the delta**, then load the delta back in. Clearing the overlap
up front guarantees the delta fully replaces those keys — no stale leftover
values, and the load is safe to re-run.

```
   bronze.VBAK_FULL                          bronze.VBAK_INCR
   ┌───────────────┐                          ┌───────────────┐
   │ VBELN  NETWR  │                          │ VBELN  NETWR  │
   │  1001   500   │                          │  1002   999   │  (changed)
   │  1002   400   │◄── overlap on VBELN ────►│  1005   120   │  (new)
   │  1003   700   │                          └───────┬───────┘
   └───────┬───────┘                                  │
           │  ① DELETE WHERE VBELN IN (delta)         │
           ▼                                          │
   ┌───────────────┐                                  │
   │  1001   500   │   1002 removed                   │
   │  1003   700   │                                  │
   └───────┬───────┘                                  │
           │  ② MERGE  ◄───────────────────────────────┘
           ▼          (no matches left → every delta row INSERTs)
   ┌───────────────┐
   │  1001   500   │
   │  1003   700   │
   │  1002   999   │   updated value
   │  1005   120   │   new row
   └───────────────┘
```

```sql
-- 1. Remove the rows the delta is about to replace
DELETE FROM bronze.VBAK_FULL
WHERE VBELN IN (SELECT DISTINCT VBELN FROM bronze.VBAK_INCR);

-- 2. MERGE the delta in (after the delete, every row is a fresh INSERT)
MERGE INTO bronze.VBAK_FULL AS tgt
USING bronze.VBAK_INCR AS src
  ON tgt.VBELN = src.VBELN
WHEN MATCHED THEN UPDATE SET
    tgt.AUART = src.AUART,
    tgt.VBTYP = src.VBTYP,
    tgt.AUDAT = src.AUDAT,
    tgt.VKORG = src.VKORG,
    tgt.VTWEG = src.VTWEG,
    tgt.SPART = src.SPART,
    tgt.VKBUR = src.VKBUR,
    tgt.VKGRP = src.VKGRP,
    tgt.KUNNR = src.KUNNR,
    tgt.NETWR = src.NETWR
WHEN NOT MATCHED THEN INSERT (
    VBELN, AUART, VBTYP, AUDAT, VKORG, VTWEG,
    SPART, VKBUR, VKGRP, KUNNR, NETWR
) VALUES (
    src.VBELN, src.AUART, src.VBTYP, src.AUDAT, src.VKORG, src.VTWEG,
    src.SPART, src.VKBUR, src.VKGRP, src.KUNNR, src.NETWR
);
```

Snowflake reports how many rows were deleted and inserted in each statement.

---

## Step 6 — Delete the overlap, then INSERT INTO

If the delta only ever brings whole rows (never partial updates), you don't even
need `MERGE`. Same first step — **delete the overlap** — then a plain
`INSERT INTO` appends the delta. Simpler, and just as idempotent.

```
   bronze.VBAK_FULL                          bronze.VBAK_INCR
   ┌───────────────┐                          ┌───────────────┐
   │ VBELN  NETWR  │                          │ VBELN  NETWR  │
   │  1001   500   │                          │  1002   999   │  (changed)
   │  1002   400   │◄── overlap on VBELN ────►│  1005   120   │  (new)
   │  1003   700   │                          └───────┬───────┘
   └───────┬───────┘                                  │
           │  ① DELETE WHERE VBELN IN (delta)         │
           ▼                                          │
   ┌───────────────┐                                  │
   │  1001   500   │   1002 removed                   │
   │  1003   700   │                                  │
   └───────┬───────┘                                  │
           │  ② INSERT INTO  ◄─────────────────────────┘
           ▼          (append ALL delta rows, no matching)
   ┌───────────────┐
   │  1001   500   │
   │  1003   700   │
   │  1002   999   │   re-added with new value
   │  1005   120   │   new row
   └───────────────┘
```

```sql
-- 1. Remove the rows the delta is about to replace
DELETE FROM bronze.VBAK_FULL
WHERE VBELN IN (SELECT DISTINCT VBELN FROM bronze.VBAK_INCR);

-- 2. Append every delta row
INSERT INTO bronze.VBAK_FULL (
    VBELN, AUART, VBTYP, AUDAT, VKORG, VTWEG,
    SPART, VKBUR, VKGRP, KUNNR, NETWR
)
SELECT
    VBELN, AUART, VBTYP, AUDAT, VKORG, VTWEG,
    SPART, VKBUR, VKGRP, KUNNR, NETWR
FROM bronze.VBAK_INCR;
```

### Verify

```sql
-- Row count after the load
SELECT COUNT(*) AS rows_after_load FROM bronze.VBAK_FULL;

-- Spot-check a document that was in the delta
SELECT * FROM bronze.VBAK_FULL
WHERE VBELN IN (SELECT VBELN FROM bronze.VBAK_INCR)
ORDER BY VBELN;
```

---

## Re-running is safe

Both patterns start by **deleting the overlap**, so re-running with the same
delta lands the table in the exact same state — the delete clears the keys, the
load puts them back once. That property is called **idempotency**, and it's what
makes incremental loads safe to retry after a failure.

| Pattern | When to use |
|---------|-------------|
| **MERGE** (Step 5) | Delta may contain partial updates; one statement does update + insert |
| **Delete + INSERT** (Step 6) | Delta always carries whole rows; simpler, no match logic |

---

## Recap

```
┌────────────────────────────┐        ┌────────────────────────┐
│  @stage/VBAK/VBAK*.parquet  │        │  @stage/VBAK_INCR.parquet │
│  full history (partitions)  │        │  delta (changed rows)   │
└──────────────┬─────────────┘        └───────────┬────────────┘
               ▼                                   ▼
        ext.VBAK_PARTITION                  ext.VBAK_INCR
               │                                   │
               ▼                                   ▼
        bronze.VBAK_FULL  ◄── DELETE overlap ── bronze.VBAK_INCR
        (current truth)     then MERGE / INSERT     (rows to apply)
                              on VBELN
```

| Step | Object | What it does |
|------|--------|-------------|
| **Full external** | `ext.VBAK_PARTITION` | One external table over many partition files (`VBAK/VBAK*.parquet`) |
| **Delta external** | `ext.VBAK_INCR` | External table over the single delta file |
| **Bronze full** | `bronze.VBAK_FULL` | 11 selected columns — the table we keep |
| **Bronze delta** | `bronze.VBAK_INCR` | Same columns from the delta |
| **Apply (Step 5)** | `DELETE` overlap + `MERGE` | Clear matching `VBELN`, then upsert the delta |
| **Apply (Step 6)** | `DELETE` overlap + `INSERT` | Clear matching `VBELN`, then append the delta |

**Next:** nobody wants to run these two statements by hand every hour —
let Snowflake run them on a schedule with **tasks**.

[← Back to Home](index.md) | [Next: Tasks & scheduling →](03-4-task-and-schedule.md)
