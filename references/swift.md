# Race Conditions in Swift

## Concurrency model

Swift has real threads (GCD/`Thread`), so **memory data races are real**: two
tasks touching the same shared mutable `class` corrupt state. Modern concurrency
closes this by construction — `actor` isolates mutable state behind an implicit
serial queue, `Sendable` marks what crosses boundaries safely, `@MainActor` pins
UI to the main thread. The danger lives in unprotected reference types (`class`),
mutable captures in concurrent closures, and legacy GCD/`NSLock`. In **Swift 6
language mode** with `-strict-concurrency=complete`, most of these data races
become **compile errors** — the language's differentiator.

## The races you'll encounter

- **Memory data race** — mutable `class` shared between tasks (counter, cache,
  array). Unsynchronized access is undefined behavior, not just a "wrong value".
  It's what `actor`, lock, or `-strict-concurrency` kill.
- **Logical race on isolated state** — even with everything `Sendable`, a `read →
  decide → write` crossing an `await` lets another task enter the window (actor
  reentrancy is the classic case).
- **Off-main UI** — mutating `UIView`/`@Published` off the main thread: crash or
  glitch. `@MainActor` solves it.
- **Persistence race** — check-then-act in Core Data/SQLite/remote database: two
  concurrent `save`s, lost update. Lives in the data layer, not on the CPU (see
  `references/databases-sql.md`).

## How to avoid it

### 1. Shared counter: `class` with data race → `actor`

```swift
// BAD: shared reference class; count++ is not atomic.
// Two tasks read the same value and write the same +1 → lost updates.
// With -strict-concurrency=complete, passing it between tasks won't even compile
// (Counter is not Sendable — error when crossing the isolation boundary).
final class Counter {
    var count = 0
    func increment() { count += 1 }
}

// GOOD: actor serializes all access to mutable state under the hood.
// Each increment() runs isolated; nobody sees a partial count.
actor Counter {
    private(set) var count = 0
    func increment() { count += 1 }
}
// call: await counter.increment()  — the await is the price of safety
```

### 2. Actor reentrancy: check-then-act split by `await`

```swift
// BAD: the await suspends the actor and RELEASES the isolation.
// Another task enters between the check and the set → two "winners".
actor Reservation {
    var taken = false
    func claim() async -> Bool {
        guard !taken else { return false }     // CHECK
        try? await network.confirm()           // window: actor reentrant here
        taken = true                           // USE: state changed during the await
        return true
    }
}

// GOOD: decide BEFORE suspending; only then do the I/O.
// State is mutated synchronously, with no await in the middle of the critical section.
actor Reservation {
    var taken = false
    func claim() async -> Bool {
        guard !taken else { return false }
        taken = true                           // synchronous commit — no window
        do { try await network.confirm(); return true }
        catch { taken = false; return false }  // undo if the I/O fails
    }
}
```

### 3. UI mutated off the main thread → `@MainActor`

```swift
// BAD: network callback runs on a background thread; mutating the label there is UB.
func load() {
    api.fetch { text in
        self.titleLabel.text = text            // off-main: crash/glitch
    }
}

// GOOD: @MainActor guarantees, at compile time, that everything here runs on the main thread.
@MainActor
final class ProfileViewModel {
    var title = ""
    func load() async {
        let text = try? await api.fetch()      // suspends off the main thread
        title = text ?? ""                     // resumes ALREADY on the main thread
    }
}
```

### 4. Get-or-create in a cache: TOCTOU → single critical section

```swift
// BAD: legacy GCD. Check and insert in separate .sync calls;
// another thread inserts the same key between the "contains" and the "insert".
func cached(_ key: String) -> Image {
    if queue.sync(execute: { store[key] }) == nil {       // CHECK
        let img = render(key)
        queue.sync { store[key] = img }                    // USE: race here
    }
    return queue.sync { store[key]! }
}

// GOOD: actor makes get-or-create a single, indivisible operation.
actor ImageCache {
    private var store: [String: Image] = [:]
    func image(for key: String) -> Image {
        if let hit = store[key] { return hit }
        let img = render(key)
        store[key] = img                                   // no window: all isolated
        return img
    }
}
```

### 5. Parallel state with await scattered around → `TaskGroup`

```swift
// BAD: mutating a shared array from inside concurrent tasks = data race.
var results: [Data] = []
for url in urls {
    Task { results.append(try await fetch(url)) }   // concurrent append: corrupts
}

// GOOD: structured concurrency collects results without shared state.
// Each child returns its value; the join happens at a single point, with no race.
let results = try await withThrowingTaskGroup(of: Data.self) { group in
    for url in urls { group.addTask { try await fetch(url) } }
    var acc: [Data] = []
    for try await data in group { acc.append(data) }  // serial by construction
    return acc
}
```

## Idiomatic guards

| Primitive | When to use |
|---|---|
| `actor` | Mutable state shared between tasks; protects by isolation, not manual lock |
| `@MainActor` | Anything touching UIKit/SwiftUI/AppKit; forces main-thread execution at compile time |
| `Sendable` / `@Sendable` | Mark types/closures safe to cross concurrency boundaries |
| `async let` / `TaskGroup` | Structured parallelism without sharing mutable state |
| `nonisolated` | Actor members not touching mutable state (avoids unnecessary `await`) |
| `Atomic<T>` (`Synchronization` module) | Low-level lock-free counter/flag; stdlib in Swift 6, or `ManagedAtomic<T>` from the `swift-atomics` package for back-deploy — without yielding to the actor |
| `OSAllocatedUnfairLock` | Modern, lightweight, `Sendable` lock (replaces raw `os_unfair_lock`) |
| `NSLock` / serial `DispatchQueue` | Legacy GCD; still valid in pre-async code, minimal scope |
| Database / Core Data | Check-then-act that persists — see `references/databases-sql.md` |

## Proving the guard

Real XCTest: launch N simultaneous tasks against the **same** actor and assert
the invariant. Without `actor`, this test detects the data race (under TSan) or
the wrong count; with it, it always passes.

```swift
import XCTest

final class CounterRaceTests: XCTestCase {
    func testConcurrentIncrementsLoseNothing() async {
        let counter = Counter()                 // actor from the "How to avoid it" section
        let n = 10_000
        // Simultaneous start: n children over the same state.
        await withTaskGroup(of: Void.self) { group in
            for _ in 0..<n { group.addTask { await counter.increment() } }
        }
        let final = await counter.count
        XCTAssertEqual(final, n)                 // invariant: nothing was lost
    }
}
```

For legacy synchronous code (no async), `DispatchQueue.concurrentPerform` blocks
until all iterations finish — launch N operations on the same object and assert
afterward. Here `legacy` is the **unsynchronized** `class` (the BAD from section
1), not the `actor`: precisely the one that should fail under TSan or lose count.

```swift
let legacy = Counter()   // the mutable class from the BAD block, without protection
DispatchQueue.concurrentPerform(iterations: 10_000) { _ in legacy.increment() }
XCTAssertEqual(legacy.count, 10_000)   // may fail: increment() is not atomic
```

Run the suite with the **Thread Sanitizer enabled** (Scheme → Diagnostics →
Thread Sanitizer): it instruments the accesses and fails on the first real data
race, even if the count happens to match.

## Lint & static detection

- **Swift 6 compiler — strict concurrency.** The strongest detector. In Swift 6
  language mode, a data race on state checked by actor/`Sendable` is a **compile
  error**. Incremental migration: `swift build -Xswiftc
  -strict-concurrency=complete`, or in `Package.swift` per target
  `.enableExperimentalFeature("StrictConcurrency")` (pre-Swift 6) /
  `swiftLanguageMode(.v6)`. Catches: non-`Sendable` mutable state crossing a
  boundary, unsafe capture in `@Sendable`, access to `@MainActor` off the main
  thread. **Doesn't catch**: logical race in persistence, nor `@unchecked
  Sendable` (you take on the risk).

- **Thread Sanitizer (Xcode/SwiftPM).** Dynamic detector, same engine as Go's
  `-race`. Covers legacy code, `@unchecked Sendable`, and GCD the compiler
  doesn't prove. `swift test --sanitize=thread` or Scheme → Diagnostics → Thread
  Sanitizer. **Doesn't catch** what doesn't run — it needs a test exercising the
  concurrent path.

Cross-cutting detail of both tools in `references/lint-detectors.md`. Neither
catches a **logical** race in the data layer — only a concurrent test and review
do.

## Swift-specific anti-patterns

- **`@unchecked Sendable` to "shut up the compiler".** Promises safety that
  doesn't exist; use only with a proven internal lock, never to make the build pass.
- **Relying on actor reentrancy.** `await` mid critical section releases the
  isolation — it's not a mutex that holds until the end. Mutate state before suspending.
- **`DispatchQueue.main.sync` on the main thread.** Immediate deadlock. Use
  `async` or `MainActor.run`.
- **`.sync` on a concurrent queue expecting a write barrier.** A barrier is
  `.async(flags: .barrier)`; plain `.sync` doesn't serialize writes.
- **Lock only in the app (NSLock/actor) in a multi-device system.** It doesn't
  serialize between instances — the real guard is in the database
  (`references/databases-sql.md`).
- **Capturing mutable `self` (class) across multiple `Task`s without isolation.**
  Classic data race — promote the type to `actor` or `@MainActor`.
