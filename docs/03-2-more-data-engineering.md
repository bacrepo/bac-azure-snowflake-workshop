---
title: "Lesson 05 — Data engineering II"
---

# Lesson 05 — Data engineering II

[← Back to Home](index.md)

---

## Dynamic tables — auto-refreshing gold layer

In the previous workshop we built `gold.Sales` as a regular table.
The problem: when source data changes, `gold.Sales` goes stale.

A **dynamic table** solves this — Snowflake keeps it fresh automatically
within your `TARGET_LAG`. No scheduling, no manual refresh.

---

## What amounts do we have?

Before aggregating, understand what's available:

| Table | Amount column | What it is |
|-------|-------------|-----------|
| `silver.VBAK` | `NetValueHeader` | Total net value of the **entire order** (header level) |
| `silver.VBAP` | `ItemNetValue` | Net value per **item line** (detail level) |
| `silver.VBEP` | *(no amount)* | Schedule lines have **quantities**, not amounts |

> Use `ItemNetValue` from VBAP for detail-level aggregation — it sums up
> correctly per item. `NetValueHeader` is the order total (header).

---

## Create the dynamic table

`gold.SalesSummary` — monthly sales amount by item, auto-refreshed.

We source from `gold.Sales` which already joins VBAK + VBAP + VBEP.
We aggregate `ItemNetValue` (detail amount from VBAP) at item grain
to avoid double-counting across schedule lines.

```sql
USE SCHEMA gold;

CREATE OR REPLACE DYNAMIC TABLE gold.SalesSummary
  TARGET_LAG = '1 minute'
  WAREHOUSE  = WORKSHOP_WH
AS
SELECT
    TO_CHAR(DocumentDate, 'YYYYMM')  AS SalesMonth,
    SalesOrg,
    DocumentCurrency,
    COUNT(DISTINCT SalesDocument)     AS OrderCount,
    SUM(DISTINCT ItemNetValue)        AS TotalItemNetValue
FROM gold.Sales
WHERE DocumentDate IS NOT NULL
GROUP BY
    TO_CHAR(DocumentDate, 'YYYYMM'),
    SalesOrg,
    DocumentCurrency;
```

### Verify

```sql
SELECT * FROM gold.SalesSummary ORDER BY SalesMonth DESC LIMIT 20;
```

### Check refresh status

```sql
-- When was the last refresh?
SELECT * FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLE_REFRESH_HISTORY(
  NAME => 'USER00_DB.GOLD.SALESSUMMARY'
)) ORDER BY REFRESH_START_TIME DESC LIMIT 5;
```

---

## What TARGET_LAG means

| Setting | Behaviour |
|---------|-----------|
| `'1 minute'` | Snowflake checks every ~1 min if source data changed; refreshes if yes |
| `'10 minutes'` | Cheaper — checks less often, data can be up to 10 min stale |
| `DOWNSTREAM` | Only refresh when a downstream dynamic table needs it |

> For a workshop, `1 minute` is fine. In production, pick the lag the
> business can tolerate — shorter = more compute cost.

---

## Clean up

```sql
-- Stop the auto-refresh and drop
DROP DYNAMIC TABLE IF EXISTS gold.SalesSummary;
```

---

## Recap

| What | Why |
|------|-----|
| `gold.Sales` (regular table) | Point-in-time snapshot — manual rebuild |
| `gold.SalesSummary` (dynamic table) | Auto-refreshed — always fresh within `TARGET_LAG` |
| `ItemNetValue` (VBAP) | Detail amount per item — use for aggregation |
| `NetValueHeader` (VBAK) | Header-level total — use for order-level reporting |
| `DocumentDate → SalesMonth (YYYYMM)` | `TO_CHAR(DocumentDate, 'YYYYMM')` for monthly grain |

[← Back to Home](index.md)
