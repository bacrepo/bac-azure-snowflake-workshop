---
title: "DEMO 11 — Excel is more difficult than anything"
---

# DEMO 11 — Excel is more difficult than anything

**Goal:** feel the pain yourself. Take one *innocent-looking* workbook from
the stage — `Target.xlsx` — load it successfully… and then watch every
normal, human edit to the file break the pipeline, one trap at a time.

[← Back to Home](index.md)

---

## The setup

CSV took us one `COPY INTO`. Parquet didn't even need a schema. Now try the
format your business actually lives in.

On the stage sits `Target.xlsx` — the sales target workbook. One sheet,
`Target`, with the targets per product per month:

| Year | Product | Jan | Feb | Mar | … | Nov |
|------|---------|----:|----:|----:|---|----:|
| 2025 | A | 69 | 71 | 11 | … | 65 |
| 2025 | B | 99 | 63 | 38 | … | 99 |
| 2025 | C | 45 | 63 | 24 | … | 68 |

We'll load it in a **Snowflake Notebook** (Python cells), because as you saw
in [Lesson 05](05-1-notebooks-snowpark.md), SQL has no native Excel loader
at all.

---

## Step 0 — SQL can't even open it

First, the honest attempt:

```sql
COPY INTO bronze.TARGET
  FROM @raw_azure.stage_azure/Target.xlsx
  FILE_FORMAT = (TYPE = 'CSV');   -- there is no TYPE = 'EXCEL'
```

This fails — or worse, "succeeds" by reading the raw ZIP bytes of the
`.xlsx` as garbage text. Excel isn't a data format, it's an application
file.

| Format | Load effort |
|--------|------------|
| Parquet | `COPY INTO` — schema comes with the file |
| CSV | `COPY INTO` + one file format |
| **Excel** | **a Python program** |

---

## Step 1 — Load it with Snowpark (and it works!)

```python
import pandas as pd
from snowflake.snowpark.context import get_active_session
session = get_active_session()

# Bring the workbook down to the notebook runtime:
session.file.get("@raw_azure.stage_azure/Target.xlsx", "/tmp/")

pdf = pd.read_excel("/tmp/Target.xlsx", sheet_name="Target")
pdf.head()
```

```
   Year Product  Jan  Feb  Mar  Apr  May  Jun  Jul  Aug  Sep  Oct  Nov
0  2025       A   69   71   11   57   90   33   39   28   88   22   65
1  2025       B   99   63   38   74   50   74   12   55   45   62   99
```

Clean header, real numbers, one sheet. One thing to fix before landing:
the shape. Months-as-columns is **human-shaped**; a table wants
**machine-shaped** — one row per Year + Product + Month:

```python
long = pdf.melt(
    id_vars=["Year", "Product"],
    var_name="Month",
    value_name="TargetQty",
)

session.create_dataframe(long).write.save_as_table(
    "bronze.TARGET", mode="overwrite",
)
```

```sql
SELECT * FROM bronze.TARGET LIMIT 5;   -- Year | Product | Month | TargetQty
```

Loaded, unpivoted, landed. *"So Excel is fine?"* — it is, **today**. But a
workbook isn't a database; it's a **shared document that humans edit
freely**. Here's what happens to your working loader over the next few
weeks. Open the file and play along — each trap is one innocent edit.

---

## The traps 🪤

### TRAP 1 — Someone renames the sheet: `Target` → `Traget`

A colleague fixes "a typo" (or makes one). Your loader:

```
ValueError: Worksheet named 'Target' not found
```

💥 **Hard break.** Annoying — but at least it's *loud*. You'll learn to fear
the quiet ones below.

### TRAP 2 — Someone adds a `README` sheet… as the **first** sheet

Helpful documentation! And deadly: any loader that reads "the first sheet"
(`sheet_name=0`, or tools that default to it) now loads the **README text**
instead of the data.

```python
pd.ExcelFile("/tmp/Target.xlsx").sheet_names
# ['README', 'Traget']        ← guess which one your tool reads first
```

💥 **Silent wrong data** — no error, your table is now full of prose.

### TRAP 3 — December arrives: a `Dec` column appears

The most *legitimate* edit possible — it's December, someone fills in the
month. Now the workbook has 14 columns where your `bronze` table expects 13:
every loader with a hardcoded column list, a `usecols="A:M"`, or a fixed
target schema **breaks** — and it breaks **on schedule, every single month**.

> The `melt()` we used survives this one — new month columns just become new
> rows. That's not luck: **long shape is schema-stable, wide shape changes
> schema every month.** One of the most useful modelling lessons of the day.

### TRAP 4 — Someone adds a `Total` row

```
   Year Product   Jan   Feb ...
8  2025       H    92    26 ...
9  2025   Total   593   433 ...     ← helpful, right?
```

No error. Nothing fails. Your `SUM(TargetQty)` is now **exactly double**.

💥 **Silent data corruption** — the worst trap on this page. You find out
when finance asks why targets jumped 100%.

### TRAP 5 — Someone adds a title header above the table

`"SALES TARGET FY2025 — CONFIDENTIAL"`, bold, merged across the top. Pretty
for humans; for pandas the real header is now mid-file:

```
  Unnamed: 0  SALES TARGET FY2025 — CONFIDENTIAL  Unnamed: 2 ...
0       Year                             Product         Jan ...
```

💥 Columns become `Unnamed: *`, every value becomes text (`object`), and
nothing downstream casts cleanly.

### TRAP 6 — Year-end: `Target` becomes `2025`, and a `2024` sheet appears

History! Now the data is **partitioned across sheets by name**, the sheet
your loader points at no longer exists, and next January a `2026` sheet will
appear. Even the *fix* has traps: loop over all sheets — but skip `README`,
and remember sheet names are typed by humans (`Traget`…).

---

## The survival loader

What your one-line `read_excel` looks like after a few months in production:

```python
frames = []
for sheet in pd.ExcelFile("/tmp/Target.xlsx").sheet_names:
    if not sheet.strip().isdigit():          # TRAP 2/6: skip README & friends
        continue
    pdf = pd.read_excel(
        "/tmp/Target.xlsx",
        sheet_name=sheet,
        skiprows=1,                          # TRAP 5: title row on top
    )
    pdf = pdf[pdf["Product"].notna()
              & (pdf["Product"] != "Total")] # TRAP 4: hand-typed totals
    long = pdf.melt(id_vars=["Year", "Product"],
                    var_name="Month", value_name="TargetQty")
    long["TargetQty"] = pd.to_numeric(long["TargetQty"], errors="coerce")
    frames.append(long)                      # TRAP 3: melt absorbs new months

session.create_dataframe(pd.concat(frames)).write.save_as_table(
    "bronze.TARGET", mode="overwrite",
)
```

Count the workarounds — and notice the loader *still* can't defend against
TRAP 1 (a misspelled sheet name silently skipped by the `isdigit()` filter)
without yet more code. **Every one of these guards exists because nothing
stops a human from editing a workbook.** A 2 GB Parquet file with 70 columns
loaded in one statement this morning; a 10 KB Excel file needs this.

---

## The lesson

| | Excel as the **source of truth** | Excel as a **last-mile view** |
|--|-------------------------------|------------------------------|
| Loading | a survival loader, growing forever | — |
| Schema | changes every December | long table — stable for years |
| Trust | whichever copy is newest | reads from a governed table |
| Failure mode | **silent** (Total row, README sheet) | loud, governed, audited |
| Verdict | ❌ what we just suffered through | ✅ perfectly fine |

The goal of this whole workshop is to make sure data **starts** in Snowflake
(or lands there straight from the source) — so nobody ever has to maintain
the survival loader again.

---

## Recap

- There is **no native Excel load** — every workbook needs Python.
- `Target.xlsx` loaded clean on day one — then every *normal human edit*
  broke it: renamed sheet, README-first, December column, Total row, title
  header, year-per-sheet.
- The **loud** breaks (missing sheet) are the kind ones; the **silent** ones
  (README loaded as data, doubled totals) reach the dashboard.
- **Melt wide → long** when you land: months-as-columns changes schema every
  month; one-row-per-month is stable forever.
- The fix isn't a smarter loader. It's **moving the source of truth into the
  platform** — which is exactly what you've built today.

**Next:** the data is clean and governed. Time for the payoff — letting the
business **chat with it**.

[← Back to Home](index.md) | [Next: Snowflake Intelligence →](07-1-snowflake-intelligence.md)
