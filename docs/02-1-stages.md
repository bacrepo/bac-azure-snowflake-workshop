---
title: "Stages - Path to Snowflake Data"
---

# Stages - Path to Snowflake Data

**Goal:** understand where files live *before* they become tables — and why
**external stages** are the standard approach in real projects.

[← Back to Home](index.md)

---

## What is a stage?

A **stage** is a pointer to a folder that Snowflake can read files from.
It is the **landing zone** between a raw file (CSV, JSON, Parquet, etc.) and a Snowflake table.

```
  raw file  ──▶  STAGE  ──▶  COPY INTO  ──▶  TABLE
```

There are two types:

| | Internal Stage | External Stage |
| --- | --- | --- |
| Where files live | Inside Snowflake (managed for you) | **Your own cloud storage** (Azure Blob, S3, GCS) |
| Setup effort | One line | URL + credentials |
| Best for | Quick demos, small ad-hoc uploads | **Production pipelines, data lakes, large loads** |
| **Used in practice** | Rarely | **Almost always** |

---

## 1 — Internal Stage

Snowflake stores the files for you. Simple, but limited — you can't share the
files with other tools, and storage is tied to Snowflake.

```sql
CREATE OR REPLACE STAGE my_internal_stage;

-- Upload via Snowsight UI:  open the stage → click "+ Files"
-- Upload via SnowSQL CLI:
PUT file:///path/to/data.csv @my_internal_stage;

LIST @my_internal_stage;   -- see what's staged
```

> Use internal stages for **workshop exercises** and **quick one-off loads**.
> For anything beyond that, use an external stage.

---

## 2 — External Stage  ⭐ Most Used

An **external stage** points Snowflake at storage **you own** — an Azure Blob
container, an AWS S3 bucket, or a GCP Cloud Storage bucket.
The data stays in your cloud; Snowflake just reads it.

**This is the standard pattern in production.**

### The easy setup

**Step 1 — Create a schema for raw data:**

```sql
CREATE DATABASE IF NOT EXISTS LEARN_DB;
CREATE SCHEMA IF NOT EXISTS LEARN_DB.RAW;
USE SCHEMA LEARN_DB.RAW;
```

**Step 2 — Create the external stage:**

**Azure (SAS token — recommended for Azure):**

```sql
CREATE OR REPLACE STAGE az_external_stage
  URL = 'azure://<storage_account>.blob.core.windows.net/<container>/<path>/'
  CREDENTIALS = ( AZURE_SAS_TOKEN = '<your-sas-token>' );

LIST @az_external_stage;   -- verify Snowflake can see your files
```

**AWS (key/secret):**

```sql
CREATE OR REPLACE STAGE s3_external_stage
  URL = 's3://<bucket>/<path>/'
  CREDENTIALS = ( AWS_KEY_ID = '<your-access-key>' AWS_SECRET_KEY = '<your-secret-key>' );

LIST @s3_external_stage;
```

> **Why SAS tokens?** A SAS (Shared Access Signature) token is the simplest way
> to grant Snowflake read access to Azure Blob storage. You generate it in the
> Azure Portal with a specific expiry and permission scope — no service principal
> or Entra ID setup required. **This is the recommended approach for workshops
> and getting started quickly.**

### For production — Storage Integration

In production, replace the inline SAS token with a **Storage Integration**.
This keeps credentials out of SQL and is centrally governed by an admin.

```sql
CREATE STORAGE INTEGRATION az_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = 'AZURE'
  ENABLED = TRUE
  AZURE_TENANT_ID = '<tenant-id>'
  STORAGE_ALLOWED_LOCATIONS = ('azure://<account>.blob.core.windows.net/<container>/');

-- Then create the stage using the integration (no inline credentials):
CREATE OR REPLACE STAGE az_external_stage
  URL = 'azure://<account>.blob.core.windows.net/<container>/<path>/'
  STORAGE_INTEGRATION = az_integration;
```

---

## 3 — The real-world pattern: push files to cloud, read via external stage

In practice, you **don't** upload files to Snowflake directly. Instead, you push
files to cloud storage with a CLI tool, and Snowflake reads them through an
external stage.

```
  your file  ──▶  cloud storage  ──▶  external stage  ──▶  COPY INTO  ──▶  TABLE
```

### Azure (this workshop)

```bash
azcopy copy local-data.csv "https://<account>.blob.core.windows.net/<container>/<path>/local-data.csv?<sas-token>"
```

### AWS

```bash
aws s3 cp local-data.csv s3://<bucket>/<path>/local-data.csv
```

### GCP

```bash
gsutil cp local-data.csv gs://<bucket>/<path>/local-data.csv
```

Then in Snowflake, the external stage picks it up:

```sql
LIST @az_external_stage;                       -- see the file
COPY INTO my_table FROM @az_external_stage;    -- load it
```

> **Note:** Snowflake also has its own CLI — **`snow`** (Snowflake CLI) — which
> can stage files and run queries. It is an alternative to SnowSQL and the
> cloud-specific CLIs above.

---

## Recap

| Concept | Key point |
| --- | --- |
| **Stage** | A pointer to a folder Snowflake reads files from before `COPY INTO` |
| **Internal stage** | Snowflake-managed — quick and easy, but rarely used in production |
| **External stage** | Points at **your** cloud storage — **the standard approach** |
| **SAS token** | Simplest way to authenticate an Azure external stage (recommended to start) |
| **Storage Integration** | Production-grade credential management — no secrets in SQL |
| **The pattern** | `azcopy` / `aws s3 cp` / `gsutil cp` → cloud storage → external stage → `COPY INTO` |

[← Back to Home](index.md)
