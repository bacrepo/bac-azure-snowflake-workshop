---
title: "Snowflake features"
---

# Snowflake features

**Goal:** a technical walk-through of the capabilities you'll use throughout
this workshop — data engineering, AI, and app development.

[← Back to Home](index.md)

---

## Data engineering

### Snowpipe — auto-load files as they land

Instead of loading files on a schedule, **Snowpipe** watches a stage and loads
new files automatically the moment they arrive — no warehouse needed, billed
by the second.

```
  file lands in stage ──▶ Snowpipe detects it ──▶ loaded into table (seconds)
```

### Snowpipe Streaming — real-time row ingestion

**Snowpipe Streaming** ingests rows directly from an application or message
queue (e.g. Kafka) into a Snowflake table with **sub-second latency**, without
staging files at all.

```
  app / Kafka ──▶ Snowpipe Streaming API ──▶ rows in table (sub-second)
```

| | Snowpipe | Snowpipe Streaming |
| --- | --- | --- |
| Input | Files on a stage | Rows via SDK / Kafka connector |
| Latency | Seconds to minutes | Sub-second |
| Use case | Batch file drops (CSV, Parquet) | Real-time event streams, IoT |
| Warehouse | Not required (serverless) | Not required (serverless) |

Both are **serverless** — Snowflake manages the compute. You pay only for what
is ingested.

### Dynamic tables — declarative pipelines

A **dynamic table** defines its contents as a query. Snowflake keeps it fresh
automatically within your chosen **target lag** — no scheduling, no orchestrator.

```sql
CREATE OR REPLACE DYNAMIC TABLE sales_by_region
  TARGET_LAG = '1 minute'
  WAREHOUSE  = LEARN_WH
AS
  SELECT region, SUM(amount) AS total_amount
  FROM sales
  GROUP BY region;
```

### Streams & tasks — change tracking and scheduling

- A **stream** captures row-level changes (inserts, updates, deletes) on a
  table — like a change log.
- A **task** runs SQL on a schedule or when a stream has data — like a
  lightweight orchestrator.

Together they form event-driven pipelines without external tools.

---

## AI & ML — Cortex

### Cortex AI — LLMs inside Snowflake

Snowflake Cortex gives you access to large language models — **directly inside
Snowflake**, without sending data to external APIs.

```sql
-- Summarize a column of customer feedback in one SQL call
SELECT SNOWFLAKE.CORTEX.SUMMARIZE(feedback_text) FROM reviews;

-- Translate text
SELECT SNOWFLAKE.CORTEX.TRANSLATE(comment, 'th', 'en') FROM tickets;

-- Classify sentiment
SELECT SNOWFLAKE.CORTEX.SENTIMENT(review_text) FROM product_reviews;

-- Use a specific model (Claude, Llama, Mistral, etc.) for any prompt
SELECT SNOWFLAKE.CORTEX.COMPLETE(
  'claude-sonnet-4-6',
  'Translate the following Thai text to English: สวัสดีครับ วันนี้อากาศดีมาก'
) AS translation;
```

Snowflake hosts models from **Anthropic (Claude), Meta (Llama), Mistral**, and
others. You pick the model by name — no API keys, no external services, no data
leaving Snowflake.

### Cortex Search — RAG without the plumbing

Retrieval-Augmented Generation (RAG) is the pattern behind most enterprise AI
assistants: retrieve relevant documents, then let an LLM answer based on them.
**Cortex Search** handles the retrieval side — vector embeddings, indexing, and
search — as a managed service. You bring your documents; Snowflake does the rest.

### Cortex Agents — AI that acts on your data

Cortex Agents can **reason, plan, and take actions** — querying tables, calling
APIs, or chaining steps together — all governed by Snowflake's security model.

---

## Cortex Code — AI assistant for DE/DA teams

**Cortex Code** is an AI coding assistant built directly into Snowsight. It's
the tool your DE and DA teams will reach for every day.

### What it does

- **Generates SQL and Python** from a plain-language description — "create a
  table that aggregates monthly sales by region" → ready-to-run SQL.
- **Explains existing queries** — paste a 200-line stored procedure and ask
  "what does this do?"
- **Debugs errors** — feed it the error message and it suggests a fix.
- **Knows your account** — it's grounded in your actual schemas, tables, and
  columns, not generic examples.

### Where you'll see it

| Location | How to use it |
| --- | --- |
| **Worksheets** | An assistant panel opens beside your SQL — ask it anything |
| **Notebooks** | Inline help alongside Python and SQL cells |
| **Ask Copilot** | Natural-language prompt bar at the top of Snowsight |

### Example workflow

```
  DE/DA: "Join sales with customers and show top 10 by revenue this quarter"
       │
       ▼
  Cortex Code  ──▶  draft SQL (using your actual table/column names)
       │
       ▼
  You review, tweak, run  ──▶  results
```

> **Always review AI-generated SQL before running it** — especially anything
> that writes, drops, or grants. The assistant drafts; you stay in control.

---

## Snowflake Intelligence — plain-English analytics for the business

**Snowflake Intelligence** (formerly Cortex Analyst) is designed for a different
audience: **business users who don't write SQL**. They ask questions in plain
English and get governed, trustworthy answers.

### How it works

1. A **DE/DA team builds a semantic model** — a YAML file that maps business
   terms ("revenue", "region", "last quarter") to actual tables and columns.
2. A business user types a question in natural language.
3. Snowflake Intelligence translates the question into SQL using the semantic
   model, runs it, and returns the answer — often with a chart.

```
  ┌─────────────────────────────────────────────────────────────┐
  │  Semantic model (built by DE/DA)                            │
  │                                                             │
  │  "revenue"      →  SUM(sales.amount)                        │
  │  "region"       →  sales.region                             │
  │  "last quarter" →  DATE_TRUNC('quarter', DATEADD(...))      │
  └─────────────────────────────────────────────────────────────┘
        ▲                                         │
        │                                         ▼
  Business user:                            SQL generated &
  "What was revenue by                      executed automatically
   region last quarter?"                          │
                                                  ▼
                                          Answer + chart
```

### Why it matters for DE/DA teams

- **You control the definitions.** The semantic model is yours — you decide what
  "revenue" means, which tables back it, and what filters apply. The AI can't
  make up numbers that don't match your logic.
- **Fewer ad-hoc requests.** Business users self-serve instead of filing tickets
  asking for "one more cut of the data."
- **Governed answers.** Snowflake's RBAC still applies — users only see data
  their role permits.

### Semantic model example (YAML)

```yaml
name: sales_model
tables:
  - name: sales
    base_table: LEARN_DB.WORKSHOP.SALES
    measures:
      - name: revenue
        expr: SUM(amount)
        description: Total sales revenue
        synonyms:
          - ยอดขาย
          - รายได้
          - sales amount
        sample_values: [1500.00, 32000.50, 780.25]
    dimensions:
      - name: region
        expr: region
        description: Sales region
        synonyms:
          - ภูมิภาค
          - พื้นที่ขาย
          - area
        sample_values: [กรุงเทพ, ภาคเหนือ, ภาคใต้, ภาคอีสาน]
    time_dimensions:
      - name: sold_on
        expr: sold_on
        description: Date of sale
        synonyms:
          - วันที่ขาย
          - วันที่
```

With `synonyms` a business user can ask *"ยอดขายภาคเหนือเดือนที่แล้วเท่าไหร่"*
and Intelligence maps **ยอดขาย → revenue**, **ภาคเหนือ → region**
automatically. `sample_values` help the model match user input to actual data
values.

> In this workshop you'll build a semantic model and connect it to an
> Intelligence app in **Lesson 08**.

---

## Development & apps

### Snowpark — code in Python, Java, or Scala

Snowpark lets you write data transformations, ML pipelines, and custom logic in
**Python, Java, or Scala** — and run them **inside Snowflake's compute**, so
your data never moves.

```python
# A Snowpark DataFrame — looks like pandas, runs in Snowflake
df = session.table("sales")
result = df.group_by("region").agg(sum("revenue"))
result.show()
```

### Notebooks — interactive data science

Snowflake Notebooks let you write Python and SQL side-by-side in an interactive
environment — like Jupyter, but inside Snowflake. No local setup, no data
downloads, no "it works on my machine" problems.

### Streamlit — data apps in minutes

Build interactive web apps that sit **on top of your Snowflake data** — with
just Python. No JavaScript, no front-end framework, no separate hosting. The app
runs inside Snowflake, secured by the same roles and policies as your tables.

---

## Feature map

| Area | Feature | You'll use it in |
| --- | --- | --- |
| **Data Engineering** | Snowpipe / Snowpipe Streaming | Lesson 03 (Stages) |
| | Dynamic tables | Lesson 04 (Data engineering I) |
| | Streams & tasks | Lesson 04 |
| **AI & ML** | Cortex AI (LLM functions) | Lesson 05 (Cortex Code) |
| | Cortex Code | Lesson 05 |
| | Cortex Search / Agents | Lesson 08 (Data agent) |
| | Snowflake Intelligence | Lesson 08 |
| **Development** | Snowpark | Lesson 06 (Notebooks & Snowpark) |
| | Notebooks | Lesson 06 |
| | Streamlit | Lesson 09 (Streamlit) |

---

## Recap

- **Data engineering:** Snowpipe (auto-load files), Snowpipe Streaming
  (real-time rows), dynamic tables (declarative refresh), streams & tasks
  (event-driven pipelines).
- **AI & ML:** Cortex AI (LLMs in SQL), Cortex Code (AI assistant), Cortex
  Search (RAG), Cortex Agents (autonomous actions), Snowflake Intelligence
  (plain-English analytics).
- **Development:** Snowpark (Python/Java/Scala), Notebooks (interactive),
  Streamlit (data apps).
- Everything runs **inside Snowflake** — no data movement, no external
  infrastructure.

[← Back to Home](index.md)
