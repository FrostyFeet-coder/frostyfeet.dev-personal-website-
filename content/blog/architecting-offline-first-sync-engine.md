---
title: "Architecting an Offline-First Sync Engine for Rich Media"
date: "2026-08-19"
description: "How to safely sync offline database records and media files without ending up with broken references."
---

Building an offline-first mobile application is notoriously difficult. If your app only syncs JSON text, you can rely on robust conflict resolution algorithms or CRDTs. But what happens when your app collects rich media—like photos, voice recordings, or videos—while completely disconnected from the internet?

Recently, I was looking back at the Android sync engine I helped architect. We use Android's `WorkManager` to handle background synchronization. Our users often go offline for days, collect dozens of audio recordings, and then briefly connect to a patchy network to sync.

Handling this sync requires a very specific sequence of operations. If you get the order wrong, you end up with corrupted data on the server.

## The Problem: The "Broken Reference" Race Condition

When a user completes a task offline, two things are saved to the device:
1. A local database row containing metadata (e.g., `{"status": "COMPLETED", "duration": 15}`).
2. A binary file saved to local storage (e.g., `recording_1.wav`).

A naive sync engine might try to upload the database row first. If the row uploads successfully, the server marks the task as "Done." Then, the app attempts to upload the `.wav` file. 

But what if the user walks into a subway or the app is killed by the OS? The server now has a database record pointing to a file that doesn't exist. The data is corrupted.

## The Solution: The 5-Step Sync Sequence

To guarantee data integrity across an unstable connection, the sync engine must follow a strict, sequential pipeline. Here is the architecture we use in our `SyncWorker`:

### 1. Prune the Local Database
Before talking to the network, clean the house. We scan the local Room database to mark any tasks that have exceeded their deadline as `EXPIRED`. This prevents the app from trying to upload answers the server will reject anyway.

### 2. Upload Media Files First
We iterate over all completed tasks and extract the local file paths. We upload **only the binary files** to the server's blob storage (like AWS S3 or GCP Cloud Storage). 
- If a file upload fails or the network drops, the worker stops. The database row remains safely marked as "offline completed" on the device.
- We also add safety checks: if a file exceeds our `MAX_FILE_SIZE_BYTES` limit, we log a warning and skip it rather than repeatedly crashing the worker with `OutOfMemory` errors.

### 3. Upload Completed Database Records
Once—and only once—all the files for a specific task have been successfully uploaded and acknowledged by the blob store, we upload the JSON database row. 

Because the files are already safely in the cloud, the database mutation on the backend is completely safe.

### 4. Download New Database Records
After pushing the user's progress up, we pull the newest state down. We ask the server: *"Give me the JSON metadata for any new tasks assigned to this user."* 

These records are saved to the local Room database. They often contain references to new media files the user will need to view (e.g., an image to annotate).

### 5. Download Input Media Files
Finally, we scan the newly downloaded database records to extract URLs for required media. We download these images/audio files to local storage. Once they are fully downloaded, the UI is updated to allow the user to start working on the new tasks entirely offline.

## Why This Architecture Works

By explicitly decoupling the **Payload (Media)** from the **Pointer (Database Row)**, and ensuring the Payload always arrives before the Pointer, we eliminate broken references. 

Furthermore, by wrapping this entire sequence in a single Coroutine inside a `WorkManager` job, Android automatically handles the retries, wake-locks, and backoff policies if the network drops midway through step 2.

Offline-first isn't just about caching data; it's about choreographing network requests to ensure that partial failures never result in permanent corruption.
