# race-condition-guard

Claude Code skill: validates and prevents **race conditions** in concurrent code —
double-submit, TOCTOU, lost update, check-then-act, in-memory data race,
deadlock, idempotency — and teaches how to **prove the guard with concurrency
tests** (not with a sequential call).

**Multi-language.** The core is agnostic; the idiomatic detail (`BAD→GOOD`
examples of how to avoid + the lint/detectors of each ecosystem) lives in
`references/<language>.md`, loaded on demand.

## Contents

- **Core (`SKILL.md`)** — mental model of the TOCTOU window, taxonomy of
  guards (constraint, atomic update, pessimistic/optimistic lock, idempotency
  key, advisory/distributed lock, upsert, atomics, ownership), concurrency
  test principle, checklist and anti-patterns.
- **Per language (`references/`)** — Python, JavaScript/TypeScript,
  Java/Kotlin, C#/.NET, Go, Rust, C/C++, Ruby, PHP, Swift, Dart/Flutter. Each
  one with concurrency model, `BAD→GOOD` examples, idiomatic guards,
  concurrency test and lint detectors.
- **Cross-cutting (`references/`)** —
  `databases-sql.md` (universal layer: isolation levels, `FOR UPDATE`,
  constraints, upsert, advisory/Redis lock),
  `lint-detectors.md` (catalog of detectors: `go -race`, ThreadSanitizer,
  SpotBugs, Infer/RacerD, clippy/Miri/loom, VS Threading Analyzers,
  rubocop-thread_safety…),
  `detection-workflow.md` (procedure to scan an existing codebase:
  detect → classify → guard → test → harden in CI).

## Structure

```
race-condition-guard/
  SKILL.md
  references/
    detection-workflow.md      lint-detectors.md      databases-sql.md
    python.md                  javascript-typescript.md
    java-kotlin.md             csharp-dotnet.md
    go.md                      rust.md                cpp.md
    ruby.md                    php.md                 swift.md
    dart.md
```

## Install

```bash
npx skills add git@github.com:FelipeOFF/race-condition-guard-skill.git -y
```

Or through the [myskills](https://github.com/FelipeOFF/my-claude-code-skills)
marketplace (`programming@myskills`), where the skill comes already vendored.

## License

MIT © 2026 Felipe Oliveira
