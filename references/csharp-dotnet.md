# Race Conditions in C# / .NET

## Concurrency model

.NET has **real threads** with shared memory (`ThreadPool`, TPL) plus
`async`/`await` multiplexing tasks over them. Races arise in two places:
**memory data race** (two threads touch a field without synchronization — torn
read of `long`/`decimal`, non-atomic `++`, non-thread-safe list) and **logical
race** (an `await` or `SELECT`→`UPDATE` mid check-then-act opens a window for
another task/request). The JIT and CPU reorder accesses without a barrier, so
"looks sequential" guarantees nothing.

## The races you will encounter

- **Memory data race**: `count++` from multiple threads, `Dictionary<,>` mutated
  concurrently (corrupts internally, not just loses a write), `long`/
  `double` field read without `Interlocked`/`Volatile` (tearing on 32-bit).
- **Check-then-act in memory**: `if (!cache.ContainsKey(k)) cache[k] = ...` —
  two threads pass the `if` and both write/initialize.
- **Logical race over I/O**: handler reads balance (`await`), decides, writes (`await`)
  — two requests enter the window between the `await`s.
- **Deadlock from synchronous blocking in async**: `.Result`/`.Wait()`/`.GetAwaiter().GetResult()`
  in a context with a `SynchronizationContext` (UI, classic ASP.NET) locks up permanently.
- **Lost update in the database**: EF Core reads entity, changes it, `SaveChanges` — without a
  concurrency token, last write wins silently.

## How to avoid

### 1. Shared counter: `++` → `Interlocked`

```csharp
// BAD: ++ is read-modify-write in 3 steps; two threads read the same value and
// one increment is lost (memory lost update).
private long _count;
public void Hit() => _count++;            // NOT atomic

// GOOD: Interlocked does the increment atomically at the CPU level, without a lock.
private long _count;
public void Hit() => Interlocked.Increment(ref _count);
public long Read() => Interlocked.Read(ref _count);  // read a long without tearing
```

### 2. Critical section with logic: `lock` (Monitor)

```csharp
// BAD: check-then-act over a shared collection — two threads see !Contains
// and both add; List<T> is not thread-safe, not even on writes.
private readonly List<int> _ids = new();
public void Add(int id) { if (!_ids.Contains(id)) _ids.Add(id); }

// GOOD: lock serializes the entire check+act. Use Interlocked for a simple counter;
// lock only when the operation is composite (multiple fields / decide-and-act).
private readonly object _gate = new();           // dedicated object, never 'this' nor a string
private readonly List<int> _ids = new();
public void Add(int id) { lock (_gate) { if (!_ids.Contains(id)) _ids.Add(id); } }
```

### 3. Get-or-create in memory: `ConcurrentDictionary`

```csharp
// BAD: ContainsKey-then-add has a window; and a Dictionary<,> mutated by 2 threads
// can CORRUPT the structure (infinite loop/IndexOutOfRange), not just lose data.
private readonly Dictionary<string, Conn> _pool = new();
Conn Get(string k) { if (!_pool.ContainsKey(k)) _pool[k] = Open(k); return _pool[k]; }

// GOOD: GetOrAdd is atomic. The factory may run twice under a race (and the loser is
// discarded), so use Lazy<T> if Open() is expensive/has side effects.
private readonly ConcurrentDictionary<string, Lazy<Conn>> _pool = new();
Conn Get(string k) => _pool.GetOrAdd(k, key => new Lazy<Conn>(() => Open(key))).Value;
// for a counter: AddOrUpdate applies add/update atomically
_counts.AddOrUpdate(k, 1, (_, n) => n + 1);
```

### 4. Never block async with `.Result`/`.Wait()` → `await`

```csharp
// BAD: .Result captures the SynchronizationContext and waits; the continuation needs
// to return to the SAME thread, which is blocked waiting → classic deadlock.
public ActionResult Get() { var data = LoadAsync().Result; return View(data); }

// GOOD: async all the way up. ConfigureAwait(false) in libs releases the context and
// avoids the deadlock even if a legacy caller blocks above.
public async Task<ActionResult> Get() { var data = await LoadAsync(); return View(data); }
// in a library, with no context dependency:
var data = await LoadAsync().ConfigureAwait(false);
```

### 5. Limit concurrency in async: `SemaphoreSlim` (not `lock`)

```csharp
// BAD: lock cannot wrap an await — it is compile error CS1996 ("Cannot await
// in the body of a lock statement"). Monitor is per-thread (Enter/Exit on the SAME
// thread), but the continuation may resume on ANOTHER → the compiler blocks it outright.
lock (_gate) { await CallApiAsync(); }   // WRONG: does not compile (CS1996)

// GOOD: SemaphoreSlim is the asynchronous "lock". WaitAsync does not block a thread;
// try/finally guarantees the Release even on exception. Initial=1 → mutual exclusion.
private readonly SemaphoreSlim _sem = new(1, 1);
public async Task DoAsync()
{
    await _sem.WaitAsync();
    try { await CallApiAsync(); }   // only one task at a time in this section
    finally { _sem.Release(); }
}
```

## Idiomatic guards

| Primitive | When to use |
|---|---|
| `Interlocked.Increment/Add/Exchange/CompareExchange` | atomic counter/flag/swap of `int`/`long`, lock-free |
| `Volatile.Read/Write`, `volatile` | publish a flag between threads without reordering (no composite atomicity) |
| `lock` (Monitor) | **synchronous** critical section with composite logic; minimal scope, dedicated object |
| `SemaphoreSlim` (+ `WaitAsync`) | exclusion/concurrency limit in **async** code |
| `ConcurrentDictionary` (`GetOrAdd`/`AddOrUpdate`) | shared map get-or-create / per-key counter |
| `ConcurrentQueue/Bag`, `Channel<T>` | producer-consumer without a manual lock |
| `Lazy<T>` (default thread-safe mode) | single, expensive initialization under a race |
| `ReaderWriterLockSlim` | many reads, few writes (expensive; only if you measure a gain) |
| `[Timestamp]` rowversion / `[ConcurrencyCheck]` (EF Core) | DB lost update → `DbUpdateConcurrencyException` + retry |
| `SELECT ... FOR UPDATE` via `FromSqlRaw` / transaction | pessimistic lock when logic does not fit in an UPDATE — see `references/databases-sql.md` |

### EF Core: optimistic lock with retry

```csharp
// [Timestamp] public byte[] RowVersion { get; set; }  ← on the entity
// EF includes  ... WHERE Id=@id AND RowVersion=@orig  in the UPDATE.
for (var attempt = 0; ; attempt++)
{
    var acc = await db.Accounts.FindAsync(id);
    acc.Balance -= amount;
    try { await db.SaveChangesAsync(); break; }      // 0 rows affected → conflict
    catch (DbUpdateConcurrencyException) when (attempt < 3)
    {
        await db.Entry(acc).ReloadAsync();            // reloads values from the database and tries again
    }
}
```

Isolation level, `FOR UPDATE`, constraints and write skew are cross-cutting:
**see `references/databases-sql.md`**. The database guard holds across all
replicas; `lock`/`Interlocked` only serialize **within a single process**.

## Prove the guard

xUnit test firing N simultaneous tasks with a `Barrier` (synchronized start),
asserting the invariant. The BAD version (`_count++`) fails; the GOOD one always passes.

```csharp
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Xunit;

public class CounterRaceTests
{
    [Fact]
    public async Task Increment_under_concurrency_loses_no_update()
    {
        const int threads = 50, perThread = 1_000;
        var counter = new Counter();                 // uses Interlocked.Increment
        var barrier = new Barrier(threads);          // all start together → maximizes contention

        var tasks = Enumerable.Range(0, threads).Select(_ => Task.Run(() =>
        {
            barrier.SignalAndWait();                 // waits for everyone at the gate
            for (var i = 0; i < perThread; i++) counter.Hit();
        }));

        await Task.WhenAll(tasks);                    // joins all the tasks
        Assert.Equal(threads * perThread, counter.Read());  // exact invariant
    }
}
```

Run with `dotnet test`. Under `_count++` the assert breaks non-deterministically
— rerun a few times or raise `threads` if it does not reproduce on the first pass.

## Lint & static detection

```xml
<!-- .csproj — Microsoft threading analyzers -->
<PackageReference Include="Microsoft.VisualStudio.Threading.Analyzers" Version="*">
  <PrivateAssets>all</PrivateAssets>
</PackageReference>
<PropertyGroup>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>   <!-- gate in CI -->
</PropertyGroup>
```

Catches (rules `VSTHRD*`): `VSTHRD002` use of `.Result`/`.Wait()`/`.GetAwaiter().GetResult()`
(sync-over-async deadlock), `VSTHRD103` synchronous call when an async version exists,
`VSTHRD111`/`VSTHRD200` missing `ConfigureAwait`/`Async` convention, `VSTHRD110`
unawaited `Task` (silent fire-and-forget). Under `dotnet build` (Roslyn) the
analyzers run in CI; `TreatWarningsAsErrors` turns the warning into a failure.

**Does not catch**: logical data race between two `await`s (check-then-act over I/O),
DB lost update, or a missing `Interlocked` on a `++`. .NET **has no
ThreadSanitizer** — the real detector for these is the **concurrency test +
database guard**. Full catalog in `references/lint-detectors.md`.

## C# / .NET-specific anti-patterns

- `lock (this)` or `lock ("string literal")` — the object is visible/interned by the
  runtime; external code can lock the same monitor → deadlock. Use a private `object`.
- `lock` around an `await` — compile error (CS1996); use `SemaphoreSlim`.
- `.Result`/`.Wait()` "just this once" — in classic ASP.NET/UI it is a guaranteed deadlock
  under load; `async` all the way up is the only cure.
- `async void` (outside an event handler) — the exception is not catchable, the task not awaitable;
  use `async Task`.
- Static mutable `Dictionary<,>`/`List<>` shared between requests without a lock —
  corrupts the structure, not just loses data. Use `ConcurrentDictionary` or an immutable one.
- `ConfigureAwait(false)` in a controller method that touches `HttpContext` afterward —
  loses the needed context; use it only in libs with no context affinity.
- EF Core `DbContext` shared between threads/requests — it is not thread-safe;
  it is scoped per request. One `await` at a time per `DbContext`.
- `Interlocked` on a field and a raw read elsewhere — the read also needs
  `Interlocked.Read`/`Volatile.Read`; atomicity applies to all accesses.
