---
title: "Lesson 05 — Querying & analytics"
---

# Lesson 05 — Querying & analytics

**Goal:** the everyday SQL you'll use to actually answer questions with your
data.

[← Previous: Loading data](05-loading-data.md) · [Home](index.md)

---

These all run against the `students` table from
[Lesson 03](04-databases.md). Make sure your context is set:

```sql
USE WAREHOUSE LEARN_WH;
USE SCHEMA LEARN_DB.WORKSHOP;
```

## Filtering & sorting

```sql
SELECT name, score
FROM students
WHERE score >= 85
ORDER BY score DESC;
```

## Aggregating

```sql
SELECT
  COUNT(*)        AS num_students,
  AVG(score)      AS avg_score,
  MIN(score)      AS lowest,
  MAX(score)      AS highest
FROM students;
```

## Grouping

```sql
SELECT
  DATE_TRUNC('month', enrolled_on) AS enrolled_month,
  COUNT(*)                          AS signups
FROM students
GROUP BY enrolled_month
ORDER BY enrolled_month;
```

## Window functions (ranking)

A window function computes across rows *without* collapsing them — perfect for
rankings and running totals:

```sql
SELECT
  name,
  score,
  RANK() OVER (ORDER BY score DESC) AS class_rank
FROM students;
```

## Snowflake conveniences worth knowing

```sql
-- QUALIFY filters on a window function directly (no subquery needed):
SELECT name, score
FROM students
QUALIFY RANK() OVER (ORDER BY score DESC) <= 2;   -- top 2 students

-- Sample a fraction of a big table:
SELECT * FROM students SAMPLE (50);   -- ~50% of rows
```

## Result caching (a nice surprise)

Run the same query twice — the second run returns instantly from Snowflake's
**result cache**, and it doesn't even need the warehouse to spin up. Great for
dashboards that re-ask the same question.

---

## Recap

- `WHERE` / `ORDER BY` filter and sort; aggregate functions summarize.
- `GROUP BY` summarizes *per group*; **window functions** rank without collapsing.
- `QUALIFY`, `SAMPLE`, and the **result cache** are Snowflake niceties.

🎓 **That's the core workshop.** You can now sign in, create compute, organize
and load data, and query it. From here, explore **roles & access control**,
**tasks & streams**, and **time travel**.

[← Previous: Loading data](05-loading-data.md) · [Home](index.md)
