# Contributing to race-condition-guard

This guide tells **any LLM or human** exactly what to change, and where, to extend
the skill — most commonly **adding a new language**, but also adding a guard
strategy, a lint detector, or a cross-cutting reference. Follow it literally;
the skill stays consistent only if every contribution touches the same points.

> The skill content is **English** (prose and code comments). Code identifiers
> are English. Keep it that way.

---

## 1. How the skill is structured

```
race-condition-guard/
  SKILL.md                       # language-agnostic core + routing + decision flow (entry point)
  CONTRIBUTING.md                # this file
  README.md                      # human-facing overview
  references/
    detection-workflow.md        # procedure to scan an existing codebase (scan→classify→reuse→guard→test→CI)
    lint-detectors.md            # cross-language catalog of static/dynamic race detectors
    databases-sql.md             # the UNIVERSAL data-layer guard (cited by every language file)
    <language>.md                # one per language: python, go, rust, dart, ...
```

**Design law — progressive disclosure.** `SKILL.md` is the only file loaded when
the skill triggers. Everything language-specific lives in `references/` and is
loaded **on demand**. So:

- Keep `SKILL.md` lean. Add detail to a `references/` file, not to `SKILL.md`.
- The `description` in `SKILL.md` frontmatter is loaded into **every** conversation
  for skill discovery — treat its tokens as precious (keep it ≤ ~700 chars).

**Non-negotiable principles for any contribution:**

1. Every `BAD` code block must **actually contain the race** it claims; every
   `GOOD` block must **actually close the window** and compile/run in that language.
2. **Real tools only.** Never invent a lint flag, package, or API. If you are not
   sure it exists, do not cite it.
3. **Do not duplicate `databases-sql.md`.** For the database layer, show the
   language's idiomatic call and point to `references/databases-sql.md`.
4. Prove guards with a **concurrency test** (N simultaneous ops → assert invariant),
   never a sequential loop.
5. English prose, English identifiers, no emojis. Tables for reference, fenced
   blocks for code, comment the **why** (not the obvious).

---

## 2. Adding a new language (the main case)

### Step 2.1 — Create `references/<language>.md`

Use the exact **7-section contract** below (canonical headings). Translate
`<Language>` to the real name. Target **130–220 lines**, dense.

```markdown
# Race Conditions in <Language>

## Concurrency model
2–4 sentences: what makes races happen HERE (shared memory? GIL? event loop?
process isolated per request? actor/isolate model?) and where the danger lives.

## The races you'll encounter
Bullets: memory data race vs logical/DB race, with this language's specific flavor.
State clearly which race classes are impossible here and which remain.

## How to avoid
3–5 `BAD → GOOD` pairs, short, correct, idiomatic, commented with the WHY.
Cover the classes that bite this language: check-then-act/lost update,
double-submit/idempotency, shared counter/atomic, lazy-init/singleton,
deadlock — whichever apply. These are the examples an LLM will copy, so they
must be exemplary.

## Idiomatic guards
Quick-reference table: ecosystem primitive/package → when to use it.

## Prove the guard
ONE runnable test in the language's test framework: prepare known state →
simultaneous launch of N operations over the SAME resource → assert the invariant.

## Lint & static detection
The REAL tools of the ecosystem: install/config, what each catches, what it misses.
Must be consistent with references/lint-detectors.md.

## <Language>-specific anti-patterns
Bullets: the traps unique to this language/ecosystem.
```

Quality rules to obey while writing the file:

- Read `SKILL.md`, `references/databases-sql.md`, and `references/lint-detectors.md`
  first, to match tone and cross-references.
- BAD/GOOD blocks must be correct and idiomatic (see principle 1).
- Only real lint tools (principle 2). Point to `databases-sql.md` for the DB layer
  (principle 3), do not copy it.

### Step 2.2 — Wire it into the skill (touch-point checklist)

A new language is **not done** until these files reference it. Edit each:

| File | What to add | Where |
|---|---|---|
| `SKILL.md` | The language name in the `description` parenthetical list `(Python, JS/TS, …)` | frontmatter `description:` — keep it ≤ ~700 chars |
| `SKILL.md` | The language in the **"Where the race lives"** table | append its name to the matching row (shared memory / process state / database) |
| `SKILL.md` | A routing row `\| <Language> (...) \| references/<language>.md \| key detector \|` | the **"Per language"** table |
| `references/lint-detectors.md` | A summary-table row for the language | the **"Summary: what exists per language"** table |
| `references/lint-detectors.md` | A `### <Language>` detail subsection | the **"Detail and how to run"** section |
| `references/detection-workflow.md` | A row in the **"Where the project's own guard tends to live"** table | inside **Step 3 — Reuse before you build** |
| `README.md` | The language in the per-language list **and** the structure tree | "Conteúdo / Content" + the tree block |

> If the language introduces a genuinely new concurrency category (not shared
> memory, process state, or database), add a new row to the "Where the race
> lives" table instead of appending to an existing one.

### Step 2.3 — Validate, then commit

Run the validation script in §5. Fix any failure. Then commit (§6).

---

## 3. Adding other things

**A new guard strategy** (e.g. a new locking pattern):
- Add a row to the **"Guard strategies"** table in `SKILL.md` (language-neutral).
- Show the idiomatic version in each relevant `references/<language>.md` `How to avoid`
  or `Idiomatic guards` section.
- If it is a data-layer pattern, document it in `references/databases-sql.md` and
  link from the language files.

**A new lint detector for an existing language:**
- Add it to that language's `### <Language>` subsection in `references/lint-detectors.md`
  (install/config, what it catches, what it misses).
- Update the language's `Lint & static detection` section in `references/<language>.md`.
- If it is the key detector, update the language's row in the `SKILL.md` routing table.

**A new cross-cutting reference** (like a `distributed-systems.md`):
- Create `references/<name>.md`.
- Link it from `SKILL.md` and from the language files that need it.
- Keep it out of `SKILL.md`'s body — link, don't inline.

**A reusable structure** (like Dart's `ClickHandler`):
- Put it in the relevant language file's `How to avoid` as a `GOOD` block,
  decoupled and idiomatic. Expose anything tests need (e.g. a `threshold`
  parameter). Add it to that file's `Idiomatic guards` table.

---

## 4. The section contract, as a copy-paste skeleton

See §2.1. The validator checks that each `references/<language>.md` has **exactly
7 `##` sections** and an `# ` H1. Minor wording variance in a heading is fine
("The races you'll encounter" vs "… you will encounter") as long as the seven
sections are present and in order.

---

## 5. Validation checklist

Run from the repo root before committing. Everything must pass.

```bash
python3 - <<'PY'
import os, re, glob, sys
files=["SKILL.md","README.md"]+sorted(glob.glob("references/*.md"))
langs=[os.path.basename(p)[:-3] for p in glob.glob("references/*.md")
       if os.path.basename(p) not in
       {"detection-workflow.md","lint-detectors.md","databases-sql.md"}]
fail=[]
# 1) code fences balanced (even count of ``` per file)
for f in files:
    n=sum(1 for l in open(f,encoding="utf-8") if l.startswith("```"))
    if n%2: fail.append(f"odd fences ({n}) {f}")
# 2) no leftover Portuguese prose (URLs excluded)
acc=re.compile(r"[ãõçáéíóúâêôàü]",re.I)
pw=re.compile(r"\b(não|você|para que|janela|guarda|corrida|saldo)\b",re.I)
for f in files:
    for i,l in enumerate(open(f,encoding="utf-8"),1):
        if "http" in l or "github" in l: continue
        if acc.search(l) or pw.search(l): fail.append(f"PT {f}:{i}")
# 3) each language file: H1 + exactly 7 H2 sections
for lg in langs:
    f=f"references/{lg}.md"; lines=open(f,encoding="utf-8").read().splitlines()
    if not lines or not lines[0].startswith("# "): fail.append(f"no H1 {f}")
    h2=sum(1 for l in lines if l.startswith("## "))
    if h2!=7: fail.append(f"H2={h2} (want 7) {f}")
# 4) cross-references resolve; no untranslated placeholder
refs=set()
for f in files:
    t=open(f,encoding="utf-8").read()
    refs.update(re.findall(r"references/[a-z-]+\.md",t))
    if "<linguagem>" in t: fail.append(f"PT placeholder in {f}")
for r in sorted(refs):
    if not os.path.isfile(r): fail.append(f"broken ref {r}")
# 5) frontmatter description budget
fm=re.match(r"^---\n(.*?)\n---\n",open("SKILL.md",encoding="utf-8").read(),re.S).group(1)
desc=re.search(r"description:\s*\|\n(.*?)(?=\n[a-z_]+:)",fm,re.S).group(1)
if len(desc)>1024: fail.append(f"description {len(desc)}>1024")
print(f"description: {len(desc)} chars | languages: {len(langs)} | refs: {len(refs)}")
print("VALIDATION: PASS" if not fail else "VALIDATION: FAIL")
[print("  -",x) for x in fail]
sys.exit(1 if fail else 0)
PY
```

A new language must also be referenced by `SKILL.md` (routing table),
`lint-detectors.md`, `detection-workflow.md`, and `README.md` — grep to confirm:

```bash
LANG=dart   # the file stem you added
grep -ril "references/$LANG.md\|$LANG" SKILL.md references/lint-detectors.md \
  references/detection-workflow.md README.md
```

---

## 6. Conventions

- **Commits:** Conventional Commits, type in English. Scope a language change as
  `docs(<language>): …`. Example: `docs(elixir): add Elixir reference (BEAM, OTP)`.
- **One excellent example beats many.** Do not add multiple variants of the same
  pattern in one language file.
- **Keep `SKILL.md` token-lean.** New depth goes into `references/`, never the body.
- **Optional, for scale:** you can author a new language file with an LLM and have
  a second LLM adversarially review it (verify every BAD races, every GOOD compiles,
  every lint tool is real) before wiring it in. This is how the existing references
  were produced and validated.
