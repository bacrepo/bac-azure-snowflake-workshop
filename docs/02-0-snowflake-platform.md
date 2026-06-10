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

Other governance building blocks you'll meet later: **masking policies** (hide
sensitive columns), **row access policies**, **tags**, and **access history**.

---

## 6 — Best practice: land in cloud storage, then let Snowflake take over

With the platform understood, one key architectural decision remains: **how
should data get into Snowflake?**

A common mistake is piping data **directly** from source systems into Snowflake.
The recommended pattern is to **land files first** on your cloud platform's
native object storage, then have Snowflake read from there.

```
  Source Systems
       │
       ▼
 ┌───────────────────────────────────────────────┐
 │  Cloud Object Storage  (the "landing zone")   │
 │                                                │
 │  Azure → ADLS Gen2 / Blob Storage             │
 │  AWS   → S3                                   │
 │  GCP   → GCS                                  │
 └───────────────┬───────────────────────────────┘
                 │  External Stage
                 ▼
 ┌───────────────────────────────────────────────┐
 │            Snowflake                           │
 │                                                │
 │  COPY INTO  /  Snowpipe  /  Snowpipe Streaming │
 │        ↓                                       │
 │  Bronze → Silver → Gold                        │
 └───────────────────────────────────────────────┘
```

### Why land first, load second?

| Benefit | Explanation |
|---------|-------------|
| **Decouple ingestion from transformation** | Source systems push files on their own schedule; Snowflake picks them up independently — no tight coupling |
| **Replay & reprocess** | Raw files stay in storage; if a load fails or logic changes, just re-ingest — no need to ask the source again |
| **Lower cost** | Object storage is far cheaper than Snowflake credits; keep the warehouse off until data is ready |
| **Platform-native tooling** | Use ADF (Azure), Glue (AWS), or Dataflow (GCP) for extraction — tools built for moving data at scale |
| **Security boundary** | Landing zone can sit behind its own network/IAM policies before Snowflake ever touches the data |

### How it works on Azure (this workshop)

1. **Land** — Azure Data Factory (or any tool) drops Parquet / CSV files into an **ADLS Gen2** container.
2. **Stage** — Snowflake's **External Stage** points to that container using a **Storage Integration** (service-principal auth, no secrets in SQL).
3. **Load** — `COPY INTO` (batch) or **Snowpipe** (auto, event-driven) brings data into Snowflake tables.

```sql
-- Example: external stage pointing to ADLS
CREATE OR REPLACE STAGE landing_stage
  URL = 'azure://myaccount.blob.core.windows.net/landing/'
  STORAGE_INTEGRATION = azure_int;

-- Batch load from the landing zone
COPY INTO raw.sales
  FROM @landing_stage/sales/
  FILE_FORMAT = (TYPE = PARQUET)
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```

> 💡 **Rule of thumb:** let each platform do what it does best — cloud
> services move data, Snowflake transforms and serves it. Don't use Snowflake
> as an ingestion engine when your cloud platform already has one.

---

## 7 — Medallion Architecture: organize data in layers

The landing-zone pattern above naturally extends into the **Medallion
Architecture** — a widely adopted way to organize data as it moves from raw
ingestion to business-ready analytics.

The idea is simple: data flows through **progressive layers**, getting cleaner
and more reliable at each step.

```
 Cloud Storage          Snowflake
 ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
 │  FILES   │───▶│  BRONZE  │───▶│  SILVER  │───▶│   GOLD   │
 │ (landing │    │ (as-is   │    │ (cleaned,│    │ (business│
 │  zone)   │    │  copy)   │    │  joined) │    │  ready)  │
 └──────────┘    └──────────┘    └──────────┘    └──────────┘
   ADLS/S3/GCS     COPY INTO       transforms      aggregates,
                   Snowpipe        & standards      models, KPIs
```

### The layers

| Layer | What happens | Data quality |
|-------|-------------|--------------|
| **Files in cloud storage** | Source systems land raw files (CSV, Parquet, JSON) in ADLS / S3 / GCS | Untouched — exactly as the source sent it |
| **Bronze** | `COPY INTO` or Snowpipe loads files as-is into Snowflake tables | Raw copy — schema matches the source, no transformation yet |
| **Silver** | Cleansing, deduplication, type casting, standardizing column names, joining reference data | Clean & consistent — ready for analysts to explore |
| **Gold** | Aggregations, business logic, KPIs, dimensional models, reporting views | Business-ready — trusted numbers for dashboards and decisions |

### Many names, same idea

Different teams call these layers different things. They all mean the same
progression from raw to refined:

| Layer | Medallion | Also called | Also called | Also called |
|-------|-----------|-------------|-------------|-------------|
| Files from source | **Stage/External** | RAW / STAGE | LANDING | SOURCE |
| After COPY INTO Snowflake | **Bronze** | STAGING | RAW | INGESTED |
| After cleansing / standardize | **Silver** | CURATED | PROCESSED | CONFORMED |
| Ready to use | **Gold** | ANALYTICS | CURATED | PRESENTATION |

> Don't get hung up on naming. Pick one convention for your project and stick
> with it. What matters is the **progression**: raw → clean → ready.

### Example: Snowflake schemas as layers

A common pattern is to use **one schema per layer** inside the same database:

```sql
-- One schema per medallion layer
CREATE SCHEMA IF NOT EXISTS bronze;   -- raw loads land here
CREATE SCHEMA IF NOT EXISTS silver;   -- cleaned & standardized
CREATE SCHEMA IF NOT EXISTS gold;     -- business-ready

-- Bronze: raw copy from landing zone
COPY INTO bronze.sales
  FROM @landing_stage/sales/
  FILE_FORMAT = (TYPE = PARQUET)
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;

-- Silver: clean and standardize
CREATE OR REPLACE TABLE silver.sales AS
SELECT
    sale_id,
    TRIM(customer_name)              AS customer_name,
    sale_amount::NUMBER(12,2)        AS sale_amount,
    TO_DATE(sale_date, 'YYYY-MM-DD') AS sale_date
FROM bronze.sales
WHERE sale_id IS NOT NULL;

-- Gold: aggregate for reporting
CREATE OR REPLACE TABLE gold.daily_sales AS
SELECT
    sale_date,
    COUNT(*)        AS total_orders,
    SUM(sale_amount) AS total_revenue
FROM silver.sales
GROUP BY sale_date;
```

> 💡 **Why separate schemas?** Each layer can have its own access roles —
> data engineers own Bronze, analysts read Silver, dashboards query Gold.
> This keeps governance clean and prevents accidental writes to curated data.

---

## Recap

| # | Topic | Key takeaway |
|---|-------|--------------|
| 1 | **Architecture** | Three layers — storage, compute, cloud services — scaling independently |
| 2 | **Editions** | Standard → Enterprise → Business Critical; each adds governance & security features |
| 3 | **Compute** | Warehouses are resizable clusters; each size step = 2× nodes and credits |
| 4 | **Costing** | Credits accrue only while a warehouse runs; auto-suspend keeps the bill low |
| 5 | **Governance** | Privileges live on roles, which are granted to users |
| 6 | **Best practice** | Land data in cloud storage (ADLS / S3 / GCS) first, then load into Snowflake |
| 7 | **Medallion Architecture** | Organize data in progressive layers — Bronze (raw) → Silver (clean) → Gold (ready) |

| # | Topic | Key takeaway |
|---|-------|--------------|
| 1 | **Architecture** | Three layers — storage, compute, cloud services — scaling independently |
| 2 | **Editions** | Standard → Enterprise → Business Critical; each adds governance & security features |
| 3 | **Compute** | Warehouses are resizable clusters; each size step = 2× nodes and credits |
| 4 | **Costing** | Credits accrue only while a warehouse runs; auto-suspend keeps the bill low |
| 5 | **Governance** | Privileges live on roles, which are granted to users |
| 6 | **Best practice** | Land data in cloud storage (ADLS / S3 / GCS) first, then load into Snowflake |

[← Back to Home](index.md)
