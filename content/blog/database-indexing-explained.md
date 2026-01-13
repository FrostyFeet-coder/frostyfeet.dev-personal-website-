---
title: "Database Indexing Explained: Speed Up Your Queries"
date: 2025-01-05T00:00:00.000+05:30
description: "Understanding how database indexes work and when to use them for optimal query performance"
tags: ["database", "sql", "performance", "backend"]
---

If you've ever wondered why some queries are blazing fast while others take forever, the answer often lies in **indexing**. Let's dive into how database indexes work and how to use them effectively.

## What is an Index?

An index is a data structure that improves the speed of data retrieval operations on a database table. Think of it like the index in a book—instead of reading every page to find a topic, you look it up in the index and go directly to the right page.

## How Indexes Work

Without an index, the database performs a **full table scan**:

```sql
-- Without index on 'email' column
SELECT * FROM users WHERE email = 'ansh@example.com';
-- Scans ALL rows: O(n) time complexity
```

With an index, the database uses a **B-tree structure**:

```sql
-- With index on 'email' column
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'ansh@example.com';
-- Uses index: O(log n) time complexity
```

## Types of Indexes

### 1. Single-Column Index
```sql
CREATE INDEX idx_users_email ON users(email);
```

### 2. Composite Index (Multi-Column)
```sql
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);
```

**Important**: Column order matters! This index helps with:
- `WHERE user_id = 1`
- `WHERE user_id = 1 AND created_at > '2025-01-01'`

But NOT with:
- `WHERE created_at > '2025-01-01'` (first column not used)

### 3. Unique Index
```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

### 4. Partial Index (PostgreSQL)
```sql
CREATE INDEX idx_active_users ON users(email) WHERE active = true;
```

## When to Create Indexes

✅ **Create indexes on:**
- Primary keys (automatic)
- Foreign keys
- Columns used in WHERE clauses frequently
- Columns used in JOIN conditions
- Columns used in ORDER BY

❌ **Avoid indexes on:**
- Small tables (full scan might be faster)
- Columns with low cardinality (e.g., boolean flags)
- Columns that are frequently updated
- Tables with heavy INSERT operations

## The Cost of Indexes

Indexes aren't free:

```
Storage: Indexes require disk space
Writes: INSERT/UPDATE/DELETE become slower
Maintenance: Indexes need to be rebuilt occasionally
```

## Analyzing Query Performance

Use `EXPLAIN` to understand how your queries use indexes:

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'ansh@example.com';
```

Look for:
- **Index Scan** ✅ - Using index
- **Seq Scan** ⚠️ - Full table scan

## Best Practices

1. **Don't over-index** - Each index has overhead
2. **Use composite indexes wisely** - Order columns by selectivity
3. **Monitor slow queries** - Use query logs to identify candidates
4. **Regularly analyze tables** - Keep statistics updated

## Conclusion

Indexing is one of the most powerful tools for database optimization. Understanding when and how to use indexes can dramatically improve your application's performance.

Happy optimizing! 🚀
