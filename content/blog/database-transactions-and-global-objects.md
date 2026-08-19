---
title: "The Danger of Mixing Transactions and Global ORM Objects"
date: "2026-08-19"
description: "Why nested reads and writes must use the transaction scope to avoid race conditions and broken locks."
---

One of the most insidious bugs you can introduce into a backend service is a broken database transaction. It usually happens quietly, passes all unit tests, and only explodes under heavy production load.

Recently, I encountered a subtle but dangerous pattern in our backend related to how we handled global ORM objects inside transactions.

## The Problem

Most Node.js applications use a global wrapper or object for database access (let's call it `db.base`). It makes fetching data easy:
```typescript
const user = await db.base.getSingle('user', { id: userId });
```

When you need atomicity, you wrap operations in a transaction callback that provides a specific transaction object:

```typescript
await db.base.transaction(async (trx) => {
    // 1. Update the user's balance
    await trx.updateSingle('user', { id: userId }, { balance: 0 });
    
    // 2. Fetch the user's metadata to create an audit log
    const meta = await db.base.getSingle('user_meta', { user_id: userId });
    
    // 3. Create the audit log
    await trx.insertRecord('audit', { user_id: userId, data: meta });
});
```

Do you see the bug? 

In step 2, the code uses `db.base.getSingle` instead of `trx.getSingle`. 

## The Consequence

By using the global `db.base` object inside the transaction block, the `getSingle` query is executed **outside** the scope of the current transaction. 

In standard Postgres configurations (like Read Committed isolation), this can cause severe issues:
1. **Race Conditions:** The global read will not see the uncommitted changes made in step 1.
2. **Broken Locks:** If step 2 was a write (`db.base.updateSingle`), it could deadlock the application because the transaction holds a lock on the row, and the global connection is waiting for that lock to be released before it can proceed—but the transaction can't finish until step 2 completes.

## The Lesson

When you open a transaction, **every single read and write** inside that block must use the transaction object (`trx`), not the global database object. 

```typescript
await db.base.transaction(async (trx) => {
    await trx.updateSingle('user', { id: userId }, { balance: 0 });
    
    // CORRECT: Using the transaction object for the read
    const meta = await trx.getSingle('user_meta', { user_id: userId });
    
    await trx.insertRecord('audit', { user_id: userId, data: meta });
});
```

Always review your transaction blocks closely during PRs to ensure global ORM references haven't slipped in!
