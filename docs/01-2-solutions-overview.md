---
title: "BAC Solutions Overview"
---

# BAC solutions overview

**Goal:** understand the problem this workshop solves — and where Snowflake fits
in the BAC data landscape on Azure.

[← Back to Home](index.md)

---

## The starting point (the POC)

Today data is spread across systems that don't talk to each other:

```
  SAP  ──┐
         ├──▶  Excel files  ──▶  SharePoint / SharePoint Lists  ──▶  ✉️ email & manual rework
  ...  ──┘
```

- **SAP** — the system of record, but hard to get analytical data out of.
- **Excel** — where numbers actually get worked on (and copied, and broken).
- **SharePoint / SharePoint Lists** — where files and small datasets are shared.

## Where we're going

The POC consolidates these into **Snowflake on Azure** as the single place to
land, engineer, govern, and serve data — then build AI agents and apps on top.

```
  SAP / Excel / SharePoint  ──▶  Stage  ──▶  Snowflake (Azure)  ──▶  Agents & Streamlit apps
```

## What we'll cover today

- **Morning** — the platform, getting files in (stages), and core data
  engineering (tables, `COPY INTO`, dynamic tables, time travel).
- **Afternoon** — Cortex Code, notebooks & Snowpark, why Excel hurts, building a
  data agent, and shipping a Streamlit app.

---

## Recap

- The pain: data scattered across **SAP, Excel, and SharePoint**.
- The plan: consolidate into **Snowflake on Azure**, then add **AI & apps**.
- Everything you build today runs in **Snowsight** in your browser.

[← Back to Home](index.md) | [Next: Setup & first query →](01-3-snowflake-ui.md)
