---
title: "Full Table Scans vs Client-Side Constraints in SQLite"
date: "2026-08-19"
description: "When is a full table scan acceptable? A look at SQLite performance on mobile devices."
---

If there's one thing drilled into backend engineers, it's this: **Full table scans are bad.** You should index your columns, optimize your queries, and never use `LIKE '%something%'` on a massive production table. 

But does this strict rule always apply to client-side databases like Android's SQLite (via Room)? Recently, I had to evaluate this exact tradeoff.

## The Query in Question

I needed to find specific tasks in the local database matching a specific string pattern. The query looked like this:

```sql
SELECT * FROM task WHERE name LIKE '%_profile_%'
```

Because of the leading wildcard (`%`), SQLite cannot use a standard B-Tree index. It is forced to evaluate every single row in the `task` table to check for a match. In a backend PostgreSQL database with millions of rows, this would be a catastrophic bottleneck.

## Context is Everything

However, on a mobile client, the rules change. 

1. **Data Constraints:** A local SQLite database for a single user rarely holds millions of rows for an active task queue. In our case, the `task` table would hold a maximum of maybe a few hundred rows at any given time.
2. **Execution Time:** A full table scan of 500 rows in SQLite on modern smartphone flash storage takes less than a millisecond.
3. **Overhead of Alternatives:** The "correct" way to solve this would be to add a dedicated boolean column `is_profile_task`, run a database migration to backfill it, and index it. That introduces schema complexity, migration risk, and APK size overhead.

## The Verdict

We decided to keep the `LIKE '%_profile_%'` query. While it technically triggers a full table scan, it is entirely acceptable given the **client-side database constraints**. 

The lesson here isn't that full table scans are suddenly good, but rather that engineering is about context. Applying massive-scale backend database rules to a mobile client's SQLite instance can lead to over-engineering. Always measure the actual impact before adding complexity.
