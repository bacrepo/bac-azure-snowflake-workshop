---
title: BAC Azure + Snowflake Workshop
---

# ❄️ BAC Azure + Snowflake Workshop

A hands-on workshop for the BAC team: moving data off SAP / Excel / SharePoint
into **Snowflake** on **Microsoft Azure**, engineering it, and putting AI and
apps on top. Each lesson is short and ends with something you can run yourself.

> **How to use this:** read a lesson top-to-bottom, then copy each snippet into
> [Snowsight](https://app.snowflake.com) (Snowflake's web UI) and run it.
> Learning sticks when you type the commands yourself.

**The shape of the day:** data engineering **first**, AI **on top**. The chat
agent and dashboard you build in the afternoon are only trustworthy because of
the pipeline you build in the morning — that ordering is the biggest lesson of
the workshop.

```
  SAP / Excel / SharePoint
     └▶ stage ─▶ bronze ─▶ silver ─▶ gold / agent.Sales     ← morning: engineering
                                       ├▶ Snowflake Intelligence (chat)   ← afternoon:
                                       └▶ Streamlit dashboard  (apps)        AI on top
```

---

## Morning — foundations & data engineering

| #  | Lesson | What you'll learn |
| -- | ------ | ----------------- |
| 01-0 | [What is Snowflake?](01-0-whatissnowflake.md) | Cloud-native SaaS data platform, Data + AI strategy |
| 01-1 | [Snowflake features](01-1-snowflake-features.md) | Technical deep dive — Snowpipe, Cortex, Snowpark, Streamlit |
| 01-2 | [BAC solutions overview](01-2-solutions-overview.md) | The POC: SAP → Excel / SharePoint → Snowflake on Azure |
| 01-3 | [Setup & first query](01-3-snowflake-ui.md) | Sign in to Snowsight, run your first query |
| 02-0 | [The Snowflake platform](02-0-snowflake-platform.md) | Compute (warehouses), costing, and governance |
| 02-1 | [Stages](02-1-stages.md) | Landing zones, internal vs. external stages, medallion architecture |
| 02-2 | [Stages demo](02-2-stages-demo.md) | Hands-on: load CSV & Parquet via internal stage |
| 02-3 | [External stage](02-3-external-stage.md) | Hands-on: connect Azure ADLS, load from external stage |
| 03-0 | [Data engineering](03-0-data-engineering.md) | The engineer's loop: land → clean → conform → serve |
| 03-1 | [Data engineering workshop](03-1-data-engineering-workshop.md) | Hands-on: SAP files → bronze → silver → `gold.Sales` |
| 03-2 | [Data engineering II](03-2-more-data-engineering.md) | Hands-on: dynamic tables — a self-refreshing gold layer |
| 03-3 | [Incremental data](03-3-incremental-data.md) | Hands-on: apply deltas with `DELETE` + `MERGE` / `INSERT` |
| 03-4 | [Tasks & scheduling](03-4-task-and-schedule.md) | Hands-on: procedures, tasks, and task graphs — pipelines that run themselves |

## Afternoon — AI on top

| #  | Lesson | What you'll learn |
| -- | ------ | ----------------- |
| 04-0 | [Cortex Code](04-0-cortex-code.md) | Snowflake's AI assistant for writing SQL & code |
| 04-1 | [Agent-ready summary](04-1-summary-data.md) | Hands-on: build `agent.Sales` — the narrow, clean table AI reasons over |
| 04-2 | [Let Cortex Code build the table](04-2-cortex-test.md) | Hands-on: same table, AI-drafted — you review, refine, verify |
| 05-1 | [Notebooks & Snowpark](05-1-notebooks-snowpark.md) | Python for data engineers — DataFrames pushed down to the warehouse |
| 05-2 | [Snowpark, end to end](05-2-snowpark-demo.md) | Hands-on: read → transform → write, `session.sql` & DataFrame API |
| 06-1 | [Why Excel is difficult](06-1-why-excel.md) | The hidden costs of spreadsheet-driven processes |
| 06-2 | [Excel is more difficult than anything](06-2-excel-is-difficult-than-anything.md) | Hands-on: load `SAMPLE01.xlsx` and feel every pain point |
| 07-1 | [Snowflake Intelligence](07-1-snowflake-intelligence.md) | Cortex Analyst, semantic models — why they make or break chat-with-data; pricing |
| 07-2 | [Snowflake Intelligence hands-on](07-2-snowflake-intelligence-hands-on.md) | Hands-on: semantic model on `agent.Sales`, then chat in Thai & English |
| 08-1 | [Governance](08-1-governance.md) | Horizon — row/column policies at platform level, data catalog, lineage |
| 08-2 | [Governance hands-on](08-2-governance-hands-on.md) | Hands-on: mask & filter `gold.Sales`, run as the read-only `BAC_DA_ROLE` |
| 09-1 | [Streamlit introduction](09-1-streamlit-introduction.md) | Data apps in pure Python — no hosting, no auth code, RBAC built in |
| 09-2 | [Streamlit hands-on](09-2-streamlit-hands-on.md) | Hands-on: AI-drafted sales dashboard on `agent.Sales` from one good prompt |

---

## Before you start

You'll need:

- A **Snowflake account** on Azure — sign in at
  [https://tknjgnq-xc47183.snowflakecomputing.com](https://tknjgnq-xc47183.snowflakecomputing.com).
- A **username and password** from your instructor:
  - **Username:** `USERXX` (replace `XX` with your assigned number)
  - **Password:** `Workshop@2026!`
- A modern web browser — everything runs in Snowsight, nothing to install.

Ready? Start with **[What is Snowflake? →](01-0-whatissnowflake.md)**
