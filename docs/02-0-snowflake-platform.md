---
title: "The Snowflake platform"
---

# The Snowflake platform

**Goal:** understand the architecture, editions, compute, costing, and
governance that make Snowflake different — then see the recommended way to
bring data in.

[← Back to Home](index.md)

---

## 1 — Architecture: storage and compute are separate

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

Because the layers are independent, you can spin up **multiple warehouses** of
different sizes that all read the **same data** — without copying anything.

---

## 2 — Editions: Standard, Enterprise & Business Critical

Before diving into compute and cost, it helps to know that Snowflake comes in
three editions. The edition you choose determines both the **features available**
and the **price per credit**.

**Standard** covers most workloads — full SQL, auto-scaling, Time Travel
(1 day), and role-based access control.

**Enterprise** adds:

- **Time Travel up to 90 days** — query or restore data as it was up to 90 days ago
- **Multi-cluster warehouses** — auto-scale out to handle concurrency spikes
- **Materialized views** — pre-computed results that Snowflake keeps in sync
- **Dynamic data masking & row access policies** — column- and row-level security
- **Search optimization** — speeds up point-lookup queries on large tables
- **Object tagging & access history** — governance and compliance tracking

**Business Critical** adds everything in Enterprise, plus:

- **Data encryption everywhere** — including data at rest with customer-managed keys (Tri-Secret Secure)
- **Private connectivity** — Azure Private Link so traffic never touches the public internet
- **Database failover & replication** — cross-region/cross-cloud disaster recovery
- **HIPAA & PCI DSS compliance** — required for regulated industries (healthcare, finance)
- **Enhanced security hardening** — additional protections for the most sensitive workloads

> 💡 **Which edition to pick?** Start with **Standard** for learning and
> development. Move to **Enterprise** when you need 90-day Time Travel,
> masking policies, or multi-cluster warehouses. Choose **Business Critical**
> only when compliance or private connectivity is a hard requirement.

---

## 3 — Compute: virtual warehouses

A **warehouse** is a cluster of compute nodes. Every query needs one to run.

### Sizing

Each size step **doubles** the nodes — and the credit burn:

| Size | Nodes | Credits / hour |
|------|------:|---------------:|
| `XSMALL` | 1 | 1 |
| `SMALL` | 2 | 2 |
| `MEDIUM` | 4 | 4 |
| `LARGE` | 8 | 8 |
| `XLARGE` | 16 | 16 |
| `2XLARGE` | 32 | 32 |
| `3XLARGE` | 64 | 64 |
| `4XLARGE` | 128 | 128 |

### Try it — create, resize, suspend

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

---

## 4 — Costing: you pay for what runs

Now that you know warehouses and sizes, here's how the bill works.

### Credits and pricing

You're billed in **credits**, only while a warehouse is **running**.
The dollar price per credit depends on your edition. List prices on
**Azure Southeast Asia (Singapore)**:

| Edition | ~USD / credit |
|---------|-------------:|
| Standard | $2.50 |
| Enterprise | $3.70 |
| Business Critical | $5.00 |

> So an `XSMALL` warehouse on Standard costs roughly **$2.50 / hour** while
> running. A `MEDIUM` (4 credits/hr) ≈ **$10 / hour**.

### Billing mechanics

- **Minimum charge:** 1 minute when a warehouse starts, then billed
  per-**second** after that.
- `AUTO_SUSPEND` + `AUTO_RESUME` are the two settings that keep costs sane.
- **Storage** is billed separately — roughly **$23 / TB / month** (compressed)
  on Azure.

### Quick cost example

| Scenario | Size | Run time | Credits | ~Cost (Std) |
|----------|------|----------|--------:|------------:|
| Workshop queries | XSMALL | 30 min | 0.5 | $1.25 |
| Daily ETL job | SMALL | 15 min | 0.5 | $1.25 |
| Heavy transform | MEDIUM | 1 hour | 4 | $10.00 |
| Idle (suspended) | any | all day | 0 | $0.00 |

```sql
-- Where did the credits go? (needs ACCOUNTADMIN or a granted role)
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
ORDER BY start_time DESC
LIMIT 20;
```

> 💡 **Cost habit:** small warehouse + short auto-suspend. For learning you
> almost never need bigger than `XSMALL`.

---

## 5 — Governance: who can do what

Access in Snowflake is **role-based**: privileges are granted to roles, and
roles are granted to users.

```sql
SELECT CURRENT_ROLE();
SHOW ROLES;

-- Switch roles to change what you're allowed to do:
USE ROLE SYSADMIN;
```

Other governance building blocks you'll meet in
[Lesson 08](08-1-governance.md): **masking policies** (hide sensitive
columns), **row access policies**, **tags**, and **access history**.

---

## Recap

| # | Topic | Key takeaway |
|---|-------|--------------|
| 1 | **Architecture** | Three layers — storage, compute, cloud services — scaling independently |
| 2 | **Editions** | Standard → Enterprise → Business Critical; each adds governance & security features |
| 3 | **Compute** | Warehouses are resizable clusters; each size step = 2× nodes and credits |
| 4 | **Costing** | Credits accrue only while a warehouse runs; auto-suspend keeps the bill low |
| 5 | **Governance** | Privileges live on roles, which are granted to users |

[← Back to Home](index.md) | [Next: Stages →](02-1-stages.md)
