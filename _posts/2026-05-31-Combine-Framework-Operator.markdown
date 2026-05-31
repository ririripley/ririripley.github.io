---
author: ripley
comments: false
date: 2026-05-31 12:11:08+00:00
layout: post
slug: Combine Framework Operators
title: Combine Framework Operators
wordpress_id: 304
categories:
- Tech
tags:
description: Combine Framework
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
# Combine Framework: Value Lane vs. Failure Lane Operators

## Conceptual Model

```
┌─────────────────────────────────────────────────────────┐
│                    Publisher Stream                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   ═══╦══════════════════════════════════════╦═══        │
│      ║         VALUE LANE (Output)         ║           │
│      ║  ──●────●────●────●────●────●──▶    ║           │
│      ╠══════════════════════════════════════╣           │
│      ║        FAILURE LANE (Error)         ║           │
│      ║  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ✗─▶   ║           │
│   ═══╩══════════════════════════════════════╩═══        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Operators That ONLY Touch the Value Lane

These transform/filter **values** and pass errors through **unchanged**:

| Operator | What It Does on Values |
|----------|----------------------|
| `map { }` | Transforms each value |
| `filter { }` | Conditionally passes values through |
| `compactMap { }` | Transforms + unwraps optionals |
| `flatMap { }` | Maps to a new publisher, flattens |
| `scan(initial) { }` | Accumulates values (like `reduce` but emits each step) |
| `reduce(initial) { }` | Accumulates into a single final value |
| `removeDuplicates()` | Suppresses consecutive equal values |
| `replaceEmpty(with:)` | Emits a default if no values arrived |
| `replaceNil(with:)` | Substitutes nil with a value |
| `collect()` | Buffers all values into one array |
| `first()` / `last()` | Takes first/last value only |
| `dropFirst(n)` | Skips the first N values |
| `drop(while:)` | Skips values while predicate is true |
| `prefix(n)` | Takes only the first N values |
| `prepend(...)` | Inserts values before the stream |
| `append(...)` | Adds values after completion |
| `min()` / `max()` | Emits the min/max value |
| `count()` | Emits the count of values |
| `contains { }` | Emits Bool if match found |
| `allSatisfy { }` | Emits Bool if all match |
| `output(at:)` | Emits only the value at index N |
| `debounce(for:)` | Waits for a pause in values |
| `throttle(for:)` | Rate-limits values |
| `delay(for:)` | Delays each value in time |
| `measureInterval` | Emits time between values |
| `removeDuplicates(by:)` | Custom equality suppression |

```
  Values:  ──●────●────●──▶   map { $0 * 2 }   ──●────●────●──▶
  Errors:  ─ ─ ─ ─ ─ ─ ✗─▶   (pass-through)   ─ ─ ─ ─ ─ ─ ✗─▶
```

---

## Operators That ONLY Touch the Failure Lane

These handle/transform **errors** and pass values through **unchanged**:

| Operator | What It Does on Errors |
|----------|----------------------|
| `mapError { }` | Transforms the error type/value |
| `catch { }` | Replaces the error with a new publisher |
| `retry(n)` | Re-subscribes on failure (up to N times) |
| `replaceError(with:)` | Substitutes error with a default value + finishes |
| `setFailureType(to:)` | Adds a failure type to a `Never`-failing publisher |

```
  Values:  ──●────●────●──▶   mapError { }     ──●────●────●──▶
  Errors:  ─ ─ ─ ─ ─ ─ ✗─▶   (transforms ✗)   ─ ─ ─ ─ ─ ─ ✗─▶
```

### Detailed Behavior:

```swift
// mapError — changes the error type, values pass through untouched
publisher
    .mapError { error -> MyAppError in
        .networkFailure(error)  // only runs if an error occurs
    }

// catch — swaps in a recovery publisher when error occurs
publisher
    .catch { error in
        Just(fallbackValue)  // replaces failed stream with this
    }

// retry — values pass through; on error, resubscribes
publisher
    .retry(3)  // tries up to 3 more times on failure

// replaceError — converts error into one final value
publisher
    .replaceError(with: defaultValue)  // Error → value + finished
```

---

## Visual Summary

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  VALUE-LANE OPERATORS          FAILURE-LANE OPERATORS        │
│  ════════════════════          ═══════════════════════        │
│                                                              │
│  ┌─────────────────┐          ┌─────────────────────┐       │
│  │ map             │          │ mapError            │       │
│  │ filter          │          │ catch               │       │
│  │ compactMap      │          │ retry               │       │
│  │ flatMap         │          │ replaceError        │       │
│  │ scan / reduce   │          │ setFailureType      │       │
│  │ removeDuplicates│          └─────────────────────┘       │
│  │ collect         │                                         │
│  │ first / last    │                                         │
│  │ drop / prefix   │          BOTH LANES (side effects)      │
│  │ prepend / append│          ═════════════════════════       │
│  │ debounce        │          ┌─────────────────────┐       │
│  │ throttle        │          │ handleEvents        │       │
│  │ delay           │          │ print               │       │
│  │ replaceEmpty    │          │ breakpoint          │       │
│  │ replaceNil      │          │ receive(on:)        │       │
│  │ min / max       │          │ subscribe(on:)      │       │
│  │ count           │          └─────────────────────┘       │
│  └─────────────────┘                                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Key Insight: The `try` Variants Bridge Lanes

Operators prefixed with `try` operate on the **value lane** but can **throw into the failure lane**:

```swift
// tryMap operates on values but can PRODUCE errors
publisher
    .tryMap { value -> NewType in
        guard isValid(value) else {
            throw MyError.invalid  // ← jumps to failure lane!
        }
        return transform(value)
    }
```

```
  Values:  ──●────●────●──▶              ──●────●──▶
                              tryMap { }
  Errors:  ─ ─ ─ ─ ─ ─ ─ ▶              ─ ─ ─ ─ ─ ✗──▶
                                          (value→error possible)
```

| try Variant | Base Operator |
|-------------|--------------|
| `tryMap` | `map` |
| `tryFilter` | `filter` |
| `tryCompactMap` | `compactMap` |
| `tryScan` | `scan` |
| `tryReduce` | `reduce` |
| `tryRemoveDuplicates` | `removeDuplicates` |
| `tryAllSatisfy` | `allSatisfy` |
| `tryContains` | `contains` |
| `tryDrop(while:)` | `drop(while:)` |
| `tryPrefix(while:)` | `prefix(while:)` |

These are value-lane operators with an **escape hatch** into the failure lane.

---
---

# Combine Pipeline Learning Notes

## Table of Contents
- [Failure vs Error](#failure-vs-error)
- [Operator Details](#operator-details)
  - [.filter](#filter)
  - [.timeout](#timeout)
  - [.map](#map)
  - [.handleEvents](#handleevents)
  - [.catch](#catch)
  - [.merge](#merge)
  - [.removeDuplicates](#removeduplicates)
- [Combining merge + removeDuplicates](#combining-merge--removeduplicates)
- [Two Lanes: Values vs Failures](#two-lanes-values-vs-failures)
- [Operator Order Matters](#operator-order-matters)
- [.sink Overloads](#sink-overloads)
- [Sources of Failure](#sources-of-failure)
- [Completion vs Cancellation](#completion-vs-cancellation)
- [Synchronous vs Asynchronous Execution](#synchronous-vs-asynchronous-execution)
- [Real Example: SessionService Pipeline](#real-example-sessionservice-pipeline)

---

## Failure vs Error

This is a crucial distinction in Combine:

### Failure (Combine-specific term)
- The **type-level** error that a publisher can produce
- Every publisher has a `Failure` associated type: `Publisher<Output, Failure>`
- When a publisher "fails," it sends a `.failure(error)` **completion event** and the stream is **permanently terminated**
- It's part of the type system: `Failure == Never` means the publisher **can never** fail

### Error (Swift general term)
- Any value conforming to the `Error` protocol (like `NSError`, custom enums, etc.)
- It's what gets **wrapped inside** a `.failure` completion

### How they relate:

```swift
// Publisher completion is an enum:
enum Subscribers.Completion<Failure: Error> {
    case finished          // Normal end, no problem
    case failure(Failure)  // Terminated with an error value
}
```

"Failure" is the **event type** (the envelope), "error" is the **value** inside it.

### Practical example:
```swift
let subject = PassthroughSubject<String, URLError>()
//                                Output ^    ^ Failure type

subject.send("hello")                          // value event
subject.send(completion: .finished)            // normal completion
subject.send(completion: .failure(URLError(.timedOut))) // failure completion
```

After **either** completion (`.finished` or `.failure`), the stream is **dead** — no more values can ever flow.

### An enum case named `.error` is NOT a failure!

```swift
enum SessionVerificationResult {
    case success
    case error    // This is just DATA, a label — not a Combine failure!
}
```

`SessionVerificationResult.error` is just a **value** flowing through the value lane. You could rename it to `.denied` or `.banana` and it would behave identically. Combine doesn't care what you name your enum cases.

---

## Operator Details

---

### .filter

#### Purpose
Only lets values through that satisfy a condition. Values that don't match are **silently dropped**.

#### Type signature
```swift
func filter(_ isIncluded: @escaping (Output) -> Bool) -> Publishers.Filter<Self>
```

- **Output type**: unchanged
- **Failure type**: unchanged

#### What flows through:

| Event type | Behavior |
|-----------|----------|
| Value that passes predicate | Passes downstream |
| Value that fails predicate | Dropped silently |
| `.finished` completion | Passes downstream |
| `.failure` completion | Passes downstream |

#### Key insight
`.filter` **never** sees or touches failures. Failures always bypass it. Only values are tested.

```swift
[1, 2, 3, 4, 5].publisher
    .filter { $0 > 3 }
    .sink { print($0) }
// Prints: 4, 5
```

#### In context:
```swift
.filter { $0 == sessionId }
```
`$0` is each UUID emitted by the upstream publisher. Only the UUID matching the current session passes through. All others are silently ignored — the timeout clock keeps ticking while waiting for a *matching* UUID.

---

### .timeout

#### Purpose
Enforces a **time limit**. If the upstream publisher doesn't emit a value within the specified interval, the timeout takes action.

#### Type signature
```swift
func timeout<S: Scheduler>(
    _ interval: S.SchedulerTimeType.Stride,
    scheduler: S,
    options: S.SchedulerOptions? = nil,
    customError: (() -> Failure)? = nil
) -> Publishers.Timeout<Self, S>
```

#### Two modes:

| `customError` | When timeout fires |
|---|---|
| **Provided** (closure) | Sends `.failure(yourError)` — stream dies with error |
| **nil** (default) | Sends `.finished` — stream ends normally, silently |

#### Timer behavior:
- Timer starts when subscribed
- **Resets** every time a value passes through
- Fires only when NO value arrives for the full interval

#### What flows through:

| Event type | Behavior |
|-----------|----------|
| Value (before timeout) | Passes downstream, timer resets |
| `.finished` (before timeout) | Passes downstream, timer cancelled |
| `.failure` (before timeout) | Passes downstream, timer cancelled |
| Timer fires (with customError) | Sends `.failure(customError())` downstream |
| Timer fires (without customError) | Sends `.finished` downstream |

#### Example:
```swift
.timeout(.milliseconds(100), scheduler: DispatchQueue.main,
         customError: { NSError(domain: "CustomTimeoutError", code: -1) })
```
"If no matching UUID arrives within 100ms, terminate the stream with a `.failure` containing that NSError."

---

### .map

#### Purpose
Transforms each **value** into a different value. Does not affect failures/completions.

#### Type signature
```swift
func map<T>(_ transform: @escaping (Output) -> T) -> Publishers.Map<Self, T>
```

- **Output type**: changes to `T`
- **Failure type**: unchanged

#### What flows through:

| Event type | Behavior |
|-----------|----------|
| Value | Transformed by closure, new value passes downstream |
| `.finished` | Passes through untouched |
| `.failure` | Passes through untouched — **`.map` closure is NOT called** |

#### Critical insight
**Failures bypass `.map` entirely.** The transform closure never executes for failures.

```swift
subject
    .map { value in
        print("Transform called")  // NEVER prints if a failure arrives
        return value * 2
    }
```

---

### .handleEvents

#### Purpose
Observe (but **never modify**) events at any stage of the pipeline. Used for side effects.

#### Type signature
```swift
func handleEvents(
    receiveSubscription: ((Subscription) -> Void)? = nil,
    receiveOutput: ((Output) -> Void)? = nil,
    receiveCompletion: ((Subscribers.Completion<Failure>) -> Void)? = nil,
    receiveCancel: (() -> Void)? = nil,
    receiveRequest: ((Subscribers.Demand) -> Void)? = nil
) -> Publishers.HandleEvents<Self>
```

- **Output type**: unchanged
- **Failure type**: unchanged

#### All hooks:

| Hook | When it fires |
|------|---------------|
| `receiveSubscription` | When a subscriber subscribes |
| `receiveOutput` | Each time a value passes through |
| `receiveCompletion` | When the publisher completes (`.finished` or `.failure`) |
| `receiveCancel` | When the subscription is cancelled |
| `receiveRequest` | When demand is requested by downstream |

#### Critical insight about `receiveOutput` vs failures
`receiveOutput` is **only called for values**, NOT for failures. If a `.failure` arrives:
- `receiveOutput` -> **NOT called**
- `receiveCompletion` -> **IS called** (with `.failure(error)`)

---

### .catch

#### Purpose
**Intercept a failure** and replace the failed publisher with a new publisher. This is error recovery.

#### Type signature
```swift
func catch<P: Publisher>(_ handler: @escaping (Failure) -> P) -> Publishers.Catch<Self, P>
    where P.Output == Output
```

- **Output type**: unchanged (must match)
- **Failure type**: changes to the replacement publisher's Failure type

#### What flows through:

| Event type | Behavior |
|-----------|----------|
| Value | Passes through, `.catch` does nothing |
| `.finished` | Passes through, `.catch` does nothing |
| `.failure(error)` | **Catches it!** Original upstream is dead. Handler returns a new publisher |

#### Key insight
`.catch` **only activates on failure**. Normal completion and values pass through invisible.

#### What happens when it catches:
1. Upstream publisher is **terminated** (dead, can never emit again)
2. Your closure receives the error value
3. You return a **new publisher** — becomes the new source
4. Downstream subscribes to the new publisher

#### Type change with `Just`:
`Just` has `Failure == Never`, so after `.catch { Just(...) }`, the downstream pipeline has `Failure == Never` — it can never fail.

---

## Two Lanes: Values vs Failures

A Combine pipeline has **two separate lanes**:

```
+-----------------------------------------------------+
|  VALUE LANE (carries data)                          |
|                                                     |
|  UUID -> filter -> timeout -> .map -> .handleEvents --> sink receives value
|                                       |             |
|                        SessionVerificationResult    |
|                        .error (just data!)          |
|                                                     |
+-----------------------------------------------------+
|  FAILURE LANE (carries termination signal)          |
|                                                     |
|  timeout fires -> .failure(NSError) ------------------> .catch intercepts
|                  (bypasses .map!)                    |
|                  (bypasses .handleEvents!)           |
|                                                     |
+-----------------------------------------------------+
```

- `.map` operates on the **value lane** only
- `.catch` operates on the **failure lane** only
- They **never** interact with each other's lane

---

## Operator Order Matters

The order of operators fundamentally changes behavior. Each operator only sees what the previous one outputs.

### Example 1: Swap `.filter` and `.timeout`

**Current: `.filter` -> `.timeout`**
```swift
.filter { $0 == sessionId }        // Drop non-matching UUIDs
.timeout(.milliseconds(100), ...)  // Timer resets ONLY when matching UUID arrives
```
Timer fires if no *matching* UUID arrives in 100ms.

**Swapped: `.timeout` -> `.filter`**
```swift
.timeout(.milliseconds(100), ...)  // Timer resets on ANY UUID
.filter { $0 == sessionId }        // Then check if it matches
```
Any UUID resets the timer — even wrong ones. A wrong UUID resets the clock but gets filtered out. You'd wait forever without a value or timeout!

### Example 2: Swap `.map` and `.catch`

**Current: `.map` -> `.catch`**
```swift
.map { _ in SessionVerificationResult.error }  // Value -> .error
.catch { _ in Just(.success) }                 // Failure -> .success
```
Two distinct paths, two distinct outcomes.

**Swapped: `.catch` -> `.map`**
```swift
.catch { _ in Just(.success) }                 // Failure -> .success VALUE
.map { _ in SessionVerificationResult.error }  // ALL values -> .error
```
Failures get caught -> become a `.success` value -> then `.map` converts ALL values to `.error`. Login ALWAYS fails!

### Example 3: Swap `.map` and `.timeout`

**Current: `.timeout` -> `.map`**
- Value arrives: `.map` converts to `.error`
- Timeout fires: `.failure` bypasses `.map`

**Swapped: `.map` -> `.timeout`**
- UUID arrives -> `.map` converts to `.error` value -> `.timeout` sees a value, resets timer
- Same final outcomes but types change (`.timeout` now operates on `SessionVerificationResult` instead of `UUID`)

### Mental Model

Think of it as a **literal assembly line**:
```
Raw Material -> Station 1 -> Station 2 -> Station 3 -> Final Product
```
Each station only receives what the previous station outputs. Rearranging stations = completely different results.

---

## .sink Overloads

### Version 1: Two closures (for publishers that CAN fail)
```swift
.sink(
    receiveCompletion: { completion in ... },  // Called on .finished or .failure
    receiveValue: { value in ... }             // Called on each value
)
```

### Version 2: One closure (ONLY for `Failure == Never`)
```swift
.sink { value in ... }  // Called ONLY on values — completions are silently ignored
```

When a publisher has `Failure == Never` (like after `.catch { Just(...) }`), the single-closure version is used. **Completions are silently ignored** — there's no closure to handle them.

### If completion arrives without a value first:
With single-closure `.sink`:
- The callback **never fires**
- The subscription silently dies
- No result is ever reported

---

## Sources of Failure

Failures don't only come from `.send(completion:)`. They can originate from:

| Source | Example |
|--------|---------|
| Explicitly sent by a Subject | `subject.send(completion: .failure(myError))` |
| Generated by an operator | `.timeout(...)` fires -> creates a `.failure` internally |
| Returned by a closure | `tryMap { throw MyError() }` -> failure |
| From an upstream system | `URLSession.dataTaskPublisher` -> network error |
| From `Future` | `Future { promise in promise(.failure(error)) }` |

A failure is any `.failure(Error)` completion event flowing through the pipeline — regardless of **who** created it.

---

## Completion vs Cancellation

| | Completion | Cancellation |
|---|---|---|
| **Who initiates** | Upstream (publisher/operator) | Downstream (subscriber) |
| **Direction** | Flows downstream | Flows upstream |
| **Callbacks fired** | `receiveCompletion` is called | `receiveCancel` (in handleEvents) |
| **End result** | Same: subscription is dead | Same: subscription is dead |
| **How it happens** | `subject.send(completion:)` or operator | `cancellable.cancel()` or `AnyCancellable` dealloc |

Both terminate the subscription, but from opposite ends:
```
        Cancellation (upstream <-)
        +---------------------------+
        |                           |
Publisher ---- Operators ---- Subscriber
        |                           |
        +---------------------------+
        Completion (downstream ->)
```

### After any completion (`.finished` or `.failure`):
- `receiveCompletion` fires one final time
- No more values will ever flow
- The subscription is effectively released
- Calling `.cancel()` afterward is redundant but harmless

---

## Synchronous vs Asynchronous Execution

```swift
Button(action: {
    // 1. verify() is called — sets up subscription, returns IMMEDIATELY
    sessionService.verify { ... }
    
    // 2. This runs RIGHT AFTER verify() returns (nanoseconds later)
    auth.passwordAttempts += 1
    
    // 3. Button action closure ends. Control returns to the run loop.
    
    // ... time passes ...
    
    // 4. MUCH LATER: completion callback fires (after UUID arrives or 100ms timeout)
})
```

The `@escaping` keyword is the hint:
```swift
func verify(completion: @escaping (SessionVerificationResult) -> Void)
//                      ^^^^^^^^
// "escaping" means: this closure will be called AFTER the function returns
```

`verify` sets up a Combine subscription and returns immediately. The completion callback fires asynchronously — when the pipeline produces a value.

---

## Real Example: SessionService Pipeline

```swift
self
    .filter { $0 == sessionId }
    .timeout(
        .milliseconds(100),
        scheduler: DispatchQueue.main,
        customError: { NSError(domain: "CustomTimeoutError", code: -1) }
    )
    .map { _ in SessionVerificationResult.error }
    .handleEvents(receiveOutput: { result in
        if case .error = result {
            verificationSubject.send(completion: .finished)
        }
    })
    .catch { error in
        Just(SessionVerificationResult.success)
    }
    .eraseToAnyPublisher()
```

### Scenario A: UUID arrives within 100ms
```
verificationSubject emits UUID
    -> .filter: UUID matches sessionId? YES -> passes through
    -> .timeout: value arrived, no timeout fires
    -> .map: UUID becomes .error (just a value label!)
    -> .handleEvents: sees .error value, shuts down verificationSubject
    -> .catch: no failure to catch, does nothing
    -> downstream receives: .error
    -> Login FAILS
```

### Scenario B: No UUID arrives within 100ms
```
verificationSubject emits nothing
    -> .filter: nothing to filter
    -> .timeout: 100ms passes, fires NSError as failure
    -> .map: SKIPPED (failures bypass .map)
    -> .handleEvents: SKIPPED (failures bypass receiveOutput)
    -> .catch: catches NSError, returns Just(.success)
    -> downstream receives: .success
    -> Login SUCCEEDS
```

### The Inverted Logic (by design):
```
UUID arrives within 100ms  ->  .map  ->  .error  ->  Login FAILS
No UUID within 100ms       ->  timeout  ->  .catch  ->  .success  ->  Login SUCCEEDS
```

### The Bug (in context):
When Login is clicked:
1. `verify()` sets up subscription (100ms clock starts)
2. `auth.passwordAttempts += 1` mutates `@Published` property
3. SwiftUI re-renders `SecurityBannerView`
4. `.execute { initializeVerificationProcess() }` fires
5. UUID is sent to `verificationSubject` within ~1ms
6. UUID arrives before 100ms timeout -> `.map` -> `.error` -> Login always fails

### The Fix:
Remove `@Published` from `passwordAttempts` -> no re-render -> no UUID sent -> timeout fires -> `.success` -> Login works.

---

## Operator Details (continued)

---

### .merge

#### Purpose
Combines multiple publishers of the **same Output and Failure type** into a single stream, emitting values from all sources in the order they arrive (interleaved by time).

#### Type signature
```swift
func merge(with other: P) -> Publishers.Merge<Self, P>
    where P: Publisher, Self.Failure == P.Failure, Self.Output == P.Output
```

Variants exist up to `Merge8`, plus `MergeMany` for dynamic collections of publishers.

#### How it works:

```
Publisher A:  --1------3------5-->
Publisher B:  ----2------4-------->
Merged:       --1-2----3-4----5-->
```

1. **Subscribes to all upstream publishers** simultaneously
2. **Forwards every value** from any upstream as soon as it arrives — no buffering, no reordering
3. **Completes only when all** upstream publishers have completed
4. **Fails immediately** if any upstream emits a failure (cancels others)

#### What flows through:

| Event type | Behavior |
|-----------|----------|
| Value from any upstream | Forwarded downstream immediately |
| `.finished` from one upstream | That upstream is done; merge waits for others |
| `.finished` from ALL upstreams | `.finished` sent downstream |
| `.failure` from any upstream | `.failure` sent downstream immediately, others cancelled |

#### Key characteristics:

| Property | Behavior |
|----------|----------|
| Demand management | Requests from downstream are forwarded to **all** upstreams |
| Backpressure | Each upstream independently respects demand |
| Ordering | Temporal — whoever emits first is forwarded first |
| Completion | Waits for **all** upstreams to complete |
| Error | Fails fast — first error terminates the merged stream |

#### Example:
```swift
let localUpdates = database.publisher(for: User.self)   // Publisher<User, Never>
let remoteUpdates = api.userStream()                     // Publisher<User, Never>

localUpdates
    .merge(with: remoteUpdates)
    .sink { user in
        print("User updated: \(user)")
    }
// Receives values from BOTH sources as they arrive
```

#### Key insight: merge vs combineLatest vs zip

| Operator | Output | When it emits |
|----------|--------|---------------|
| `merge` | Same type as inputs (single value) | Every time ANY upstream emits |
| `combineLatest` | Tuple of latest from each | Every time ANY upstream emits (after all have emitted at least once) |
| `zip` | Tuple pairing values 1-to-1 | When ALL upstreams have a new value to pair |

```swift
// merge: "give me everything from both"
A: --1----3-->     merged: --1-2-3-4-->
B: ---2----4->

// combineLatest: "give me latest from each whenever either changes"
A: --1----3-->     combined: --(1,2)-(3,2)-(3,4)-->
B: ---2----4->

// zip: "pair them up in order"
A: --1----3-->     zipped: --(1,2)----(3,4)-->
B: ---2----4->
```

---

### .removeDuplicates

#### Purpose
Suppresses **consecutive** duplicate values, only forwarding a value if it differs from the immediately preceding one.

#### Type signatures
```swift
// For Equatable outputs:
func removeDuplicates() -> Publishers.RemoveDuplicates<Self>
    where Self.Output: Equatable

// With custom predicate:
func removeDuplicates(by predicate: @escaping (Output, Output) -> Bool)
    -> Publishers.RemoveDuplicates<Self>
```

- **Output type**: unchanged
- **Failure type**: unchanged

#### How it works:

```
Input:   --1--1--2--2--1--1--3-->
Output:  --1-----2-----1-----3-->
```

1. **Stores the last emitted value** (exactly one element of state)
2. For each new value from upstream, **compares it** to the stored value using `==` (or the custom predicate)
3. If **equal** → the value is dropped (not forwarded downstream)
4. If **different** → the value is forwarded and becomes the new stored value
5. The **first value** is always forwarded (there's nothing to compare against)

#### What flows through:

| Event type | Behavior |
|-----------|----------|
| Value (different from previous) | Passes downstream, becomes new "previous" |
| Value (same as previous) | Dropped silently |
| First value ever | Always passes (nothing to compare against) |
| `.finished` completion | Passes downstream |
| `.failure` completion | Passes downstream |

#### Critical insight: consecutive only, NOT global
```swift
[1, 1, 2, 2, 3, 3, 1, 1].publisher
    .removeDuplicates()
    .sink { print($0) }
// Output: 1, 2, 3, 1
//                   ^ emitted again! Only consecutive duplicates are suppressed
```

This is NOT a "unique" filter — it only compares against the **immediately preceding** value.

#### Custom predicate example:
```swift
struct UserLocation {
    let lat: Double
    let lon: Double
}

locationPublisher
    .removeDuplicates { prev, current in
        // Return true = "these are duplicates" = SUPPRESS
        // Return false = "these are different" = PASS THROUGH
        distance(prev, current) < 10.0
    }
    .sink { location in
        updateMap(location)  // Only fires when user moves > 10 meters
    }
```

#### Predicate semantics (easy to confuse!):

| Predicate returns | Meaning | Action |
|-------------------|---------|--------|
| `true` | "These are equal/duplicates" | **Suppress** (drop the new value) |
| `false` | "These are different" | **Pass through** |

This is the **opposite** of `.filter`! In `.filter`, `true` = keep. In `.removeDuplicates(by:)`, `true` = drop.

#### Common use with @Published properties:
```swift
class ViewModel: ObservableObject {
    @Published var searchText = ""
}

viewModel.$searchText
    .removeDuplicates()   // Don't re-search if text hasn't actually changed
    .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
    .sink { text in
        performSearch(text)
    }
```

#### For global uniqueness (NOT what removeDuplicates does):
If you need to filter out ALL previously seen values (not just consecutive):
```swift
.scan((Set<Output>(), nil as Output?)) { state, value in
    var (seen, _) = state
    if seen.contains(value) { return (seen, nil) }
    seen.insert(value)
    return (seen, value)
}
.compactMap { $0.1 }
```

---

### Combining merge + removeDuplicates

A common real-world pattern — merge multiple sources and deduplicate:

```swift
let local = database.publisher(for: User.self)   // Publisher<User, Never>
let remote = api.fetchUser()                      // Publisher<User, Never>

local
    .merge(with: remote)
    .removeDuplicates()       // suppress if both sources emit the same User
    .assign(to: \.user, on: viewModel)
```

#### Execution flow:
```
local:               --UserV1------------------->
remote:              ----------UserV1---UserV2-->
merged:              --UserV1--UserV1---UserV2-->
removeDuplicates:    --UserV1----------UserV2-->
                               ^ suppressed (consecutive duplicate)
```

This works because after merging, if both sources happen to emit the same value in sequence, `removeDuplicates` catches the redundancy.
