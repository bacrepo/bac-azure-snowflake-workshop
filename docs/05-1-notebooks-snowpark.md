---
title: "Notebooks & Snowpark — Python for data engineers"
---

# Notebooks & Snowpark — Python for data engineers

**Goal:** do the data engineer's job — **land → clean → conform → write** — in
**Python** instead of SQL, using **Snowpark** inside a **Snowflake Notebook**,
and pull in a source SQL can't read natively: an **Excel file from the stage**.

[← Back to Home](index.md)

---

## Why Python, when SQL already works?

You've built the whole pipeline in SQL — `COPY INTO`, `MERGE`, dynamic tables,
tasks. Snowpark doesn't replace that; it gives you a second tool for the jobs
SQL is awkward at:

| Reach for SQL | Reach for Snowpark (Python) |
|---------------|------------------------------|
| Set-based loads, joins, aggregates | Logic that's easier as loops / branching |
| `COPY INTO` of CSV & Parquet | Formats SQL can't load — **Excel**, JSON-from-an-API |
| Dynamic tables, MERGE | Reusable transforms, pandas / ML libraries |

The important part: **Snowpark runs on your warehouse, not your laptop**. A
`session.table(...)` is a *plan*, not data — it executes on Snowflake and only
the result comes back. Same compute, same governance, same `WORKSHOP_WH` you've
been paying for all morning.

---

## Notebooks in Snowflake

A **Snowflake Notebook** runs SQL *and* Python cells against your account, with
results inline. Create one in Snowsight: **Projects → Notebooks → + Notebook**.

- Pick the **database/schema** (`USERXX.bronze`) and **warehouse** (`WORKSHOP_WH`)
  when you create it — same costing rules as everywhere else.
- Mix SQL and Python in the same notebook; a SQL cell's result is available to
  the next Python cell.
- Perfect for the data engineer's exploration → prototype → productionise loop.

---

## Snowpark — DataFrames pushed down to the engine

First, set the role and warehouse the engineer works under — same context as
the rest of the workshop:

```sql
USE ROLE BAC_DE_ROLE;
USE WAREHOUSE WORKSHOP_WH;
```

Inside a notebook you already have a `session`. Grab it, then work with the
`gold.Sales` table (the joined VBAK + VBAP + VBEP view we built earlier) as a
DataFrame:

```python
from snowflake.snowpark.context import get_active_session
session = get_active_session()

# A handle to the gold table we built earlier — nothing has run yet:
sales = session.table("gold.Sales")

# Build a transform. Still just a plan — pushed down, not pulled local:
from snowflake.snowpark.functions import col, sum as sum_

by_org = (
    sales
    .filter(col("ItemNetValue") > 1000)            # WHERE
    .group_by("SalesOrg", "DocumentCurrency")      # GROUP BY
    .agg(sum_("ItemNetValue").alias("TOTAL"))      # SUM(item net value)
    .sort(col("TOTAL").desc())
)

by_org.show()        # NOW it executes on WORKSHOP_WH; only rows come back
```

That `by_org` is the **exact** logic as a SQL `SELECT SalesOrg,
DocumentCurrency, SUM(ItemNetValue) ... GROUP BY SalesOrg, DocumentCurrency`,
written in Python. `.show()` (or `.collect()`, `.to_pandas()`) is what triggers
the run.

### Write the result back — Python as an ELT step

A transform you'd persist with `CREATE TABLE AS` in SQL is `save_as_table` in
Snowpark — landing the result straight into a schema:

```python
by_org.write.save_as_table(
    "gold.Sales_By_Org",
    mode="overwrite",     # like CREATE OR REPLACE
)
```

```sql
-- Queryable like anything else you built today:
SELECT * FROM gold.Sales_By_Org ORDER BY TOTAL DESC;
```

---

## Two DataFrames: Snowpark (Spark-style) vs. pandas

There are **two** DataFrame worlds in a notebook, and a data engineer uses both:

| | Snowpark DataFrame | pandas DataFrame |
|--|--------------------|------------------|
| API style | **Spark-like** — `select`, `filter`, `group_by`, `agg` | the classic `pdf[...]`, `.groupby()` |
| Runs on | **the warehouse** (pushed down, lazy) | **the notebook runtime** (in memory, eager) |
| Good for | big tables, set logic, ELT | small results, ML libs, Excel parsing |

### Read with the Snowpark (Spark-style) API

This is the default — it stays on `WORKSHOP_WH` and only moves results when you
ask:

```python
sales = session.table("gold.Sales")     # Spark-style, lazy, runs on the warehouse
sales.select("SalesOrg", "ItemNetValue").filter(col("ItemNetValue") > 0).show()
```

### Read with pandas (convert from Snowpark first)

pandas can't read a Snowflake table directly — you **convert** a Snowpark
DataFrame to pandas with `.to_pandas()`. That pulls the rows **into the notebook
runtime**, so do your filtering/aggregating in Snowpark *first* and only convert
the small result:

```python
# Aggregate on the warehouse, THEN bring the small result local:
pdf = by_org.to_pandas()        # Snowpark ➜ pandas (rows come to the runtime)

print(type(pdf))                # <class 'pandas.core.frame.DataFrame'>
pdf.head()                      # now use any pandas / matplotlib / ML library
```

> ⚠️ `to_pandas()` on a raw `session.table("gold.Sales")` would drag the **whole
> table** into memory. Push the work down first, convert last.

### Save either one back to a table

```python
# Snowpark DataFrame ➜ table (pushed down, never leaves the warehouse):
by_org.write.save_as_table("gold.Sales_By_Org", mode="overwrite")

# pandas DataFrame ➜ table (wrap it back into Snowpark, then write):
session.create_dataframe(pdf).write.save_as_table("gold.Sales_By_Org", mode="overwrite")
```

The round trip is the pattern: **read with Snowpark → (optional) convert to
pandas for libraries Snowflake doesn't have → wrap back with
`create_dataframe` → `save_as_table`.**

### What about external tables?

`save_as_table` writes a **native** Snowflake table — data lives inside the
account, fast to query. That's different from the **external tables** (`ext.*`)
you built earlier, which only *read* Parquet in place on the Azure stage and
never copy it in. Rule of thumb:

| | External table (`ext.*`) | `save_as_table` (native) |
|--|--------------------------|---------------------------|
| Data location | stays on Azure Blob | inside Snowflake |
| Direction | **read-only** over the stage | **write** the result of a transform |
| Use it for | querying source Parquet without loading | persisting a cleaned/aggregated result |

So a typical Snowpark flow reads from an **external table** over the stage,
transforms, and lands a **native** table with `save_as_table` — exactly the
medallion hop from raw to bronze/silver/gold.

---

## Land an Excel file from the stage

CSV and Parquet load with `COPY INTO`. **Excel isn't a native load format** —
so the source files on the stage (`SAMPLE01.xlsx`, `POC_EXCEL_DATA_3.xlsx`)
can't be `COPY`'d. This is the classic case for Snowpark: read the workbook with
pandas, then write it into a table like any other source.

```
@stage_azure/SAMPLE01.xlsx ──► session.file.get ──► pd.read_excel ──► save_as_table ──► bronze.SAMPLE01
   (Excel on Azure Blob)        (down to runtime)    (parse sheet)     (land to bronze)   (now SQL-queryable)
```

```python
import pandas as pd

# 1. Pull the staged .xlsx into the notebook's local runtime:
session.file.get("@raw_azure.stage_azure/SAMPLE01.xlsx", "/tmp/")

# 2. Parse it with pandas (needs openpyxl for .xlsx):
pdf = pd.read_excel("/tmp/SAMPLE01.xlsx", sheet_name="Sheet1")

# 3. Land it as a real Snowflake table — into the bronze (raw) layer:
session.create_dataframe(pdf).write.save_as_table(
    "bronze.SAMPLE01",
    mode="overwrite",
)
```

```sql
-- It's now part of the warehouse, queryable and joinable:
SELECT * FROM bronze.SAMPLE01 LIMIT 10;
```

### Clean & conform it — the engineer's next move

Landing is **bronze**. The same medallion discipline applies: clean the raw
Excel, conform the names, promote to **silver**. You can do it in Snowpark…

```python
clean = (
    session.table("bronze.SAMPLE01")
    .drop_duplicates()                          # clean: dedupe
    .filter(col("AMOUNT").is_not_null())        # clean: drop bad rows
    .with_column_renamed("AMOUNT", "NET_AMOUNT")  # conform: standard name
)
clean.write.save_as_table("silver.SAMPLE01", mode="overwrite")
```

…or in a SQL cell — pick whichever reads clearer for the transform.

> 💡 Multiple sheets, merged cells, and stray header rows make Excel loads
> fiddly — `sheet_name`, `skiprows`, and `header` on `pd.read_excel` are your
> levers. **Why** Excel is this painful as a data source is the next lesson.

---

## Recap

The same data-engineer loop — **land → clean → conform → write** — now in Python:

| Step | Snowpark | SQL equivalent |
|------|----------|----------------|
| Read a table (Spark-style) | `session.table("gold.Sales")` | `SELECT ... FROM` |
| Transform | `.filter().group_by().agg()` | `WHERE` / `GROUP BY` / `SUM` |
| Run it | `.show()` / `.collect()` / `.to_pandas()` | (query executes) |
| To pandas | `df.to_pandas()` (pulls rows local — convert *after* aggregating) | — |
| Back from pandas | `session.create_dataframe(pdf)` | — |
| Write back (native) | `.write.save_as_table("gold.X", mode="overwrite")` | `CREATE OR REPLACE TABLE AS` |
| Read-only over stage | `ext.*` **external table** | `CREATE EXTERNAL TABLE` |
| Load Excel | `file.get` → `pd.read_excel` → `save_as_table` | *(not possible — no native Excel load)* |

- **Notebooks** mix SQL + Python on `WORKSHOP_WH`, results inline.
- **Snowpark** is DataFrame-style Python **pushed down** to Snowflake — same
  compute and governance as your SQL.
- **Excel** is the reason Snowpark earns its place in the pipeline: stage it,
  `session.file.get`, `pd.read_excel`, then `save_as_table` into **bronze**, and
  clean it up into **silver**.

[← Back to Home](index.md) | [Next: Snowpark, end to end →](05-2-snowpark-demo.md)