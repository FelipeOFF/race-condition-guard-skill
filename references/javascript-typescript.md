# Race Conditions in JavaScript / TypeScript

## Concurrency model

Node and the browser run JavaScript on a **single-thread event loop**: two
synchronous stretches never run at once, so ordinary code has **no memory data
race**. The danger is the **logical race**: every `await` (or I/O callback) is a
*yield* point where another task steps in. Read a value, `await`, then write based
on that read, and you opened a TOCTOU window — another request ran in between. The
exception is `worker_threads` + `SharedArrayBuffer`: genuine shared memory and a
**hardware data race**, tamed only with `Atomics`.

## The races you'll run into

- **Read-modify-write crossing an `await`** — `x = await load(); x.n++; await save(x)`.
  Two executions read the same `x`, both increment, one write is lost
  (lost update). The case ESLint's `require-atomic-updates` targets.
- **Double-submit** — user clicks twice, or the client resends the POST; two
  handlers create two records / charge twice. A race between **requests**.
- **Concurrent cache / lazy-init** — N requests see the empty cache before the
  first fills it, all firing the same expensive fetch (cache stampede).
- **Real data race in `SharedArrayBuffer`** — a counter shared across
  `worker_threads` without `Atomics`: the non-atomic increment loses writes on the
  hardware, just like C. Only here is the problem memory, not logic.
- **Persisted state** — any of the above touching the database needs the guard
  **in the database**; an in-memory lock of 1 process is worthless for N replicas.

## How to avoid it

```ts
// BAD: read-modify-write with an await in the middle → lost update between requests
async function deposit(id: string, amt: number) {
  const acc = await db.account.findUnique({ where: { id } }); // CHECK: reads balance
  // another request runs here, reads the SAME balance
  await db.account.update({                                   // USE: writes stale
    where: { id }, data: { balance: acc.balance + amt },
  });
}

// GOOD: atomic update in the database — relative increment, no read first
async function deposit(id: string, amt: number) {
  await db.account.update({
    where: { id },
    data: { balance: { increment: amt } }, // the database resolves the concurrency
  });
}
```

```ts
// BAD: check-then-act in memory — two requests pass the if before any write
type Req = { idempotencyKey: string };
const seen = new Set<string>();
async function handle(req: Req) {
  if (seen.has(req.idempotencyKey)) return cached(req);
  await process(req);          // await opens the window; 2nd request already passed the if
  seen.add(req.idempotencyKey);
}

// GOOD: uniqueness lives in the database; conflict = "lost the race"
async function handle(req: Req) {
  try {
    await db.operation.create({ data: { idempotencyKey: req.idempotencyKey } });
  } catch (e) {
    if (isUniqueViolation(e)) return cached(req); // 2nd call returns the 1st
    throw e;
  }
  await process(req);
}
```

```ts
// BAD: double click fires two POSTs — without a lock, two orders are created
button.onclick = async () => { await api.placeOrder(cart); };

// GOOD: disable immediately (synchronous, before the await) and re-enable in finally
button.onclick = async () => {
  if (button.disabled) return;
  button.disabled = true;       // state becomes synchronous → 2nd click hits the guard
  try { await api.placeOrder(cart); }
  finally { button.disabled = false; }
};
// UI defense does not replace the server guard (idempotency key + UNIQUE).
```

```ts
import { Mutex } from "async-mutex";

// BAD: concurrent lazy-init — N requests see null and all fetch (stampede)
let config: Config | null = null;
async function getConfig() {
  if (config) return config;
  config = await fetchConfig(); // await: other requests enter here too
  return config;
}

// GOOD: serialize the initialization; only the 1st fetches, the rest wait on the lock
const mutex = new Mutex();
let cfg: Config | null = null;
async function getConfig() {
  if (cfg) return cfg;                       // fast-path without lock
  return mutex.runExclusive(async () => {
    if (cfg) return cfg;                      // double-check inside the lock
    cfg = await fetchConfig();
    return cfg;
  });
}
```

```ts
// BAD: REAL data race — non-atomic increment on shared memory
const view = new Int32Array(sharedArrayBuffer);
view[0] += 1;            // read+add+write: two workers overwrite each other

// GOOD: atomic hardware increment (worker_threads / SharedArrayBuffer)
Atomics.add(view, 0, 1); // indivisible across workers; only here is Atomics necessary
```

## Idiomatic guards

| Primitive | When to use |
|---|---|
| Atomic ORM update (`{ increment }`, `updateMany` + conditional `where`) | Persisted counter/balance/stock — closes the window in the database (1st choice) |
| `UNIQUE` in the schema + handle conflict error | Double-submit, idempotency, unique slug — see `references/databases-sql.md` |
| `$transaction` (Prisma) / `BEGIN…FOR UPDATE` | Check-then-act too big for 1 UPDATE — pessimistic row lock |
| `Mutex` / `Semaphore` (`async-mutex`) | Serialize a critical section **within 1 process** (cache init, local queue) |
| Serial queue (`p-queue` concurrency: 1) | Order work by key in a single process/worker |
| `disabled`/synchronous flag + `finally` | Double-submit on the front end (UX defense, not server) |
| `Atomics.*` over `SharedArrayBuffer` | Counter/flag shared across `worker_threads` (real data race) |
| Redis `SET NX PX` + fencing token | Distributed lock across replicas — see `references/databases-sql.md` |

Rule: for **persisted state**, the definitive guard is always in the database. An
in-memory `Mutex` serializes within one process only — with N replicas it
serializes nothing. Use the in-memory lock for a local resource; the database
for the rest.

## Proving the guard

A race won't reproduce sequentially. Fire N simultaneous operations with
`Promise.all` over the **same** resource and assert the invariant (Vitest/Jest).

```ts
import { describe, it, expect, beforeEach } from "vitest";

describe("withdraw under concurrency", () => {
  beforeEach(async () => {
    await db.account.upsert({
      where: { id: "a1" }, create: { id: "a1", balance: 100 }, update: { balance: 100 },
    });
  });

  it("never leaves the balance negative with 20 simultaneous withdrawals", async () => {
    // conditional atomic withdrawal: only debits if there is balance
    const withdraw = () =>
      db.account.updateMany({
        where: { id: "a1", balance: { gte: 10 } },
        data: { balance: { decrement: 10 } },
      });

    // simultaneous start: 20 withdrawals of 10 against balance 100
    const results = await Promise.all(Array.from({ length: 20 }, withdraw));

    const ok = results.filter((r) => r.count === 1).length; // count=0 → lost the race
    const acc = await db.account.findUniqueOrThrow({ where: { id: "a1" } });

    expect(ok).toBe(10);          // exactly 10 withdrawals won
    expect(acc.balance).toBe(0);  // invariant: never negative
  });
});
```

Run it with repetition (`--retry`/loop) or against a real database — in-memory
SQLite serializes too much and can hide the race. The point is the `Promise.all`:
a joint start.

## Lint & static detection

TypeScript does **not** detect races (the type system doesn't model time). The
ecosystem's lint is weak for concurrency — rely on **concurrent test + database
guard**. What exists:

```jsonc
// eslint.config / .eslintrc — relevant ESLint core rules
{ "rules": {
    // reassigning a variable after an await may discard a concurrent update
    "require-atomic-updates": "error",
    // sequential await in a loop = didn't parallelize; flags serial read-modify-write
    "no-await-in-loop": "warn"
}}
```

- **`require-atomic-updates`** catches the `x = await ...; x = ... x ...` pattern on
  the same variable/property — the classic intra-function lost update. False
  negatives: it misses a race **between** functions and at the database layer.
- **`no-await-in-loop`** is not a race rule, but flags loops where you should
  probably use `Promise.all` — and where serial read-modify-write lives.
- There is no ThreadSanitizer for JS; a data race in `SharedArrayBuffer` is **not**
  caught by a linter. The defense is using `Atomics` by construction.

Cross-cutting catalog in `references/lint-detectors.md`. A green linter doesn't
clear review: a logical database race shows up only in the concurrent test and by eye.

## JavaScript / TypeScript-specific anti-patterns

- **"Single-thread, therefore safe"** — the event loop only protects the CPU; each
  `await` is an entry point for another request. The logical race is alive.
- **In-memory `Mutex` in a service with N replicas** — doesn't serialize across
  processes/instances. For shared state, the guard is in the database or a
  distributed lock (Redis with fencing).
- **App-level `findOrCreate` / `upsert` without `UNIQUE` underneath** — has a TOCTOU
  window; creates a duplicate under a race. Ensure the constraint in the schema.
- **`Promise.all` without a limit over a contended resource** — fires 10k
  simultaneous writes and blows the pool/connection; throttle with `p-limit`/`p-queue`.
- **`forEach` with an `async` callback** — doesn't await the promises; the function
  "finishes" before the writes, hiding the window. Use `for…of` + `await` or `Promise.all`.
- **Swallowing a `UNIQUE`/conflict error without picking a winner** — a silent
  `catch {}` turns the race into silent corruption; treat it as "lost, return the 1st".
- **Forgotten `Atomics`** — any read-modify-write on `SharedArrayBuffer` without
  `Atomics` is a real data race, even if it "looks atomic".
