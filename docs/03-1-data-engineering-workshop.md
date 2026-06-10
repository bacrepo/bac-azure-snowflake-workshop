---
title: "Lesson 04 — Data engineering workshop"
---

# Lesson 04 — Data engineering workshop

[← Back to Home](index.md)

---

## Setup — create schemas (medallion architecture)

Each schema is a layer in the pipeline. Data flows left to right, gaining value
at every step.

```
raw_azure → ext → bronze → silver → gold
(stage)    (all   (pick    (rename   (join,
            cols)  what     to biz    enrich,
                   matters) names)    serve)
```

```sql
USE DATABASE USER00_DB;

CREATE SCHEMA IF NOT EXISTS raw_azure;   -- external stage lives here
CREATE SCHEMA IF NOT EXISTS ext;         -- external tables (all columns)
CREATE SCHEMA IF NOT EXISTS bronze;      -- selected columns only
CREATE SCHEMA IF NOT EXISTS silver;      -- conformed / business-friendly names
CREATE SCHEMA IF NOT EXISTS gold;        -- joined, enriched, query-ready
```

---

## Step 1 — External stage (`raw_azure`)

Point Snowflake at the Azure Data Lake container.

```sql
USE SCHEMA raw_azure;

CREATE OR REPLACE STAGE stage_azure
  URL = 'azure://dlsbacpocsnowflake.blob.core.windows.net/rawbgc/'
  CREDENTIALS = ( AZURE_SAS_TOKEN = 'sv=2026-02-06&ss=bf&srt=sco&sp=rlx&se=2026-06-17T16:48:05Z&st=2026-06-10T08:33:05Z&spr=https&sig=tihpb7YnzyKqm7Va0bvjTD6WCnvNsMiRJi42vkEtTPY%3D' );
LIST @stage_azure;
```

You will see files like `VBAK.parquet`, `VBAP.parquet`, `VBEP.parquet`, and more.

---

## Step 2 — External tables (`ext`)

Source files are **Parquet** → create **external tables**.  
Snowflake reads directly from the stage — no data is copied.

These tables keep **every column** from SAP. Who needs all of them? Nobody —
but we keep them here as the raw truth, and pick what matters in the next layer.

### SAP table: **VBAK** — Sales Document: Header Data

Full columns in source:
`MANDT, VBELN, ERDAT, ERZET, ERNAM, ANGDT, BNDDT, AUDAT, VBTYP, TRVOG,
AUART, AUGRU, GWLDT, SUBMI, LIFSK, FAKSK, NETWR, WAERK, VKORG, VTWEG,
SPART, VKGRP, VKBUR, GSBER, GSKST, GUEBG, GUEEN, KNUMV, VDATU, VPRGR,
AUTLF, VBKLA, VBKLT, KALSM, VSBED, FAKSD, BSARK, KOSTL, STAFO, STWAE,
AEDAT, KVGR1, KVGR2, KVGR3, KVGR4, KVGR5, KOKRS, KKBER, KNKLI, GRUPP,
SBGRP, CTLPC, CMWAE, CMFRE, CMNUP, CMNGV, AMTBL, HITYP_PR, ABRVW,
VGBEL, VGTYP, OBJNR, BUKRS_VF, TAXK1, XBLNR, ZUONR, KUNNR, BSTNK,
BSTDK, FMBDAT, VSNMR_V, HANDLE, PROLI, CONT_DG, CRM_GUID`

```sql
USE SCHEMA ext;

CREATE OR REPLACE EXTERNAL TABLE VBAK
  USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
    FROM TABLE(
      INFER_SCHEMA(
        LOCATION => '@raw_azure.stage_azure/',
        FILE_FORMAT => 'raw.format_parquet',
        FILES => 'VBAK.parquet'
      )
    )
  )
  LOCATION = @raw_azure.stage_azure/
  FILE_FORMAT = raw.format_parquet
  PATTERN = 'VBAK.parquet';

SELECT * FROM ext.VBAK LIMIT 10;
```

### SAP table: **VBAP** — Sales Document: Item Data

This is where the **detail-level amount** lives (`NETWR` per item line).

Full columns in source:
`MANDT, VBELN, POSNR, MATNR, MATWA, PMATN, CHARG, MATKL, ARKTX, PSTYV,
POSAR, LFREL, FKREL, ZIEME, ZMENG, KWMENG, UMVKZ, UMVKN, VRKME, MEINS,
NTGEW, BRGEW, GEWEI, VOLUM, VOLEH, NETWR, WAERK, ANTLF, KZTLF, CHSPL,
WERKS, LGORT, VSTEL, ROUTE, STKEY, STDAT, OBJNR, MTVFP, PRODH, PRCTR,
VKGRP, VKBUR, MVGR1, MVGR2, MVGR3, MVGR4, MVGR5, ABGRU, CMPRE, CMTFG,
ERDAT, ERNAM, AEDAT`

```sql
CREATE OR REPLACE EXTERNAL TABLE VBAP
  USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
    FROM TABLE(
      INFER_SCHEMA(
        LOCATION => '@raw_azure.stage_azure/',
        FILE_FORMAT => 'raw.format_parquet',
        FILES => 'VBAP.parquet'
      )
    )
  )
  LOCATION = @raw_azure.stage_azure/
  FILE_FORMAT = raw.format_parquet
  PATTERN = 'VBAP.parquet';

SELECT * FROM ext.VBAP LIMIT 10;
```

### SAP table: **VBEP** — Sales Document: Schedule Line Data

Full columns in source:
`MANDT, VBELN, POSNR, ETENR, ETTYP, LFREL, PLART, ABART, EDATU, WMENG,
BMENG, MEINS, VRKME, WADAT, MBDAT, TDDAT, LDDAT, WADAT_IST, WAUHR,
MBUHR, LIFSP, BANFN, BSART, BNFPO, ESTKZ, RSNUM, EMPST, ABLAD, MATKL,
BWART, VLAUF, VLAUE, DRUHR, DRDAT, LMENG, TZONUO`

```sql
CREATE OR REPLACE EXTERNAL TABLE VBEP
  USING TEMPLATE (
    SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*))
    FROM TABLE(
      INFER_SCHEMA(
        LOCATION => '@raw_azure.stage_azure/',
        FILE_FORMAT => 'raw.format_parquet',
        FILES => 'VBEP.parquet'
      )
    )
  )
  LOCATION = @raw_azure.stage_azure/
  FILE_FORMAT = raw.format_parquet
  PATTERN = 'VBEP.parquet';

SELECT * FROM ext.VBEP LIMIT 10;
```

### Verify

```sql
SELECT * FROM ext.VBAK LIMIT 5;
SELECT * FROM ext.VBAP LIMIT 5;
SELECT * FROM ext.VBEP LIMIT 5;
```

> **50–70+ columns per table.** Who needs all of them? Not the analyst, not the
> dashboard. That's what bronze is for — **pick only what matters**.

---

## Step 3 — Bronze (selected columns)

**External → Bronze:** keep the columns the business actually uses.

### VBAK — header (20 columns out of 70+)

```sql
USE SCHEMA bronze;

CREATE OR REPLACE TABLE bronze.VBAK AS
SELECT
    VBELN,       -- Sales Document Number
    AUART,       -- Sales Document Type
    VBTYP,       -- SD Document Category
    ERDAT,       -- Created On
    AUDAT,       -- Document Date
    VKORG,       -- Sales Organization
    VTWEG,       -- Distribution Channel
    SPART,       -- Division
    VKBUR,       -- Sales Office
    VKGRP,       -- Sales Group
    KUNNR,       -- Sold-to Party
    NETWR,       -- Net Value
    WAERK,       -- Document Currency
    VDATU,       -- Requested Delivery Date
    BSTNK,       -- Customer Purchase Order Number
    BSTDK,       -- Customer PO Date
    AUGRU,       -- Order Reason
    LIFSK,       -- Delivery Block
    FAKSK,       -- Billing Block
    KNUMV        -- Condition Number
FROM ext.VBAK;
```

### VBAP — items (13 columns out of 50+)

```sql
CREATE OR REPLACE TABLE bronze.VBAP AS
SELECT
    VBELN,       -- Sales Document
    POSNR,       -- Item Number
    MATNR,       -- Material Number
    ARKTX,       -- Item Description
    MATKL,       -- Material Group
    PSTYV,       -- Item Category
    WERKS,       -- Plant
    NETWR,       -- Net Value (item level!)
    WAERK,       -- Currency
    KWMENG,      -- Order Quantity
    VRKME,       -- Sales Unit
    MEINS,       -- Base Unit
    ABGRU        -- Rejection Reason
FROM ext.VBAP;
```

### VBEP — schedule lines (14 columns out of 36)

```sql
CREATE OR REPLACE TABLE bronze.VBEP AS
SELECT
    VBELN,       -- Sales Document
    POSNR,       -- Item Number
    ETENR,       -- Schedule Line Number
    ETTYP,       -- Schedule Line Category
    EDATU,       -- Schedule Line Date
    WMENG,       -- Requested Quantity
    BMENG,       -- Confirmed Quantity
    LMENG,       -- Required Quantity
    MEINS,       -- Base Unit of Measure
    MBDAT,       -- Material Availability Date
    LDDAT,       -- Loading Date
    TDDAT,       -- Transportation Planning Date
    WADAT,       -- Goods Issue Date
    BANFN        -- Purchase Requisition
FROM ext.VBEP;
```

---

## Step 4 — Silver (conformed / business-friendly names)

**Bronze → Silver:** rename SAP technical codes to readable business names.
Anyone can understand `SalesDocument` — nobody remembers what `VBELN` means.

### VBAK

```sql
USE SCHEMA silver;

CREATE OR REPLACE TABLE silver.VBAK AS
SELECT
    VBELN  AS SalesDocument,
    AUART  AS SalesDocumentType,
    VBTYP  AS DocumentCategory,
    ERDAT  AS CreatedOn,
    AUDAT  AS DocumentDate,
    VKORG  AS SalesOrg,
    VTWEG  AS DistributionChannel,
    SPART  AS Division,
    VKBUR  AS SalesOffice,
    VKGRP  AS SalesGroup,
    KUNNR  AS SoldToParty,
    NETWR  AS NetValueHeader,
    WAERK  AS DocumentCurrency,
    VDATU  AS RequestedDeliveryDate,
    BSTNK  AS CustomerPONumber,
    BSTDK  AS CustomerPODate,
    AUGRU  AS OrderReason,
    LIFSK  AS DeliveryBlock,
    FAKSK  AS BillingBlock,
    KNUMV  AS ConditionNumber
FROM bronze.VBAK;
```

### VBAP

```sql
CREATE OR REPLACE TABLE silver.VBAP AS
SELECT
    VBELN   AS SalesDocument,
    POSNR   AS ItemNumber,
    MATNR   AS Material,
    ARKTX   AS ItemDescription,
    MATKL   AS MaterialGroup,
    PSTYV   AS ItemCategory,
    WERKS   AS Plant,
    NETWR   AS ItemNetValue,
    WAERK   AS DocumentCurrency,
    KWMENG  AS OrderQuantity,
    VRKME   AS SalesUnit,
    MEINS   AS BaseUnit,
    ABGRU   AS RejectionReason
FROM bronze.VBAP;
```

### VBEP

```sql
CREATE OR REPLACE TABLE silver.VBEP AS
SELECT
    VBELN  AS SalesDocument,
    POSNR  AS ItemNumber,
    ETENR  AS ScheduleLineNumber,
    ETTYP  AS ScheduleLineCategory,
    EDATU  AS ScheduleLineDate,
    WMENG  AS RequestedQuantity,
    BMENG  AS ConfirmedQuantity,
    LMENG  AS RequiredQuantity,
    MEINS  AS BaseUnitOfMeasure,
    MBDAT  AS MaterialAvailabilityDate,
    LDDAT  AS LoadingDate,
    TDDAT  AS TransportationPlanningDate,
    WADAT  AS GoodsIssueDate,
    BANFN  AS PurchaseRequisition
FROM bronze.VBEP;
```

---

## Step 5 — Gold (joined, enriched, query-ready)

**Silver → Gold:** join header + items + schedule lines into one table that
answers business questions directly.

```sql
USE SCHEMA gold;

CREATE OR REPLACE TABLE gold.Sales AS
SELECT
    -- Keys (schedule-line grain)
    S.SalesDocument,
    S.ItemNumber,
    S.ScheduleLineNumber,

    -- Customer / Org
    H.SoldToParty,
    H.SalesOrg,
    H.DistributionChannel,
    H.Division,
    H.SalesOffice,
    H.SalesGroup,

    -- Product (from VBAP)
    I.Material,
    I.ItemDescription,
    I.MaterialGroup,
    I.ItemCategory,
    I.Plant,

    -- Time
    H.DocumentDate,
    H.CreatedOn,
    H.RequestedDeliveryDate,
    S.ScheduleLineDate,
    S.MaterialAvailabilityDate,
    S.GoodsIssueDate,

    -- Amount
    I.ItemNetValue,              -- detail amount (per item line)
    H.NetValueHeader,            -- header amount (whole order)
    H.DocumentCurrency,

    -- Quantities
    I.OrderQuantity,
    I.SalesUnit,
    S.RequestedQuantity,
    S.ConfirmedQuantity,
    S.RequiredQuantity,
    S.BaseUnitOfMeasure,

    -- Context
    H.SalesDocumentType,
    H.DocumentCategory,
    H.OrderReason,
    H.DeliveryBlock,
    H.BillingBlock,
    H.CustomerPONumber,
    I.RejectionReason
FROM silver.VBEP AS S
INNER JOIN silver.VBAP AS I
    ON  S.SalesDocument = I.SalesDocument
    AND S.ItemNumber    = I.ItemNumber
INNER JOIN silver.VBAK AS H
    ON S.SalesDocument = H.SalesDocument;
```

### Verify

```sql
SELECT * FROM gold.Sales LIMIT 10;
```

---

## Summary — the full picture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Azure Blob (ADLS)                                                  │
│  VBAK.parquet  VBAP.parquet  VBEP.parquet  ...                      │
└──────────────┬──────────────────────────────────────────────────────┘
               │  External Stage (raw_azure.stage_azure)
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ext.VBAK / ext.VBAP / ext.VBEP                                     │
│  ALL columns — raw truth, query in place                             │
└──────────────┬──────────────────────────────────────────────────────┘
               │  Pick what matters
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  bronze.VBAK (20 cols) / bronze.VBAP (13 cols) / bronze.VBEP (14)   │
│  Only the columns the business needs                                │
└──────────────┬──────────────────────────────────────────────────────┘
               │  Rename to business-friendly names
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  silver.VBAK / silver.VBAP / silver.VBEP                             │
│  SalesDocument, ItemNetValue, Material, SoldToParty ...              │
└──────────────┬──────────────────────────────────────────────────────┘
               │  Join header + items + schedule lines
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  gold.Sales                                                          │
│  One table — header + items + schedule lines — ready for dashboards  │
└─────────────────────────────────────────────────────────────────────┘
```

| Layer | Schema | What happens | Example |
|-------|--------|-------------|---------|
| **Stage** | `raw_azure` | Connect to Azure Blob | `stage_azure` |
| **Ext** | `ext` | External table — all columns, no copy | `ext.VBAK`, `ext.VBAP`, `ext.VBEP` |
| **Bronze** | `bronze` | Pick only useful columns | `bronze.VBAP` (13 cols from 50+) |
| **Silver** | `silver` | Rename SAP codes → business names | `NETWR` → `ItemNetValue` |
| **Gold** | `gold` | Join VBAK + VBAP + VBEP, serve | `gold.Sales` |

[← Back to Home](index.md)
