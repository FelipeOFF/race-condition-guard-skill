# Race Conditions in Ruby

## Concurrency model

In MRI (CRuby) there is the **GVL** (Global VM Lock): only one thread executes
Ruby bytecode at a time, but the GVL is **released on I/O** (database query, HTTP,
file read). That does not save you: a read-modify-write that crosses an I/O call
that releases the GVL — or that is not a single atomic bytecode operation —
lets another thread enter the window. In **JRuby** and **TruffleRuby** there is no
GVL: threads run in real parallel and any shared mutable state is a true data
race. The danger
lives in web servers with a thread pool (Puma, Sidekiq) and in any `@@var`,
mutable constant, or singleton shared between requests.

## The races you will encounter

- **Memory data race** — `@@counter += 1`, a `Hash`/`Array` shared between
  threads, lazy memoization in a singleton (`@config ||= load`). On MRI it gives
  the wrong result; on JRuby/TruffleRuby it can even corrupt the structure.
- **Logical / database race** — the bread and butter in Rails: `find_or_create_by`
  creates a duplicate, check-then-act on balance/stock causes a lost update,
  form double-submit creates two records. The GVL covers none of this — the race
  is between Puma processes and replicas, in the database.
- **Lost update from a stale object** — two requests load the same `ActiveRecord`,
  both mutate different attributes and save: the second `save` overwrites the
  first entirely (not just the field that changed).

## How to avoid

```ruby
# BAD — find_or_create_by has a TOCTOU window: two requests pass the empty
# SELECT and both do INSERT → two records that should have been one.
user = User.find_or_create_by(email: email)

# GOOD — unique index in the database + rescue: the race loser gets the error
# and re-reads the winner. The index is what actually closes the window (the
# rescue only handles the conflict). Migration: add_index :users, :email, unique: true
begin
  user = User.create!(email: email)
rescue ActiveRecord::RecordNotUnique
  user = User.find_by!(email: email)  # someone won the race; use theirs
end
```

```ruby
# BAD — read-modify-write in Ruby: reads 100, decides, writes. Two requests read
# 100 and both debit → lost update. .save persists the entire stale object.
account = Account.find(id)
account.balance -= amount if account.balance >= amount
account.save!

# GOOD — atomic conditional UPDATE in the database, without reading first (closes the window).
# Returns affected rows: 0 = insufficient balance OR lost the race.
rows = Account.where(id: id).where("balance >= ?", amount)
              .update_all("balance = balance - #{amount.to_i}")
raise InsufficientFunds if rows.zero?
# see references/databases-sql.md (Pattern 1) for the detail of the conditional UPDATE
```

```ruby
# BAD — missing pessimistic lock: logic between check and act runs on data that
# another transaction can change before the save.
account = Account.find(id)
apply_complex_discount_rules(account)   # decision depends on the current state
account.save!

# GOOD — with_lock opens a transaction and does SELECT ... FOR UPDATE: serializes
# per row. No other transaction reads/writes that row until the block finishes.
account.with_lock do            # SELECT * FROM accounts WHERE id = ? FOR UPDATE
  apply_complex_discount_rules(account)
  account.save!
end                             # COMMIT releases the lock
```

```ruby
# BAD — incrementing via the object: reads, adds 1, saves. Classic lost update
# under concurrency. A counter maintained by hand in the app has this window;
# Rails' native counter_cache does NOT, because it already does an atomic UPDATE in the database.
post.comments_count += 1
post.save!

# GOOD — increment! generates an atomic UPDATE ... SET x = x + 1 in the database (does not read first).
# For counter cache use the class method, not the in-memory mutation.
post.increment!(:comments_count)        # UPDATE posts SET comments_count = comments_count + 1
# or: Post.update_counters(post.id, comments_count: 1)
```

```ruby
# BAD — class variable accumulates state between requests/threads. On JRuby it is
# a data race; on MRI it still gives the wrong count because += is not atomic.
class Metrics
  @@hits = 0
  def self.track = @@hits += 1   # non-atomic read-modify-write
end

# GOOD — concurrent-ruby: AtomicFixnum does an atomic compare-and-swap on any
# runtime. For shared maps use Concurrent::Map (thread-safe without an explicit
# lock), never a bare Hash.
require "concurrent"
class Metrics
  HITS = Concurrent::AtomicFixnum.new(0)
  def self.track = HITS.increment   # atomic even without GVL
end
```

## Idiomatic guards

| Primitive | When to use |
|---|---|
| `Mutex#synchronize` | serialize access to in-memory mutable state within the process (does not cover N replicas) |
| `Concurrent::AtomicFixnum` | numeric counter/flag shared between threads — atomic increment |
| `Concurrent::Map` | shared thread-safe dictionary (replaces a bare `Hash` under threads) |
| `Concurrent::AtomicReference` | swap an entire reference atomically (compare-and-set) |
| unique index + `rescue ActiveRecord::RecordNotUnique` | get-or-create / double-submit / idempotency key |
| `record.with_lock { ... }` | pessimistic lock (`SELECT FOR UPDATE`) for multi-step check-then-act |
| optimistic locking (`lock_version`) | low contention; catches concurrent write via `StaleObjectError` |
| `update_all` / `increment!` / `update_counters` | atomic conditional UPDATE without loading the object |
| `with_advisory_lock` (gem) | serialize by logical key across processes (`pg_advisory_xact_lock`) |

**Rails native optimistic locking**: add a `lock_version` column
(integer, default 0). ActiveRecord then starts appending `AND lock_version = :v` to
every UPDATE and raises `ActiveRecord::StaleObjectError` when 0 rows are
affected (someone wrote first). Handle it with reload + retry. Cross-cutting detail
of the version pattern in `references/databases-sql.md` (Pattern 4).

## Prove the guard

A test that fires N threads with a simultaneous start (manual barrier) and asserts the
invariant. Runnable with RSpec or Minitest — here RSpec. Use a real database
(in-memory SQLite has no real concurrency; run against Postgres/MySQL).

```ruby
# spec/models/account_concurrency_spec.rb
require "rails_helper"

RSpec.describe "Concurrent Account debit", type: :model do
  it "never leaves the balance negative under N simultaneous debits" do
    account = Account.create!(balance: 100)
    threads_count = 20
    start = Queue.new                      # manual start barrier

    threads = Array.new(threads_count) do
      Thread.new do
        start.pop                          # blocks until the start
        ActiveRecord::Base.connection_pool.with_connection do
          # guard under test: atomic conditional UPDATE
          Account.where(id: account.id).where("balance >= ?", 10)
                 .update_all("balance = balance - 10")
        end
      end
    end

    threads_count.times { start << :go }   # releases all at once
    threads.each(&:join)

    # invariant: 100 / 10 = at most 10 debits can pass; balance >= 0
    expect(account.reload.balance).to eq(0)
    expect(account.balance).to be >= 0
  end
end
```

Without the guard (`account.balance -= 10; account.save!`) the balance ends up negative or
above zero non-deterministically. Run the spec several times — one pass
may not open the window.

## Lint & static detection

Ruby has no ThreadSanitizer; the real static detection of antipatterns is
**`rubocop-thread_safety`**.

```yaml
# .rubocop.yml
require:
  - rubocop-thread_safety
```

```bash
gem install rubocop-thread_safety
rubocop --only ThreadSafety       # runs only the thread safety cops
```

**What it catches**: class/module-level mutable state
(`ThreadSafety/ClassAndModuleAttributes`,
`ThreadSafety/InstanceVariableInClassMethod`,
`ThreadSafety/ClassInstanceVariable`), mutable literal assigned to a class instance
variable (`ThreadSafety/MutableClassInstanceVariable`), `Dir.chdir` with a
process-wide effect (`ThreadSafety/DirChdir`), and a loose `Thread.new`
(`ThreadSafety/NewThread`). Catches the classic Rails antipatterns under
Puma/Sidekiq.

**What it does NOT catch**: **logical** database race (`find_or_create_by` without a unique
index, lost update, check-then-act). No Ruby linter sees this — it only shows up in the
**concurrency test** and in review. Catalog detail in
`references/lint-detectors.md`.

## Ruby-specific anti-patterns

- **"I have the GVL, therefore it's thread-safe"** — the GVL serializes bytecode, it does not
  close the I/O window nor protect on JRuby/TruffleRuby. `@@x += 1` still gets it wrong.
- **`find_or_create_by` / `first_or_create` without a unique index** — guaranteed TOCTOU
  window; creates a duplicate under concurrency. Always `add_index ..., unique: true`.
- **Lazy memoization in a singleton** (`@cache ||= build`) shared between
  threads — two threads can run `build` and/or see `@cache` half-built.
  Use `Concurrent::Map`/`Mutex` or initialize at boot (eager).
- **`@@class_variable` for application state** — shared by all
  threads/subclasses; the mutation is not atomic. Prefer `Concurrent::*`.
- **`save!` over a stale object** — persists the entire record with the values that
  were in memory, overwriting concurrent changes. Use `update_all`,
  `increment!`, `with_lock`, or optimistic locking.
- **Application mutex in a service with N processes** (Puma cluster, several dynos)
  — it only serializes within one process. To serialize across processes use an
  advisory lock or a database constraint, not `Mutex`.
- **Swallowing `RecordNotUnique`/`StaleObjectError` without deciding a winner** — re-read and
  reuse the winner's record (or retry on the stale); never ignore the error.
