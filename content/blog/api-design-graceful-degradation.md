---
title: "API Design: Graceful Degradation over Hard Errors"
date: "2026-08-19"
description: "Why returning empty states is often better than returning 400 errors for missing relational rows."
---

While designing an API endpoint for a "request a language" feature, I ran into a classic API design dilemma: How should the backend respond when related relational data doesn't exist yet?

## The Scenario

We needed a dedicated endpoint (`GET /api/learn_available_languages`) to fetch a catalog of available languages for a user to learn. 

In our database schema, the available languages are derived from a `task` table, but the user's progress is stored in a `learn_worker_stats` table. The dilemma was: What happens if a new user hits this endpoint, but their `learn_worker_stats` row hasn't been created yet?

## The Trap of the 400 Error

Initially, it's tempting to think strictly relationally: *"The user doesn't have a stats row, therefore they are in an invalid state for this module. I should return a `400 Bad Request` or `404 Not Found`."*

However, this creates a terrible developer experience for the frontend/mobile client. If the Android app receives a `400 Bad Request`, it has to write complex error-handling logic just to figure out *why* it failed, and then likely show an empty UI anyway. 

## Graceful Degradation

The better approach is **Graceful Degradation**. 

Instead of treating the missing `learn_worker_stats` row as a hard error, the API should recognize that a missing row simply means the user hasn't started yet. 

The API should return a `200 OK` status with an empty array:
```json
{
  "available_languages": []
}
```

By decoupling the language catalog response from the strict existence of the user's stats row, we achieved a much more resilient system:
1. The Android client doesn't need to wrap the network call in complex error handling. It just renders the empty state `[]` naturally.
2. The client can call this endpoint safely upon entering the module, without worrying about race conditions regarding when the `learn_worker_stats` row is created.

When designing APIs, always ask: *Can the client do something useful with an empty state?* If yes, return the empty state. Reserve 400/500 errors for truly unrecoverable or illegal states.
