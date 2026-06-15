# Race Conditions in Python

## Concurrency model

The **GIL serializes bytecode**, so two opcodes never corrupt a Python object —
but it **releases** on I/O calls (socket, disk, DB driver) and at every `await`.
Every Python race is **logical**: a read-decide-write crosses a blocking call or an
`await`, another thread/task enters the window, the invariant breaks.
`multiprocessing` has no GIL across processes — that is where shared
memory/resource is real.

## The races you will encounter

- **`threading`**: non-atomic read-modify-write (`counter += 1`,
  `if key not in cache: cache[key] = compute()`). `+=` is *load → add → store* in
  separate opcodes; the scheduler switches threads mid-sequence (every
  `sys.setswitchinterval`, ~5ms by default since CPython 3.2, or on I/O).
- **`asyncio`**: the most treacherous window. Between `x = await read()` and
  `await write(x)` the event loop runs another task — classic check-then-act. No
  await in the middle, no interruption.
- **`multiprocessing`**: no shared GIL — an in-memory counter is a real race;
  needs `Value`/`Array(lock=True)` or `Manager`.
- **Logical database/cache/file race**: universal, GIL-independent. The most common
  one in Django/FastAPI backends. Cured at the data layer — see
  `references/databases-sql.md`.

## How to avoid

```python
# BAD — threading: non-atomic read-modify-write. "+=" is load/add/store in
# separate opcodes; the scheduler switches threads between them and loses increments.
def worker():
    global counter
    for _ in range(100_000):
        counter += 1                 # LOSES updates under N threads

# GOOD — serialize the read-modify-write with a minimally-scoped Lock.
lock = threading.Lock()
def worker():
    global counter
    for _ in range(100_000):
        with lock:                   # only the += needs to be inside the lock
            counter += 1
```

```python
# BAD — asyncio: the await OPENS the window between check and act. Two tasks read the
# same balance 100 (await), both decide it is fine, both debit → lost update.
async def withdraw(account_id, amount):
    acc = await db.fetch_account(account_id)   # CHECK (yields the loop here)
    if acc.balance >= amount:
        acc.balance -= amount
        await db.save(acc)                     # USE — another task already read 100

# GOOD — a per-key asyncio.Lock serializes the tasks touching the SAME account.
# (Holds within 1 process; with N replicas, use the database guard — below.)
_locks: dict[int, asyncio.Lock] = defaultdict(asyncio.Lock)
async def withdraw(account_id, amount):
    async with _locks[account_id]:             # closes the await window
        acc = await db.fetch_account(account_id)
        if acc.balance < amount:
            raise InsufficientFunds
        acc.balance -= amount
        await db.save(acc)
```

```python
# BAD — Django: check-then-act in two queries. Under concurrency, lost update.
acc = Account.objects.get(id=pk)               # reads 100
if acc.balance >= amount:
    acc.balance -= amount                      # decides in memory
    acc.save()                                 # two requests write 70

# GOOD — F() expression: the subtraction and the condition become ONE atomic UPDATE in
# the database. Nothing is read first; 0 rows affected = insufficient balance OR lost.
from django.db.models import F
updated = Account.objects.filter(id=pk, balance__gte=amount).update(
    balance=F("balance") - amount
)
if updated == 0:
    raise InsufficientFunds                    # see references/databases-sql.md
```

```python
# BAD — get_or_create without UNIQUE in the database: the window between the SELECT and
# the INSERT lets two requests create the SAME user (duplicate).
user, created = User.objects.get_or_create(email=email)

# GOOD — UNIQUE(email) in the schema + transaction.atomic; the database rejects the 2nd INSERT
# and get_or_create catches the IntegrityError and re-reads the winner. Without UNIQUE, the guard
# does not exist — get_or_create IS safe ONLY with the constraint underneath.
class User(models.Model):
    email = models.EmailField(unique=True)     # the constraint is what closes the window
with transaction.atomic():
    user, created = User.objects.get_or_create(email=email)
```

```python
# BAD — file TOCTOU: check-then-use leaves a window between the exists and the open.
import os
if not os.path.exists(path):
    with open(path, "w") as f:                 # another process created it in the middle
        f.write(data)

# GOOD — exclusive open: "x" fails atomically if the file already exists (O_EXCL).
try:
    with open(path, "x") as f:                 # create-or-fail in one syscall
        f.write(data)
except FileExistsError:
    pass                                       # lost the race — already exists
```

## Idiomatic guards

| Primitive | When to use |
|---|---|
| `threading.Lock` / `RLock` | Serialize in-memory read-modify-write across threads; minimal scope |
| `threading.Barrier` | Simultaneous start of N threads (concurrency test) |
| `queue.Queue` | Pass work between threads without sharing mutable state |
| `asyncio.Lock` | Close an `await` window in check-then-act (per key) |
| `asyncio.Semaphore` | Limit concurrency (N tasks on the resource at once) |
| `asyncio.gather` | Fire N simultaneous coroutines (test or fan-out) |
| `multiprocessing.Value/Array(lock=True)` | Counter/buffer shared across processes |
| `multiprocessing.Manager` | Structures (dict/list) shared with a lock across processes |
| Django `F()` | Atomic UPDATE of a counter/balance without reading first |
| Django `select_for_update()` | Pessimistic row lock for logic between read and write |
| Django `transaction.atomic` | Wrap read+write in one transaction (no mid-commit) |
| SQLAlchemy `with_for_update()` | `SELECT ... FOR UPDATE` in the ORM |
| `UNIQUE` constraint | Idempotency/uniqueness — the database rejects the duplicate |

Pessimistic lock when the logic does not fit in an `F()` UPDATE (isolation levels and
write skew: see `references/databases-sql.md`):

```python
# Django — locks the row; only valid inside transaction.atomic
with transaction.atomic():
    acc = Account.objects.select_for_update().get(id=pk)
    if acc.balance < amount:
        raise InsufficientFunds
    acc.balance -= amount
    acc.save()

# SQLAlchemy — SELECT ... FOR UPDATE in the ORM
with Session(engine) as s, s.begin():
    acc = s.execute(
        select(Account).where(Account.id == pk).with_for_update()
    ).scalar_one()
    acc.balance -= amount
```

## Prove the guard

A race does not reproduce sequentially. The test launches **N threads at once** with
`threading.Barrier` and asserts the invariant. Run it against the BAD version to see
it fail; against the GOOD one to pass.

```python
import threading

def test_concurrent_increment_no_lost_update():
    box = {"n": 0}
    lock = threading.Lock()
    N = 50
    barrier = threading.Barrier(N)              # simultaneous start

    def bump():
        barrier.wait()                          # all threads start together
        for _ in range(1_000):
            with lock:                          # remove the lock → test FAILS
                box["n"] += 1

    threads = [threading.Thread(target=bump) for _ in range(N)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    assert box["n"] == N * 1_000                # invariant: no update lost
```

```python
# asyncio — N simultaneous requests with gather; asserts the balance never goes < 0
import asyncio

def test_concurrent_withdraw_never_negative():
    async def run():
        await seed_account(account_id=1, balance=100)
        # 100 withdrawals of 10 over balance 100: at most 10 can succeed
        results = await asyncio.gather(
            *(withdraw_request(account_id=1, amount=10) for _ in range(100)),
            return_exceptions=True,
        )
        acc = await fetch_account(1)
        assert acc.balance >= 0                  # invariant: never negative
        assert sum(r is True for r in results) == 10
    asyncio.run(run())
```

## Lint & static detection

Python has no ThreadSanitizer; lint catches little of races. The real detector is a
**concurrency test + database guard**. Full catalog in
`references/lint-detectors.md`. What exists:

```bash
flake8 --select=ASYNC .   # flake8-async: blocking call/sleep in a coroutine, await on the wrong lock
bandit -r .               # security; catches file TOCTOU (B108 tmp, FS race patterns)
ruff check .              # covers part of the flake8-async/bugbear rules (ASYNC*, B*)
```

- **Catches**: `time.sleep` or a blocking call inside `async def`, insecure temp
  `open`, some filesystem TOCTOU.
- **Does NOT catch**: database lost update, logical check-then-act, missing
  `select_for_update`, `get_or_create` without `UNIQUE`. None of it shows in the
  linter — only in a concurrent test and review.

## Python-specific anti-patterns

- **"I have the GIL, so it is thread-safe"** — the GIL protects the opcode, not the
  sequence. `counter += 1` is 3 opcodes; I/O and `await` yield mid-sequence.
- **`get_or_create` / `update_or_create` without `UNIQUE`** in the schema — creates a
  duplicate under a race. The constraint is the guard; the method alone is not.
- **`select_for_update()` outside `transaction.atomic`** — Django raises an error;
  without the transaction the lock does not last. A pessimistic lock is valid only inside it.
- **Read-decide-write in memory, then `.save()`** (instead of `F()`) — the canonical
  ORM lost update.
- **A global `asyncio.Lock` for everything** — serializes the whole service. Use a
  **per-key** lock (`defaultdict(asyncio.Lock)`) to lock only the account/resource.
- **`threading.Lock`/`asyncio.Lock` in a service with N workers/replicas**
  (gunicorn, uvicorn `--workers`) — does not serialize across processes. The guard must
  live in the database (`select_for_update`, `UNIQUE`).
- **`multiprocessing` expecting a shared mutable global** — each process has its own
  copy; use `Value`/`Array(lock=True)` or `Manager`.
- **Testing concurrency with a sequential `for`** — it never opens the window. Use
  `threading.Barrier` or `asyncio.gather` for a simultaneous start.
