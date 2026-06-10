---
title: "Lesson 00 — Setup & first query"
---

# Lesson 00 — Setup & first query

**Goal:** sign in to Snowflake, run your very first query, and learn your way
around the interface.

[← Back to home](index.md) · [Next: Architecture →](02-architecture.md)

---

## 1. Sign in to Snowsight

Snowsight is Snowflake's web interface — where you'll spend most of this
workshop.

1. Open the account URL your instructor gave you (it looks like
   `https://<account>.snowflakecomputing.com`).
2. Enter your **username** and **password**.
3. You'll land in **Snowsight**.

## 2. Open a worksheet

A *worksheet* is where you write and run SQL.

- Click **Projects → Worksheets** in the left sidebar.
- Click **+ Worksheet** (top right).

## 3. Run your first query

Paste this in and press **Run** (or `Cmd/Ctrl + Enter`):

```sql
SELECT VERSION();
```

You should see the Snowflake version number. 🎉 You just ran a query against a
cloud data warehouse.

## 4. A couple more to get oriented

```sql
-- Who am I, and what am I using right now?
SELECT CURRENT_USER(), CURRENT_ROLE(), CURRENT_WAREHOUSE();

-- What's the current date and time on the server?
SELECT CURRENT_TIMESTAMP();
```

If `CURRENT_WAREHOUSE()` returns `NULL`, you don't have compute selected yet —
that's exactly what the next lessons fix.

---

## Recap

- **Snowsight** = the web UI. **Worksheets** = where you run SQL.
- `SELECT VERSION();` confirms you're connected.
- `CURRENT_USER / ROLE / WAREHOUSE` tell you the context every query runs in.

[← Back to home](index.md) · [Next: Architecture →](02-architecture.md)
