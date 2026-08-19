---
title: "Catch Explicitly: Why Generic Error Handling Destroys Data"
date: "2026-08-19"
description: "How catching generic errors instead of specific database exceptions led to accidental state deletion during transient timeouts."
---

"Catch all errors and move on." It sounds like defensive programming, but it's actually a massive vulnerability. 

I recently tracked down a bug where a background sync job was aggressively deleting user records that were supposed to be completely healthy. The culprit? A generic `catch (error)` block.

## The Flawed Logic

The background job's logic was designed to clean up orphaned records. It would query the database for a parent record. If the parent record didn't exist, it assumed the child record was orphaned and deleted it.

The code looked something like this:

```typescript
try {
    const parent = await db.getSingle('parent_table', { id: parentId });
    // Process parent...
} catch (error) {
    // We assume the parent doesn't exist anymore!
    await db.delete('child_table', { parent_id: parentId });
}
```

## What Went Wrong

The developer assumed that `getSingle()` throwing an error meant `RecordNotFound`. 

However, `getSingle()` can throw errors for dozens of reasons that have nothing to do with missing data:
- Transient database connection timeouts
- Network partitions
- Postgres running out of memory
- Query syntax errors (after a bad migration)

When the database experienced a tiny, 2-second network hiccup, `getSingle()` threw a `ConnectionTimeoutError`. The generic `catch` block blindly caught it, assumed the record was missing, and permanently deleted the child records. 

A transient infrastructure hiccup was translated into permanent data loss.

## The Fix: Catch Explicitly

You should never perform destructive actions based on a generic error catch. You must assert exactly *what* went wrong before deciding how to handle it.

```typescript
try {
    const parent = await db.getSingle('parent_table', { id: parentId });
    // Process parent...
} catch (error) {
    if (error instanceof DbRecordNotFoundError) {
        // Safe to assume it's orphaned.
        await db.delete('child_table', { parent_id: parentId });
    } else {
        // It's a timeout, a connection error, or something else.
        // DO NOT delete data. Throw the error up the chain so the job retries.
        throw error; 
    }
}
```

By explicitly checking for `DbRecordNotFoundError` (or whatever specific exception your ORM throws for missing rows), we ensure that connection drops simply cause the job to fail and retry later, rather than silently nuking the database.
