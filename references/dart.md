# Race Conditions in Dart

## Concurrency model

Dart runs code in an **isolate**: single-threaded, with its own **event loop**
and **own heap**. Isolates **do not share mutable memory** — they communicate by
message (`SendPort`/`ReceivePort`, `Isolate.spawn`, `compute()`), actor-model. So
the classic **shared-memory data race is impossible across isolates**: the
runtime copies (or transfers) the message, never sharing the object. **Within**
an isolate (including the Flutter main isolate), the single-threaded event loop
makes the race **logical**: every `await` is a *yield* point where another
microtask/event runs in between — like JavaScript. Read a value, `await`, then
write based on it, and the TOCTOU window opens.

## The races you'll encounter

- **No data race across isolates** — isolated heaps + message passing mean
  `Isolate.spawn`/`compute()` can't corrupt shared memory. Cost: copying the
  message. Payoff: no hardware corruption to tame.
- **Read-modify-write crossing `await`** — `final n = state.count; await io();
  state.count = n + 1;` on a shared singleton/provider of the isolate. Two
  executions read the same `n`, both write, one lost (lost update).
- **Double-submit** — the user taps twice (or `onPressed` fires before the 1st
  `Future` finishes); two `placeOrder` run, two orders created.
- **Async check-then-act over cache/file** — N calls see the empty cache and all
  fire the same expensive fetch (stampede); or `if (!await file.exists())`, two
  writes race.
- **Flutter: `BuildContext` after async gap / `setState` after `dispose`** —
  using `context` or `setState` after an `await`, once the widget left the tree.
  The bug `use_build_context_synchronously` catches.
- **Logical race in the database/backend** — universal; an application lock with
  N instances serializes nothing. The definitive guard is in the database — see
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

// GOOD: a ClickHandler decoupled from the widget — wire it to ANY button's
// onPressed, so you keep full control of the widget (no wrapper widget, no
// flag at the call site). A time threshold drops rapid repeat taps; an optional
// secondaryCheck adds an app-level gate (e.g. "not already submitting", "form valid").
class ClickHandler {
  ClickHandler({this.threshold = const Duration(milliseconds: 600)});

  final Duration threshold;     // exposed so tests can set a deterministic window
  DateTime? _lastClickTime;

  /// Runs [callback] only if enough time passed since the last accepted click
  /// (>= threshold) AND [secondaryCheck] (when given) returns true. Otherwise
  /// the tap is dropped. Returns whether the click was accepted.
  Future<bool> handleClick(
    Future<void> Function() callback, {
    bool Function()? secondaryCheck,
  }) async {
    final now = DateTime.now();
    final last = _lastClickTime;

    if (last != null && now.difference(last) < threshold) return false; // repeat tap → ignore
    if (secondaryCheck != null && !secondaryCheck()) return false;       // gate rejected → ignore

    _lastClickTime = now;        // accepted: stamp BEFORE awaiting (closes the sync window)
    await callback();
    return true;
  }
}

// Usage — your own fully-customizable button; the handler guards the tap.
final _clicks = ClickHandler();
ElevatedButton(
  onPressed: () => _clicks.handleClick(
    () => api.placeOrder(cart),
    secondaryCheck: () => !controller.isSubmitting,   // optional app-level gate
  ),
  child: const Text('Pay'),
);
// UI defense doesn't replace the server guard: idempotency key + UNIQUE in the database.
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
| Synchronous mutation + `await` after | RMW on isolate state: write with no await in the middle (1st choice) |
| `Lock` (`package:synchronized`) | Serialize an **async critical region within 1 isolate** (cache init, local queue) |
| Single-flight (cache the in-flight `Future` in a `Map`) | Deduplicate concurrent fetches for the same key (cache/API stampede) |
| `Completer<T>` | Coordinate producer/consumer; expose 1 `Future` resolved by an external event |
| `ClickHandler` (threshold + `secondaryCheck`) | Double-submit in Flutter: reusable, widget-agnostic handler debounces taps + app-level gate (UX defense, not server) |
| `mounted` / `context.mounted` before `setState`/using `context` | After async gap in Flutter — avoids `setState` after `dispose` |
| `Isolate.spawn` / `compute()` | Heavy CPU off the main isolate — no shared state, message passing |
| `unawaited(...)` (`dart:async`) | **Conscious** fire-and-forget — silences `unawaited_futures` by showing intent |
| `UNIQUE` + atomic UPDATE in the database | Persisted state, idempotency, unique slug — see `references/databases-sql.md` |

Rule: for **persisted state**, the definitive guard is always in the database.
`Lock` and single-flight serialize within **one** isolate only — with N app
instances (or N backend replicas) they serialize nothing. Use the in-memory lock
for a local resource; the database for the rest.

## Proving the guard

A race doesn't reproduce sequentially. Fire N simultaneous operations with
`Future.wait` over the **same** state and assert the invariant. Use the `test`
package (or `flutter_test`).

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

Without the `Lock`, the `await` between read and write lets the 100 microtasks
read the same `value`, leaving the result well below 100 — that's how the test
**fails the unguarded version**. For a deterministic event loop, `fakeAsync`
(package `fake_async`, separate dev_dependency — `flutter_test` uses FakeAsync
under the hood via `tester.pump` but doesn't re-export the `fakeAsync` function)
controls virtual time, making the interleaving reproducible rather than
scheduler-dependent.

## Lint & static detection

Dart has **no** data race detector — and needs none: with no shared mutable
memory across isolates, there's no hardware data race to find. The defense is
**concurrency testing + database guard**. What exists and helps with the
**logical** race:

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
  `flutter_lints`) — catch unawaited `Future`s. A serialized RMW or accidental
  fire-and-forget usually shows up here; mark the intentional one `unawaited(...)`.
- **`use_build_context_synchronously`** (in `flutter_lints`) — catches
  `BuildContext` crossing an async gap, the classic Flutter bug. Solve with
  `if (!mounted) return;` or `if (context.mounted)` after the `await`.
- **`custom_lint`** + **`riverpod_lint`** — for Riverpod-managed state; catch
  misuse of `ref`/providers that hides state races.

Limit: the analyzer doesn't see a race **between** calls nor at the database
layer. A green linter doesn't clear review — a logical database race surfaces
only in the concurrent test and the eye. Cross-cutting catalog in
`references/lint-detectors.md`.

## Dart-specific anti-patterns

- **"Isolates don't share memory, so everything is safe"** — true only for the
  data race. Within the isolate the **logical** race lives at every `await`, and
  the database remains shared state across instances.
- **`Lock`/single-flight in an app with N instances** — serializes one isolate
  only. For persisted state, the guard is in the database (UNIQUE + atomic
  UPDATE) or a distributed lock — see `references/databases-sql.md`.
- **`setState` after `await` without checking `mounted`** — if the widget left
  the tree, "setState() called after dispose()" blows up. Always `if (mounted)`
  after the gap.
- **`BuildContext` stored and used after the gap** — capture what you need
  (`Navigator`, `ScaffoldMessenger`) **before** the `await`, or check
  `context.mounted` after.
- **`Future.forEach`/`for` with `await` in series when it should be parallel** —
  hides a serialized RMW; the inverse (`Future.wait` without a limit) fires too
  many writes, overflowing the pool. For throttling, slice into batches.
- **Forgetting the `await` (accidental fire-and-forget)** — the method
  "finishes" before the write happens, hiding the window. Use `await`, or
  `unawaited(...)` if intentional. The `unawaited_futures` lint is your ally.
- **Swallowing a `UNIQUE` conflict without deciding a winner** — a silent
  `catch (_) {}` turns the race into silent corruption; treat it as "lost,
  return the 1st".
- **Heavy CPU on the main isolate** — not a race, but it freezes the event loop;
  the jank masks event order. Send it to `compute()`/`Isolate.spawn`.
