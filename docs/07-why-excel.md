---
title: "Lesson 07 — Why Excel is difficult"
---

# Lesson 07 — Why Excel is difficult

**Goal:** name the real costs of spreadsheet-driven processes — so the case for
moving to Snowflake is concrete, not just "it's modern."

[← Back to Home](index.md)

---

## A short workshop discussion

Think about a report your team maintains in Excel today. As we go through each
point below, ask: *does this happen to us?*

## Where Excel hurts

- **No single source of truth.** The "real" file is whichever copy is in
  someone's inbox. `report_final_v3_REALLY_final.xlsx`.
- **Manual = error-prone.** Copy-paste, dragged formulas, and a wrong range
  silently change the numbers. Nobody sees it until it's wrong downstream.
- **Doesn't scale.** Hundreds of thousands of rows crawl or crash; you can't
  join across files easily.
- **No governance.** No row-level security, no masking, no real audit of who
  changed what.
- **Hard to automate.** Refreshing means a person opening files and clicking.
- **Structure is messy.** Merged cells, multiple sheets, headers mid-table,
  hidden columns — none of it maps cleanly to a database table.

## How Snowflake answers each one

| Excel pain | Snowflake |
| -- | -- |
| Many copies | One governed table, one truth |
| Manual edits | Pipelines & **dynamic tables** refresh themselves |
| Size limits | Scales to billions of rows |
| No security | Roles, masking, row policies, audit |
| Manual refresh | Scheduled / declarative pipelines |
| Mistakes are permanent | **Time travel** undoes them |

> The goal isn't "no more Excel" — it's making Snowflake the **system of record**
> and using Excel/Streamlit only as a *view* on top of governed data.

---

## Recap

- Excel's costs are **truth, errors, scale, governance, and automation**.
- Snowflake addresses each with **governed tables, pipelines, security, and time
  travel**.
- Keep spreadsheets as a *presentation layer*, not the source of truth.

[← Back to Home](index.md)
