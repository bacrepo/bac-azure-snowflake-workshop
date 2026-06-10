---
title: BAC Azure + Snowflake Workshop
---

# ❄️ BAC Azure + Snowflake Workshop

Welcome! This is a hands-on workshop for learning **Snowflake**, the cloud data
platform, running on **Microsoft Azure**. No prior data-warehouse experience is
assumed — each lesson is short, builds on the last, and ends with something you
can run yourself.

> **How to use this:** read a lesson top-to-bottom, then copy each SQL snippet
> into [Snowsight](https://app.snowflake.com) (Snowflake's web UI) and run it.
> Learning sticks when you type the commands yourself.

---

## Curriculum

| #  | Lesson | What you'll learn |
| -- | ------ | ----------------- |
| 00 | [Setup & first query](01-setup.md) | Sign in, run `SELECT VERSION()`, understand the UI |
| 01 | [Snowflake architecture](02-architecture.md) | Storage vs. compute vs. cloud services — why Snowflake is different |
| 02 | [Warehouses (compute)](03-warehouses.md) | Create, size, suspend & resume virtual warehouses |
| 03 | [Databases, schemas & tables](04-databases.md) | Organize data; create your first table |
| 04 | [Loading data](05-loading-data.md) | Stages, file formats, and `COPY INTO` |
| 05 | [Querying & analytics](06-querying.md) | SQL you'll actually use day-to-day |

*(More lessons are added as the workshop grows.)*

---

## Before you start

You'll need:

- A **Snowflake account** (your instructor provides the account URL — see the
  workshop account in `.env.public`).
- A **username and password** from your instructor.
- A modern web browser. That's it — everything runs in Snowsight, nothing to
  install.

Ready? Start with **[Lesson 00 — Setup & first query →](01-setup.md)**
