---
title: "What is Snowflake?"
---

# What is Snowflake?

**Goal:** understand what Snowflake is, why it matters, and how it fits into a
modern Data + AI strategy.

[← Back to Home](index.md)

---

## In one sentence

Snowflake is a **cloud-native data platform** — built from scratch for the
cloud — that lets you store, engineer, share, and run AI on your data, all in
one place.

---

## It's SaaS — not software you install

Snowflake is delivered as **Software-as-a-Service (SaaS)**. There is nothing to
install, nothing to tune, and nothing to patch. You open a browser, sign in, and
start working — Snowflake handles all the infrastructure behind the scenes.

- **No servers to provision** — Snowflake runs the infrastructure.
- **No licenses to manage** — you don't buy seats or cores.
- **Automatic updates** — new features and patches roll out without downtime.
- **Pay for what you use** — credits (compute) + storage. The meter runs only
  while a warehouse is active.

---

## Why Snowflake is different

| Concept | What it means |
| --- | --- |
| **Storage and compute are separated** | Your data sits in cheap cloud storage; compute spins up only when needed. Ten teams can query the same data without slowing each other down. |
| **Zero administration** | No indexing, no vacuuming, no partitioning, no capacity planning. You focus on data and logic, not infrastructure. |
| **Any data, any structure** | Tables, JSON, Parquet, Avro, XML, CSV, Excel — stage it, and Snowflake reads it natively. |
| **One platform, many workloads** | Data warehousing, data engineering, data lake, data sharing, apps, and AI & ML — all in one place. |

---

## How Snowflake fits the Data + AI world

Traditional analytics was linear: **store → transform → report**. The modern
stack is a loop — and Snowflake aims to run the entire loop in one platform:

```
         ┌──────────────────────────────────────────────────┐
         │                                                  │
   Store ──▶ Engineer ──▶ Analyze ──▶ AI / ML ──▶ Act ─────┘
   (stages)  (SQL /       (BI /       (Cortex /   (agents /
              Snowpark)    queries)    notebooks)  Streamlit)
```

Snowflake covers the full loop — from ingestion (Snowpipe) through engineering
(SQL, Snowpark) to AI (Cortex) and apps (Streamlit). Each capability is covered
in detail in **[Snowflake features →](01-1-snowflake-features.md)**.

---

## How BAC pays for Snowflake

Snowflake is purchased through a **Cloud Private Offer** on the **Microsoft
Azure Marketplace**:

```
  Snowflake (SaaS)
       │
       ▼
  Azure Marketplace ──▶ Private Offer (negotiated rates)
       │
       ▼
  Your Azure invoice (consolidated billing)
```

- **Billing goes through Azure** — appears on your existing Azure invoice.
- **Committed-use pricing** — negotiated rates, typically discounted vs.
  on-demand.
- **Counts toward your MACC** — Microsoft Azure Consumption Commitment.
- **No separate procurement** — no new vendor contract; flows through your Azure
  agreement.

---

## Where Snowflake runs

Snowflake runs on **all three major clouds**:

| Cloud | Regions |
| --- | --- |
| **Microsoft Azure** | This is us — BAC runs on Azure |
| AWS | The most Snowflake regions |
| Google Cloud | Growing footprint |

In this workshop, everything runs on **Snowflake on Azure**. From your
perspective, it's just a URL in a browser.

---

## Recap

- Snowflake is a **cloud-native data platform** delivered as **pure SaaS**.
- **Storage and compute are separated** — cheap storage, elastic compute, zero
  admin.
- It has evolved into a full **Data + AI platform** — Cortex AI, Cortex Code,
  Intelligence, Agents, Snowpark, Notebooks, Streamlit, and Snowpipe.
- BAC pays via **Azure Marketplace Private Offer** — consolidated on your Azure
  invoice.

[← Back to Home](index.md) | [Next: Snowflake features →](01-1-snowflake-features.md)
