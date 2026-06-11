---
title: "DEMO 12 — Snowflake Intelligence hands-on"
---

# DEMO 12 — Snowflake Intelligence hands-on

**Goal:** build a **semantic model** on top of `agent.Sales`, wire it into a
**Snowflake Intelligence agent**, and ask it questions in plain English (and
Thai) — watching the SQL it writes.

[← Back to Home](index.md)

---

## Prerequisite

You need `agent.Sales` from [DEMO 08](04-1-summary-data.md). Quick check:

```sql
USE ROLE BAC_DE_ROLE;
USE WAREHOUSE WORKSHOP_WH;
SELECT COUNT(*) FROM USERXX.agent.Sales;
DESC TABLE USERXX.agent.Sales;
```

12 columns, clean `DATE` / `NUMBER` types. If not, go back and run DEMO 08
first — this whole demo stands on that table.

> 🧭 Snowsight menus move between releases. Everything below lives under
> **AI & ML** in the left sidebar — if a button isn't where described, look
> around that section.

---

## Step 1 — Create the semantic model

1. In Snowsight: **AI & ML → Cortex Analyst** (semantic models /
   semantic views).
2. **+ Create new** → choose a location for the model
   (database `USERXX`, schema `agent`).
3. Name it **`sales_model`**, description:
   *"Sales orders from SAP (VBAK/VBAP), one row per order item."*
4. Select the base table: **`USERXX.AGENT.SALES`**.
5. Select **all 12 columns** when prompted.

Snowsight drafts a model, guessing which columns are measures and which are
dimensions. Review its guesses against what you know:

| Columns | Should be |
|---------|-----------|
| `Quantity`, `NetAmount` | **measures** (default aggregation: SUM) |
| `OrderDate` | **time dimension** |
| everything else | **dimensions** |

---

## Step 2 — Teach it the business language

This is the step that separates a demo from a tool people trust. For each
field, add a **description** and **synonyms** — every word your colleagues
would actually use:

| Field | Description | Synonyms to add |
|-------|-------------|-----------------|
| `NetAmount` | Net order value per item line | `revenue`, `sales`, `amount`, `ยอดขาย`, `รายได้` |
| `Quantity` | Ordered quantity in sales units | `qty`, `units`, `volume`, `จำนวน` |
| `Customer` | Sold-to party (customer code) | `client`, `buyer`, `account`, `ลูกค้า` |
| `ProductName` | Item description | `item`, `product description`, `สินค้า` |
| `OrderDate` | Date the order was placed | `date`, `order day`, `วันที่สั่งซื้อ` |
| `SalesOrg` | Sales organization | `sales org`, `org`, `หน่วยงานขาย` |

**Save** the model.

---

## Step 3 — Test inside the editor

The semantic model editor has a test chat panel. Try:

> **What is total revenue by month?**

Check three things in the response:

1. The **SQL** it generated — `SUM(NetAmount)` grouped by a month truncation
   of `OrderDate`. Your definitions, not a guess.
2. The **result** — sanity-check it against
   `SELECT SUM(NetAmount) FROM agent.Sales;`
3. Re-ask with a synonym: *"ยอดขายรวมต่อเดือน"* — same SQL should come back.

### Add a verified query

When an answer is exactly right, click **verify / save as verified query**.
Do it for the revenue-by-month question. From now on, similar questions reuse
your blessed SQL instead of regenerating from scratch — accuracy goes up,
token cost goes down.

---

## Step 4 — Create the agent in Snowflake Intelligence

1. **AI & ML → Agents** → **+ Create agent**.
2. Name: **`BAC_SALES_AGENT`**, display name *"BAC Sales Assistant"*.
3. Description / instructions: *"You answer questions about BAC sales
   orders. Amounts are in document currency. Be concise; always show
   numbers with their currency."*
4. Under **Tools**, add **Cortex Analyst** and point it at the
   **`sales_model`** semantic model you just built.
5. Grant usage to the workshop role, save.

Open **AI & ML → Snowflake Intelligence**, pick *BAC Sales Assistant* — and
you're chatting with this morning's pipeline.

---

## Step 5 — Interview your data

Work through these, from easy to mean:

```
1.  What is our total revenue?
2.  Top 5 customers by revenue
3.  Monthly revenue trend for 2024 — show a chart
4.  Which product group sells the most by quantity?
5.  ลูกค้าคนไหนซื้อเยอะที่สุดในปีนี้
6.  Compare revenue of order type ZOR vs ZRE by month
7.  What was the average order value last quarter?
```

For each answer, click to expand the **generated SQL** and read it. That
habit — *trust, but verify* — is what you'll teach your business users.

### What to notice

- Question 5 works because of your **Thai synonyms** (Step 2).
- "Last quarter" works because `OrderDate` is a real `DATE` — the cast you
  did in DEMO 08 is earning its keep.
- Ask something the model *can't* know — *"what is our profit margin?"* —
  and watch it decline rather than invent. There's no `profit` measure in
  the model; no measure, no number. **That refusal is a feature.**

---

## Step 6 — Watch the meters (optional)

The [two meters from the last lesson](07-1-snowflake-intelligence.md), live:

```sql
-- ② Compute: each chat question ran SQL on a warehouse like any query
SELECT query_text, warehouse_name, total_elapsed_time
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
ORDER BY start_time DESC
LIMIT 10;

-- ① LLM tokens: serverless Cortex usage (ACCOUNTADMIN or granted role;
--    data appears with some latency)
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY
ORDER BY start_time DESC
LIMIT 10;
```

---

## Recap

```
  agent.Sales (DEMO 08)
       │
       ▼
  sales_model            ← measures/dimensions + synonyms + verified queries
       │
       ▼
  BAC_SALES_AGENT        ← agent with Cortex Analyst tool
       │
       ▼
  Snowflake Intelligence ← business users chat, in Thai or English
```

| Step | What you did | Why it matters |
|------|-------------|----------------|
| 1 | Semantic model on `agent.Sales` | Gives the LLM *your* definitions |
| 2 | Descriptions + synonyms (TH/EN) | Users speak human, not column names |
| 3 | Test + verified query | Accuracy you can bless and reuse |
| 4 | Agent in Intelligence | The front door for business users |
| 5 | Interview the data | Trust, but verify the SQL |

**Next:** the data is now one chat away from *everyone* — which makes
**who may see what** an urgent question. Time for **governance**.

[← Back to Home](index.md) | [Next: Governance →](08-1-governance.md)
