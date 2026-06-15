# Race Conditions in PHP

## Concurrency model

PHP-FPM is **shared-nothing per request**: each request runs in an isolated process,
without sharing memory. For this reason **memory data races barely exist** in the
process — there is no concurrent thread touching the same variable. The race lives
**outside**: in the **database**, in the **shared cache** (Redis/Memcached), in the
**session** and in **files**. Two simultaneous requests (or the same POST
resubmitted) cross the TOCTOU window in that external state. Swoole/parallel/
pthreads bring real threads and then yes there is a memory data race — a separate case at the end.

## The races you will encounter

- **Memory data race** — rare. Only in long-running runtimes with real threads:
  ext-`parallel`, Swoole coroutines/workers, `pcntl_fork` sharing a resource.
  In classic PHP-FPM, it **does not happen** (state dies at the end of the request).
- **Logical race (the general rule)** — check-then-act crossing I/O:
  - **Lost update** in the database: `SELECT balance` → decide in PHP → `UPDATE`.
  - **Double-submit**: two POSTs create two records that should be one.
  - **Broken idempotency**: webhook/retry processed twice.
  - **Cache stampede**: N requests find the cache empty and recompute together.
  - **Session**: two requests from the same session without a lock corrupt `$_SESSION`.
  - **File**: two processes writing the same file without `flock`.

## How to avoid

### 1) Check-then-act in the database → conditional atomic UPDATE

```php
// BAD: read, decide in PHP, write — two requests read 100 and both debit
$stmt = $pdo->prepare('SELECT balance FROM accounts WHERE id = ?');
$stmt->execute([$id]);
$balance = $stmt->fetchColumn();
if ($balance >= $amount) {                          // TOCTOU window opens here
    $pdo->prepare('UPDATE accounts SET balance = ? WHERE id = ?')
        ->execute([$balance - $amount, $id]);       // lost update under concurrency
}

// GOOD: condition and mutation in a single command — the database closes the window
// (distinct placeholders: reusing the same name breaks with native prepares/pgsql)
$stmt = $pdo->prepare(
    'UPDATE accounts SET balance = balance - :amt
      WHERE id = :id AND balance >= :min'
);
$stmt->execute(['amt' => $amount, 'id' => $id, 'min' => $amount]);
if ($stmt->rowCount() === 0) {            // 0 rows = insufficient balance OR lost the race
    throw new InsufficientFundsException();
}
// see references/databases-sql.md (Pattern 1) for the cross-cutting detail
```

### 2) Logic between read and write → transaction + SELECT ... FOR UPDATE

```php
// GOOD: when the rule does not fit in an UPDATE, serialize the ROW inside the transaction
$pdo->beginTransaction();
try {
    $stmt = $pdo->prepare('SELECT * FROM accounts WHERE id = ? FOR UPDATE');
    $stmt->execute([$id]);               // another request blocks here until the COMMIT
    $acc = $stmt->fetch(PDO::FETCH_ASSOC);

    if ($acc['balance'] < $amount) {
        $pdo->rollBack();
        throw new InsufficientFundsException();
    }
    $pdo->prepare('UPDATE accounts SET balance = balance - ? WHERE id = ?')
        ->execute([$amount, $id]);
    $pdo->commit();
} catch (\Throwable $e) {
    if ($pdo->inTransaction()) $pdo->rollBack();   // never leave the transaction hanging
    throw $e;
}
// FOR UPDATE only serializes INSIDE a transaction; without beginTransaction it locks nothing
```

### 3) Uniqueness / double-submit → UNIQUE index + catch of the PDOException

```php
// BAD: classic TOCTOU — two POSTs pass the SELECT before any INSERT
$exists = $pdo->prepare('SELECT 1 FROM users WHERE email = ?');
$exists->execute([$email]);
if (!$exists->fetchColumn()) {
    $pdo->prepare('INSERT INTO users (email) VALUES (?)')->execute([$email]);
}                                        // result: two users with the same email

// GOOD: the UNIQUE index (in the schema) rejects the duplicate; the catch decides the loser
// ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);
try {
    $pdo->prepare('INSERT INTO users (email) VALUES (?)')->execute([$email]);
} catch (\PDOException $e) {
    if ($e->getCode() === '23000') {     // SQLSTATE 23000 = integrity constraint violation
        // lost the race: fetch and return the existing record (idempotent)
        return $this->findByEmail($email);
    }
    throw $e;
}
```

### 4) Cache stampede → atomic lock in Redis (single-flight)

```php
// BAD: N requests find the cache cold at the same time and recompute together (stampede)
$value = $redis->get($key);
if ($value === false) {
    $value = expensiveCompute();         // all requests enter here at once
    $redis->set($key, $value, 300);
}

// GOOD: only one request recomputes; SET NX is the atomic acquisition operation
$value = $redis->get($key);
if ($value === false) {
    // SET lock NX (only if free) + PX (TTL so it doesn't lock forever if the process dies)
    if ($redis->set("lock:$key", '1', ['NX', 'PX' => 5000])) {
        try {
            $value = expensiveCompute();
            $redis->set($key, $value, 300);
        } finally {
            $redis->del("lock:$key");    // release in the finally, even if compute throws
        }
    } else {
        usleep(50_000);                  // lost the lock: wait and re-read the now-warm cache
        $value = $redis->get($key);
    }
}
```

### 5) Laravel → DB::transaction + lockForUpdate (or atomic Cache::lock)

```php
// BAD: Laravel's 'unique' validation does NOT prevent a race — it does a SELECT
//      and two requests pass validation before any INSERT.
$request->validate(['email' => 'unique:users']);   // TOCTOU: not enough on its own
User::create(['email' => $request->email]);          // needs a UNIQUE index in the schema

// GOOD (check-then-act with logic): transaction + lockForUpdate serializes the row
DB::transaction(function () use ($id, $amount) {
    $acc = Account::where('id', $id)->lockForUpdate()->first();  // SELECT ... FOR UPDATE
    abort_if($acc->balance < $amount, 422, 'Insufficient funds');
    $acc->decrement('balance', $amount);             // decrement is already an atomic UPDATE
});

// GOOD (serialize by logical key across requests/replicas): atomic lock
Cache::lock("close-books:$day", 10)->block(5, function () {
    // only one process at a time enters here; blocks up to 5s trying to acquire
    closeBooks($day);
});
```

## Idiomatic guards

| Primitive | Layer | When to use |
|---|---|---|
| `UPDATE ... WHERE x >= n` + `rowCount()` | Database (PDO) | Balance/stock/counter — closes the window without a lock |
| `PDO::beginTransaction` + `SELECT ... FOR UPDATE` | Database (PDO) | Check-then-act with logic that does not fit in an UPDATE |
| `UNIQUE` index + catch `PDOException` 23000 | Database | Double-submit, uniqueness, idempotency key |
| `DB::transaction(...)` | Laravel | Wraps read+write; rolls back on throw |
| `->lockForUpdate()` | Eloquent | Pessimistic row lock inside the transaction |
| `->sharedLock()` | Eloquent | Lock for consistent read (others read, nobody writes) |
| `Model::increment()/decrement()` | Eloquent | Atomic counter UPDATE (does not read first) |
| `Cache::lock($k, $ttl)->block()/->get()` | Laravel (Redis/DB) | Serialize by logical key across processes/replicas |
| `$redis->set($k,$v,['NX','PX'=>$ms])` | Redis (phpredis) | Raw distributed lock, anti-stampede |
| `flock($fp, LOCK_EX)` | File | Serialize writes to the same file across processes |
| `session_write_close()` | Session | Release the session lock early (avoid serializing requests pointlessly) |

## Prove the guard

PHP-FPM is one process per request — the test **must** fire real and
simultaneous HTTP requests against the shared state. A sequential PHPUnit loop
does not reproduce the race. Use a concurrent Guzzle pool and assert the invariant.

```php
// tests/Feature/ConcurrentDebitTest.php  (PHPUnit + Guzzle pool)
use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Psr7\Request;
use PHPUnit\Framework\TestCase;

final class ConcurrentDebitTest extends TestCase
{
    public function test_concurrent_debits_never_overdraw(): void
    {
        // 1) prepare: account with balance 100, debit of 10 → at most 10 successes
        seedAccount(id: 1, balance: 100);

        $client   = new Client(['base_uri' => 'http://localhost:8080']);
        $requests = function () {
            for ($i = 0; $i < 50; $i++) {                 // 50 simultaneous debits
                yield new Request('POST', '/accounts/1/debit', [], json_encode(['amount' => 10]));
            }
        };

        $ok = 0;
        // 2) fire all 50 at once (high concurrency = the window opens)
        $pool = new Pool($client, $requests(), [
            'concurrency' => 50,
            'fulfilled'   => function ($resp) use (&$ok) {
                if ($resp->getStatusCode() === 200) $ok++;
            },
        ]);
        $pool->promise()->wait();

        // 3) invariant: balance never negative AND at most 10 debits went through
        $this->assertLessThanOrEqual(10, $ok, 'more debits than the balance allowed');
        $this->assertGreaterThanOrEqual(0, currentBalance(1), 'balance went negative');
    }
}
```

Quick alternative without a test framework, for a local smoke test: `seq 50 | xargs
-P50 -I{} curl -s -X POST localhost:8080/accounts/1/debit -d 'amount=10'` and
then check the balance. `-P50` is the parallel start.

## Lint & static detection

PHPStan and Psalm focus on **types and flow**, not on concurrency — there is no
ThreadSanitizer nor `-race` for PHP. They **do not catch** lost update, database
TOCTOU, nor cache stampede. What they do catch is indirect and still useful:

```bash
vendor/bin/phpstan analyse src --level=max   # unhandled exception, ignored error branch
vendor/bin/psalm                              # swallowed PDOException, unchecked return
```

- **PHPStan/Psalm/Larastan** flag a missing `catch` around the `INSERT` and a
  discarded `rowCount()`/return — a type heuristic, not race detection.
- **There is no** PHP linter that detects a logical database/cache race. The real
  defense is **a guard in the database (UNIQUE/FOR UPDATE/atomic UPDATE) + a lock in Redis + a
  concurrency test**. See `references/databases-sql.md` and `references/lint-detectors.md`.

## PHP-specific anti-patterns

- **`unique:users` (validation) as the only guard** — it is a `SELECT`; two
  requests pass before the `INSERT`. It only closes with a **UNIQUE index** in the schema.
- **`firstOrCreate` / `updateOrCreate` without a UNIQUE index** — they have a TOCTOU window
  underneath (SELECT then INSERT). Under a race they create a duplicate; they need the
  unique index for the INSERT to collide and Eloquent to re-resolve.
- **`lockForUpdate()` outside `DB::transaction`** — without an open transaction the lock
  holds nothing; the `FOR UPDATE` is released at the end of the statement.
- **Forgetting `rowCount()`** after the atomic UPDATE — without checking affected
  rows, "lost the race" passes as a silent success.
- **Cache::lock only in the application with the `array`/`file` driver per node** — it does not
  serialize across replicas. Use the Redis/database driver for a real distributed lock.
- **Transaction opened without `rollBack` in the catch** — the connection is left with a hanging
  transaction and the next use of the pool inherits dirty state.
- **Session lock holding requests** — `$_SESSION` is file-locked until
  the request ends; parallel AJAX from the same session serializes. Call
  `session_write_close()` early when you will no longer write to the session.
- **Writing a file without `flock($fp, LOCK_EX)`** — two FPM processes corrupt
  the content. `file_put_contents(..., LOCK_EX)` or explicit `flock`.
- **(Swoole/parallel) treating a long-running worker variable as request-scoped**
  — in a persistent runtime the state **survives** between requests and becomes shared
  memory; there Swoole's mutex/atomic apply, not the habits of PHP-FPM.
