---
title: "Lessons Learned Building an Organization Onboarding Flow"
date: "2026-08-19"
description: "War stories from building an org code flow spanning an Android client and a Postgres backend, from 500 cast errors to tricky JWT token claims."
---

One of the more complex features I worked on recently was an "Organization Code" onboarding flow. The requirement sounded simple: allow users entering the Android app to submit a short code (like `ABCD-1234`) to bind their profile to a specific organization, or let them skip it and continue as a "guest". 

Under the hood, however, this spanned our Android client, backend handlers, local Room databases, and token-based authentication. Here are the main problems I faced and how I solved them.

## 1. The PostgreSQL BigInt Cast Crash

When a user submitted an org code, the backend needed to associate them with the organization's specific "Profile Task" (a task used to collect demographic data). 

Initially, my backend route `updateUserOrgCode` kept throwing a massive `500 Internal Server Error`. Digging into the logs, I found a Postgres cast error. The system was trying to insert or evaluate an empty string `""` into a `BIGINT` foreign key column representing the `ONBOARDING_PROJECT_ID`. 

**The Fix:** 
Environment variables aren't strictly typed. If `ONBOARDING_PROJECT_ID` wasn't explicitly defined in the server's environment, it defaulted to an empty string instead of `null` or a sentinel `0`. I added an explicit guard to check if the variable was actually set and numeric before attempting the database query.

## 2. The "User Has Not Registered Fully" 401 Loop

Once the 500 error was fixed, I hit a second wall. Users who selected the "Continue as Guest" option (`PUT /user/org_guest`) were getting blocked on subsequent API calls with a `401 Unauthorized` error: *"User has not registered fully yet."*

In our backend authentication middleware, we check the JWT ID token for a custom claim: `is_registered: true`. This claim is dynamically generated based on whether the `profile_updated_at` column in the database is set. 

Because guest users were skipping the standard demographic profile task, `profile_updated_at` remained `NULL`. When the backend generated their new auth token, it silently flagged them as unregistered.

**The Fix:**
I had to explicitly set `profile_updated_at = NOW()` inside both the `updateUserOrgCode` and `markUserAsGuest` backend handlers. This satisfied the database constraint and ensured the newly minted ID tokens contained the correct `is_registered: true` claim, allowing users to pass the 401 middleware.

## 3. The Client-Side Sync Race Condition

On the Android client, successfully submitting an org code means the user's workload entirely changes. We had to:
1. Receive and save the new auth token.
2. Delete any "stale" default profile tasks sitting in the local Room database.
3. Restart the `SyncWorkManager` to sync the new organization-specific profile tasks.

Initially, if the app was closed between step 1 and 2, the user was left in a ghost state—they had an org token, but were seeing the default profile task. 

**The Fix:**
As discussed in my previous post on [Atomic Operations in Android Room](/blog/understanding-atomic-operations-in-android-room/), I wrapped the token update and task deletion in a `@Transaction`. 

I also had to write a query to find the old profile tasks using `LIKE '%_profile_%'`. While this triggered a full table scan in SQLite, I evaluated the tradeoff and accepted it given the extremely small row count on the client-side database.

## Takeaway

What seems like a simple UI feature ("just add a text box for an org code") often requires deep architectural changes across the entire stack. From mitigating database cast errors to deeply understanding how your JWT middleware infers user states, end-to-end features require you to be a detective at every layer.
