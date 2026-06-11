---
title: "DEMO 10 — Snowpark, end to end"
---

# DEMO 10 — Snowpark, end to end

**Goal:** in a notebook, **read `gold.Sales` with `session.sql`** (Snowpark's
`spark.sql`), add a couple of columns, and **land `gold.SalesDemo`** — the
read → transform → write loop, all on `WORKSHOP_WH`.

[← Back to Home](index.md)

Run these cells top-to-bottom in a **Snowflake Notebook**
(**Projects → Notebooks → + Notebook**), with the database/schema set to
`USERXX.gold` and the warehouse to `WORKSHOP_WH`.

---

## Step 0 — Context

A SQL cell to set the role and warehouse the data engineer works under.

```sql
USE ROLE BAC_DE_ROLE;
USE WAREHOUSE WORKSHOP_WH;
```

---

## Step 1 — Grab the session

Inside a notebook the connection already exists — get a handle to it.

```python
from snowflake.snowpark.context import get_active_session

session = get_active_session()
```

> In Snowpark, `session.sql("...")` is the equivalent of Spark's
> `spark.sql("...")` — pass it a SQL string, get back a lazy DataFrame that runs
> on `WORKSHOP_WH`.

---

## Step 2 — Read & shape with `session.sql`

One SQL string does the read **and** the transform. We:

- keep every existing column (`*`),
- add a literal column **`new_column`** with value `'hello'`,
- build **`CustomerDisplay`** by concatenating `CustomerID` and `CustomerName`
  (in Snowflake, strings concat with `||`, not `+`),
- filter to rows on or after **2023-01-01**.

```python
sales_demo = session.sql("""
    SELECT
        *,
        'hello'                            AS new_column,
        CustomerID || ' ' || CustomerName  AS CustomerDisplay
    FROM gold.Sales
    WHERE DocumentDate >= '2023-01-01'
""")

sales_demo.show()                 # runs on WORKSHOP_WH; only rows come back
print(sales_demo.count(), "rows")
```

> `session.sql(...)` is **lazy** — the query is a plan until an action
> (`.show()`, `.count()`, `.collect()`, `save_as_table`) triggers it.

---

## Step 2b — The same, the Spark-style DataFrame way

Don't want to write SQL? The **Snowpark DataFrame API** expresses the exact same
transform with methods — just like Spark. `lit` makes a literal column, `concat`
joins strings. Use whichever reads clearer; the result is identical.

```python
from snowflake.snowpark.functions import col, lit, concat

sales_demo = (
    session.table("gold.Sales")
    .filter(col("DocumentDate") >= "2023-01-01")                 # WHERE
    .with_column("new_column", lit("hello"))                     # 'hello' AS new_column
    .with_column(
        "CustomerDisplay",
        concat(col("CustomerID"), lit(" "), col("CustomerName")) # CustomerID || ' ' || CustomerName
    )
)

sales_demo.show()
```

> Same plan, same pushdown to `WORKSHOP_WH`. `session.sql` (Step 2) and the
> DataFrame API (here) are two ways to build the **same** `sales_demo` — pick
> one to feed Step 3.

---

## Step 3 — Save as `gold.SalesDemo`

`save_as_table` persists the result as a **native** table — like
`CREATE OR REPLACE TABLE ... AS`. The data never leaves the warehouse.

```python
sales_demo.write.save_as_table(
    "gold.SalesDemo",
    mode="overwrite",        # replace if it already exists
)
```

---

## Step 4 — Verify

A SQL cell — the table written from Python is queryable like anything else.
Check the two new columns landed:

```sql
SELECT CustomerDisplay, new_column, DocumentDate
FROM gold.SalesDemo
ORDER BY DocumentDate
LIMIT 20;
```

```sql
-- Confirm it's a real, native table in the schema:
SHOW TABLES LIKE 'SalesDemo' IN SCHEMA gold;
```

---

## (Optional) Step 5 — Hand off to pandas

Need a chart or a Python library? Convert the result to pandas.

```python
pdf = sales_demo.to_pandas()     # Snowpark ➜ pandas (rows come local)
pdf.head()
```

---

## Clean up

```sql
DROP TABLE IF EXISTS gold.SalesDemo;
```

---

## Recap

```
gold.Sales  ──►  session.sql(SELECT …)  ──►  + new_column / CustomerDisplay  ──►  save_as_table()  ──►  gold.SalesDemo
(joined         (read, spark.sql style,      WHERE DocumentDate >= 2023-01-01     (write native)        (query in SQL)
 source)         lazy on WORKSHOP_WH)
```

| Step | Snowpark |
|------|----------|
| Context | `USE ROLE BAC_DE_ROLE; USE WAREHOUSE WORKSHOP_WH;` |
| Session | `get_active_session()` |
| Read + shape (SQL) | `session.sql("SELECT *, 'hello' AS new_column, CustomerID \|\| ' ' \|\| CustomerName AS CustomerDisplay FROM gold.Sales WHERE DocumentDate >= '2023-01-01'")` |
| Read + shape (Spark API) | `session.table("gold.Sales").filter(col("DocumentDate") >= "2023-01-01").with_column("new_column", lit("hello")).with_column("CustomerDisplay", concat(col("CustomerID"), lit(" "), col("CustomerName")))` |
| Write | `.write.save_as_table("gold.SalesDemo", mode="overwrite")` |
| Verify | `SELECT * FROM gold.SalesDemo` |
| To pandas | `sales_demo.to_pandas()` |

[← Back to Home](index.md) | [Next: Why Excel is difficult →](06-1-why-excel.md)
