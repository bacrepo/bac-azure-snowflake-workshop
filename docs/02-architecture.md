---
title: "Lesson 01 — Snowflake architecture"
---

# Lesson 01 — Snowflake architecture

**Goal:** understand the one idea that makes Snowflake different — the
separation of **storage** and **compute**.

[← Previous: Setup](01-setup.md) · [Home](index.md) · [Next: Warehouses →](03-warehouses.md)

---

## The big idea

Traditional databases bolt storage and compute together: to query more data
faster, you buy a bigger server, and storage and CPU scale together whether you
want them to or not.

Snowflake splits them into **three independent layers**:

```
┌─────────────────────────────────────────────┐
│  Cloud Services                              │  ← brains: auth, optimizer,
│  (security, metadata, query optimization)    │     metadata, transactions
├─────────────────────────────────────────────┤
│  Compute  (Virtual Warehouses)               │  ← muscle: runs your queries.
│  [ WH_A ] [ WH_B ] [ WH_C ] ...              │     Many, independent, resizable
├─────────────────────────────────────────────┤
│  Storage  (your data, on Azure Blob)         │  ← one shared copy of the data
└─────────────────────────────────────────────┘
```

## Why it matters

- **Storage is one shared copy.** All your data lives once in cloud storage
  (on this workshop's account, that's **Azure Blob Storage**). You pay for what
  you store, compressed.
- **Compute is separate and elastic.** A *virtual warehouse* is a cluster of
  compute you spin up to run queries. Need it faster? Resize it. Done? Suspend
  it and stop paying. You can run many warehouses on the **same data at the same
  time** without them slowing each other down.
- **Cloud services tie it together** — login, the query optimizer, metadata,
  and result caching — and you don't manage any of it.

> **Mental model:** the data sits still; you point as much (or as little)
> compute at it as you need, only when you need it.

## On Azure specifically

Snowflake runs on AWS, Azure, and Google Cloud. This workshop's account is
hosted on **Azure** — so under the hood, your storage layer is Azure Blob
Storage and your compute runs on Azure VMs. As a user, the SQL is identical
across clouds; the cloud is just where it physically lives.

---

## Recap

- Three layers: **storage**, **compute (warehouses)**, **cloud services**.
- Storage and compute scale **independently** — that's the whole trick.
- This account lives on **Azure**, but you'll never write Azure-specific SQL.

[← Previous: Setup](01-setup.md) · [Home](index.md) · [Next: Warehouses →](03-warehouses.md)
