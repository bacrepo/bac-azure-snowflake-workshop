---
title: "Lesson 03 — Databases, schemas & tables"
---

# Lesson 03 — Databases, schemas & tables

**Goal:** learn how Snowflake organizes data, and create your first table.

[← Previous: Warehouses](03-warehouses.md) · [Home](index.md) · [Next: Loading data →](05-loading-data.md)

---

## The hierarchy

Snowflake organizes objects in a simple three-level tree:

```
DATABASE
└── SCHEMA
    ├── TABLE
    ├── VIEW
    └── ...
```

You refer to objects by their full path: `DATABASE.SCHEMA.TABLE`.

## Create a database and schema

```sql
CREATE DATABASE IF NOT EXISTS LEARN_DB;
CREATE SCHEMA   IF NOT EXISTS LEARN_DB.WORKSHOP;

-- Set them as your defaults so you can skip the full path:
USE DATABASE LEARN_DB;
USE SCHEMA WORKSHOP;
```

## Create a table

```sql
CREATE OR REPLACE TABLE students (
  id          INT,
  name        STRING,
  enrolled_on DATE,
  score       NUMBER(5,2)
);
```

## Put some rows in

```sql
INSERT INTO students (id, name, enrolled_on, score) VALUES
  (1, 'Anya',  '2026-01-15', 88.5),
  (2, 'Boon',  '2026-02-01', 92.0),
  (3, 'Chai',  '2026-02-20', 79.5);

SELECT * FROM students;
```

## Inspect your objects

```sql
SHOW DATABASES;
SHOW SCHEMAS IN DATABASE LEARN_DB;
SHOW TABLES IN SCHEMA LEARN_DB.WORKSHOP;
DESCRIBE TABLE students;   -- column names & types
```

> 💡 `CREATE OR REPLACE` is great in a workshop — re-running a lesson rebuilds
> the table cleanly. In production, prefer `CREATE TABLE IF NOT EXISTS` so you
> don't drop real data.

---

## Recap

- Objects live in **database → schema → table**.
- `USE DATABASE` / `USE SCHEMA` set defaults so you can use short names.
- `SHOW` and `DESCRIBE` let you explore what exists.

[← Previous: Warehouses](03-warehouses.md) · [Home](index.md) · [Next: Loading data →](05-loading-data.md)
