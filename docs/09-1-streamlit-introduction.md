---
title: "Streamlit — data apps in pure Python"
---

# Streamlit — data apps in pure Python

**Goal:** understand what Streamlit is, why **Streamlit in Snowflake** means
you can ship a real data app with zero front-end stack — and where it fits
next to the chat agent you just built.

[← Back to Home](index.md)

---

## What is Streamlit?

Streamlit is a Python framework that turns a **script** into a **web app**.
No HTML, no JavaScript, no REST API, no deployment pipeline:

```python
import streamlit as st

st.title("Hello BAC 👋")
number = st.slider("Pick a number", 0, 100)
st.write(f"You picked {number}")
```

That's a complete, interactive web app. Each interaction re-runs the script
top to bottom; whatever you declare gets rendered. If you can write the
Python you used in [Lesson 05](05-1-notebooks-snowpark.md), you can build an
app.

---

## Streamlit *in Snowflake*

Normally a data app needs hosting, auth, and a database connection to
secure. **Streamlit in Snowflake (SiS)** removes all three — the app runs
*inside* your Snowflake account:

```
  Classic stack:                          Streamlit in Snowflake:

  React front-end                         one .py file
  + REST API                              (runs in Snowflake,
  + auth server          ──becomes──▶      opens with session already
  + DB credentials                          connected, secured by RBAC)
  + hosting/DevOps
```

| Concern | Classic web app | Streamlit in Snowflake |
|---------|----------------|------------------------|
| Hosting | your problem | Snowflake runs it |
| Login | build it | Snowflake login, automatically |
| Data access | manage credentials | `get_active_session()` — no secrets |
| Permissions | build them | **your existing roles** apply |
| Sharing | deploy somewhere | grant the app to a role, done |
| Compute | app servers | the same warehouse you've used all day |

The security point is the big one: a user opening your app sees only what
their **role** permits — the same governance as every query today. The data
never leaves Snowflake.

---

## Chat or dashboard? Both.

You now have two ways to serve `agent.Sales` to the business:

| | Snowflake Intelligence (Lesson 07) | Streamlit (this lesson) |
|--|------------------------------------|--------------------------|
| Interaction | ask anything, free-form | curated views, always the same |
| Best for | exploration, ad-hoc "why?" questions | KPIs on a screen, daily monitoring |
| Built by | semantic model + agent | a Python script |
| Audience trust | verify the generated SQL | you wrote the SQL — it's fixed |

They're complements: Intelligence answers the questions you *didn't*
anticipate; a dashboard nails the ones you *did*.

---

## The 80% of the API you'll ever need

```python
import streamlit as st
from snowflake.snowpark.context import get_active_session

session = get_active_session()                  # already authenticated

# Layout & text
st.title("📊 Sales Dashboard")
st.header("Monthly view")
col1, col2, col3 = st.columns(3)                # side-by-side layout

# KPI cards
col1.metric("Revenue", "12.4M THB", "+8%")

# Inputs (every widget re-runs the script with the new value)
org   = st.selectbox("Sales org", ["All", "1000", "2000"])
dates = st.date_input("Date range", [])

# Data → display
df = session.sql("SELECT * FROM agent.Sales LIMIT 100").to_pandas()
st.dataframe(df)                                # interactive table
st.bar_chart(df, x="PRODUCTGROUP", y="NETAMOUNT")
st.line_chart(df, x="ORDERDATE",   y="NETAMOUNT")

# Cache query results so widget clicks don't re-hit the warehouse
@st.cache_data
def load_data(query: str):
    return session.sql(query).to_pandas()
```

That's the whole mental model: **widgets in, dataframes out, charts on
screen** — and `@st.cache_data` so the warehouse isn't queried on every
click.

> 💡 **Cost intuition:** a Streamlit app runs on a warehouse while in use,
> like a worksheet. Same rules as
> [Lesson 02](02-0-snowflake-platform.md) — `XSMALL` + auto-suspend is
> plenty, and caching keeps the meter quiet.

---

## Where you'll build it

In Snowsight: **Projects → Streamlit → + Streamlit App** → pick database
(`USERXX`), schema, and `WORKSHOP_WH`. You get a split screen — code editor
left, live app right. Edit, run, see it instantly.

---

## Recap

- **Streamlit** = Python script → web app; widgets re-run the script.
- **Streamlit in Snowflake** = no hosting, no auth code, no credentials —
  RBAC and warehouses you already understand.
- Dashboards and chat agents are **complements**: fixed views vs. free-form
  questions, both on the same `agent.Sales`.
- Core API: `st.metric`, `st.selectbox`, `st.dataframe`,
  `st.bar_chart` / `st.line_chart`, `@st.cache_data`.

**Next:** build the BAC sales dashboard on `agent.Sales` — and let AI write
the first draft from one good prompt.

[← Back to Home](index.md) | [Next: Streamlit hands-on →](09-2-streamlit-hands-on.md)
