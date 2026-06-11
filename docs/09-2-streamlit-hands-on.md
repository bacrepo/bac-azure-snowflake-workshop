---
title: "DEMO 14 — Streamlit hands-on: the BAC sales dashboard"
---

# DEMO 14 — Streamlit hands-on: the BAC sales dashboard

**Goal:** ship a real dashboard on **`agent.Sales`** — and do it the modern
way: write **one good prompt**, let AI draft the app, then review and refine
like an engineer.

[← Back to Home](index.md)

---

## The baseline table

Everything draws from the table you built in [DEMO 08](04-1-summary-data.md):

```sql
CREATE OR REPLACE TABLE agent.Sales AS
SELECT
    -- ── identifiers ──────────────────────────────────────────────
    H.SalesDocument                              AS SalesOrder,
    TO_DATE(H.DocumentDate::STRING, 'YYYYMMDD')  AS OrderDate,

    -- ── who / where ──────────────────────────────────────────────
    H.SoldToParty                                AS Customer,
    H.SalesOrg                                   AS SalesOrg,

    -- ── product ──────────────────────────────────────────────────
    I.Material                                   AS Product,
    I.ItemDescription                            AS ProductName,
    I.MaterialGroup                              AS ProductGroup,
    H.SalesDocumentType                          AS OrderType,

    -- ── measures ─────────────────────────────────────────────────
    CAST(I.OrderQuantity AS NUMBER(15,3))        AS Quantity,
    I.SalesUnit                                  AS SalesUnit,
    CAST(I.ItemNetValue  AS NUMBER(15,2))        AS NetAmount,
    H.DocumentCurrency                           AS Currency
FROM silver.VBAP AS I
INNER JOIN silver.VBAK AS H
    ON I.SalesDocument = H.SalesDocument;
```

Grain: **one row per order item**. Measures: `NetAmount`, `Quantity`. Time:
`OrderDate`. Everything else: dimensions to slice by.

---

## Step 1 — Create the app

In Snowsight: **Projects → Streamlit → + Streamlit App**

- **Name:** `BAC_SALES_DASHBOARD`
- **Database / schema:** `USERXX.agent`
- **Warehouse:** `WORKSHOP_WH`

You land in a split view — code left, running app right. Delete the example
code.

---

## Step 2 — The dashboard prompt

You *could* hand-write the app. But you have AI assistants now — Cortex Code
in Snowsight, or any LLM. The skill that matters is the **prompt**. A weak
prompt ("make a dashboard for my sales table") forces the AI to guess your
schema, your audience, and your platform — and you'll spend longer fixing
the guesses than writing the code.

A good dashboard prompt has five parts: **role & audience, exact schema,
layout spec, technical constraints, and data quirks**. Here is one for
`agent.Sales` — copy it into Cortex Code and adapt:

```text
You are building a Streamlit in Snowflake (SiS) sales dashboard for BAC
managers. They check it daily; it must answer "how are sales going?"
in under 10 seconds, with no training.

DATA
One table: agent.Sales — SAP sales orders, one row per ORDER ITEM.
Columns (note: to_pandas() returns them UPPERCASE):
- SALESORDER  (text)  order number — count DISTINCT for "number of orders"
- ORDERDATE   (date)  order date
- CUSTOMER    (text)  customer code
- SALESORG    (text)  sales organization
- PRODUCT     (text)  material code
- PRODUCTNAME (text)  item description — use this in charts, not PRODUCT
- PRODUCTGROUP(text)  material group
- ORDERTYPE   (text)  SAP order type (e.g. ZOR = order, ZRE = return)
- QUANTITY    (number) ordered quantity
- SALESUNIT   (text)  unit of QUANTITY
- NETAMOUNT   (number) net value of the item line — THE revenue measure
- CURRENCY    (text)  document currency

LAYOUT (top to bottom)
1. Title "📊 BAC Sales Dashboard" + caption naming the source table.
2. Sidebar filters: date range (default = full range), Sales org
   (selectbox with "All"), Product group (multiselect). All charts and
   KPIs respect every filter.
3. KPI row, 4 st.metric cards: Total net revenue / Distinct orders /
   Total quantity / Average value per order.
4. Line chart: monthly net revenue (resample ORDERDATE by month).
5. Two columns side by side: Top 10 products by revenue (bar,
   PRODUCTNAME) | Top 10 customers by revenue (bar).
6. Bar chart: revenue by ORDERTYPE.
7. Expandable detail table of filtered rows, newest first.

TECHNICAL
- Streamlit in Snowflake: use
  snowflake.snowpark.context.get_active_session() — no credentials,
  no st.connection.
- Read the table ONCE via session.sql(...).to_pandas() inside a
  @st.cache_data function; filter in pandas after that.
- Format money with thousands separators, no decimals.
- Guard against division by zero when no rows match the filters, and
  show st.warning("No data for selected filters") instead of crashing.

DATA QUIRKS
- NETAMOUNT is item-level — summing it is correct (no double counting).
- Returns (ORDERTYPE ZRE) can be negative — do not filter them out.
- Mixed currencies may exist: if more than one CURRENCY is present in
  the filtered data, show a caption warning that totals mix currencies.
```

> 💡 **Why this prompt works:** the AI never has to guess. Schema with
> *meanings* ("THE revenue measure", "count DISTINCT for orders") prevents
> wrong aggregates; the layout is numbered so you can diff the result
> against the spec; the quirks section encodes what only *you* know — the
> uppercase columns, the grain, the returns. That last section is pure data
> engineering knowledge. **The prompt is only as good as your understanding
> of the data — which is why this is the last lesson, not the first.**

---

## Step 3 — Review the draft (reference implementation)

Whatever the AI produces, review it like you reviewed Cortex Code's SQL in
[DEMO 09](04-2-cortex-test.md). It should look close to this — a complete,
working app you can also paste directly:

```python
import streamlit as st
import pandas as pd
from snowflake.snowpark.context import get_active_session

st.set_page_config(page_title="BAC Sales Dashboard", layout="wide")
session = get_active_session()

@st.cache_data
def load_sales() -> pd.DataFrame:
    df = session.sql("SELECT * FROM agent.Sales").to_pandas()
    df["ORDERDATE"] = pd.to_datetime(df["ORDERDATE"])
    return df

df = load_sales()

st.title("📊 BAC Sales Dashboard")
st.caption("Source: agent.Sales — SAP orders (VBAK + VBAP), one row per order item")

# ── Sidebar filters ──────────────────────────────────────────────
with st.sidebar:
    st.header("Filters")
    d_min, d_max = df["ORDERDATE"].min(), df["ORDERDATE"].max()
    date_range = st.date_input("Order date", (d_min, d_max))
    org = st.selectbox("Sales org", ["All"] + sorted(df["SALESORG"].dropna().unique()))
    groups = st.multiselect("Product group", sorted(df["PRODUCTGROUP"].dropna().unique()))

f = df.copy()
if len(date_range) == 2:
    f = f[(f["ORDERDATE"] >= pd.Timestamp(date_range[0]))
          & (f["ORDERDATE"] <= pd.Timestamp(date_range[1]))]
if org != "All":
    f = f[f["SALESORG"] == org]
if groups:
    f = f[f["PRODUCTGROUP"].isin(groups)]

if f.empty:
    st.warning("No data for selected filters")
    st.stop()

if f["CURRENCY"].nunique() > 1:
    st.caption("⚠️ Totals mix multiple currencies: "
               + ", ".join(f["CURRENCY"].dropna().unique()))

# ── KPI row ──────────────────────────────────────────────────────
revenue = f["NETAMOUNT"].sum()
orders  = f["SALESORDER"].nunique()
qty     = f["QUANTITY"].sum()
aov     = revenue / orders if orders else 0

c1, c2, c3, c4 = st.columns(4)
c1.metric("Net revenue", f"{revenue:,.0f}")
c2.metric("Orders", f"{orders:,}")
c3.metric("Quantity", f"{qty:,.0f}")
c4.metric("Avg value / order", f"{aov:,.0f}")

# ── Monthly trend ────────────────────────────────────────────────
st.subheader("Monthly net revenue")
monthly = (f.set_index("ORDERDATE")
             .resample("MS")["NETAMOUNT"].sum()
             .reset_index())
st.line_chart(monthly, x="ORDERDATE", y="NETAMOUNT")

# ── Top products | top customers ────────────────────────────────
left, right = st.columns(2)
with left:
    st.subheader("Top 10 products by revenue")
    top_products = (f.groupby("PRODUCTNAME")["NETAMOUNT"]
                      .sum().nlargest(10).sort_values())
    st.bar_chart(top_products)
with right:
    st.subheader("Top 10 customers by revenue")
    top_customers = (f.groupby("CUSTOMER")["NETAMOUNT"]
                       .sum().nlargest(10).sort_values())
    st.bar_chart(top_customers)

# ── Order type breakdown ─────────────────────────────────────────
st.subheader("Revenue by order type")
by_type = f.groupby("ORDERTYPE")["NETAMOUNT"].sum().sort_values(ascending=False)
st.bar_chart(by_type)

# ── Detail ───────────────────────────────────────────────────────
with st.expander("Detail rows"):
    st.dataframe(f.sort_values("ORDERDATE", ascending=False),
                 use_container_width=True)
```

Review checklist (the same discipline as always):

- [ ] Revenue = `SUM(NETAMOUNT)` — item grain, no double counting
- [ ] Orders = `nunique(SALESORDER)` — not row count
- [ ] Data loaded **once** behind `@st.cache_data` — filters don't re-query
- [ ] Empty-filter case handled, mixed currencies flagged

---

## Step 4 — Run, filter, break it

Click **Run**. Then act like a manager:

- Narrow the date range — every KPI and chart should follow.
- Pick a single product group — do the top-customer bars change?
- Pick a range with no data — you should get the warning, not a crash.

Then act like an engineer: cross-check one number.

```sql
SELECT SUM(NetAmount) FROM agent.Sales;   -- matches the unfiltered KPI?
```

---

## Step 5 — Share it

The whole point of Streamlit in Snowflake — no deployment:

In the app, **Share** → grant to a role — try the read-only `BAC_DA_ROLE`
from [DEMO 13](08-2-governance-hands-on.md). Anyone with that role opens
the same URL and sees the app — and if you left the governance policies in
place, their numbers are quietly masked and filtered by the platform. No
server, no credentials handed out, no governance code in the app.

---

## Recap — and the whole day on one page

```
SAP / Excel / SharePoint
   └▶ stage ─▶ bronze ─▶ silver ─▶ gold / agent.Sales      Lessons 02–05  (engineering)
                                        ├▶ semantic model ─▶ Intelligence  Lesson 07   (chat)
                                        ├▶ policies: who sees what         Lesson 08   (governance)
                                        └▶ Streamlit dashboard             Lesson 09   (apps)
```

| Step | What you did |
|------|-------------|
| Prompt | Specified audience, schema **with meanings**, layout, constraints, quirks |
| Draft | AI generated the app — you reviewed grain, aggregates, caching |
| Verify | Cross-checked the KPI against plain SQL |
| Share | Granted the app to a role — shipped, governed, done |

🎉 **That's the workshop.** You took raw SAP files and ended with a chat
agent *and* a live dashboard — and every step in between was the data
engineering that makes those last two trustworthy. Keep the order in mind
when you build the real thing: **engineer first, AI on top.**

[← Back to Home](index.md)
