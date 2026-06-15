# Race Conditions in Java / Kotlin (JVM)

## Concurrency model

The JVM runs **real OS threads** with shared memory — no GIL nor event
loop that serializes the CPU. The danger lives in the **Java Memory Model (JMM)**: without a
*happens-before* (via `volatile`, `synchronized`, `java.util.concurrent`, `final`),
a write from one thread may **never become visible** to another, or be
reordered by the compiler/CPU. Race here combines **memory data race**
(visibility/atomicity within the process) and **logical race** (check-then-act on the database).
Kotlin compiles to the same JVM — coroutines do **not** swap out the JMM by magic.

## The races you will encounter

- **Memory data race** — mutable field without `volatile`/lock: stale value
  (stale), lost `i++`, corrupted `HashMap` during `resize` (classic infinite loop).
- **Check-then-act** — `containsKey`-then-`put`, `if (instance == null) new ...`:
  the window between test and action lets two threads act together.
- **Lazy-init / DCL** without `volatile`: another thread sees the reference published before
  the constructor finishes (a "half-constructed" object).
- **Compound action on a thread-safe collection** — `ConcurrentHashMap` is safe per
  operation, but `get`-then-`put` is still check-then-act.
- **Logical race on the database** — lost update / double-submit; the JMM does not help, the guard is
  transaction/constraint (see `references/databases-sql.md`).
- **Kotlin coroutines** — `state++` in two `launch` over the same var on the default
  dispatcher (multi-thread) loses updates; and even when confined, read-decide-write that
  crosses a `suspend` lets another coroutine enter the window.

## How to avoid

### 1. Visibility — a flag shared between threads needs `volatile`

```java
// BAD: the run() thread may NEVER see running=false — without happens-before the JIT
// can hoist the read out of the loop and treat it as a constant (infinite loop).
class Worker {
    private boolean running = true;
    void run()  { while (running) { /* ... */ } }
    void stop() { running = false; }           // write may stay invisible
}

// GOOD: volatile creates happens-before between the write and each read.
class Worker {
    private volatile boolean running = true;   // every read sees the latest write
    void run()  { while (running) { /* ... */ } }
    void stop() { running = false; }
}
```

### 2. Check-then-act on a map — `compute`, not `get`-`put`

```java
// BAD: containsKey + put is not atomic; two threads pass the if and the 2nd
// overwrites (or initializes an expensive object twice).
ConcurrentHashMap<String, List<Order>> byUser = new ConcurrentHashMap<>();
if (!byUser.containsKey(id)) byUser.put(id, new ArrayList<>());  // window here
byUser.get(id).add(order);   // and ArrayList is not thread-safe for concurrent add

// GOOD: computeIfAbsent is atomic per key; the lambda runs at most once.
ConcurrentHashMap<String, Queue<Order>> byUser = new ConcurrentHashMap<>();
byUser.computeIfAbsent(id, k -> new ConcurrentLinkedQueue<>()).add(order);
// For read-modify-write of the VALUE use compute/merge: they hold the key's bin in the lambda.
```

### 3. Counter — `AtomicLong`/`LongAdder`, never `long++`

```java
// BAD: count++ is read-increment-write (3 steps); increments get lost.
private long count = 0;
void hit() { count++; }            // even with volatile it stays non-atomic

// GOOD (high contention): LongAdder spreads the counter across cells and sums in sum().
private final LongAdder count = new LongAdder();
void hit() { count.increment(); }
long total() { return count.sum(); }
// AtomicLong.incrementAndGet() works when you need the value at each step.
// volatile long does NOT help: it gives visibility, but ++ stays read-add-write.
```

### 4. Lazy singleton — Holder idiom (no lock, no manual `volatile`)

```java
// BAD: double-checked locking WITHOUT volatile on the field — the reference can be
// published before the constructor finishes; another thread sees a half-built object.
class Config {
    private static Config instance;                 // missing volatile
    static Config get() {
        if (instance == null)
            synchronized (Config.class) {
                if (instance == null) instance = new Config();  // unsafe publication
            }
        return instance;
    }
}

// GOOD: the JVM guarantees the Holder class is initialized exactly once, thread-safe, lazily.
class Config {
    private Config() { /* expensive */ }
    private static class Holder { static final Config INSTANCE = new Config(); }
    static Config get() { return Holder.INSTANCE; }  // no lock on the hot path
}
// If you NEED DCL (instance field, not static), the field MUST be volatile.
```

### 5. Kotlin coroutines — shared state needs a `Mutex` or confinement

```kotlin
// BAD: coroutines do not protect state. On the default dispatcher (multi-thread) the
// 1000 launch run in parallel and the ++ (read-add-write) loses updates.
var counter = 0
coroutineScope {
    repeat(1000) { launch { counter++ } }   // data race: ++ is not atomic
}

// GOOD: Mutex from kotlinx.coroutines (suspends instead of blocking the thread).
val mutex = Mutex()
var counter = 0
coroutineScope {
    repeat(1000) { launch { mutex.withLock { counter++ } } }
}
// Alternatives: confine to a single dispatcher (limitedParallelism(1)) or
// AtomicInteger. NEVER synchronized/ReentrantLock inside suspend (blocks the
// dispatcher thread → starvation, and deadlock if the pool is small);
// Mutex.withLock suspends, it does not block.
```

### 6. JPA — optimistic locking with `@Version`

```kotlin
// GOOD: @Version makes JPA emit UPDATE ... WHERE id=? AND version=?.
// If another tx committed first, 0 rows → OptimisticLockException → retry.
@Entity
class Account(
    @Id val id: Long,
    var balance: BigDecimal,
    @Version var version: Long = 0,   // managed by the provider; do not set by hand
)
// Check-then-act that does not fit optimistic → pessimistic lock:
//   em.find(Account::class.java, id, LockModeType.PESSIMISTIC_WRITE)  // = SELECT FOR UPDATE
```

Cross-cutting detail of the database layer in `references/databases-sql.md`.

## Idiomatic guards

| Primitive | When to use |
|---|---|
| `volatile` | flag/single-word state published between threads; **visibility only** |
| `synchronized` / `ReentrantLock` | composite critical section (check-then-act in memory) |
| `AtomicInteger`/`AtomicLong`/`AtomicReference` | counter/CAS with the value at each step |
| `LongAdder`/`LongAccumulator` | high-contention counter (write-heavy, rarely read) |
| `ConcurrentHashMap.compute*`/`merge` | atomic read-modify-write per key |
| `CopyOnWriteArrayList` | list read a lot, rarely written (listeners) |
| Holder idiom / `Lazy` (Kotlin) | thread-safe lazy initialization without a lock |
| `kotlinx.coroutines.sync.Mutex` | exclusion between coroutines (suspends, does not block) |
| `@Version` (JPA) / `LockModeType.PESSIMISTIC_WRITE` | lost update on the database (optimistic/pessimistic) |
| immutable `record` / `data class` + confinement | prefer not sharing mutable state over locking it |

`ReentrantLock` over `synchronized` when you need `tryLock(timeout)`, fairness
or an interruptible lock — always `lock()` in `try` / `unlock()` in `finally`.

## Prove the guard

Simultaneous start: an `ExecutorService` fires N threads, a `CountDownLatch` holds them
to start together (maximizes the window), and the test asserts the invariant at the end.

```java
@Test
void counter_under_concurrency_loses_no_increment() throws Exception {
    final int threads = 50, perThread = 1000;
    final AtomicLong safe = new AtomicLong();        // the guard under test

    ExecutorService pool = Executors.newFixedThreadPool(threads);
    CountDownLatch ready = new CountDownLatch(threads);
    CountDownLatch start = new CountDownLatch(1);     // start barrier
    CountDownLatch done  = new CountDownLatch(threads);

    for (int t = 0; t < threads; t++) {
        pool.submit(() -> {
            ready.countDown();
            try { start.await(); } catch (InterruptedException ignored) {}
            for (int i = 0; i < perThread; i++) safe.incrementAndGet();
            done.countDown();
        });
    }
    ready.await();                 // wait for all to be ready
    start.countDown();             // release all at the same time (maximizes the window)
    done.await();
    pool.shutdown();

    assertEquals((long) threads * perThread, safe.get());  // invariant: nothing lost
}
```

Run under stress (repeat, increase N) — a single pass may not open the window. To
prove the JMM at the limit (reordering, visibility) use **jcstress** (OpenJDK), which
generates interleavings and classifies the results as `ACCEPTABLE`/`FORBIDDEN`.

## Lint & static detection

Full catalog in `references/lint-detectors.md`. Java/Kotlin summary:

```bash
# SpotBugs — multithreading category (non-synchronized fields, etc.)
spotbugs -textui -effort:max -bugCategories MT_CORRECTNESS target/classes
# Infer / RacerD — inter-procedural race analysis (does not run the binary)
infer run -- ./gradlew build
# Error Prone (plugin net.ltgt.errorprone): @GuardedBy("lock") on fields →
# access outside the declared lock becomes a compile error.
```

- **SpotBugs `MT_CORRECTNESS`** catches inconsistent `synchronized`, non-synchronized
  static-field lazy-init, `wait()` outside a loop. Does not catch database races.
- **Error Prone `@GuardedBy`** proves at compile time that every access to the field uses the
  declared lock — cheap and precise, but only within the method/class.
- **Infer/RacerD** is the strongest for inter-procedural races; moderate noise.
- **None** catch lost update on the database nor cover coroutines well — that is concurrency
  testing + a guard at the data layer + Mutex/confinement by construction.

## Java / Kotlin-specific anti-patterns

- **`volatile` ≠ atomic**: using `volatile long count` and doing `count++` expecting
  atomicity. `volatile` only gives visibility; `++` keeps losing updates.
- **DCL without `volatile`** on the instance field: unsafe publication; use the Holder
  idiom (static) or `volatile` (instance) — never DCL without a barrier.
- **`Collections.synchronizedMap` + iteration** without externally synchronizing the loop,
  or thinking `ConcurrentHashMap` makes `get`+`put` atomic (it does not).
- **`GlobalScope.launch`** without structured concurrency: the coroutine leaks, writes
  after the scope dies and exceptions vanish. Use `coroutineScope`/`supervisorScope`.
- **`@Transactional` on a `private`/self-invoked method** (Spring): the proxy does not
  intercept, the transaction does not open, and the "pessimistic lock" runs outside a transaction.
- **Swallowed `OptimisticLockException`**: it must become a retry or an explicit error,
  never an empty `catch` (silent corruption, see `databases-sql.md`).
- **`@Async`/`@Scheduled` mutating a singleton bean field** without a guard: the bean is
  shared across all requests/threads.
