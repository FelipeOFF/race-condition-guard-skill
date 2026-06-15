# Race condition detection and remediation workflow

Procedure to apply the skill to an existing codebase — scan, classify, guard,
prove, and harden in CI. Follow in order; each step points to the concrete
artifact in the language reference.

## Step 0 — Establish the concurrency model

Before hunting races, answer for the target:

- **Is there shared memory across real threads?** (Java, C#, Go, C/C++, Rust,
  Kotlin, Swift) → there is a **data race** risk on top of the logical race.
- **Does the runtime serialize CPU?** (Python/GIL, Node/event loop, Ruby/GIL,
  PHP-FPM per request) → a memory data race is rare, but the **logical** race at
  the database/cache/file layer still applies.
- **How many processes/replicas serve the same state?** An in-memory lock in 1
  process does not serialize N replicas — only a distributed lock or a database
  lock does.

This decides whether you need atomics/mutex (memory) or
constraint/transaction/lock (database) — or both.

## Step 1 — Locate the risk points (grep + reading)

Look for the patterns that open a TOCTOU window. Useful commands (adjust to the
language):

```bash
# check-then-act: "exists? then create/update"
rg -n "get_or_create|find_or_create|first_or_create|firstOrCreate|exists\(|count\(\) == 0"

# read-modify-write of balance/counter/stock
rg -n "balance|stock|counter|quantity|amount" -g '!*test*'

# uniqueness only in the application (no constraint)
rg -n "if not .*exists|already.*taken|slug.*available"

# shared memory without a guard (real-thread languages)
rg -n "static .*=|global |std::thread|new Thread|go func\(|Task\.Run|@Volatile|synchronized"

# missing idempotency in POST/webhook
rg -n "webhook|callback|/payments|/charge|retry" -g '!*test*'
```

For each hit, ask: **do two concurrent callers break the invariant?**

## Step 2 — Classify each finding

| Symptom in the code | Class | Preferred guard |
|---|---|---|
| `if exists: return; create(...)` | check-then-act / TOCTOU | `UNIQUE` constraint in the database |
| `x = read(); x.n -= v; save(x)` | lost update | conditional atomic UPDATE or lock |
| payment POST without a key | double-submit | idempotency key (`UNIQUE`) |
| counter/flag in a shared variable | memory data race | atomic / mutex |
| `lazy = lazy or compute()` on a shared field | lazy-init race | `Once`/double-checked with barrier / lock |
| two locks in different orders | deadlock | total acquisition order |
| cache "check then set" | cache stampede / lost update | per-key lock or atomic set |

## Step 3 — Reuse before you build

Before writing a new guard, **search the repo for one that already exists** — in
any language. Reusing the project's own abstraction beats a fresh guard that
drifts from it. Look for two things: an existing version of the guard you need,
and existing implementations of the *other* guards you could apply instead.

```bash
# locks / mutual exclusion already in the codebase
rg -in "mutex|\bLock\(|synchronized|select_for_update|lockForUpdate|FOR UPDATE|with_lock"

# debounce / throttle / single-flight / click-handler
rg -in "debounce|throttle|single.?flight|in.?flight|isSubmitting|ClickHandler"

# idempotency / dedup middleware or keys
rg -in "idempoten|dedup|idempotency.?key"

# atomic update / upsert / optimistic version + unique constraints
rg -in "ON CONFLICT|upsert|@Version|lock_version|ConcurrencyCheck|UNIQUE|unique=True|unique: true"
```

Where the project's own guard tends to live, by ecosystem:

| Ecosystem | Look for |
|---|---|
| Python | `select_for_update`, `F()` helpers, a shared `Lock`/`asyncio.Lock` util, `unique=True` |
| JS/TS | `async-mutex`, a `Mutex`/`Semaphore` util, an idempotency middleware, a debounce in utils |
| Java/Kotlin | `@Version` entities, `ReentrantLock` wrappers, `@GuardedBy` fields, a coroutine `Mutex` |
| C#/.NET | `SemaphoreSlim` helpers, `[ConcurrencyCheck]`/`rowversion`, an `Interlocked` util |
| Go | a `sync.Mutex`/`errgroup` helper, `singleflight.Group`, `SELECT ... FOR UPDATE` |
| Rust | `Arc<Mutex<…>>` wrappers, an `AtomicUsize` util, a `sqlx` atomic-update helper |
| Ruby/Rails | `with_lock`, `lock_version` columns, `find_or_create_by` + unique index |
| PHP/Laravel | `lockForUpdate`, `Cache::lock`, unique-index migrations |
| Dart/Flutter | `package:synchronized` `Lock`, a debounce/`ClickHandler`, a single-flight util |

If found, **apply the existing one** and stop. Add a new guard only when none
exists — and put it where the project keeps shared utilities, not inline.

## Step 4 — Apply the idiomatic guard

Go to `references/<language>.md`, find the `BAD→GOOD` example for the class and
adapt it. Golden rule per layer:

- **Persisted in a database** → prefer **constraint + atomic UPDATE**;
  pessimistic lock only when you need logic between reading and writing.
  Cross-cutting details in `references/databases-sql.md`.
- **State in shared memory** → atomic for a simple counter/flag; **minimal
  scope** mutex for regions; even better, **don't share**
  (messages/channels/actors/immutability).
- **Across processes/replicas** → distributed lock with a **fencing token** or
  serialization in the database. Never a process mutex.

## Step 5 — Prove it with a concurrency test

Without a concurrent test, the fix is faith. Write the test **before/alongside**
the fix:

1. Run the test against the code **with the bug** → it must **fail** (reproduces the race).
2. Apply the guard → the test **passes**.
3. Run the test many times / under stress (`--count=100`, `-race`, loop) — a
   single pass may not open the window.

The skeleton in each language's framework is in `references/<language>.md`
(section "Prove the guard"). Universal pattern: set up state → simultaneous
launch of N operations → assert the invariant.

## Step 6 — Wire the detector into CI

Add the language detector to the pipeline (`references/lint-detectors.md`):

- Go: `go test -race ./...`
- C/C++/Swift: build with `-fsanitize=thread` and run the suite.
- Java/Kotlin: SpotBugs `MT_CORRECTNESS` + Error Prone `@GuardedBy`; optional Infer.
- C#: `Microsoft.VisualStudio.Threading.Analyzers` package (fail the build on warning).
- Rust: `cargo clippy` + tests under `loom` for the synchronization code.
- Ruby: `rubocop-thread_safety`.

A green detector does **not** prove the absence of a logical race in the
database — only the concurrency test and the review catch that. Use both.

## Step 7 — Final review

Run the `SKILL.md` checklist. Points that most often slip through:

- Application lock in a service with N replicas (does not serialize).
- `get_or_create` without a `UNIQUE` underneath.
- Version/constraint conflict swallowed without deciding the winner.
- Transaction that commits **in the middle** of the check→act window.
- Detector running only locally, absent from CI.

## Priority order when there are many findings

1. **Money and security first** (balance, charge, permission, vote).
2. **Identity uniqueness** (user, slug, idempotency key).
3. **Counters/stock** with overselling risk.
4. **Caches and flags** with lower consistency impact.

Fix from top to bottom; each fix lands with its concurrency test.
