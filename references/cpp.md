# Race Conditions in C / C++

## Concurrency model

C and C++ have **real threads** and genuine shared memory — there is no GIL nor
event loop serializing you. Worse: a **data race** (two threads access the
same object without synchronization, at least one writing) is **undefined
behavior (UB)** by the standard (`[intro.races]` in C++, §5.1.2.4 in C11). UB is
not "wrong value": the compiler may reorder, fuse reads, or assume the
race never happens and optimize your code into something you never intended. The
danger lives in any `T x` read/written by more than one thread without `std::atomic`, mutex, or
another *happens-before* relation.

## The races you will encounter

- **Memory data race (UB)** — `count++` counter (non-atomic read-modify-write),
  `bool ready` flag, pointer published without a barrier, write to a
  `std::vector`/`std::map` from multiple threads. It is not "sometimes wrong"; it is UB.
- **Logical check-then-act** — `if (!ptr) ptr = create();` (lazy-init), `if
  (map.count(k)) ...` followed by insert, double-checked locking done wrong.
- **Torn read/write** — `int64_t`/structs larger than the CPU word read
  halfway. Even a `bool` can be cached in a register and never reread memory.
- **Deadlock** — two mutexes locked in opposite orders by two threads.
- **Database/external-resource race** — read-modify-write crossing the DB; the
  guard is the same as in any language, see `references/databases-sql.md`.

## How to avoid

```cpp
// === Shared counter ===
// BAD: ++ is load+add+store; two threads interleave and lose increments. UB.
int count = 0;
void worker() { for (int i = 0; i < N; ++i) ++count; }   // data race

// GOOD: std::atomic makes the RMW indivisible. fetch_add is an atomic instruction.
std::atomic<int> count{0};
void worker() { for (int i = 0; i < N; ++i) count.fetch_add(1, std::memory_order_relaxed); }
// relaxed is enough for a pure counter: we only want atomicity, not to order
// other accesses relative to it.
```

```cpp
// === Critical section (state that doesn't fit in an atomic) ===
// BAD: two threads mutate the same container concurrently → UB / corruption.
std::map<std::string, int> tally;
void add(const std::string& k) { tally[k]++; }   // no lock

// GOOD: lock_guard acquires in the constructor and releases in the destructor (RAII).
std::mutex m;
std::map<std::string, int> tally;
void add(const std::string& k) {
    std::lock_guard<std::mutex> lk(m);   // exception-safe: releases the lock even if add throws
    tally[k]++;
}
```

```cpp
// === Lazy-init / flag publication ===
// BAD: thread A writes data and ready=true; B sees ready=true but stale data.
// Without a barrier, the compiler/CPU can reorder the two writes. UB when reading data.
int data = 0; bool ready = false;
void producer() { data = 42; ready = true; }
void consumer() { while (!ready) {} use(data); }   // may read data == 0

// GOOD: atomic<bool> with release/acquire creates happens-before.
std::atomic<bool> ready{false}; int data = 0;
void producer() { data = 42; ready.store(true, std::memory_order_release); }
void consumer() {
    while (!ready.load(std::memory_order_acquire)) {}  // acquire "sees" everything before the release
    use(data);                                         // data == 42 guaranteed
}
```

```cpp
// === Single initialization (singleton / expensive resource) ===
// BAD: naive double-checked locking — the second if reads ptr without synchronization,
// and the write to ptr may be seen before the object is constructed. Classic UB.
Widget* ptr = nullptr; std::mutex m;
Widget* get() {
    if (!ptr) { std::lock_guard<std::mutex> lk(m); if (!ptr) ptr = new Widget(); }
    return ptr;
}

// GOOD: std::call_once guarantees exactly one execution, with the correct synchronization.
std::once_flag flag; std::unique_ptr<Widget> ptr;
Widget* get() {
    std::call_once(flag, []{ ptr = std::make_unique<Widget>(); });
    return ptr.get();
}
// In modern C++, a local `static Widget w;` is already thread-safe by default ([stmt.dcl]).
```

```cpp
// === Multiple locks without deadlock ===
// BAD: transfer(a,b) locks a→b and transfer(b,a) locks b→a → cross deadlock.
void transfer(Account& a, Account& b, int v) {
    std::lock_guard<std::mutex> la(a.m);
    std::lock_guard<std::mutex> lb(b.m);   // order depends on the arguments
    a.bal -= v; b.bal += v;
}

// GOOD: std::scoped_lock locks all mutexes at once with a deadlock-free
// algorithm (lock + back-off). Argument order no longer matters.
void transfer(Account& a, Account& b, int v) {
    std::scoped_lock lk(a.m, b.m);   // C++17: acquires both without risk of deadlock
    a.bal -= v; b.bal += v;
}
```

## Idiomatic guards

| Primitive | When to use |
|---|---|
| `std::atomic<T>` + `memory_order` | shared counter/flag/pointer without a critical section |
| `std::mutex` + `std::lock_guard` | simple critical section, a single lock, RAII scope |
| `std::scoped_lock` (C++17) | acquire **two or more** mutexes without deadlock |
| `std::unique_lock` | need an unlockable lock / with `condition_variable` |
| `std::shared_mutex` + `shared_lock` | many readers, few writers |
| `std::call_once` + `std::once_flag` | single init (singleton, expensive resource) |
| local `static` (C++11+) | simple singleton — thread-safe init by the language |
| `std::condition_variable` | wait for a condition without busy-wait |
| `std::barrier`/`std::latch` (C++20) | coordinate start/join of N threads |
| C11: `_Atomic`, `mtx_t`/`mtx_lock` (`threads.h`), `pthread_mutex_t` | equivalents in pure C |
| `std::counting_semaphore` (C++20) | limit concurrency to N |

In pure C: `_Atomic int x;` with `<stdatomic.h>` (`atomic_fetch_add`,
`atomic_load_explicit`), `pthread_mutex_lock`/`unlock` or the `mtx_*` from
`<threads.h>` (C11), and `call_once`/`once_flag` also exist in `<threads.h>`.

## Prove the guard

A test that launches N simultaneous threads over the **same** counter and asserts the total.
Use `std::barrier` (C++20) for a simultaneous start — without it, threads finish
before the next is born and the window never opens. Run **under TSan** (`-fsanitize=thread`):
the raw `int` version fails TSan and/or gets the total wrong; the `atomic` one passes.

```cpp
// test_counter.cpp  — compile: g++ -std=c++20 -fsanitize=thread -g -O1 test_counter.cpp
#include <atomic>
#include <barrier>
#include <cassert>
#include <thread>
#include <vector>

int main() {
    constexpr int THREADS = 16;
    constexpr int PER_THREAD = 100'000;
    std::atomic<int> counter{0};

    std::barrier sync(THREADS);                    // all wait here and start together
    std::vector<std::thread> pool;
    for (int t = 0; t < THREADS; ++t) {
        pool.emplace_back([&] {
            sync.arrive_and_wait();                // simultaneous start → maximizes the race
            for (int i = 0; i < PER_THREAD; ++i)
                counter.fetch_add(1, std::memory_order_relaxed);
        });
    }
    for (auto& th : pool) th.join();

    assert(counter.load() == THREADS * PER_THREAD);   // invariant: no lost increment
    return 0;
}
```

Swap `std::atomic<int>` for raw `int` and `fetch_add` for `++` to watch the test
**fail** (and TSan flag the data race) — proving that the test actually detects
the race, rather than passing by luck. Run it several times; one pass may not open the window.

## Lint & static detection

| Tool | Type | Command | Catches / doesn't catch |
|---|---|---|---|
| **ThreadSanitizer** | dynamic | `g++ -fsanitize=thread -g -O1` (or `clang++`) | **catches** real data races at runtime, some deadlocks and invalid mutex use; does **not** catch unexecuted paths nor database races |
| **Helgrind / DRD** | dynamic | `valgrind --tool=helgrind ./app` | races + inconsistent lock order; slower, more false positives than TSan |
| **Clang Static Analyzer** | static | `clang --analyze` / `scan-build make` | partial lock/leak bugs; race analysis is limited |
| **Infer / RacerD** | static | `infer run -- make` | inter-procedural races without running the binary; good on C++/ObjC with annotations |
| **clang-tidy** | static | `clang-tidy --checks='bugprone-*,concurrency-*'` | `concurrency-mt-unsafe`, use of non-thread-safe APIs; does **not** prove absence of races |

```bash
# TSan is the reference dynamic detector (same engine as Go's -race)
g++ -std=c++20 -fsanitize=thread -g -O1 app.cpp -o app && ./app
valgrind --tool=helgrind ./app          # alternative without recompiling with a sanitizer
infer run -- make                        # static, inter-procedural
```

Golden rules: TSan **only sees what executes** — the test must exercise the
concurrent path, ideally under load and several times. Compile with `-O1 -g`
(readable symbols, no aggressive optimization that masks it). Never run TSan and
AddressSanitizer in the same binary (incompatible). None of these tools
catch a **logical** race at the data layer — for that, see `references/databases-sql.md`.

## C / C++-specific anti-patterns

- **`volatile` as synchronization** — `volatile` creates neither atomicity nor
  ordering between threads; it is for MMIO, not for concurrency. Use `std::atomic`.
- **Double-checked locking with a raw pointer** — the canonical pre-C++11 UB. Use
  `std::call_once` or a local `static`.
- **Copying a `std::mutex`/`std::atomic`** — they are not copyable/movable; structs
  that contain them need care. `go vet` has `copylocks`; in C++ the compiler
  blocks it, but dangling references to a destroyed lock slip through.
- **Lock with too broad a scope** — holding the mutex during I/O or a call that
  may throw/block kills throughput and invites deadlock. Minimal scope.
- **Forgetting `notify`/using `wait` without a predicate** — `cv.wait(lk)` without a predicate
  suffers *spurious wakeup* and *lost wakeup*. Always `cv.wait(lk, []{ return ready; })`.
- **`std::lock_guard` in manual order for 2 locks** — easy to invert and deadlock.
  Use `std::scoped_lock`, which is deadlock-free by construction.
- **Thinking `memory_order_relaxed` orders other accesses** — relaxed only guarantees
  atomicity of the object itself. To publish data, use release/acquire.
- **Detached thread touching state that goes out of scope** — `t.detach()` and the capture
  by reference dies before the thread. Prefer `join()` or `std::jthread` (C++20).
