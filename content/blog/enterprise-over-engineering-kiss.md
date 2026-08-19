---
title: "Avoiding Enterprise Over-Engineering: The KISS Principle"
date: "2026-08-19"
description: "Why you don't need another IManager or Factory class, and how to keep your backend functional, flat, and simple."
---

One of my core programming philosophies—and one I rigorously enforce on my backend projects—is an absolute aversion to "Enterprise Over-engineering." 

As developers, we are constantly told to write code that is scalable, decoupled, and future-proof. Unfortunately, this advice is often misinterpreted as an instruction to wrap everything in layers of abstraction.

## The Abstraction Trap

You start with a simple requirement: *Save user preferences to the database.* 

Instead of writing a simple function, the "enterprise" approach looks like this:
1. Create an `IPreferencesManager` interface.
2. Implement a `PreferencesManagerImpl` class.
3. Inject an `IPreferencesRepository` into the manager via a `PreferencesManagerFactory`.
4. Wrap the database call in an `AbstractBaseController`.

Suddenly, a 5-line database query spans 4 files, 3 interfaces, and 2 factories. 

## Keep It Simple, Stupid (KISS)

The reality is that **the best code is the code you never wrote.** 

When designing backend systems, I prefer functional, flat, and extremely simple code structures over deep object-oriented hierarchies. 

Here are the rules I try to follow:
- **Functions over Classes:** If a class only has one method (`execute` or `handle`), it shouldn't be a class. It should just be a function.
- **No YAGNI (You Aren't Gonna Need It):** Never build an abstraction because you *might* need to swap out the underlying technology later. You won't. If the day comes when you actually migrate from Postgres to DynamoDB, you will rewrite the data layer anyway. An `IDatabase` interface won't save you.
- **Modify Only What Is Strictly Necessary:** When fixing a bug, do not introduce heavy design patterns just because they look nice. Keep the blast radius of your diff small.

By keeping code flat and avoiding `IManager`, `Engine`, and `AbstractBase` boilerplate, the codebase remains readable. A new developer can trace the execution path in seconds without needing to CMD+Click through six layers of interfaces. 

Write code to solve today's problem, not a hypothetical problem three years from now.
