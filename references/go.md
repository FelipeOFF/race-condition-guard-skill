# Race Conditions in Go

## Concurrency model

Go has **real threads** (goroutines multiplexed over OS threads) and **truly
shared memory**: two goroutines touching the same variable without
synchronization cause a **real data race**, with corrupted read/write and
undefined behavior. The danger lives in any mutable state reachable by
more than one goroutine — counters, `map`, slices, struct fields, caches. The
idiomatic philosophy is "*share memory by communicating*" (pass ownership through
**channels** instead of protecting state with a lock), but when the lock is simpler
`sync/atomic` and `sync.Mutex` are equally idiomatic.

## The races you'll encounter

- **Memory data race** — N goroutines increment an `int` or write to a
  `map` without synchronization. The `map` is the most brutal case: concurrent
  write to a `map` **triggers `fatal error: concurrent map writes`** and brings down
  the process (not recoverable with `recover`).
- **Duplicate lazy-init** — two goroutines see `conn == nil` and both initialize
  (connection/cache created twice). Classic `sync.Once` case.
- **Lost update via weak lock** — `Lock`/`Unlock` poorly scoped, or read-outside /
  write-inside the lock, leaving the window open.
- **Logical race in the database** — check-then-act in separate transactions (balance, unique
  slug, idempotency). A goroutine won't save you from this: the window is in the database. Cross-cutting
  detail in `references/databases-sql.md`.
- **Goroutine leak / missing cancellation** — without `context`, goroutines leak and
  keep mutating state after the request has died.

## How to avoid

### 1) Concurrent counter — `sync/atomic`

```go
// BAD: counter++ is load-add-store; two goroutines lose increments (real data race)
var counter int
for i := 0; i < 1000; i++ {
    go func() { counter++ }() // -race flags here
}

// GOOD: atomic increment — the hardware guarantees an indivisible read-modify-write
var counter atomic.Int64       // Go 1.19+: atomic type, no loose &var to pass
for i := 0; i < 1000; i++ {
    go func() { counter.Add(1) }()
}
_ = counter.Load()             // reads must also be atomic
```

### 2) Concurrent map — minimal-scope mutex (or `sync.Map`)

```go
// BAD: concurrent write to a native map → "fatal error: concurrent map writes" (kills the process)
cache := map[string]int{}
go func() { cache["a"] = 1 }()
go func() { cache["b"] = 2 }() // non-recoverable crash, not just a "silent" race

// GOOD: serialize access to the map with a mutex; the lock covers every access (read and write)
type Cache struct {
    mu sync.Mutex            // protects m; never read m outside the lock
    m  map[string]int
}
func (c *Cache) Set(k string, v int) {
    c.mu.Lock()
    defer c.mu.Unlock()      // defer guarantees unlock even on panic
    c.m[k] = v
}
// sync.Map is an alternative only when the pattern is write-once / read-many disjoint by key.
```

### 3) Single init — `sync.Once`

```go
// BAD: two goroutines pass the nil-check together and open two connections (duplicate lazy-init)
var conn *DB
func Get() *DB {
    if conn == nil {        // CHECK and USE in distinct goroutines → TOCTOU window
        conn = connect()
    }
    return conn
}

// GOOD: sync.Once runs the function exactly once, even with N concurrent goroutines
var (
    once sync.Once
    conn *DB
)
func Get() *DB {
    once.Do(func() { conn = connect() }) // Do blocks the others until the 1st finishes
    return conn
}
```

### 4) Worker pool by channel — "share by communicating" instead of lock

```go
// BAD: N goroutines contend for a lock to push onto a shared slice (contention + race if you forget the lock)
var mu sync.Mutex
var out []Result
for _, job := range jobs {
    go func(j Job) { mu.Lock(); out = append(out, process(j)); mu.Unlock() }(job)
}

// GOOD: each worker owns its work; the channel transfers the result without shared state
results := make(chan Result)
for _, job := range jobs {
    go func(j Job) { results <- process(j) }(job) // no lock: the channel synchronizes
}
out := make([]Result, 0, len(jobs))
for range jobs {
    out = append(out, <-results) // only the collector touches the slice → no race
}
```

### 5) Cancellation — `context` instead of orphan goroutine

```go
// BAD: a goroutine with no stop signal stays alive (and mutating state) after the request dies
func handle(ch chan int) {
    go func() { for { ch <- expensive() } }() // leaks forever
}

// GOOD: context propagates cancellation; the goroutine ends when the caller cancels
func handle(ctx context.Context, ch chan int) {
    go func() {
        for {
            select {
            case <-ctx.Done(): return        // releases the goroutine on cancel/timeout
            case ch <- expensive():
            }
        }
    }()
}
```

### 6) Check-then-act in the database — atomic UPDATE (not `database/sql` read-decide-write)

```go
// BAD: reads balance, decides in Go, writes → lost update under concurrency
row := db.QueryRowContext(ctx, "SELECT balance FROM accounts WHERE id=$1", id)
// ... if balance >= amt { UPDATE ... } — two goroutines/replicas enter the window

// GOOD: condition and mutation in a single atomic UPDATE; check affected rows
res, err := db.ExecContext(ctx,
    "UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1", amt, id)
n, _ := res.RowsAffected()
if n == 0 { return ErrInsufficientFunds } // 0 rows = insufficient balance OR lost the race
```

Cross-cutting detail (isolation levels, `FOR UPDATE`, uniqueness constraint,
optimistic lock) in `references/databases-sql.md`.

## Idiomatic guards

| Primitive | When to use |
|---|---|
| `atomic.Int64`/`atomic.Bool`/`atomic.Pointer[T]` | counter, flag, swapped pointer — read-modify-write without lock |
| `sync.Mutex` (+ `defer Unlock`) | protect struct/map/slice; minimal scope, always covers every access |
| `sync.RWMutex` | many readers, few writers (read-heavy); don't use it by reflex |
| `sync.Once` | single init of connection/singleton/lazy cache |
| `sync.Map` | write-once/read-many cache with disjoint keys; not a generic mutex |
| `chan T` + owner goroutine | transfer ownership ("share by communicating"); pipelines, fan-out/fan-in |
| `context.Context` | cancellation/timeout — propagates stop, avoids goroutine leak |
| `errgroup.Group` | fan-out of goroutines with aggregated error/cancellation |
| Constraint / atomic UPDATE in the database | logical race between replicas — `references/databases-sql.md` |

## Prove the guard

`sync.WaitGroup` releases N goroutines onto the same state and the test asserts the invariant.
Always run **with `-race`** — without it, the BAD version may pass by luck.

```go
package counter

import (
	"sync"
	"sync/atomic"
	"testing"
)

// Invariant: after N concurrent increments, the counter equals exactly N.
func TestCounterIsRaceFree(t *testing.T) {
	const n = 1000
	var counter atomic.Int64
	var start, wg sync.WaitGroup
	start.Add(1) // barrier: releases all goroutines at once (simultaneous start)

	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			start.Wait()      // wait for the start → maximizes the race window
			counter.Add(1)
		}()
	}

	start.Done() // releases the n goroutines simultaneously
	wg.Wait()    // wait for all to finish

	if got := counter.Load(); got != n {
		t.Fatalf("invariant violated: expected %d, got %d (lost update)", n, got)
	}
}
```

```bash
go test -race ./...   # run it like this: without -race a data race may not fail the test
```

## Lint & static detection

**`go test -race` is the gold standard** — ThreadSanitizer built into the toolchain. It
**instruments every memory access** at runtime and fails on the first real data race
observed (with stacks of the two conflicting goroutines). Cost: ~2-10× more
slow, ~5-10× more memory; that's why it runs in the suite, not in production.

```bash
go test -race ./...      # fails on the first real data race exercised by the tests
go build -race ./...     # instrumented binary to run in staging/canary
go vet ./...             # copylocks: catches a struct with sync.Mutex copied by value
```

`go vet` (copylocks) catches the classic error of **copying a struct that contains a
`sync.Mutex`** — the copy carries a "different" lock and the protection evaporates; that's why
methods that mutate must have a pointer receiver. `staticcheck` (external
tool, `staticcheck ./...`) adds lints such as misuse of `sync` and
loop var pointer in a goroutine (pre-Go 1.22).

**What `-race` does NOT catch:** only what **executes** — if the test doesn't exercise the
concurrent path, the race slips through. And it **doesn't catch a logical race in the database**
(lost update in separate transactions): that only shows up in a concurrency test +
database guard. Full catalog in `references/lint-detectors.md`.

## Go-specific anti-patterns

- **Capturing the loop variable in a goroutine** (`for _, v := range … { go func(){ use(v) }() }`)
  — before Go 1.22 all saw the **last** `v`. In Go 1.22+ the scope is per
  iteration; in old code, pass it as an argument: `go func(v T){…}(v)`.
- **Reading a field outside the lock and writing inside** — the lock only holds if it covers
  **every** access (read and write) to the protected variable.
- **`sync.RWMutex` by reflex** — under write-heavy load the `RWMutex` is slower than
  `Mutex`; it's only worth it for real read-heavy.
- **`sync.Map` as a generic mutex** — it's optimized for write-once/read-many; for
  general access, `Mutex` + `map` is faster and clearer.
- **Copying `sync.Mutex`/`WaitGroup`/`Once` by value** (passing the struct by value,
  reassigning) — breaks the protection. `go vet` warns; use a pointer.
- **`time.Sleep` to "synchronize" goroutines** — that's not synchronization, it's a disguised
  race. Use a channel, `WaitGroup` or `context`.
- **Channel without closing / without `select` on `ctx.Done()`** — goroutines block
  forever and leak. Always have an exit path via `context`.
- **`recover` to "handle" `concurrent map writes`** — it's a `fatal error`, not a panic;
  `recover` doesn't catch it. The only cure is to synchronize access to the map.
