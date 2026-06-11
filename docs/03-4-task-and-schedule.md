---
title: "DEMO 07 — Tasks & scheduling"
---

# DEMO 07 — Tasks & scheduling

**Goal:** automate the incremental load — wrap each step in a stored
procedure, then schedule it with a single task or a task graph (DAG).
This completes the data engineering story: a pipeline that runs itself.

[← Back to Home](index.md)

---

## From a hand-run query to an automated pipeline

In the last lesson we applied the delta by hand — **STEP 1** `DELETE` the
overlap, **STEP 2** `INSERT` the new rows. That works once, but nobody wants to
run two statements every hour. We want Snowflake to do it **on a schedule**.

A **task** runs SQL on a timer (or on demand). We'll build it two ways:

| Way | Shape | When |
|-----|-------|------|
| **One big task** | a single task that runs both steps via a stored procedure | the whole load is one logical unit |
| **A task graph (DAG)** | two tasks — STEP 1, then STEP 2 `AFTER` it | you want each step visible, retryable, monitored on its own |

```
  ONE BIG TASK                    TASK GRAPH (DAG)

  ┌──────────────┐               ┌──────────────┐   ┌──────────────┐
  │   BIG TASK   │               │  STEP 1 TASK │──►│  STEP 2 TASK │
  │ CALL proc()  │               │   DELETE     │   │   INSERT     │
  │ DELETE+INSERT│               │  (root,      │   │  (AFTER      │
  └──────────────┘               │   scheduled) │   │   step 1)    │
                                 └──────────────┘   └──────────────┘
```

---

## Script or procedure?

A task runs **one statement**. STEP 1 and STEP 2 are two statements, so each step
becomes its own **stored procedure**, and a task `CALL`s it.

| | Anonymous script | Stored procedure |
|--|------------------|------------------|
| Defined with | `EXECUTE IMMEDIATE $$ ... $$` | `CREATE PROCEDURE ... AS $$ ... $$` |
| Lives | runs once, nothing saved | **saved in the schema**, callable by name |
| A task can run it? | no | **yes** — `CALL` it |

We'll build **three** procedures:

| Procedure | Does |
|-----------|------|
| `bronze.proc_vbak_delete` | STEP 1 — delete the overlap |
| `bronze.proc_vbak_insert` | STEP 2 — append the delta |
| `bronze.proc_vbak` | **master** — calls delete then insert |

---

## Build the procedures

### STEP 1 — `proc_vbak_delete`

```sql
USE SCHEMA bronze;

CREATE OR REPLACE PROCEDURE bronze.proc_vbak_delete()
  RETURNS STRING
  LANGUAGE SQL
AS
$$
BEGIN
  DELETE FROM bronze.VBAK_FULL
  WHERE VBELN IN (SELECT DISTINCT VBELN FROM bronze.VBAK_INCR);

  RETURN 'STEP 1 delete: ' || SQLROWCOUNT || ' rows removed';
END;
$$;
```

### STEP 2 — `proc_vbak_insert`

```sql
CREATE OR REPLACE PROCEDURE bronze.proc_vbak_insert()
  RETURNS STRING
  LANGUAGE SQL
AS
$$
BEGIN
  INSERT INTO bronze.VBAK_FULL (
      VBELN, AUART, VBTYP, AUDAT, VKORG, VTWEG,
      SPART, VKBUR, VKGRP, KUNNR, NETWR
  )
  SELECT
      VBELN, AUART, VBTYP, AUDAT, VKORG, VTWEG,
      SPART, VKBUR, VKGRP, KUNNR, NETWR
  FROM bronze.VBAK_INCR;

  RETURN 'STEP 2 insert: ' || SQLROWCOUNT || ' rows added';
END;
$$;
```

### MASTER — `proc_vbak` (calls both, in order)

```sql
CREATE OR REPLACE PROCEDURE bronze.proc_vbak()
  RETURNS STRING
  LANGUAGE SQL
AS
$$
BEGIN
  CALL bronze.proc_vbak_delete();   -- STEP 1
  CALL bronze.proc_vbak_insert();   -- STEP 2
  RETURN 'VBAK delta applied';
END;
$$;

-- Test the whole thing
CALL bronze.proc_vbak();
```

> Idempotent: STEP 1 clears the overlap first, so re-calling `proc_vbak` lands
> the table in the same state every time.

---

## Step 1 — The BIG TASK (one task, master procedure)

A single task that `CALL`s the **master** procedure. It needs a warehouse and a
schedule.

```sql
CREATE OR REPLACE TASK bronze.task_vbak
  WAREHOUSE = WORKSHOP_WH
  SCHEDULE  = '60 MINUTE'
AS
  CALL bronze.proc_vbak();
```

> **A new task is created SUSPENDED** — it won't run on its schedule until you
> resume it (Step 4). That's on purpose: test before you arm it.

---

## Step 2 — The TASK GRAPH (two step-tasks, chained)

Instead of one task, give each step its own task and **chain** them. The first
task is the **root** (it carries the `SCHEDULE`); the second runs `AFTER` it and
only fires when STEP 1 succeeds.

```sql
-- STEP 1 task — the ROOT, carries the schedule
CREATE OR REPLACE TASK bronze.task_vbak_step1_delete
  WAREHOUSE = WORKSHOP_WH
  SCHEDULE  = '60 MINUTE'
AS
  CALL bronze.proc_vbak_delete();

-- STEP 2 task — runs AFTER step 1 (no schedule of its own)
CREATE OR REPLACE TASK bronze.task_vbak_step2_insert
  WAREHOUSE = WORKSHOP_WH
  AFTER bronze.task_vbak_step1_delete
AS
  CALL bronze.proc_vbak_insert();
```

> **A child task only runs in order if STEP 1 finishes without error** — if the
> `DELETE` fails, the `INSERT` never fires. That ordering is the whole point of
> a task graph.

---

## Step 3 — Execute on demand (run it once, now)

You don't have to wait for the schedule. `EXECUTE TASK` runs it **once, right
now** — even while suspended. For a graph, execute the **root** and the whole
chain runs.

```sql
-- Big task
EXECUTE TASK bronze.task_vbak;

-- Task graph — run the root; step 2 follows automatically
EXECUTE TASK bronze.task_vbak_step1_delete;
```

Check what ran:

```sql
SELECT NAME, STATE, SCHEDULED_TIME, COMPLETED_TIME, RETURN_VALUE, ERROR_MESSAGE
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY())
ORDER BY SCHEDULED_TIME DESC
LIMIT 10;
```

---

## Step 4 — Schedule it ON (resume)

A new task is suspended. **Resume** to start it firing on the timer.

For a **task graph, resume children first, then the root** — Snowflake won't let
you resume a child whose root isn't ready, and resuming the root arms the graph.

```sql
-- Big task
ALTER TASK bronze.task_vbak RESUME;

-- Task graph — children first, root last
ALTER TASK bronze.task_vbak_step2_insert RESUME;
ALTER TASK bronze.task_vbak_step1_delete RESUME;
```

### Change the schedule

Set an interval, or a CRON expression for a specific clock time. You must
**suspend** a task before altering it, then resume.

```sql
ALTER TASK bronze.task_vbak_step1_delete SUSPEND;

-- every 30 minutes
ALTER TASK bronze.task_vbak_step1_delete SET SCHEDULE = '30 MINUTE';

-- or: every day at 02:00 Asia/Bangkok
ALTER TASK bronze.task_vbak_step1_delete
    SET SCHEDULE = 'USING CRON 0 2 * * * Asia/Bangkok';

ALTER TASK bronze.task_vbak_step1_delete RESUME;
```

| CRON `min hr dom mon dow tz` | Runs |
|------------------------------|------|
| `0 * * * *` | top of every hour |
| `0 2 * * *` | every day at 02:00 |
| `0 8 * * 1` | every Monday at 08:00 |
| `*/15 * * * *` | every 15 minutes |

---

## Step 5 — Disable the schedule (turn it OFF)

`SUSPEND` stops a task from firing on its timer — the definition stays, it just
goes dormant. A suspended task costs nothing.

```sql
-- Big task
ALTER TASK bronze.task_vbak SUSPEND;

-- Task graph — suspend the ROOT to stop the whole chain
ALTER TASK bronze.task_vbak_step1_delete SUSPEND;
ALTER TASK bronze.task_vbak_step2_insert SUSPEND;
```

> Suspending the **root** is enough to stop the graph from running on schedule —
> children never fire without it. Suspend the children too if you want them
> fully parked.

Remove them entirely when you're done:

```sql
DROP TASK IF EXISTS bronze.task_vbak_step2_insert;
DROP TASK IF EXISTS bronze.task_vbak_step1_delete;
DROP TASK IF EXISTS bronze.task_vbak;
```

---

## Monitor

```sql
-- State + schedule of every task in the schema
SHOW TASKS IN SCHEMA bronze;

-- Recent runs (success / failure / return value)
SELECT NAME, STATE, SCHEDULED_TIME, RETURN_VALUE, ERROR_MESSAGE
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY())
ORDER BY SCHEDULED_TIME DESC
LIMIT 20;
```

---

## Recap

```
PROCEDURES
   proc_vbak_delete()  = STEP 1 DELETE
   proc_vbak_insert()  = STEP 2 INSERT
   proc_vbak()         = MASTER → CALL delete; CALL insert

  ── ONE BIG TASK ────────────────────────────────────────────────
     task_vbak   SCHEDULE '60 MINUTE'   AS CALL proc_vbak()

  ── TASK GRAPH (DAG) ────────────────────────────────────────────
     task_vbak_step1_delete  (root, SCHEDULE '60 MINUTE')  CALL proc_vbak_delete()
            │ AFTER
            ▼
     task_vbak_step2_insert  (no schedule)                 CALL proc_vbak_insert()

  Lifecycle:  CREATE → (suspended) → EXECUTE TASK (test once)
                                  → RESUME  (schedule ON)
                                  → SUSPEND (schedule OFF)
```

| Action | SQL |
|--------|-----|
| STEP 1 procedure | `CREATE PROCEDURE bronze.proc_vbak_delete() ...` |
| STEP 2 procedure | `CREATE PROCEDURE bronze.proc_vbak_insert() ...` |
| Master procedure | `CREATE PROCEDURE bronze.proc_vbak() ...` (calls both) |
| One big task | `CREATE TASK task_vbak SCHEDULE='60 MINUTE' AS CALL proc_vbak()` |
| Step-1 task (root) | `CREATE TASK task_vbak_step1_delete SCHEDULE='60 MINUTE' AS CALL proc_vbak_delete()` |
| Step-2 task (child) | `CREATE TASK task_vbak_step2_insert AFTER task_vbak_step1_delete AS CALL proc_vbak_insert()` |
| Run once now | `EXECUTE TASK <root>` |
| Schedule ON | `ALTER TASK ... RESUME` (children before root) |
| Schedule OFF | `ALTER TASK ... SUSPEND` (root stops the chain) |
| Monitor | `SHOW TASKS IN SCHEMA bronze` / `TASK_HISTORY()` |

**You've completed the data engineering core.** 🎉 You can now land, clean,
conform, refresh, and schedule data — the foundation everything else stands on.
From here, the workshop puts **AI on top** of the pipeline you just built.

[← Back to Home](index.md) | [Next: Cortex Code →](04-0-cortex-code.md)
