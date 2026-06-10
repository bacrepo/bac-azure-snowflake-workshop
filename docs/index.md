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
| 03-0 | [Data engineering](03-0-data-engineering.md) | Tables, `COPY INTO`, scripting, dynamic tables, time travel |

## Afternoon — AI, notebooks & apps

| #  | Lesson | What you'll learn |
| -- | ------ | ----------------- |
| 05 | [Cortex Code](05-cortex-code.md) | Snowflake's AI assistant for writing SQL & code |
| 06 | [Data engineering II](06-notebooks-snowpark.md) | Notebooks, Snowpark, reading Excel from a stage |
| 07 | [Why Excel is difficult](07-why-excel.md) | The hidden costs of spreadsheet-driven processes |
| 08 | [Building a data agent](08-data-agent.md) | Cortex Agent, semantic models, Snowflake Intelligence |
| 09 | [Building apps with Streamlit](09-streamlit.md) | Streamlit in Snowflake — data apps with no front-end stack |

---

## Before you start

You'll need:

- A **Snowflake account** on Azure (your instructor provides the account URL).
- A **username and password** from your instructor.
- A modern web browser — everything runs in Snowsight, nothing to install.

Ready? Start with **[What is Snowflake? →](01-0-whatissnowflake.md)**
