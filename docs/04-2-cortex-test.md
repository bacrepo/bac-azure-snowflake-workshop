---
title: "DEMO 09 — Let Cortex Code build the table"
---

# DEMO 09 — Let Cortex Code build the table

**Goal:** instead of hand-writing the join, **ask Cortex Code** to pick the most
important columns from the two silver tables and build a gold table for us —
`gold.salescoco`. You describe the outcome; Cortex drafts the SQL.

[← Back to Home](index.md)

---

## The idea

In [DEMO08](04-1-summary-data.md) we hand-picked 12 columns and wrote the join
ourselves. Here we let the AI do the first draft. Same destination — a narrow,
business-ready table — but driven by a **plain-language prompt**.

```
  "give me the 12 most important columns from VBAP + VBAK"
            │
            ▼
   Cortex Code  ──▶  draft SQL  ──▶  you review & run  ──▶  gold.salescoco
```

---

## Step 1 — open Cortex Code

In Snowsight, open a **worksheet** (or a notebook **SQL cell**) and open the
**Cortex Code** assistant panel. Make sure your context is set:

```sql
USE DATABASE USERXX;
USE ROLE BAC_DE_ROLE;
USE WAREHOUSE WORKSHOP_WH;
CREATE SCHEMA IF NOT EXISTS gold;
```

---

## Step 2 — ask for it in plain language

Type this prompt into Cortex Code:

> **I want most 12 important columns from**
> **`silver.VBAP`**
> **`silver.VBAK`**
>
> **To Create/Replace `gold.salescoco`**

Cortex inspects both tables, decides which 12 columns matter, works out the
join key, and drafts a `CREATE OR REPLACE TABLE` for you.

---

## Step 3 — review the draft

Cortex will produce something close to this — **read it before you run it**:

```sql
CREATE OR REPLACE TABLE gold.salescoco AS
SELECT
    -- ── identifiers ──────────────────────────────────────────────
    H.SalesDocument                              AS SalesOrder,
    TO_DATE(H.DocumentDate::STRING, 'YYYYMMDD')  AS OrderDate,

    -- ── who / where ──────────────────────────────────────────────
    H.SoldToParty                                AS Customer,
    H.SalesOrg                                   AS SalesOrg,

    -- ── product ──────────────────────────────────────────────────
    I.Material                                   AS Product,
    I.ItemDescription                            AS ProductName,
    I.MaterialGroup                              AS ProductGroup,
    H.SalesDocumentType                          AS OrderType,

    -- ── measures ─────────────────────────────────────────────────
    CAST(I.OrderQuantity AS NUMBER(15,3))        AS Quantity,
    I.SalesUnit                                  AS SalesUnit,
    CAST(I.ItemNetValue  AS NUMBER(15,2))        AS NetAmount,
    H.DocumentCurrency                           AS Currency
FROM silver.VBAP AS I
INNER JOIN silver.VBAK AS H
    ON I.SalesDocument = H.SalesDocument;
```

> 💡 **Cortex drafts; you decide.** Your draft may differ — Cortex picks
> columns from what it sees in your schema. Check the **join key**, the **grain**
> (one row per order item), and the **casts** before running. If a date column
> is already a real `DATE`, drop the `TO_DATE()` wrapper.

---

## Step 4 — run and verify

```sql
SELECT * FROM gold.salescoco LIMIT 10;   -- peek
DESC TABLE gold.salescoco;               -- confirm the types
SELECT COUNT(*) AS rows FROM gold.salescoco;
```

---

## Step 5 — refine: only 2024 onward

You don't start over — just **follow up** in the same Cortex Code chat. It
remembers the table it just built. Type:

> **I want only Year 2024 onward**

Cortex adds a date filter to the build. Review the draft:

```sql
CREATE OR REPLACE TABLE gold.salescoco AS
SELECT
    H.SalesDocument                              AS SalesOrder,
    TO_DATE(H.DocumentDate::STRING, 'YYYYMMDD')  AS OrderDate,
    H.SoldToParty                                AS Customer,
    H.SalesOrg                                   AS SalesOrg,
    I.Material                                   AS Product,
    I.ItemDescription                            AS ProductName,
    I.MaterialGroup                              AS ProductGroup,
    H.SalesDocumentType                          AS OrderType,
    CAST(I.OrderQuantity AS NUMBER(15,3))        AS Quantity,
    I.SalesUnit                                  AS SalesUnit,
    CAST(I.ItemNetValue  AS NUMBER(15,2))        AS NetAmount,
    H.DocumentCurrency                           AS Currency
FROM silver.VBAP AS I
INNER JOIN silver.VBAK AS H
    ON I.SalesDocument = H.SalesDocument
WHERE TO_DATE(H.DocumentDate::STRING, 'YYYYMMDD') >= '2024-01-01';   -- ← new
```

> 💡 The only change is the `WHERE` clause. Cortex filters on the **order date**
> so the table now holds **2024 and later** only.

---

## Step 6 — ask a question

Now query the table in plain language. Type:

> **With `salescoco` table, find me customer who bought us the most net amount.**

Cortex drafts an aggregate query:

```sql
SELECT
    Customer,
    SUM(NetAmount) AS TotalNetAmount
FROM gold.salescoco
GROUP BY Customer
ORDER BY TotalNetAmount DESC
LIMIT 10;
```

> 💡 The customer at the **top row** is your biggest buyer by net amount. Drop
> the `LIMIT 10` to `LIMIT 1` if you only want the single top customer.

---

## DEMO08 by hand vs. DEMO09 with Cortex

| | DEMO08 — `agent.Sales` | DEMO09 — `gold.salescoco` |
|--|------------------------|----------------------------|
| Who writes the SQL | **you** | **Cortex Code** (first draft) |
| Column choice | you pick the 12 | Cortex proposes the 12 |
| Join key | you specify | Cortex infers, you confirm |
| Your job | type & run | **review** & run |

Same shape of result — a narrow, well-named, well-typed sales table. The
difference is **who drafts it**.

---

## Recap

- Cortex Code turns a **plain-language ask** into a `CREATE OR REPLACE TABLE`.
- It picks columns, infers the join, and casts types from what it sees in your
  schema.
- **Always review** the draft — check join key, grain, and casts before running.

**Next:** SQL and Cortex Code cover a lot — but some sources (like Excel)
need **Python**. That's where Notebooks and Snowpark come in.

[← Back to Home](index.md) | [Next: Notebooks & Snowpark →](05-1-notebooks-snowpark.md)
