---
title: "DEMO 08 — Agent-ready summary (Notebook)"
---

# DEMO 08 — Agent-ready summary (Notebook)

**Goal:** boil the wide `gold.Sales` table down to a **narrow, clean, well-typed**
table — `agent.Sales` — that **Cortex Analyst** and **Streamlit** can sit on top
of. We do it in a **Snowflake Notebook** using **SQL cells**.

[← Back to Home](index.md)

---

## Why summarise?

The **silver** tables (`VBAK` + `VBAP`) carry **50+ columns** between them, with
SAP-style names and Parquet-inherited types. Great for engineering — but an AI
agent reasons best over a **small, business-named, correctly-typed** table.

| | silver `VBAK` + `VBAP` | `agent.Sales` |
|--|------------------------|----------------|
| Columns | 50+ across two tables | **12** — only what matters |
| Names | `SalesDocument`, `ItemNetValue` | `SalesOrder`, `NetAmount` |
| Types | inherited from Parquet (dates as text, amounts as float) | **clean** `DATE` / `NUMBER` |
| Audience | data engineers | **Cortex Analyst & Streamlit** |

> Fewer, well-named, well-typed columns → the LLM picks the right field, the
> right metric, and the right filter far more reliably.

---

## Open a Notebook

In Snowsight: **Notebooks → + Notebook**. Set:

- **Database / schema:** `USERXX` (any schema — we create `agent` below)
- **Warehouse:** `WORKSHOP_WH`

Add each block below as its own **SQL cell** and run top to bottom.

---

## Cell 1 — context + create the `agent` schema

```sql
USE DATABASE USERXX;
USE ROLE BAC_DE_ROLE;
USE WAREHOUSE WORKSHOP_WH;
CREATE SCHEMA IF NOT EXISTS agent;
USE SCHEMA agent;
```

---

## Cell 2 — build `agent.Sales` (pick · rename · retype)

Join **silver** header (`VBAK`) + items (`VBAP`) at **item grain** — one row per
order line. Keep the **12 important columns**, give them **good names**, and
**cast** dates and numbers into clean types.

```sql
CREATE OR REPLACE TABLE agent.Sales AS
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

> **Grain:** one row per **order item** (`VBAP`). We skip `VBEP` (schedule lines)
> on purpose — joining it would repeat each item per schedule line and
> double-count `NetAmount`.

> **Date format:** SAP dates land as text like `20260115`.
> `TO_DATE(..., 'YYYYMMDD')` turns them into a real `DATE` so the agent can do
> "last month", "this quarter", etc. If your column is already a `DATE`, drop the
> `TO_DATE()` wrapper.

---

## Cell 3 — verify

```sql
SELECT * FROM agent.Sales LIMIT 10;   -- peek
DESC TABLE agent.Sales;               -- confirm the clean types
SELECT COUNT(*) AS rows FROM agent.Sales;
```

---

## What `agent.Sales` contains

| Column | Source (silver) | Type | Role |
|--------|-----------------|------|------|
| `SalesOrder` | `VBAK.SalesDocument` | VARCHAR | identifier |
| `OrderDate` | `VBAK.DocumentDate` | **DATE** | time dimension |
| `Customer` | `VBAK.SoldToParty` | VARCHAR | dimension |
| `SalesOrg` | `VBAK.SalesOrg` | VARCHAR | dimension |
| `Product` | `VBAP.Material` | VARCHAR | dimension |
| `ProductName` | `VBAP.ItemDescription` | VARCHAR | dimension |
| `ProductGroup` | `VBAP.MaterialGroup` | VARCHAR | dimension |
| `OrderType` | `VBAK.SalesDocumentType` | VARCHAR | dimension |
| `Quantity` | `VBAP.OrderQuantity` | **NUMBER(15,3)** | **measure** |
| `SalesUnit` | `VBAP.SalesUnit` | VARCHAR | dimension |
| `NetAmount` | `VBAP.ItemNetValue` | **NUMBER(15,2)** | **measure** |
| `Currency` | `VBAK.DocumentCurrency` | VARCHAR | dimension |

Two **measures** (`Quantity`, `NetAmount`), one **time** dimension (`OrderDate`),
and the rest are **dimensions** to slice by — exactly the shape Cortex Analyst
wants for a semantic model.

---

## Recap

```
   silver.VBAK  +  silver.VBAP   (50+ cols, raw SAP names/types)
        │
        │   SQL notebook:  join (item grain) · select 12 · rename · cast
        ▼
   agent.Sales  (12 cols, business names, clean DATE/NUMBER)
        │
        ▼
   Cortex Analyst  →  Streamlit app
```

| Step | Cell | What it does |
|------|------|-------------|
| Schema | `CREATE SCHEMA IF NOT EXISTS agent` | Home for the agent-facing table |
| Build | `CREATE TABLE agent.Sales AS SELECT ... FROM silver.VBAP JOIN silver.VBAK` | Join, pick 12, rename, cast types |
| Verify | `SELECT` / `DESC TABLE` | Confirm rows and clean types |

`agent.Sales` is the table the rest of the workshop builds on — the semantic
model in Lesson 07 and the Streamlit dashboard in Lesson 09 both sit on top
of it.

[← Back to Home](index.md) | [Next: Let Cortex Code build the table →](04-2-cortex-test.md)
