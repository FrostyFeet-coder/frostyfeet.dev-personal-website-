---
title: "Clean Architecture: Stop Putting Business Logic in Repositories"
date: "2026-08-19"
description: "Why orchestrating WorkManager jobs and complex data flows belongs in a UseCase, not an Android Repository."
---

If you look at the architecture of most modern Android applications, the **Repository Pattern** is ubiquitous. It’s designed to be the single source of truth for data, hiding the complexity of whether data comes from a local Room database or a remote network call. 

But as apps grow, Repositories often become the dumping ground for complex business logic. I recently had to refactor a massive codebase where this exact anti-pattern had taken root, and it taught me a valuable lesson about the true purpose of a UseCase.

## The Bloated Repository

We had an onboarding flow where a user submitted an "Organization Code." This wasn't just a simple network call; it was a complex orchestration of events:
1. Make an API call to verify the organization code.
2. Save the newly issued authentication token.
3. Delete all stale, default "guest" tasks from the local Room database.
4. Enqueue an Android `WorkManager` background chain to download the new organization's specific tasks.

Initially, all of this logic was stuffed inside an `OnboardingRepository.kt` file. 

This created several massive architectural problems:
- **Tight Coupling:** The repository was now directly dependent on Android's `WorkManager` APIs.
- **Violation of Single Responsibility:** The repository was no longer just fetching or storing data; it was orchestrating application behavior.
- **Testing Nightmare:** To test the network call, we had to mock out the entire `WorkManager` scheduling system.

## The Refactor: Introducing the UseCase

To fix this, I extracted the orchestration logic out of the Repository and into a dedicated `SubmitOrgCodeUseCase`.

A UseCase (or Interactor) in Clean Architecture sits between your ViewModel and your Repositories. Its entire job is to orchestrate business logic.

Here is what the refactored architecture looked like:

```kotlin
class SubmitOrgCodeUseCaseImpl(
    private val authRepository: AuthRepository,
    private val taskRepository: TaskRepository,
    private val workManager: WorkManager
) : SubmitOrgCodeUseCase {

    override suspend operator fun invoke(orgCode: String) {
        // 1 & 2. The AuthRepository ONLY handles the network & token storage
        authRepository.verifyAndSaveOrgCode(orgCode)
        
        // 3. The TaskRepository ONLY handles the database operations
        taskRepository.deleteStaleGuestTasks()
        
        // 4. The UseCase orchestrates the background sync
        val syncRequest = OneTimeWorkRequestBuilder<SyncWorker>().build()
        workManager.enqueue(syncRequest)
    }
}
```

## Why This Matters

By moving the orchestration to `SubmitOrgCodeUseCase`, the Repositories went back to being "dumb" data pipes. 
- `AuthRepository` doesn't know about offline tasks.
- `TaskRepository` doesn't know about network authentication.
- The ViewModel simply calls `submitOrgCodeUseCase(code)` and observes the result.

If you find your Android Repositories enqueuing background workers, firing off analytics events, or coordinating multiple other repositories, it's time to extract that logic. **Repositories map data. UseCases orchestrate behavior.**
