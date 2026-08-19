---
title: "Understanding Atomic Operations in Android Room"
date: "2026-08-19"
description: "Exploring the importance of atomicity when performing multiple database operations in Android Room."
---

When building offline-first Android applications, managing the local database state correctly is critical. Over the past year, one of the key lessons I learned while working with the Room persistence library is the importance of **atomicity**.

## The Problem: Interrupted Flows

Consider a scenario where a user submits a new organization code in an app. The app needs to perform two separate local operations:
1. Update the user's authentication token.
2. Delete the old "profile task" from the local database.

If these operations are executed sequentially without being grouped together, what happens if the app crashes, is killed by the OS, or the user forcefully closes it right after step 1 but before step 2?

You end up in an **inconsistent state**. The user has the new token, but the old profile task remains in the database. This can lead to bugs that are extremely hard to reproduce, like the app trying to sync an old profile task using a new organization token.

## The Solution: `@Transaction`

In SQLite and Room, an atomic operation guarantees that a series of database operations either all succeed entirely, or all fail entirely. If something goes wrong halfway through, the database is rolled back to its previous state.

Room makes this easy using the `@Transaction` annotation. 

```kotlin
@Dao
interface UserDao {
    @Query("UPDATE user_table SET token = :newToken WHERE id = :userId")
    suspend fun updateToken(userId: String, newToken: String)

    @Query("DELETE FROM task_table WHERE task_type = 'profile'")
    suspend fun deleteProfileTasks()

    @Transaction
    suspend fun updateTokenAndClearProfile(userId: String, newToken: String) {
        // Either both of these succeed, or neither do.
        updateToken(userId, newToken)
        deleteProfileTasks()
    }
}
```

By wrapping the logic in a transaction, we ensure that the local database remains perfectly consistent, regardless of app interruptions. Designing for failure—assuming the app *will* be killed at the worst possible moment—is a crucial mindset for robust mobile development.
