# Race Conditions in Dart

## Concurrency model

Dart runs code in an **isolate**: each isolate is single-threaded, has its own
**event loop** and its **own heap**. Isolates **do not share mutable memory** —
they communicate by message (`SendPort`/`ReceivePort`, `Isolate.spawn`,
`compute()`), in the actor model. So the classic **shared-memory data race
is impossible across isolates**: the runtime copies (or transfers) the
message, it does not give access to the same object. **Within** an isolate (including
the Flutter main isolate), the event loop is single-threaded and the race is **logical**:
every `await` is a *yield* point where another microtask/event runs in between —
just like JavaScript. If you read a value, `await`, and write based on what
you read, the TOCTOU window opened.

## The races you'll encounter

- **No data race across isolates** — isolated heaps + message passing mean
  that `Isolate.spawn`/`compute()` don't corrupt shared memory. The cost is
  copying the message; the payoff is that there's no hardware corruption to tame.
- **Read-modify-write crossing `await`** — `final n = state.count; await io();
  state.count = n + 1;` on a shared singleton/provider of the isolate. Two
  executions read the same `n`, both write, one is lost (lost update).
- **Double-submit** — the user taps the button twice (or `onPressed` fires
  before the 1st `Future` finishes); two `placeOrder` run, two orders are created.
- **Async check-then-act over cache/file** — N calls see the empty cache and
  all fire the same expensive fetch (stampede); or `if (!await file.exists())` and
  two writes race.
- **Flutter: `BuildContext` after async gap / `setState` after `dispose`** — using the
  `context` or calling `setState` after an `await`, when the widget has already left the
  tree. It's the bug `use_build_context_synchronously` catches.
- **Logical race in the database/backend** — universal; an application lock in an app with N
  instances serializes nothing. The definitive guard is in the database — see
  `references/databases-sql.md`.

## How to avoid

```dart
// BAD: read-modify-write with await in the middle → lost update in isolate state
class Counter {
  int value = 0;
  Future<void> bump() async {
    final current = value;        // CHECK: read the current value
    await persist(current);       // await: another microtask enters and reads the SAME value
    value = current + 1;          // USE: writes stale; one increment is lost
  }
}

// GOOD: synchronous and relative increment; no await separates read from write
class Counter {
  int value = 0;
  Future<void> bump() async {
    value += 1;                   // atomic mutation on the single-threaded event loop
    await persist(value);         // the await comes AFTER — the window doesn't open on the state
  }
}
```

```dart
import 'package:synchronized/synchronized.dart';

// BAD: concurrent lazy-init — N calls see null and all fetch (stampede)
Config? _config;
Future<Config> getConfig() async {
  if (_config != null) return _config!;
  _config = await fetchConfig();  // await: other calls already passed the if and fetch too
  return _config!;
}

// GOOD: package:synchronized serializes the async critical region; only the 1st fetches
final _lock = Lock();
Config? _cfg;
Future<Config> getConfig() async {
  if (_cfg != null) return _cfg!;          // fast-path without lock
  return _lock.synchronized(() async {
    if (_cfg != null) return _cfg!;        // double-check inside the lock
    _cfg = await fetchConfig();
    return _cfg!;
  });
}
```

```dart
// BAD: double-submit — second tap fires before the first await finishes
ElevatedButton(
  onPressed: () async {
    await api.placeOrder(cart);   // 2nd tap enters here again → duplicate order
  },
  child: const Text('Pay'),
);

// GOOD: a reusable AsyncButton — the click handler owns its in-flight state, so
// call sites pass an async action, never an ad-hoc bool. Guards re-entrancy AND self-disables.
class AsyncButton extends StatefulWidget {
  const AsyncButton({super.key, required this.onPressed, required this.child});
  final Future<void> Function() onPressed;
  final Widget child;
  @override
  State<AsyncButton> createState() => _AsyncButtonState();
}

class _AsyncButtonState extends State<AsyncButton> {
  bool _busy = false;
  Future<void> _handleTap() async {
    if (_busy) return;                          // sync re-entrancy guard, before any await
    setState(() => _busy = true);
    try {
      await widget.onPressed();
    } finally {
      if (mounted) setState(() => _busy = false); // mounted: no setState after dispose
    }
  }
  @override
  Widget build(BuildContext context) => ElevatedButton(
        onPressed: _busy ? null : _handleTap,   // null disables the button while busy
        child: _busy
            ? const SizedBox.square(dimension: 16, child: CircularProgressIndicator(strokeWidth: 2))
            : widget.child,
      );
}

// Call site: no flag, no setState, no try/finally — just the action.
AsyncButton(onPressed: () => api.placeOrder(cart), child: const Text('Pay'));
// UI defense doesn't replace the server guard: idempotency key + UNIQUE in the database.
```

For non-widget code (an MVVM controller, a service), the same single-flight as a
plain primitive — the call site has no bool and the view watches `isBusy`:

```dart
// AsyncGuard: a 2nd call while one is in flight joins it instead of re-running.
class AsyncGuard {
  Future<void>? _inFlight;
  bool get isBusy => _inFlight != null;
  Future<void> run(Future<void> Function() action) {
    final existing = _inFlight;
    if (existing != null) return existing;            // 2nd call joins the in-flight run
    return _inFlight = action().whenComplete(() => _inFlight = null);
  }
}

// In a controller: the guard dedups taps; expose isBusy to drive the button state.
final _placeOrder = AsyncGuard();
Future<void> submit() => _placeOrder.run(() => api.placeOrder(cart));
```

```dart
// BAD: uses BuildContext after an async gap — the widget may have already been removed
Future<void> save(BuildContext context) async {
  await repo.save();                                  // async gap
  Navigator.of(context).pop();                        // context may be dead → crash
}

// GOOD: check context.mounted after the await; capture what you need BEFORE the gap
Future<void> save(BuildContext context) async {
  final navigator = Navigator.of(context);            // capture before the await
  await repo.save();
  if (!context.mounted) return;                       // BuildContext.mounted (Flutter 3.7+)
  navigator.pop();                                    // safe: tree still alive
}
// Inside a State, the getter is mounted (State.mounted); here we only have the
// BuildContext from the argument, so the correct guard is context.mounted.
```

## Idiomatic guards

| Primitive / package | When to use |
|---|---|
| Synchronous mutation + `await` after | Read-modify-write on isolate state: do the write without await in the middle (1st choice) |
| `Lock` (`package:synchronized`) | Serialize an **async critical region within 1 isolate** (cache init, local queue) |
| Single-flight (cache the in-flight `Future` in a `Map`) | Deduplicate concurrent fetches for the same key (cache/API stampede) |
| `Completer<T>` | Coordinate producer/consumer; expose 1 `Future` resolved by an external event |
| `AsyncButton` widget / `AsyncGuard` single-flight | Double-submit in Flutter: a reusable click handler owns the in-flight state (UX defense, not server) |
| `mounted` / `context.mounted` before `setState`/using `context` | After async gap in Flutter — avoids `setState` after `dispose` |
| `Isolate.spawn` / `compute()` | Heavy CPU off the main isolate — no shared state, message passing |
| `unawaited(...)` (`dart:async`) | **Conscious** fire-and-forget — silences `unawaited_futures` by showing intent |
| `UNIQUE` + atomic UPDATE in the database | Persisted state, idempotency, unique slug — see `references/databases-sql.md` |

Rule: for **persisted state**, the definitive guard is always in the database. `Lock`
and single-flight only serialize within **one** isolate — with N instances of the app
(or N backend replicas) they serialize nothing. Use the in-memory lock for a
local resource; the database for the rest.

## Proving the guard

A race doesn't reproduce with a sequential call. Fire N simultaneous operations with
`Future.wait` over the **same** state and assert the invariant. Use the
`test` package (or `flutter_test`).

```dart
import 'package:test/test.dart';
import 'package:synchronized/synchronized.dart';

// Counter serialized by Lock: the async critical region never interleaves.
class SafeCounter {
  final _lock = Lock();
  int value = 0;
  Future<void> bump() => _lock.synchronized(() async {
        final current = value;       // read
        await Future<void>.delayed(Duration.zero); // forces an event loop yield
        value = current + 1;         // write — protected by the lock, no interleaving
      });
}

void main() {
  test('100 simultaneous bumps lose no increment', () async {
    final counter = SafeCounter();

    // simultaneous start: 100 concurrent bumps over the SAME counter
    await Future.wait(
      List.generate(100, (_) => counter.bump()),
    );

    // invariant: each bump counted exactly once
    expect(counter.value, 100);
  });
}
```

Without the `Lock`, the `await` between read and write lets the 100 microtasks read the same
`value` and the final result stays well below 100 — that's how the test
**fails the unguarded version**. For a deterministic event loop, `fakeAsync`
(package `fake_async`, separate dev_dependency — `flutter_test` uses FakeAsync under
the hood via `tester.pump`, but doesn't re-export the `fakeAsync` function) controls virtual
time and makes the interleaving reproducible instead of depending on the real scheduler.

## Lint & static detection

Dart does **not** have a data race detector — and doesn't need one: with no shared mutable
memory across isolates, there's no hardware data race to find. The
defense is **concurrency testing + database guard**. What exists for real and
helps with the **logical** race:

```yaml
# analysis_options.yaml — enable the lint set and key rules
include: package:flutter_lints/flutter.yaml   # or package:lints/recommended.yaml (pure Dart)

linter:
  rules:
    - unawaited_futures            # Future ignored in an async body (forgot the await?)
    - discarded_futures            # Future discarded outside an async context
    # use_build_context_synchronously already comes in flutter_lints (Flutter)
```

```bash
dart analyze            # runs the analyzer + configured lints; PR gate
flutter analyze         # same engine, in Flutter projects
```

- **`unawaited_futures`** / **`discarded_futures`** (packages `lints` /
  `flutter_lints`) — catch unawaited `Future`s. A serialized RMW or an accidental
  fire-and-forget usually shows up here; mark the intentional one with
  `unawaited(...)`.
- **`use_build_context_synchronously`** (in `flutter_lints`) — catches `BuildContext`
  crossing an async gap, the classic Flutter bug. Solve with `if
  (!mounted) return;` or `if (context.mounted)` after the `await`.
- **`custom_lint`** + **`riverpod_lint`** — for Riverpod-managed state; they
  catch misuse of `ref`/providers that hides state races.

Limit: the analyzer doesn't see a race **between** calls nor at the database layer.
A green linter doesn't clear review — a logical database race only comes out in the concurrent test
and the eye. Cross-cutting catalog in `references/lint-detectors.md`.

## Dart-specific anti-patterns

- **"Isolates don't share memory, so everything is safe"** — true only for
  data race. Within the isolate the **logical** race is alive at every `await`, and the
  database remains shared state across instances.
- **`Lock`/single-flight in an app with N instances** — serializes within one
  isolate only. For persisted state, the guard is in the database (UNIQUE + atomic
  UPDATE) or a distributed lock — see `references/databases-sql.md`.
- **`setState` after `await` without checking `mounted`** — if the widget left the
  tree, "setState() called after dispose()" blows up. Always `if (mounted)` after the gap.
- **`BuildContext` stored and used after the gap** — capture what you need
  (`Navigator`, `ScaffoldMessenger`) **before** the `await`, or check
  `context.mounted` after.
- **`Future.forEach`/`for` with `await` in series when it should be parallel** —
  hides a serialized RMW; and the inverse (`Future.wait` without a limit) fires too many
  writes and overflows the pool. For throttling, slice into batches.
- **Forgetting the `await` (accidental fire-and-forget)** — the method "finishes" before the
  write happens, hiding the window. Use `await`, or `unawaited(...)` if it's
  intentional. The `unawaited_futures` lint is your ally here.
- **Swallowing a `UNIQUE` conflict without deciding a winner** — a silent `catch (_) {}`
  turns the race into silent corruption; treat it as "lost, return the 1st".
- **Heavy CPU on the main isolate** — it's not a race, but it freezes the event loop and the jank
  masks the event order. Send it to `compute()`/`Isolate.spawn`.
