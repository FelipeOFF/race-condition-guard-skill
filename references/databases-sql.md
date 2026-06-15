# Race conditions at the database layer (universal)

Applies to **every** language: no matter how much the runtime serializes CPU (GIL, event
loop) or how many N replicas you run, the database is the shared point of truth.
Most backend races are resolved **here** — and resolved well, because the database already
has atomic primitives. This is the cross-cutting reference cited by all the
others.

## Why the database is the right place for the guard

- It is **single** even with N processes/replicas (an application lock is not).
- It has **transactional atomicity** and **constraints** that the app does not.
- `UPDATE ... WHERE`, `INSERT ... ON CONFLICT`, and `SELECT ... FOR UPDATE`
  close the TOCTOU window without you inventing locking.

## Isolation levels — what each one gives you

| Level | Prevents | Still allows | Typical use |
|---|---|---|---|
| READ COMMITTED (default PG/Oracle) | dirty read | non-repeatable read, lost update, phantom | most cases; **needs** atomic UPDATE/explicit lock |
| REPEATABLE READ (default MySQL InnoDB) | + non-repeatable read | phantom (PG: no), write skew | consistent read |
| SERIALIZABLE | everything (write skew included) | nothing — may abort with serialization error | multi-row invariant; **handle the retry** |

Rule of thumb: in READ COMMITTED, **check-then-act is not safe by default** —
you need `FOR UPDATE`, a conditional UPDATE, or a constraint. SERIALIZABLE
resolves write skew (e.g., "at most 2 doctors on call") but requires a **retry on the
serialization error** (`40001` / deadlock).

## Pattern 1 — Conditional atomic UPDATE (lost update)

The window closes in the database: nothing is read beforehand in the app.

```sql
-- BAD: app reads, decides, writes (lost update under concurrency)
SELECT balance FROM accounts WHERE id = 1;     -- 100
-- app: if balance >= 30 ...
UPDATE accounts SET balance = 70 WHERE id = 1;  -- two requests write 70

-- GOOD: condition and mutation in a single atomic command
UPDATE accounts
   SET balance = balance - 30
 WHERE id = 1 AND balance >= 30;
-- affected rows == 0  → insufficient balance OR lost the race → explicit error
```

Always **check the affected rows**. 0 rows is the "lost" signal.

## Pattern 2 — Uniqueness constraint (check-then-act / double-submit)

Don't trust `if not exists`. Let the database reject the duplicate.

```sql
ALTER TABLE users ADD CONSTRAINT uq_email UNIQUE (email);
-- idempotency:
ALTER TABLE operations ADD CONSTRAINT uq_idem UNIQUE (idempotency_key);
```

```sql
-- 2nd request with the same key violates the constraint → treat as "lost the race"
INSERT INTO operations (idempotency_key, ...) VALUES (:key, ...);
-- on conflict/duplicate → fetch and return the result of the 1st
```

`EXCLUDE` (Postgres) enforces uniqueness by overlap (e.g., room reservations without
time clash) — `EXCLUDE USING gist (room WITH =, period WITH &&)`.

## Pattern 3 — Pessimistic lock (`SELECT ... FOR UPDATE`)

When you need **logic between read and write** that doesn't fit in an UPDATE.

```sql
BEGIN;
  SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- serializes per row
  -- app logic with the locked data
  UPDATE accounts SET balance = balance - 30 WHERE id = 1;
COMMIT;
```

- `FOR UPDATE SKIP LOCKED` — work queues: each worker grabs a free row.
- `FOR UPDATE NOWAIT` — fail fast instead of waiting for the lock.
- **Always lock in the same order** across transactions to avoid deadlock.

## Pattern 4 — Optimistic lock (`version` column)

Low contention; avoids holding a lock. Detects the conflict at write time.

```sql
-- reads version=7; on write requires that nobody touched it
UPDATE docs SET body = :new, version = 8 WHERE id = 1 AND version = 7;
-- 0 rows → someone wrote first → reload and try again (or error)
```

ORMs already do this: Hibernate/JPA `@Version`, Django `select_for_update` vs
version fields, Rails `lock_version` (native optimistic locking), EF Core
`[ConcurrencyCheck]`/`rowversion`.

## Pattern 5 — Atomic upsert (get-or-create)

```sql
-- Postgres
INSERT INTO counters (key, n) VALUES (:k, 1)
ON CONFLICT (key) DO UPDATE SET n = counters.n + 1
RETURNING n;

-- MySQL
INSERT INTO counters (key, n) VALUES (:k, 1)
ON DUPLICATE KEY UPDATE n = n + 1;
```

Replaces the app's `get_or_create` (which has a TOCTOU window) with a single
atomic operation.

## Pattern 6 — Advisory lock (serialize by logical key)

When what you want to serialize is not a row (e.g., "only one closing job
per day").

```sql
-- Postgres: lock by key, released at the end of the transaction
SELECT pg_advisory_xact_lock(hashtext('close-books-2026-06-15'));
-- ... work serialized by this key ...
COMMIT;  -- releases
```

## Distributed lock (Redis) — when there is no transactional database in the path

Use `SET key token NX PX <ttl>` to acquire and a **fencing token** to prevent
an expired lock and a new owner from writing out of order. Caution: a distributed
lock is more fragile than the database — prefer the guard in the database when the
state already lives there.

```
ok = SET lock:res <token> NX PX 30000      # acquire only if free, with TTL
# ... short critical section ...
# release only if the token is still yours (Lua compare-and-del script)
```

## Pitfalls

- **READ COMMITTED + check-then-act** without `FOR UPDATE`/atomic UPDATE = lost update.
- **Write skew** (two sides read, each decides, together they break the invariant)
  only disappears in SERIALIZABLE — and requires handling the serialization error with retry.
- **`get_or_create` without `UNIQUE`** underneath creates a duplicate under a race.
- **Long transaction** holding a heavy lock kills throughput and causes deadlock.
- **Commit in the middle** of the check→act window reopens the race.
- **Lock only in Redis** without a fencing token: clock/TTL can leave two owners.

## Data-layer checklist

- [ ] Check-then-act became a conditional UPDATE, `FOR UPDATE`, or constraint.
- [ ] Uniqueness/idempotency has a real `UNIQUE` in the schema.
- [ ] Affected rows / conflict are checked and turn into an explicit error.
- [ ] Multi-row invariant (write skew) uses SERIALIZABLE **with retry**.
- [ ] Pessimistic lock locks in the same order on all paths (no deadlock).
- [ ] Transaction is short and encompasses read+write without committing in the middle.
