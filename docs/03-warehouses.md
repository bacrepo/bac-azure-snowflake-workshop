---
title: "Lesson 02 — Warehouses (compute)"
---

# Lesson 02 — Warehouses (compute)

**Goal:** create the compute that runs your queries, and learn to control its
cost.

[← Previous: Architecture](02-architecture.md) · [Home](index.md) · [Next: Databases →](04-databases.md)

---

## What is a warehouse?

A **virtual warehouse** is a cluster of compute. Queries need one to run.
Warehouses come in **t-shirt sizes** — `XSMALL`, `SMALL`, `MEDIUM`, … — and each
size up roughly **doubles** the compute (and the cost per second).

You only pay while a warehouse is **running**. Suspend it and the meter stops.

## Create one

```sql
CREATE WAREHOUSE IF NOT EXISTS LEARN_WH
  WAREHOUSE_SIZE = 'XSMALL'   -- smallest & cheapest; perfect for learning
  AUTO_SUSPEND   = 60         -- pause after 60 seconds of inactivity
  AUTO_RESUME    = TRUE       -- wake up automatically on the next query
  INITIALLY_SUSPENDED = TRUE; -- don't start (or bill) until first use
```

`AUTO_SUSPEND` + `AUTO_RESUME` are the two settings that keep costs sane: the
warehouse sleeps when idle and wakes itself when you run something.

## Use it

```sql
USE WAREHOUSE LEARN_WH;

-- Now this query has compute to run on:
SELECT 'compute is on!' AS status;
```

## Manage it

```sql
-- Resize on the fly (takes effect on the next query):
ALTER WAREHOUSE LEARN_WH SET WAREHOUSE_SIZE = 'SMALL';

-- Manually pause / resume:
ALTER WAREHOUSE LEARN_WH SUSPEND;
ALTER WAREHOUSE LEARN_WH RESUME;

-- See all warehouses and their state:
SHOW WAREHOUSES;
```

> 💡 **Cost habit:** small warehouse + short auto-suspend. For learning, you
> almost never need bigger than `XSMALL`.

---

## Recap

- A **warehouse** is compute; queries can't run without one.
- **Size** trades speed for cost (each step ≈ 2×).
- `AUTO_SUSPEND` / `AUTO_RESUME` stop you from paying for idle compute.

[← Previous: Architecture](02-architecture.md) · [Home](index.md) · [Next: Databases →](04-databases.md)
