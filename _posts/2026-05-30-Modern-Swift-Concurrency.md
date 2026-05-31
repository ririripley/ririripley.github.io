---
author: ripley
comments: false
date: 2026-05-31 12:12:08+00:00
layout: post
slug: iOSDevelopment
title: Modern Swift Concurrency
wordpress_id: 304
categories:
- Tech
tags:
description: Concurrency
---
<style>
.article_body {
  color: #243447;
}

.article_body h1,
.article_body h2,
.article_body h3,
.article_body h4,
.article_body h5,
.article_body h6 {
  color: #111827;
}

.article_body code {
  color: #1f2937;
  background-color: #f6f8fa;
}

.article_body pre,
.article_body .highlight {
  color: #1f2937;
  background-color: #ffffff;
}

.article_body pre code,
.article_body .highlight code,
.article_body .highlight pre {
  color: inherit;
  background: transparent;
}

.article_body .highlight .err,
.article_body .highlight .esc,
.article_body .highlight .g,
.article_body .highlight .l,
.article_body .highlight .n,
.article_body .highlight .o,
.article_body .highlight .x,
.article_body .highlight .p,
.article_body .highlight .gd,
.article_body .highlight .ge,
.article_body .highlight .gr,
.article_body .highlight .gh,
.article_body .highlight .gi,
.article_body .highlight .gp,
.article_body .highlight .gs,
.article_body .highlight .gu,
.article_body .highlight .gt,
.article_body .highlight .ld,
.article_body .highlight .nb,
.article_body .highlight .nc,
.article_body .highlight .nd,
.article_body .highlight .ni,
.article_body .highlight .ne,
.article_body .highlight .nl,
.article_body .highlight .nn,
.article_body .highlight .nx,
.article_body .highlight .py,
.article_body .highlight .ow,
.article_body .highlight .bp {
  color: #1f2937;
}
</style>

# Modern Swift Concurrency

## Bridge GCD and Modern Swift Concurrency

### GCD → Modern Swift Concurrency

Wrap a completion-handler API using `withCheckedContinuation` (or `withCheckedThrowingContinuation` for throwing variants):

```swift
// Before: GCD / completion-handler style
func fetchUser(id: String, completion: @escaping (User?, Error?) -> Void) {
    DispatchQueue.global().async {
        let result = database.query(id)
        DispatchQueue.main.async {
            completion(result, nil)
        }
    }
}

// After: Modern Swift Concurrency
func fetchUser(id: String) async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        fetchUser(id: id) { user, error in
            if let user {
                continuation.resume(returning: user)
            } else {
                continuation.resume(throwing: error ?? FetchError.unknown)
            }
        }
    }
}
```

### Modern Swift Concurrency → GCD

Expose an async function to callers that still use completion handlers:

```swift
// Async function
func fetchUser(id: String) async throws -> User {
    let data = try await URLSession.shared.data(from: endpoint(for: id))
    return try JSONDecoder().decode(User.self, from: data.0)
}

// Wrapper for GCD / completion-handler consumers
func fetchUser(id: String, completion: @escaping (Result<User, Error>) -> Void) {
    Task {
        do {
            let user = try await fetchUser(id: id)
            DispatchQueue.main.async { completion(.success(user)) }
        } catch {
            DispatchQueue.main.async { completion(.failure(error)) }
        }
    }
}
```

## Task

`Task` creates an **unstructured** unit of async work — fire and forget. It **inherits the actor isolation context** from where it is defined (e.g. a `Task` created inside a `@MainActor` function runs on the main actor), but its lifetime is not bound to a parent scope.

### Cancellation with `checkCancellation`

Cancellation is **cooperative** — a task must explicitly check for it. Call `try Task.checkCancellation()` at appropriate points to throw `CancellationError` when the task has been cancelled.

```swift
let task = Task {
    for i in 0..<1000 {
        try Task.checkCancellation() // throws CancellationError if cancelled
        await process(item: i)
    }
}

// Later...
task.cancel()
```

### `withTaskCancellationHandler`

Use `withTaskCancellationHandler` to bridge Swift concurrency cancellation to non-async APIs that have their own cancel mechanism (e.g. `URLSessionTask`, Core Location, etc.).

```swift
func download(from url: URL) async throws -> Data {
    let request = URLRequest(url: url)

    return try await withTaskCancellationHandler {
        try await URLSession.shared.data(for: request).0
    } onCancel: {
        // Bridge cancellation to a non-cooperative API
        URLSession.shared.invalidateAndCancel()
    }
}
```

A more transparent example — bridging a callback-based operation with an explicit cancel handle:

```swift
class ImageProcessor {
    private var isCancelled = false

    func process(data: Data, completion: @escaping (UIImage?) -> Void) {
        DispatchQueue.global().async {
            for i in 0..<data.count {
                if self.isCancelled { completion(nil); return }
                // ... expensive pixel work ...
            }
            completion(UIImage())
        }
    }

    func cancel() { isCancelled = true }
}

func processImage(data: Data) async throws -> UIImage {
    let processor = ImageProcessor()

    return try await withTaskCancellationHandler {
        try await withCheckedThrowingContinuation { continuation in
            processor.process(data: data) { image in
                if let image {
                    continuation.resume(returning: image)
                } else {
                    continuation.resume(throwing: CancellationError())
                }
            }
        }
    } onCancel: {
        processor.cancel() // propagates cancellation down to the processor
    }
}
```

> **Anti-pattern:** Don't create an unstructured `Task` only to immediately `await` its `.value` inside `withTaskCancellationHandler` — that's redundant. Just call the async function directly; structured concurrency propagates cancellation automatically.

> **When is `withTaskCancellationHandler` necessary?** When you're awaiting work that won't be cancelled automatically by Swift's structured concurrency. Two key cases:
> 1. **Unstructured tasks (`Task {}`)** — they are detached from the task tree, so even if the calling site is cancelled, the unstructured task keeps running unless you manually cancel it.
> 2. **Non-async/external APIs** (callbacks, C libraries, legacy APIs) — they have their own cancel mechanism that Swift doesn't know about.
>
> If you're calling a pure `async` function in a structured context (e.g. inside `TaskGroup`, or directly with `await`), cancellation propagates automatically and you don't need this.

## Task Group (Structured Concurrency)

`TaskGroup` spawns child tasks whose lifetimes are bound to the group scope. All child tasks are guaranteed to finish before the group returns. Cancellation propagates automatically to all children.

### Spawn concurrent tasks and gather results in order

```swift
func processChunks(data: [Data]) async throws -> [Result] {
    try await withThrowingTaskGroup(of: (Int, Result).self) { group in
        for (index, chunk) in data.enumerated() {
            group.addTask {
                let result = try await process(chunk)
                return (index, result) // tag with index to preserve order
            }
        }

        var results = Array<Result?>(repeating: nil, count: data.count)
        for try await (index, result) in group {
            results[index] = result
        }
        return results.compactMap { $0 }
    }
}
```

### Limit concurrency with `group.next()`

Use `group.next()` to wait for a child to finish before spawning another, capping the number of in-flight tasks.

```swift
func downloadAll(urls: [URL], maxConcurrency: Int) async throws -> [Data] {
    try await withThrowingTaskGroup(of: Data.self) { group in
        var results: [Data] = []
        var index = 0

        // Fill up to the concurrency limit
        for _ in 0..<min(maxConcurrency, urls.count) {
            let url = urls[index]
            group.addTask { try await download(from: url) }
            index += 1
        }

        // As each completes, spawn the next
        while let data = try await group.next() {
            results.append(data)
            if index < urls.count {
                let url = urls[index]
                group.addTask { try await download(from: url) }
                index += 1
            }
        }

        return results
    }
}
```

### Set a time limit for a task

Use `withThrowingTaskGroup` and a deadline child task to race against a timeout.

```swift
func withTimeout<T: Sendable>(
    seconds: UInt64,
    operation: @escaping @Sendable () async throws -> T
) async throws -> T {
    try await withThrowingTaskGroup(of: T.self) { group in
        group.addTask {
            try await operation()
        }

        group.addTask {
            try await Task.sleep(nanoseconds: seconds * 1_000_000_000)
            throw TimeoutError()
        }

        // First to finish wins; cancel the loser
        let result = try await group.next()!
        group.cancelAll()
        return result
    }
}

struct TimeoutError: Error {}
```

### `async let` — lightweight structured concurrency

Use `async let` when you have a fixed number of concurrent tasks known at compile time. Each `async let` starts immediately and is implicitly awaited (and cancelled if unused) at the end of its scope.

```swift
func loadDashboard(userId: String) async throws -> Dashboard {
    async let profile = fetchProfile(userId)
    async let posts = fetchPosts(userId)
    async let friends = fetchFriends(userId)

    // All three run concurrently; await them together
    return try await Dashboard(
        profile: profile,
        posts: posts,
        friends: friends
    )
}
```

> When you create a task group and add child tasks using `group.addTask { ... }`, the closure you pass in is marked as `@Sendable`.
> * **The Hop:** Because the closure is `@Sendable`, it is non-isolated by default. Even if you start the Task Group on the `@MainActor`, the child tasks spawned inside `addTask` will immediately cross over and execute concurrently on the **global pool** (unless you explicitly mark the closure with `@MainActor`).

### How structured concurrency cancellation works

In structured concurrency, cancellation propagates **top-down** automatically:

1. When a parent task is cancelled, all its child tasks are marked as cancelled.
2. When a child task throws inside a `withThrowingTaskGroup`, the group **automatically cancels all remaining siblings** and rethrows the error.
3. With `async let`, if the enclosing scope exits (via throw or return) before awaiting a binding, the implicit child task is cancelled automatically.

Cancellation is still **cooperative** — a child task is only *marked* as cancelled, it must check `Task.isCancelled` or call `try Task.checkCancellation()` to actually stop.

### When you still need to cancel child tasks manually

Even with automatic propagation, there are scenarios where you must call `group.cancelAll()` explicitly:

**1. Race pattern (first result wins):**

The group doesn't know you only need one result. After getting the first result, cancel the rest yourself.

```swift
try await withThrowingTaskGroup(of: Response.self) { group in
    for server in mirrors {
        group.addTask { try await fetch(from: server) }
    }

    let fastest = try await group.next()!
    group.cancelAll() // manually cancel remaining children
    return fastest
}
```

**2. Conditional early exit without throwing:**

Automatic sibling cancellation only triggers on a thrown error. If you find your answer and want to stop early in a non-throwing group, you must cancel manually.

```swift
await withTaskGroup(of: (Int, Bool).self) { group in
    for (index, item) in items.enumerated() {
        group.addTask { (index, await validate(item)) }
    }

    for await (index, isValid) in group {
        if !isValid {
            group.cancelAll() // stop remaining validations on first failure
            return index
        }
    }
    return nil
}
```

**3. Non-throwing task group:**

`withTaskGroup` (non-throwing) never auto-cancels siblings since no error is thrown. You must use `group.cancelAll()` to stop remaining work.

## The `nonisolated` Keyword

`nonisolated` opts a method or property out of its enclosing actor's isolation. It can be called synchronously from outside without `await`, but cannot access the actor's mutable state.

### `nonisolated` method

```swift
actor BankAccount {
    let accountId: String  // immutable, safe
    var balance: Double

    // No actor-isolated state is accessed, so no isolation needed
    nonisolated func displayName() -> String {
        "Account #\(accountId)"
    }
}

let account = BankAccount(accountId: "123", balance: 1000)
// No `await` needed — the method is nonisolated
print(account.displayName())
```

### `nonisolated` computed property (protocol conformance)

```swift
actor Temperature {
    let id: UUID
    var celsius: Double

    // Conforming to a non-async protocol requirement
    nonisolated var description: String {
        "Sensor \(id)"  // only accesses immutable `let` state
    }
}

extension Temperature: CustomStringConvertible {}
```

### `nonisolated` on a `let` property

Stored `let` properties on actors are implicitly safe to access without isolation (immutable after init). You can mark them explicitly `nonisolated` to make intent clear, or rely on the compiler's implicit behavior (Swift 5.10+).

```swift
actor Server {
    nonisolated let host: String  // explicitly nonisolated
    nonisolated let port: Int
    var connections: [Connection] = []

    init(host: String, port: Int) {
        self.host = host
        self.port = port
    }
}

let server = Server(host: "localhost", port: 8080)
// Direct access without `await` — these are nonisolated let properties
print("\(server.host):\(server.port)")
```
