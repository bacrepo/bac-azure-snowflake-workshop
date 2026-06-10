---
title: "Lesson 02 — The Snowflake platform"
---

# Lesson 02 — The Snowflake platform

**Goal:** understand the three things that make Snowflake different —
**compute**, **costing**, and **governance**.

[← Back to Home](index.md)

---

## The big idea: storage and compute are separate

Traditional databases bolt storage and compute together. Snowflake splits them
into **three independent layers**:

```
┌─────────────────────────────────────────────┐
│  Cloud Services                              │  ← brains: auth, optimizer,
│  (security, metadata, query optimization)    │     metadata, transactions
├─────────────────────────────────────────────┤
│  Compute  (Virtual Warehouses)               │  ← muscle: runs your queries.
│  [ WH_A ] [ WH_B ] [ WH_C ] ...              │     Many, independent, resizable
├─────────────────────────────────────────────┤
│  Storage  (your data, on Azure Blob)         │  ← one shared copy of the data
└─────────────────────────────────────────────┘
```

On this account, storage is **Azure Blob Storage** and compute runs on **Azure
VMs** — but the SQL is identical across clouds.

## Compute — virtual warehouses

A **warehouse** is a cluster of compute. Queries need one to run. Sizes go
`XSMALL → SMALL → MEDIUM → …`, each step roughly **2×** the power and cost.

```sql
CREATE WAREHOUSE IF NOT EXISTS LEARN_WH
  WAREHOUSE_SIZE = 'XSMALL'    -- smallest & cheapest; perfect for learning
  AUTO_SUSPEND   = 60          -- pause after 60s idle (the meter stops)
  AUTO_RESUME    = TRUE        -- wake up on the next query
  INITIALLY_SUSPENDED = TRUE;  -- don't start (or bill) until first use

USE WAREHOUSE LEARN_WH;
```

```sql
-- Resize on the fly, pause/resume, and inspect:
ALTER WAREHOUSE LEARN_WH SET WAREHOUSE_SIZE = 'SMALL';
ALTER WAREHOUSE LEARN_WH SUSPEND;
ALTER WAREHOUSE LEARN_WH RESUME;
SHOW WAREHOUSES;
```

## Costing — you pay for what runs

- You're billed in **credits**, only while a warehouse is **running**.
- `AUTO_SUSPEND` + `AUTO_RESUME` are the two settings that keep costs sane.
- Storage is billed separately and cheaply (compressed).

```sql
-- Where did the credits go? (needs ACCOUNTADMIN or a granted role)
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
ORDER BY start_time DESC
LIMIT 20;
```

> 💡 **Cost habit:** small warehouse + short auto-suspend. For learning you
> almost never need bigger than `XSMALL`.

## Governance — who can do what

Access in Snowflake is **role-based**: privileges are granted to roles, and
roles are granted to users.

```sql
SELECT CURRENT_ROLE();
SHOW ROLES;

-- Switch roles to change what you're allowed to do:
USE ROLE SYSADMIN;
```

Other governance building blocks you'll meet later: **masking policies** (hide
sensitive columns), **row access policies**, **tags**, and **access history**.

---

## Recap

- Three layers: **storage**, **compute (warehouses)**, **cloud services** —
  storage and compute scale **independently**.
- **Costing:** credits accrue only while a warehouse runs; auto-suspend protects
  you.
- **Governance:** privileges live on **roles**, which are granted to users.

[← Back to Home](index.md)
