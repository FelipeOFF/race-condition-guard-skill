# Race Conditions in Rust

## Concurrency model

Rust is the rare case where the **compiler blocks memory data races at compile
time**. The borrow checker guarantees "one mutable reference XOR several
immutable ones", and the `Send`/`Sync` traits control what can cross threads: a
type only moves to another thread if it is `Send`, and is only shared by
reference if it is `Sync`. `Rc<T>` is not `Send` (non-atomic refcount), so the
code simply does not compile — you are forced to `Arc<T>`. What **remains** is
everything the type cannot see: **logical** race (check-then-act over data that
each one locks in isolation), **deadlock** by lock order, and the race at the
**database** layer.

## The races you will encounter

- **Memory data race** — in safe code it is **impossible by construction**;
  what exists is in `unsafe`, FFI, or wrong use of `*const`/`*mut`. For that
  residue there is Miri.
- **Logical race** — two locks/atomics individually correct, but the sequence
  between them is not atomic (e.g.: `if map.lock().is_empty() { ... }` and then a
  second lock based on that already stale read). The compiler does not see it.
- **Deadlock** — two `Mutex` locked in opposite orders in different threads;
  or locking the **same** `std::sync::Mutex` twice in the same thread (it is not
  reentrant → locks forever).
- **Database race** — N processes/replicas: check-then-act in SQL is a lost update,
  exactly like in any other language. See `references/databases-sql.md`.

## How to avoid

```rust
// 1) Shared counter — atomic, not Mutex<usize>
// BAD: data race? does not compile. But Rc does not even cross threads; and read-add-write
//      with separate load/store is a LOGICAL race even in AtomicUsize.
let n = AtomicUsize::new(0);
let cur = n.load(Ordering::Relaxed);
n.store(cur + 1, Ordering::Relaxed); // window between load and store: lost update

// GOOD: fetch_add is a single atomic read-modify-write operation — no window.
let n = AtomicUsize::new(0);
n.fetch_add(1, Ordering::Relaxed); // Relaxed is enough for a pure counter (no syncing of other data)
```

```rust
// 2) Shared mutable state — Arc<Mutex<T>>, minimal-scope lock
// BAD: holding the lock across heavy work serializes everything and invites deadlock.
let guard = state.lock().unwrap();
do_expensive_io(&guard); // I/O with the lock held: throughput dies, deadlock easy
// GOOD: copy what you need, drop the lock, then work.
let snapshot = { let g = state.lock().unwrap(); g.clone() }; // guard dropped at the end of the block
do_expensive_io(&snapshot);
```

```rust
// 3) Check-then-act on a Mutex — one critical section, not two locks
// BAD: the "is_empty" and the "insert" are two locks; another thread enters the window.
if shared.lock().unwrap().is_empty() {
    shared.lock().unwrap().insert(key, val); // two clients see empty → both insert
}
// GOOD: lock ONCE; decide and act under the same guard.
let mut g = shared.lock().unwrap();
if g.is_empty() {
    g.insert(key, val); // atomic check+act inside the same critical section
}
```

```rust
// 4) Share by communicating — channel instead of locked state
// BAD: N workers contending over Arc<Mutex<Vec<T>>> to push results.
let out = Arc::new(Mutex::new(Vec::new()));
// each worker: out.lock().unwrap().push(item);  // contention + undefined order
// GOOD: each worker sends over the channel; ONE owner consumes. No shared lock.
let (tx, rx) = mpsc::channel();
// worker: tx.clone().send(item).unwrap();
let results: Vec<_> = rx.iter().collect(); // ownership transferred via the message
```

```rust
// 5) Async (tokio) — holding a lock across .await
// BAD: with tokio::sync::Mutex this COMPILES (guard is Send), but holding the guard
//      across the .await serializes all tasks on that lock — that is the trap.
//      (With std::sync::Mutex you can't even: .lock() returns Result, not a future to
//      .await; and its guard, non-Send, refuses to cross await on a multi-thread runtime.)
let mut g = state.lock().await;
remote_call().await;           // guard alive during the await: blocks other tasks
g.update();
// GOOD: close the critical section before the await, or do the I/O outside the lock.
{ let mut g = state.lock().await; g.prepare(); } // guard dropped here
let resp = remote_call().await;                  // I/O without lock
state.lock().await.apply(resp);                  // re-lock only for the write
```

## Idiomatic guards

| Primitive | When to use |
|---|---|
| `AtomicUsize`/`AtomicBool` + `fetch_add`/`compare_exchange` | counter, flag, simple lock-free; a single value with no associated data |
| `Arc<Mutex<T>>` | mutable state shared across threads; serialized writes |
| `Arc<RwLock<T>>` | many reads, few writes; readers run concurrently with each other |
| `mpsc::channel` / `crossbeam::channel` | share-by-communicating; transfers ownership, avoids shared lock |
| `tokio::sync::Mutex` / `RwLock` | async: guard is `Send` and can (but avoid) cross `.await` |
| `tokio::sync::mpsc` / `oneshot` | pass values between async tasks |
| `OnceLock` / `OnceCell` / `LazyLock` | single, thread-safe init (without the manual double-check) |
| `parking_lot::Mutex` | faster mutex, no `PoisonError`; guard is not reentrant |
| `compare_exchange` / `compare_exchange_weak` | CAS for lock-free lost update (retry in a loop) |
| `Ordering::Relaxed` / `Acquire`+`Release` / `SeqCst` | pure counter / publish-read data via flag / when in doubt |

`Ordering` rule: `Relaxed` only when the atomic does not synchronize **any other
data** (isolated counter). To publish data ("I wrote the buffer, now I set the
flag"), use `Release` on the store and `Acquire` on the load. When in doubt, `SeqCst`.

## Prove the guard

Launch N simultaneous threads with `Barrier` over the **same** resource and assert
the invariant. Without a barrier the threads may not even overlap.

```rust
use std::sync::{Arc, Barrier};
use std::sync::atomic::{AtomicUsize, Ordering};
use std::thread;

#[test]
fn counter_has_no_lost_updates() {
    const THREADS: usize = 50;
    const PER_THREAD: usize = 1_000;

    let counter = Arc::new(AtomicUsize::new(0));
    let barrier = Arc::new(Barrier::new(THREADS)); // simultaneous start

    let handles: Vec<_> = (0..THREADS)
        .map(|_| {
            let counter = Arc::clone(&counter);
            let barrier = Arc::clone(&barrier);
            thread::spawn(move || {
                barrier.wait(); // all threads start together → maximizes the window
                for _ in 0..PER_THREAD {
                    counter.fetch_add(1, Ordering::Relaxed);
                }
            })
        })
        .collect();

    for h in handles {
        h.join().unwrap();
    }

    // invariant: no add was lost
    assert_eq!(counter.load(Ordering::Relaxed), THREADS * PER_THREAD);
}
```

Swap `fetch_add` for a separate `load`/`store` to watch the test fail — that is
how you confirm it actually detects the race. Run with
`cargo test --release` and several times; debug mode sometimes does not open the window.

## Lint & static detection

The compiler is already the strongest detector: a memory data race in safe code
**does not compile** (`Send`/`Sync` + borrow checker). The rest:

```bash
cargo clippy --all-targets -- -D warnings
# catches: await_holding_lock (Mutex guard alive across .await),
#       mutex_atomic (Mutex<bool>/<usize> where an Atomic is enough), and the like.

cargo +nightly miri test
# Miri: detects UB in `unsafe` code (pointers, data race on raw pointers,
#       out-of-borrow accesses). Does NOT run code that depends on FFI/syscalls.
```

For **lock-free code** (structures with hand-rolled atomics and `Ordering`), `loom`
exhaustively tests the interleavings allowed by the memory model — it catches
ordering bugs that `cargo test` never reproduces on your machine:

```rust
// Cargo.toml: loom = "0.7" under [target.'cfg(loom)'.dependencies]
// imports swap std::sync::* for loom::sync::* under cfg(loom)
#[test]
fn lock_free_is_sound() {
    loom::model(|| {
        // exercises 2 threads; loom enumerates ALL Ordering interleavings
    });
}
// run: RUSTFLAGS="--cfg loom" cargo test --release
```

What they do **NOT** catch: **logical** race (two correct locks in a wrong
sequence) and **database** race. Clippy does not model the sequence between critical
sections; Miri/loom only look at memory, not SQL. That comes out in a concurrency
test + review.

## Rust-specific anti-patterns

- **`Ordering::Relaxed` on everything** because "it compiled". Relaxed does not
  establish happens-before; publishing data via a Relaxed flag lets the reader see
  garbage. Use `Release`/`Acquire`.
- **Holding a `MutexGuard` across `.await`** with `tokio::sync::Mutex`:
  serializes all tasks on that lock. Close the critical section before the await.
- **`std::sync::Mutex` in async code** — blocks the entire runtime thread;
  if locked twice in the same task, deadlock. Use `tokio::sync::Mutex` when
  the guard crosses await, `std::sync::Mutex` only for very short sections without await.
- **Locking the same `Mutex` twice in the same thread** — `std::sync::Mutex` is not
  reentrant: immediate deadlock, not panic.
- **Two `Mutex` in inconsistent order** across threads → classic deadlock.
  Impose a total acquisition order (e.g.: always by increasing address/id).
- **`Mutex<bool>`/`Mutex<usize>` for a single value** — clippy warns
  (`mutex_atomic`); an `AtomicBool`/`AtomicUsize` is cheaper and has no poison.
- **`.lock().unwrap()` ignoring `PoisonError`** — a panic with the lock held
  poisons the mutex and propagates the panic to everyone. Handle it, or use `parking_lot`.
- **Manual double-checked locking** for single init — use `OnceLock`/`LazyLock`;
  the hand-rolled version gets the `Ordering` wrong almost every time.
- **Reimplementing a distributed lock in memory** in a service with N replicas — the
  `Arc<Mutex>` only serializes **one** process. The multi-replica guard is in the database:
  `UPDATE ... WHERE`, `UNIQUE` constraint, `SELECT ... FOR UPDATE`. With `sqlx`,
  the check-then-act becomes a single atomic `query!` — see `references/databases-sql.md`.
