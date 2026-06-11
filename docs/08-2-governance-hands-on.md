---
title: "DEMO 13 — Governance hands-on: row & column policies"
---

# DEMO 13 — Governance hands-on: row & column policies

**Goal:** protect the `gold.Sales` table you built in
[DEMO 04](03-1-data-engineering-workshop.md) with a **masking policy**
(column) and a **row access policy** (rows), then prove they work by
**running as** the read-only `BAC_DA_ROLE` — same queries, different
answers per role.

[← Back to Home](index.md)

---

## Meet `BAC_DA_ROLE` — the reader

So far you've been `BAC_DE_ROLE`, the builder. `BAC_DA_ROLE` is a **data
analyst** role: it can read and query — and that's about it. No creating
tables, no loading data, no changing anything. Most real users look like
this.

### Run as the analyst — feel the ceiling

`USE ROLE` is your **run as**: same user (you), different hat. Try it:

```sql
USE ROLE BAC_DA_ROLE;
USE WAREHOUSE WORKSHOP_WH;

SELECT COUNT(*) FROM USERXX.gold.Sales;           -- ✅ works — reading is the job

CREATE TABLE USERXX.gold.test_table (id INT);     -- ❌ fails
DELETE FROM USERXX.gold.Sales;                    -- ❌ fails
SELECT * FROM USERXX.silver.VBAK LIMIT 5;         -- ❌ fails — no grant on silver
```

The errors are the governance working. Note the last one: DA can see the
serving layer (`gold`) but not the engineering layers (`bronze`, `silver`) —
exactly the schema-per-layer access design from
[Lesson 02](02-1-stages.md).

```sql
USE ROLE BAC_DE_ROLE;   -- back to the builder hat for the next steps
```

---

## Step 1 — Masking policy: hide the customer from analysts

`SoldToParty` is the customer code — commercially sensitive. Analysts can
aggregate, but shouldn't read it. Create the policy and attach it to the
column:

```sql
USE SCHEMA USERXX.gold;

CREATE OR REPLACE MASKING POLICY mask_customer AS (val STRING)
  RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() = 'BAC_DE_ROLE' THEN val   -- builders see it
    ELSE '***MASKED***'                            -- everyone else doesn't
  END;

ALTER TABLE gold.Sales
  MODIFY COLUMN SoldToParty SET MASKING POLICY mask_customer;
```

> If `CREATE MASKING POLICY` fails on privileges, these are Enterprise
> features gated behind `CREATE MASKING POLICY` / `APPLY` grants — ask the
> instructor (or use `BAC_PLATFORM_ROLE`).

### Test it — run as each role

```sql
-- As the builder:
USE ROLE BAC_DE_ROLE;
SELECT SoldToParty, ItemNetValue FROM gold.Sales LIMIT 5;   -- real codes

-- As the analyst:
USE ROLE BAC_DA_ROLE;
SELECT SoldToParty, ItemNetValue FROM gold.Sales LIMIT 5;   -- ***MASKED***
```

Same table, same query, different truth per role. Note what still works for
the analyst:

```sql
-- Aggregation is fine — it just groups the masked value:
SELECT SoldToParty, SUM(DISTINCT ItemNetValue) AS total
FROM gold.Sales
GROUP BY SoldToParty
ORDER BY total DESC LIMIT 5;
```

One row, `***MASKED***`, with the grand total — the analyst can still get
*numbers*, just not *identities*. If analysts should distinguish customers
without naming them, mask with `SHA2(val)` instead of a constant.

> (Why `SUM(DISTINCT ...)`? `gold.Sales` is at **schedule-line grain** —
> plain `SUM(ItemNetValue)` would double-count, as you learned in
> [DEMO 05](03-2-more-data-engineering.md).)

---

## Step 2 — Row access policy: analysts see one sales org only

First, look at what orgs exist (as DE):

```sql
USE ROLE BAC_DE_ROLE;
SELECT SalesOrg, COUNT(*) FROM gold.Sales GROUP BY SalesOrg;
```

Pick one your data actually has — we'll use `'1000'` below; substitute
yours. Now the policy:

```sql
CREATE OR REPLACE ROW ACCESS POLICY rap_salesorg AS (org STRING)
  RETURNS BOOLEAN ->
  CURRENT_ROLE() = 'BAC_DE_ROLE'    -- builders: every row
  OR org = '1000';                  -- everyone else: org 1000 only

ALTER TABLE gold.Sales
  ADD ROW ACCESS POLICY rap_salesorg ON (SalesOrg);
```

### Test it — the count changes with the hat

```sql
USE ROLE BAC_DE_ROLE;
SELECT COUNT(*) AS rows_de FROM gold.Sales;     -- everything

USE ROLE BAC_DA_ROLE;
SELECT COUNT(*) AS rows_da FROM gold.Sales;     -- only org 1000
SELECT DISTINCT SalesOrg FROM gold.Sales;       -- proof: one value
```

The analyst gets **no error and no hint** — the other orgs simply don't
exist from where they stand.

> 💡 In production, hard-coding roles in the policy doesn't scale. Use a
> **mapping table** (`role_name`, `allowed_org`) and have the policy
> `EXISTS`-check it — then onboarding a new team is an `INSERT`, not a
> policy change.

### The payoff: every tool inherits it

This is the platform-level argument from the
[previous lesson](08-1-governance.md), live: the policy lives **on
`gold.Sales`**, so a worksheet, a notebook, a semantic model, a dashboard —
anything that queries the table as `BAC_DA_ROLE` — sees org 1000 only,
with zero governance code in the tool.

Want to see it with the chat agent? The agent from
[DEMO 12](07-2-snowflake-intelligence-hands-on.md) reads `agent.Sales`, so
attach the same two policies there (columns: `Customer`, `SalesOrg`) and
ask it *"total revenue by sales org"* as the analyst — the generated SQL
runs under your role, and the answer quietly shrinks to org 1000.

> 📝 **Good to know — `EXECUTE AS`:** stored procedures
> ([DEMO 07](03-4-task-and-schedule.md)) run either as the **caller**
> (`EXECUTE AS CALLER` — policies see the caller's role, so DA gets masked,
> filtered data) or as the **owner** (`EXECUTE AS OWNER` — policies see the
> owner's role, so full data even when DA calls it). That's why your
> pipeline tasks see everything — and why owner's-rights procedures granted
> to readers should be reviewed as carefully as `GRANT`s.

---

## Step 3 — Audit: who is governed by what?

```sql
USE ROLE BAC_DE_ROLE;

-- Which policies are attached to gold.Sales?
SELECT POLICY_NAME, POLICY_KIND, REF_COLUMN_NAME
FROM TABLE(USERXX.INFORMATION_SCHEMA.POLICY_REFERENCES(
  REF_ENTITY_NAME => 'USERXX.GOLD.SALES',
  REF_ENTITY_DOMAIN => 'TABLE'
));

-- Who has been querying, as which role?
SELECT user_name, role_name, query_text, start_time
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE query_text ILIKE '%gold.sales%'
ORDER BY start_time DESC LIMIT 10;
```

(Account-wide and column-level audit lives in
`SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` — the lineage/audit trail from the
[previous lesson](08-1-governance.md).)

---

## Step 4 — Clean up

Policies must be **detached before they can be dropped**:

```sql
USE ROLE BAC_DE_ROLE;

ALTER TABLE gold.Sales DROP ROW ACCESS POLICY rap_salesorg;
ALTER TABLE gold.Sales MODIFY COLUMN SoldToParty UNSET MASKING POLICY;

DROP ROW ACCESS POLICY IF EXISTS rap_salesorg;
DROP MASKING POLICY    IF EXISTS mask_customer;
```

(Leave them in place if you want tomorrow's dashboard demo to show
per-role numbers — just remember they're there when totals look "wrong".)

---

## Recap

```
                          gold.Sales
                              │
              ┌───────────────┼────────────────┐
              │ mask_customer │ rap_salesorg   │   ← policies live ON the table
              │ (SoldToParty) │ (SalesOrg)     │
              └───────────────┼────────────────┘
        ┌─────────────┬───────┴──────┬──────────────┐
   worksheet      notebook      Intelligence     Streamlit
        └─────────────┴──────────────┴──────────────┘
          BAC_DE_ROLE: all rows, real customers
          BAC_DA_ROLE: org 1000, ***MASKED***  — everywhere, automatically
```

| Action | SQL |
|--------|-----|
| Run as another role | `USE ROLE BAC_DA_ROLE;` |
| Mask a column | `CREATE MASKING POLICY` + `ALTER TABLE ... SET MASKING POLICY` |
| Filter rows | `CREATE ROW ACCESS POLICY` + `ALTER TABLE ... ADD ROW ACCESS POLICY` |
| What's attached? | `INFORMATION_SCHEMA.POLICY_REFERENCES(...)` |
| Detach / drop | `DROP ROW ACCESS POLICY` / `UNSET MASKING POLICY`, then `DROP` |

**Next:** the data is engineered, chat-enabled, and now governed. Last step:
put a **dashboard** on it — which every policy you just wrote will quietly
keep honest.

[← Back to Home](index.md) | [Next: Streamlit introduction →](09-1-streamlit-introduction.md)
