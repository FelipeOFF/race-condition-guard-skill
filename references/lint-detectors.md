# Lint & race condition detector catalog

Tools that help **find** races by language — static (they read the
code) and dynamic (they instrument execution). Each has a ceiling: none
catches **logical** races at the database/cache layer. A detector complements
concurrency testing, never replaces it.

## Summary: what exists per language

| Language | Dynamic detector | Static detector | Strength |
|---|---|---|---|
| **Go** | `go test -race` (built-in TSan) | `go vet` (copylocks), `staticcheck` | **High** — gold standard, low friction |
| **C / C++** | ThreadSanitizer (`-fsanitize=thread`), Helgrind/DRD (Valgrind) | Clang Static Analyzer, Coverity, Infer/RacerD | **High** — TSan catches real data races |
| **Rust** | Miri (UB in `unsafe` code), `loom` (concurrency model in test) | **the compiler** (`Send`/`Sync`), `clippy` | **High** — data race is a compile error |
| **Swift** | ThreadSanitizer (Xcode "Thread Sanitizer") | Swift 6 compiler (`-strict-concurrency=complete`, `Sendable`) | **High** (Swift 6) — data race becomes an error |
| **Java / Kotlin** | Java Flight Recorder (indirect), jcstress (tests) | SpotBugs `MT_CORRECTNESS`, Error Prone `@GuardedBy`, **Infer/RacerD**, IntelliJ inspections | **Medium** |
| **C# / .NET** | — | `Microsoft.VisualStudio.Threading.Analyzers`, Roslyn analyzers, NDepend | **Medium** |
| **Ruby** | — | **`rubocop-thread_safety`** | **Low-Medium** — catches common antipatterns |
| **Python** | — (no TSan; concurrent `pytest` is the way) | `bandit`, `flake8-async`, `flake8-bugbear`, `ruff` (partial) | **Low** — rely on test + database |
| **JavaScript / TS** | — | ESLint (`require-atomic-updates`, `no-await-in-loop`), types | **Low** — race is logical/DB |
| **PHP** | — | PHPStan, Psalm (not focused on concurrency) | **Low** — rely on a database lock |
| **Dart / Flutter** | — (no shared memory between isolates) | `dart analyze`: `unawaited_futures`, `discarded_futures`, `use_build_context_synchronously` | **Low** — race is logical/event-loop/DB |

## Detail and how to run

### Go — `go test -race` (the best on the market)

Instruments every memory access and **fails the test on the first real data
race** observed at runtime. Cost: ~2-10× slower, ~5-10× more memory.

```bash
go test -race ./...
go build -race ./...      # instrumented binary to run in staging
go vet ./...              # copylocks: structs with mutex copied by value
```

Limit: only detects what **executes**. The test must exercise the concurrent
path. Combine with `staticcheck` for static analysis.

### C / C++ — ThreadSanitizer + Helgrind

```bash
clang++ -fsanitize=thread -g -O1 app.cpp -o app && ./app   # TSan
g++    -fsanitize=thread -g -O1 app.cpp -o app && ./app
valgrind --tool=helgrind ./app                              # alternative
```

TSan is the same engine as Go's `-race`. It catches data races, some deadlocks,
and invalid mutex uses. **Infer/RacerD** (`infer run -- make`) does
inter-procedural static analysis without running the binary.

### Rust — the compiler does the heavy lifting

A data race between threads is a **compile error** (`Send`/`Sync` types). What
remains:

```bash
cargo clippy                       # lints, incl. concurrency patterns
cargo +nightly miri test           # UB in unsafe code
# loom: exhaustive interleaving test for lock-free structures
cargo test --features loom
```

### Swift — Swift 6 data-race safety

```bash
swift build -Xswiftc -strict-concurrency=complete   # warns/errors on non-Sendable state
# Xcode: Scheme → Diagnostics → Thread Sanitizer
```

In Swift 6 language mode the compiler **proves** the absence of data races in
code checked by actor/`Sendable`. TSan covers legacy/`@unchecked` code.

### Java / Kotlin

```bash
# SpotBugs with the multithreading category
spotbugs -textui -effort:max -bugCategories MT_CORRECTNESS target/classes

# Error Prone: annotates fields guarded by a lock and checks at compile time
# (build.gradle) errorprone + @GuardedBy("lock") on the fields

# Infer / RacerD — inter-procedural race analysis
infer run -- ./gradlew build
```

jcstress (OpenJDK) writes micro-tests that stress the memory model.
IntelliJ has useful "Concurrency" inspections in the editor.

### C# / .NET

```xml
<!-- csproj: analyzers that catch async deadlock, .Result/.Wait, etc. -->
<PackageReference Include="Microsoft.VisualStudio.Threading.Analyzers" Version="*" />
```

They catch `.Result`/`.Wait()` (deadlock in a synchronization context), missing
`ConfigureAwait`, access to an unawaited `Task`. Treat warnings as errors in CI
(`<TreatWarningsAsErrors>true`).

### Ruby — `rubocop-thread_safety`

```bash
gem install rubocop-thread_safety
# .rubocop.yml:  require: rubocop-thread_safety
rubocop --only ThreadSafety
```

Catches class variables (`@@x`), mutation of shared constants/structures,
and mutable global instances — the classic Rails antipatterns under threads.

### Python — weak lint, strong test

There is no ThreadSanitizer for Python. Use:

```bash
ruff check .                     # catches some async patterns
flake8 --select=ASYNC            # flake8-async: await under a lock, sleep in async
bandit -r .                      # security, includes some file TOCTOU
```

The real race detector in Python is the **concurrency test** (threads/asyncio
with a barrier) + **database guard**.

### JavaScript / TypeScript

```jsonc
// .eslintrc — rules relevant to concurrency
{ "rules": {
    "require-atomic-updates": "error",   // reassignment after await can lose an update
    "no-await-in-loop": "warn"
}}
```

In Node the race is almost always **logical** (await in the middle of a
read-modify-write) or in the **database**. TypeScript does not detect races.
Rely on `async-mutex`/locks and on a database guard.

### PHP

PHPStan and Psalm focus on types, not concurrency. Since each request runs
isolated (PHP-FPM), the race lives in the **shared database/cache** — the defense
is `SELECT ... FOR UPDATE`, an atomic lock in Redis, or a constraint. No linter
replaces that.

### Dart / Flutter

Isolates do not share mutable memory (they communicate by message), so there is
no memory data race — and no dynamic detector for that. The race is **logical**
(a window in the `await` inside the isolate) or in the **database**. The static
analyzer catches the most dangerous async antipatterns:

```yaml
# analysis_options.yaml
linter:
  rules:
    - unawaited_futures            # unawaited Future: lost ordering/error
    - discarded_futures            # Future discarded in a sync context
    - use_build_context_synchronously  # Flutter: BuildContext after an async gap (classic bug)
```

```bash
dart analyze            # runs the lints from analysis_options.yaml
flutter analyze         # same in a Flutter project
```

`use_build_context_synchronously` catches the real case of using `BuildContext`
after an `await` (the widget may have been unmounted in the meantime). The real
race detector in Dart is the **concurrency test** (`Future.wait` of N operations)
+ a database guard.

## Recommended CI policy

1. Run the language's **dynamic** detector on the suite (`-race`, TSan) — that's
   what catches a real data race.
2. Run the **static** one (SpotBugs/Infer/analyzers/clippy) as a PR gate.
3. Keep at least **one concurrency test** per critical invariant
   (money, uniqueness, stock) running in CI.
4. A green linter does **not** clear review: a logical database race only shows
   up through careful reading and the concurrency test.
