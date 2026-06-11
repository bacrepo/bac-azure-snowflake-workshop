---
title: "Governance — Horizon, policies, catalog & lineage"
---

# Governance — Horizon, policies, catalog & lineage

**Goal:** before you open the platform to more people, decide **who may see
what** — and enforce it once, at the platform level. Meet **row access
policies**, **column-level (masking) policies**, the **data catalog**,
**data lineage**, and the umbrella over all of it: **Snowflake Horizon**.

[← Back to Home](index.md)

---

## Why now?

Look at what you just built. `gold.Sales` and its agent-ready cousin are no
longer tables only engineers query — they feed a **chat agent** anyone can
talk to, and next lesson a **dashboard** anyone can open. The moment data gets easy
to consume, the question stops being *"can people get to the data?"* and
becomes *"are people seeing only what they should?"*

And the audience is changing too: so far you've worked as `BAC_DE_ROLE`,
which can build anything. Real organizations have many more **readers** than
builders — meet **`BAC_DA_ROLE`**, a data analyst role that can `SELECT` and
nothing else. The hands-on next lesson uses it as our test subject.

---

## Govern at the platform level, not in every tool

There are two places you can enforce "Somchai must not see other regions'
sales":

```
  Governance in each tool:                Governance in the platform:

  ┌─────────┐ ┌─────────┐ ┌─────────┐     ┌──────────────────────────────┐
  │ BI tool │ │Dashboard│ │ Export  │     │  Snowflake                   │
  │ filter  │ │ filter  │ │  ???    │     │  policy ON THE TABLE         │
  └────┬────┘ └────┬────┘ └────┬────┘     └──────────────┬───────────────┘
       │           │           │                          │
   3 rule sets, 3 admins,  1 leak ◄──     worksheet, notebook, chat agent,
   forever out of sync                    Streamlit, BI tool, export —
                                          ALL filtered, automatically
```

| | Governance per tool | Governance at platform level |
|--|--------------------|------------------------------|
| Rules defined | once per tool, per dashboard | **once, on the data** |
| New consumption path (e.g. the agent you built) | re-implement the rules | already covered |
| Direct SQL / exports | bypasses everything | still enforced |
| Audit | scattered logs | one `ACCOUNT_USAGE` trail |
| Drift | guaranteed | impossible — there's one rule |

This is the same argument as the semantic model in
[Lesson 07](07-1-snowflake-intelligence.md): push the definition **down to
the platform**, and every tool above it inherits the behaviour. A policy on
`gold.Sales` filters the worksheet, the notebook, **Snowflake Intelligence's
generated SQL**, and the Streamlit dashboard — with zero code in any of them.

---

## Row access policy — *which rows* you see

A **row access policy** is a filter Snowflake attaches to the table itself.
Every query, from anyone, through anything, passes through it.

```sql
-- "DE sees everything; everyone else sees only sales org 1000"
CREATE ROW ACCESS POLICY rap_salesorg
  AS (org STRING) RETURNS BOOLEAN ->
    CURRENT_ROLE() = 'BAC_DE_ROLE' OR org = '1000';

ALTER TABLE gold.Sales
  ADD ROW ACCESS POLICY rap_salesorg ON (SalesOrg);
```

From now on, `SELECT COUNT(*) FROM gold.Sales` returns a **different
number** depending on who asks — and the user can't tell rows were
filtered, can't opt out, and can't work around it with a different tool.

In production the role-to-org rules usually live in a **mapping table**
(`role`, `allowed_org`) that the policy reads — so granting access is an
`INSERT`, not a policy rewrite.

## Column-level security — *which columns* you see clearly

Column access control in Snowflake is a **masking policy**: the column stays
queryable, but its *value* is redacted unless your role qualifies.

```sql
-- "Customer codes are visible to DE; masked for everyone else"
CREATE MASKING POLICY mask_customer AS (val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() = 'BAC_DE_ROLE' THEN val
    ELSE '***MASKED***'
  END;

ALTER TABLE gold.Sales
  MODIFY COLUMN SoldToParty SET MASKING POLICY mask_customer;
```

Same query, different result per role — `GROUP BY SoldToParty` still *works*
for the analyst, it just groups masked values. Variants you'll meet later:
**conditional masking** (mask column A based on column B), **tag-based
masking** (tag a column `PII`, and one policy masks every column carrying
the tag, account-wide).

> 💡 **Policies vs. grants:** RBAC (`GRANT SELECT`) is all-or-nothing on the
> object. Policies are the fine grain *inside* the object — which rows,
> which column values. You need both.

> ⚠️ Row access and masking policies are **Enterprise edition** features —
> one of the reasons editions matter
> ([Lesson 02](02-0-snowflake-platform.md)).

---

## Data catalog — *what data do we have?*

Governance isn't only about hiding things. The flip side: people can't use —
or trust — data they can't **find**. A data catalog answers: *what tables
exist, what do they mean, who owns them, which are sensitive?*

In Snowflake the catalog is built in, fed by metadata you already have plus
what you add:

- **Descriptions** — `COMMENT ON TABLE gold.Sales IS 'SAP sales orders —
  one row per schedule line …'` shows up everywhere the table does. The
  descriptions you wrote in the semantic model do the same job for the AI.
- **Tags** — `cost_center`, `data_owner`, `sensitivity = 'PII'` attached to
  databases, tables, columns. Tags are queryable — *"list every PII column
  in the account"* is one query.
- **Universal Search** in Snowsight — search by business meaning ("sales
  order value") and find tables, columns, views, even marketplace data.
- **Classification** — Snowflake can scan a table and *suggest* which
  columns look like names, emails, national IDs — then you confirm and tag.

## Data lineage — *where did this number come from?*

You built quite a chain today: stage → `ext.VBAK` → `bronze` → `silver` →
`gold.Sales` → `agent.Sales` → dashboard. Lineage is that chain, recorded
automatically and queryable:

- **Snowsight lineage graph** — open a table → **Lineage** and see what
  feeds it and what depends on it, hop by hop.
- **`ACCESS_HISTORY`** (in `SNOWFLAKE.ACCOUNT_USAGE`) — every query's reads
  *and* writes, column-level. Auditors love it; so will you.

Why it matters in practice:

| Question | Answered by lineage |
|----------|--------------------|
| "Can I drop `bronze.VBAK_INCR`?" | What's downstream of it? |
| "This KPI looks wrong" | Trace it back to the exact source file |
| "GDPR: where does customer data flow?" | Follow the PII tag through the graph |
| "Schema change in SAP" | Impact analysis before it breaks prod |

---

## Snowflake Horizon — the umbrella

**Horizon** is Snowflake's name for all of this as one built-in suite — you
don't buy a separate catalog tool, lineage tool, and policy engine and
stitch them together:

| Horizon pillar | What's in it | You've now seen |
|----------------|--------------|------------------|
| **Catalog & discovery** | Universal Search, descriptions, tags, classification | this lesson |
| **Governance & privacy** | RBAC, row access policies, masking, tag-based policies | this lesson + [Lesson 02](02-0-snowflake-platform.md) |
| **Lineage & audit** | lineage graph, `ACCESS_HISTORY`, access audit | this lesson |
| **Security & compliance** | Trust Center (posture checks), network policies, MFA monitoring | mostly admin territory |
| **Collaboration** | secure data sharing, marketplace — share *access*, not copies | a teaser for later |

The point of the umbrella: **the policies travel with the data**. Share a
table with a partner account, query it from a new BI tool, point a new AI
agent at it — Horizon's rules come along, because they live in the platform,
not in the consuming tool.

---

## Recap

- Enforce **who-sees-what once, at the platform level** — every tool above
  (worksheets, notebooks, Intelligence, Streamlit) inherits it. Per-tool
  rules drift; platform rules can't.
- **Row access policy** = which *rows* a role sees. **Masking policy** =
  which *column values* a role sees clearly. Both Enterprise-edition.
- **Catalog** (descriptions, tags, search, classification) makes data
  findable and labeled; **lineage** (`ACCESS_HISTORY`, lineage graph) makes
  every number traceable to its source.
- **Horizon** is the umbrella: catalog + governance + lineage + security +
  sharing, built into the platform.

**Next:** make it real — mask a column, filter rows, and watch the same
query return different answers for `BAC_DE_ROLE` and the read-only
`BAC_DA_ROLE`.

[← Back to Home](index.md) | [Next: Governance hands-on →](08-2-governance-hands-on.md)
