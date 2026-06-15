---
name: race-condition-guard
description: |
  Use when writing or reviewing concurrent code that mutates shared state —
  endpoints, workers, threads, goroutines, coroutines, async/await — or when
  building tests that fire simultaneous operations and assert invariants. Covers
  double-submit, TOCTOU, lost update, check-then-act, data race, deadlock, and
  idempotency across many languages (Python, JS/TS, Java/Kotlin, C#, Go, Rust,
  C/C++, Ruby, PHP, Swift, Dart) with per-language guards, examples, and lint
  detectors. Triggers: "race condition", "concurrency", "data race", "double
  submit", "lost update", "TOCTOU", "deadlock", "thread safety", "idempotency",
  "lock", "mutex", "atomic".
source: authored
upstream: https://github.com/FelipeOFF/race-condition-guard-skill
license: MIT
added: 2026-06-05
---

# Race Condition Guard

Guardrail for **race conditions** in concurrent code. A race is born when
two or more lines of execution (requests, threads, goroutines, tasks, workers)
touch the same state at almost the same time and the result depends on **who
arrives first** — violating an invariant (negative balance, double stock, two
records that should have been one, lost write, corrupted memory).

This skill is **multi-language**: the core below is agnostic (mental model,
guard taxonomy, testing principle). The idiomatic detail — `BAD→GOOD` examples
of how to avoid them and the **lint/detectors** of each ecosystem — lives in
`references/<language>.md`, loaded on demand.

## When to activate

- Endpoint/handler reads state, decides based on it, and writes (**check-then-act**).
- Mutation of a shared resource: balance, stock, counter, status, unique slug.
- Operations that **must** be idempotent (payment, creation by external key).
- Code with threads/goroutines/coroutines/async that shares memory or state.
- Webhooks/retries that may arrive duplicated or out of order.
- PR review that adds concurrency, lock, cache, or counter.

## Usage flow (follow in order)

1. **Detect the language and the concurrency model** of the target (shared
   memory? GIL? event loop? process isolated per request?).
2. **Classify the race** by the taxonomy below (check-then-act, lost update,
   data race, double-submit, lazy-init...).
3. **Reuse before you build.** Search the codebase for an existing guard or
   utility (lock helper, debounce/click-handler, idempotency middleware,
   atomic-update wrapper, unique constraint) and prefer it — in any language.
   Only add a new one if none exists. See `references/detection-workflow.md`.
4. **Choose the guard** in the strategy table and apply the idiomatic version
   for the language — see `references/<language>.md` for the `BAD→GOOD` example.
5. **Prove with a concurrency test** that fires simultaneous operations and
   asserts the invariant (a sequential call does not reproduce the race).
6. **Turn on the language's** static/dynamic detector (`references/lint-detectors.md`).

Detailed procedure for applying this to an existing codebase: `references/detection-workflow.md`.

## The mental model: the TOCTOU window

Every race is born from a **window** between *Time Of Check* and *Time Of Use*:

```
# BAD — check-then-act without atomicity (pseudocode, applies to any language)
acc = load(id)                 # CHECK: reads balance 100
if acc.balance >= amount:      #        decides based on the read value
    acc.balance -= amount      # USE:   writes 100-amount
    save(acc)                  # two concurrent executions read 100,
                               # both debit → wrong balance (lost update)
```

The cure is to **close the window**: make check+act a single atomic operation, or
serialize access to the resource. Never trust "it's too fast to race"
— under load, the window always opens.

### Where the race lives (changes by language)

| Layer | Who suffers | Defense mechanism |
|---|---|---|
| **Shared memory** | Java, C#, Go, C/C++, Rust, Kotlin, Swift (real threads) | atomics, mutex, memory barriers, `Sync`/`Sendable` types |
| **Process state** (GIL/event loop doesn't save you) | Python, Node.js, Ruby, PHP-async, Dart (isolate) | even so there is a logical race in I/O, cache, counters |
| **Database** (universal) | *all* — N replicas/processes | transaction, constraint, row lock, isolation level |

Even in runtimes with a GIL or single-thread event loop, the **logical** race exists
at the database/cache/file layer: read-decide-write crosses an `await`/IO and
another request enters the window. Per-language detail in `references/<language>.md`.

## Guard strategies (choose by case)

| Strategy | When | Mechanism |
|---|---|---|
| **Database constraint** | Uniqueness (slug, idempotency key, 1 vote/user) | `UNIQUE`/`EXCLUDE` + treat conflict as "lost the race" |
| **Conditional atomic update** | Counter, balance, stock | `UPDATE ... SET x = x - n WHERE x >= n` and check affected rows |
| **Pessimistic lock** | Check-then-act that doesn't fit in 1 UPDATE | `SELECT ... FOR UPDATE` inside a transaction |
| **Optimistic lock** | Low contention, avoid long lock | `version` column; `UPDATE ... WHERE version = :v`; retry on conflict |
| **Idempotency key** | POST that may be resent (payment, webhook) | unique key per operation; 2nd call returns the 1st result |
| **Advisory/distributed lock** | Serialize by logical key across processes | `pg_advisory_xact_lock(key)`, Redis lock with fencing token |
| **Atomic upsert** | Get-or-create | `INSERT ... ON CONFLICT DO UPDATE` |
| **Atomic in-memory** | Counter/flag in shared memory | `atomic`/`Interlocked`/`sync/atomic`, minimal-scope mutex |
| **Immutability / ownership** | Avoid sharing mutable state | messages/channels, `Arc`+`Mutex`, actors, immutable data |

Pseudocode of the two most common guards (idiomatic version per language in the references):

```
# 1) Conditional atomic update — without reading first (closes the window in the database)
n = UPDATE accounts SET balance = balance - :amt WHERE id = :id AND balance >= :amt
if n == 0: raise InsufficientFunds   # 0 rows = insufficient balance OR lost the race

# 2) Pessimistic lock — when you need logic between read and write
begin transaction
  acc = SELECT * FROM accounts WHERE id = :id FOR UPDATE   # serializes by row
  if acc.balance < amt: rollback; raise InsufficientFunds
  UPDATE accounts SET balance = balance - :amt WHERE id = :id
commit
```

## Per language

Load the target's reference. Each one brings: concurrency model,
**`BAD→GOOD`** examples of how to avoid them, idiomatic guard table, concurrency
test, and the ecosystem's **lint/detectors**.

| Language / runtime | Reference | Key detector |
|---|---|---|
| Python (asyncio, threads, Django/SQLAlchemy) | `references/python.md` | concurrent pytest, `bandit`, `flake8-async` |
| JavaScript / TypeScript (Node, browser, workers) | `references/javascript-typescript.md` | ESLint, `async-mutex`, DB locks |
| Java / Kotlin (JVM, coroutines) | `references/java-kotlin.md` | SpotBugs (`MT_*`), Error Prone `@GuardedBy`, Infer/RacerD |
| C# / .NET (async, TPL, EF Core) | `references/csharp-dotnet.md` | VS Threading Analyzers, Roslyn analyzers |
| Go (goroutines, channels) | `references/go.md` | **`go test -race`** (gold standard), `go vet` |
| Rust (async, threads) | `references/rust.md` | compiler (`Send`/`Sync`), `clippy`, Miri, `loom` |
| C / C++ (threads, atomics) | `references/cpp.md` | **ThreadSanitizer**, Helgrind, Clang Analyzer |
| Ruby (Rails, threads, GIL) | `references/ruby.md` | `rubocop-thread_safety`, `with_lock` |
| PHP (request-isolated, Laravel) | `references/php.md` | PHPStan, DB/cache locks |
| Swift (actors, structured concurrency) | `references/swift.md` | Swift 6 data-race safety, TSan |
| Dart / Flutter (isolates, async, event loop) | `references/dart.md` | `dart analyze`, `use_build_context_synchronously`, `package:synchronized` |
| **Database (universal)** | `references/databases-sql.md` | isolation levels, `FOR UPDATE`, constraints |

Didn't find the exact language? Use the file of the closest runtime + the
`databases-sql.md` (the database layer applies to all).

## Lint & static detection

Before claiming "it's safe", **turn on the language's detector** — full catalog
(install, config, coverage) in `references/lint-detectors.md`. Quick signal:
**Go** `go test -race` and **C/C++/Swift** ThreadSanitizer catch real data races;
**Java/Kotlin/C/C++** add Infer/RacerD and SpotBugs (`MT_CORRECTNESS`); **C#** the
VS Threading Analyzers; **Rust** the compiler + `clippy`/Miri/`loom`; **Ruby**
`rubocop-thread_safety`. **Python/JS/PHP** have weak race lint — rely on the
concurrency test + database guard. A detector complements but never replaces the
concurrency test: a logical database race is caught only by test and review.

## Validate with a concurrency test (the guardrail)

A race doesn't reproduce with a sequential call. The test **must** fire
simultaneous operations and assert the invariant afterward. Universal principle:

1. Prepare a resource with a known state (balance 100, free slug).
2. Fire **N concurrent operations** over the **same** resource, with a
   simultaneous start (barrier/`gather`/`WaitGroup`/`CountDownLatch`).
3. Assert the invariant (balance never negative, exactly 1 winner, correct count).
4. Run **many times** or under stress — a single pass may not open the window.

The concrete example in each language's test framework is in
`references/<language>.md`. Property-based testing (generating random
interleavings and asserting the invariant for any order) catches cases that fixed
examples don't cover.

## Review checklist

- [ ] Every check-then-act over shared state is atomic (conditional UPDATE, lock, or constraint).
- [ ] Uniqueness is guaranteed by a **database constraint**, not just by `if not exists`.
- [ ] Resendable POSTs (payment/webhook) have an idempotency key.
- [ ] An optimistic conflict becomes an **explicit error** or retry — never a silent write.
- [ ] Shared mutable state in memory is under atomic/lock (or is not shared).
- [ ] There is a test that fires **concurrent** operations and asserts the invariant.
- [ ] The transaction encompasses read+write (doesn't commit in the middle of the window).
- [ ] The lock has minimal scope (row/key), not the whole table/object — and consistent ordering (avoids deadlock).
- [ ] The language's detector ran in CI (`-race`, TSan, SpotBugs, analyzer...).

## Anti-patterns

- "It's fast, no race" — under real load the window opens. Always.
- "I have a GIL/event loop, so it's safe" — the logical race is in the database/cache, not in the CPU.
- `get_or_create`/`find_or_create` without `UNIQUE` in the database — creates duplicates under concurrency.
- Lock only in the application (mutex in 1 process) in a service with N replicas — doesn't serialize across processes.
- Read-decide-write in separate transactions (commit between check and act).
- Swallowing a constraint/version conflict without deciding winner/loser (silent corruption).
- Nested locks in inconsistent order across paths — deadlock.
- Testing concurrency with a sequential loop — doesn't reproduce the race.
- Relying only on the linter — it doesn't catch a logical race at the data layer.
