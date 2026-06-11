---
title: "Snowflake Intelligence — chat with your data"
---

# Snowflake Intelligence — chat with your data

**Goal:** understand the pieces behind "chat with data" — **Snowflake
Intelligence**, **Cortex Analyst**, and the **semantic model** — and why the
semantic model (not the LLM) is the part that decides success or failure.
Plus: what it costs.

[← Back to Home](index.md)

---

## The payoff moment

Everything you built today leads here:

```
  SAP files ──▶ stage ──▶ bronze ──▶ silver ──▶ agent.Sales   (this morning)
                                                    │
                                              semantic model   (this lesson)
                                                    │
  Business user: "ยอดขายเดือนนี้เท่าไหร่?" ──▶  Snowflake Intelligence
                                                    │
                                              answer + chart
```

A business user types a question in plain Thai or English and gets a
governed, correct answer — **because** a data engineer prepared the table and
defined what the words mean. No engineering, no answer. That's why this is
lesson 07 and not lesson 01.

---

## The three layers

| Layer | What it is | Who touches it |
|-------|-----------|----------------|
| **Snowflake Intelligence** | The chat UI + agents — where users ask questions | Business users |
| **Cortex Analyst** | The engine — translates a question into SQL, runs it, explains the result | Nobody directly (it's a service) |
| **Semantic model** | The dictionary — maps business words to your tables, columns, and formulas | **You (DE/DA team)** |

### Snowflake Intelligence

The front door: a chat interface in Snowsight (and via API) where users talk
to **agents**. An agent can use Cortex Analyst for structured data (tables),
Cortex Search for documents, and chain multiple steps — all inside
Snowflake's security model. Users only ever see data their **role** permits.

### Cortex Analyst

The text-to-SQL engine underneath. Given a question and a semantic model, it:

1. Reads the question ("top 5 products by revenue this quarter").
2. Maps the business words to real columns **using the semantic model**.
3. Generates and runs the SQL on your warehouse.
4. Returns the answer — with the SQL shown, so it's verifiable.

The critical design choice: **the LLM never sees your rows.** It sees the
question and the semantic model, writes SQL, and Snowflake executes it under
the user's role like any other query.

---

## What is a semantic model?

A YAML file (or a **semantic view** in newer accounts) that describes one or
more tables *in business terms*:

```yaml
name: sales_model
tables:
  - name: sales
    base_table: USERXX.AGENT.SALES        # ← the table you built in DEMO 08
    dimensions:
      - name: customer
        expr: Customer
        description: Sold-to party (customer code)
        synonyms: ["client", "buyer", "ลูกค้า"]
      - name: product_name
        expr: ProductName
        synonyms: ["item", "สินค้า"]
    time_dimensions:
      - name: order_date
        expr: OrderDate
        description: Date the sales order was placed
        synonyms: ["date", "วันที่สั่งซื้อ"]
    measures:
      - name: net_amount
        expr: NetAmount
        default_aggregation: sum
        description: Net order value per item line
        synonyms: ["revenue", "sales", "ยอดขาย", "รายได้"]
      - name: quantity
        expr: Quantity
        default_aggregation: sum
        synonyms: ["qty", "units", "จำนวน"]
```

Three building blocks, the same three roles you saw in
[DEMO 08](04-1-summary-data.md):

| Block | In `agent.Sales` | The user says |
|-------|-----------------|----------------|
| **Measures** (numbers to aggregate) | `NetAmount`, `Quantity` | "revenue", "ยอดขาย" |
| **Dimensions** (things to slice by) | `Customer`, `ProductName`, `SalesOrg` … | "by customer", "per product" |
| **Time dimensions** | `OrderDate` | "last month", "this quarter" |

Plus two trust features:

- **Synonyms** — every way a human might say it, in every language they use.
- **Verified queries** — question/SQL pairs you bless by hand. When a user
  asks something similar, Analyst reuses your verified logic instead of
  improvising.

---

## Why the semantic model is THE success factor

Here's the uncomfortable truth about chat-with-data: **the LLM is not the
hard part.** Every vendor has a good model. Projects fail or succeed on the
semantic layer.

### Without a semantic model

Ask an LLM "what was revenue last month?" against raw `silver.VBAK` and it
must *guess*:

- Is revenue `NETWR`? Header or item level? (You know from
  [DEMO 05](03-2-more-data-engineering.md) that picking wrong
  **double-counts**.)
- Which of the nine date columns means "the" date?
- Does "last month" use calendar month? Fiscal?

The killer is that the answer **looks** right — a confident number, a tidy
chart. Wrong numbers that look right are how a chat-with-data project dies:
the first time finance catches a discrepancy, nobody trusts the tool again.

### With a semantic model

- "Revenue" can only mean `SUM(NetAmount)` — **you** wrote that down. The
  model can't invent a different formula.
- The questionable columns aren't in the model at all, so they can't be
  picked.
- "ยอดขาย" resolves via synonyms; "last month" resolves via the declared
  time dimension.
- Common questions hit **verified queries** — your SQL, not generated SQL.

```
  LLM alone:            question ──▶ guess over 50 raw columns ──▶ plausible answer
  LLM + semantic model: question ──▶ YOUR definitions ──▶ governed answer
```

> 💡 **The success formula:**
> **clean narrow table** (DEMO 08) **+ semantic model** (next demo) **= trustworthy chat-with-data.**
> Both halves are data engineering work. This is why the morning mattered.

---

## Pricing scheme — Compute + LLM tokens

Snowflake Intelligence has **two meters**, and it helps to know which one is
spinning:

```
  user question
       │
       ▼
  ① LLM tokens   — Cortex Analyst / agent reads the question + semantic
       │            model and writes SQL  (serverless, billed per token
       │            in credits — no warehouse involved)
       ▼
  ② Compute      — the generated SQL runs on a warehouse
                    (normal per-second warehouse billing, like any query)
```

| Meter | What you pay for | How it's billed | You control it by |
|-------|-----------------|-----------------|-------------------|
| **LLM tokens** | Understanding the question, generating SQL, summarizing the answer | Serverless Cortex credits, per token in/out | Concise semantic model, verified queries (less back-and-forth) |
| **Compute** | Executing the SQL over your data | Warehouse credits per second, same as all day today | Small warehouse + auto-suspend; narrow tables like `agent.Sales` scan less |

Two practical notes:

- A chat question typically costs **fractions of a credit** — the token side
  is small. The compute side is the same discipline you learned in
  [Lesson 02](02-0-snowflake-platform.md): right-size the warehouse,
  auto-suspend.
- Track it like everything else — Cortex usage appears in
  `SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY`, warehouse usage
  in `WAREHOUSE_METERING_HISTORY`. No separate bill, no per-user licence:
  **pay per use, on the same Snowflake credits.**

---

## Recap

- **Snowflake Intelligence** = the chat UI; **Cortex Analyst** = the
  text-to-SQL engine; the **semantic model** = the dictionary you write.
- The LLM never touches raw rows — it writes SQL from **your** definitions,
  and RBAC still applies.
- The semantic model is the **key success factor**: it turns "plausible
  answers" into "governed answers". Synonyms and verified queries are where
  the trust comes from.
- Pricing = **LLM tokens** (serverless, per question) **+ compute** (the SQL
  runs on a normal warehouse). Both are pay-per-use credits.

**Next:** build it — semantic model on `agent.Sales`, then ask questions in
plain language.

[← Back to Home](index.md) | [Next: Snowflake Intelligence hands-on →](07-2-snowflake-intelligence-hands-on.md)
