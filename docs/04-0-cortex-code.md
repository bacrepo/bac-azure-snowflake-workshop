---
title: "Cortex Code"
---

# Cortex Code

**Goal:** use Snowflake's built-in AI assistant to write and explain SQL and
code — without leaving Snowsight.

> **Welcome to the AI half of the workshop.** Notice the order we did things:
> data engineering *first*, AI *second*. AI on top of messy data gives
> confident wrong answers; AI on top of the clean silver/gold tables you built
> this morning is genuinely useful. That ordering is the biggest lesson of
> the day.

[← Back to Home](index.md)

---

## The concept

**Cortex Code** is an AI assistant embedded in Snowflake. You describe what you
want in plain language; it drafts the SQL/Python, explains existing code, and
helps debug — grounded in your account's schemas and objects.

```
  "show total sales by region for last month"
            │
            ▼
   Cortex Code  ──▶  draft SQL  ──▶  you review & run
```

## Where you'll see it

- **In a worksheet** — an assistant panel to generate or explain a query.
- **In notebooks** — alongside your Python/SQL cells.
- **Ask Copilot / Cortex** — natural-language prompts about your data.

## Cortex functions you can call directly

The same AI is available as SQL functions — handy in pipelines:

```sql
-- Free-form generation:
SELECT SNOWFLAKE.CORTEX.COMPLETE(
  'claude-3-5-sonnet',
  'Summarize this in one sentence: ' || review_text
) AS summary
FROM customer_reviews
LIMIT 5;

-- Purpose-built helpers:
SELECT SNOWFLAKE.CORTEX.SENTIMENT(review_text)        AS sentiment,
       SNOWFLAKE.CORTEX.SUMMARIZE(review_text)        AS short_summary
FROM customer_reviews
LIMIT 5;
```

> 💡 **Always review AI-generated SQL before running it** — especially anything
> that writes, drops, or grants. The assistant drafts; you stay in control.

---

## Recap

- **Cortex Code** = an in-Snowsight AI assistant that drafts and explains code.
- The same models are callable as **`SNOWFLAKE.CORTEX.*`** SQL functions.
- Treat output as a **draft** — review before you run.

[← Back to Home](index.md) | [Next: Agent-ready summary →](04-1-summary-data.md)
