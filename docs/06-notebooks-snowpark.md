---
title: "Lesson 06 — Data engineering II"
---

# Lesson 06 — Data engineering II

**Goal:** work in **notebooks**, use **Snowpark** (Python on Snowflake), and
read an **Excel file straight from a stage**.

[← Back to Home](index.md)

---

## Notebooks in Snowflake

A **Snowflake Notebook** runs SQL *and* Python cells against your account, with
results inline. Create one in Snowsight: **Projects → Notebooks → + Notebook**.

- Mix SQL and Python in the same notebook.
- Each notebook runs on a **warehouse** you pick (same costing rules apply).
- Great for exploration, data prep, and prototyping pipelines.

## Snowpark — Python on Snowflake

**Snowpark** lets you work with Snowflake data as DataFrames in Python, with the
work pushed down to the engine. Inside a notebook you already have a `session`.

```python
from snowflake.snowpark.context import get_active_session
session = get_active_session()

df = session.table("LEARN_DB.WORKSHOP.SALES")
df.filter(df["AMOUNT"] > 1000) \
  .group_by("REGION") \
  .sum("AMOUNT") \
  .show()
```

## Read an Excel file from a stage

Excel isn't a native load format like CSV, so we read it with pandas via
Snowpark, then write it to a table.

```python
import pandas as pd

# 1. Pull the staged .xlsx down to the notebook's local runtime:
session.file.get("@my_stage/report.xlsx", "/tmp/")

# 2. Read it with pandas (needs openpyxl for .xlsx):
pdf = pd.read_excel("/tmp/report.xlsx", sheet_name="Sheet1")

# 3. Persist it as a real Snowflake table:
session.create_dataframe(pdf).write.save_as_table(
    "LEARN_DB.WORKSHOP.REPORT_FROM_EXCEL",
    mode="overwrite",
)
```

```sql
-- Now it's queryable like anything else:
SELECT * FROM LEARN_DB.WORKSHOP.REPORT_FROM_EXCEL LIMIT 10;
```

> 💡 Multiple sheets, merged cells, and stray headers make Excel loads fiddly —
> which is exactly the next lesson's topic.

---

## Recap

- **Notebooks** mix SQL + Python on a warehouse, results inline.
- **Snowpark** gives you DataFrame-style Python pushed down to Snowflake.
- Read **Excel** by staging it, `session.file.get`, `pd.read_excel`, then
  `save_as_table`.

[← Back to Home](index.md)
