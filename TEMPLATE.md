# Using this template

**Forking `bloom-filter` to start a new boru library? Read this first, then
delete it.**

This repo is a GitHub *template* for a small, single-purpose **boru library**.
It is also a real, runnable library (a bloom filter), so everything here —
tests, docs, CI, and the agent configuration — is a working example you adapt
rather than a skeleton you fill in. Clone it with **“Use this template”**, then
walk the checklist below.

The pair repo [`trie`](https://github.com/voxgig-boru/trie) follows the same
structure for a *multi-module* library; look there if your library ships
several modules/namespaces.

---

## The shape you’re inheriting

```
<lib>.aql                     the library — one module exporting one namespace
boru.jsonic                   package manifest (name, main, files)
api.json                      machine-readable API manifest (for agents)
AGENTS.md                     the canonical agent/human calling guide
CLAUDE.md                     Claude Code entrypoint; @-imports AGENTS.md
README.md                     human landing page
TEMPLATE.md                   this file — delete after instantiation
LICENSE                       MIT
dx-report.md                  boru-runtime gotchas hit while building THIS library
.gitignore
.claude/
  settings.json               registers the SessionStart hook
  hooks/session-start.sh      builds boru @ boru-lang/boru main HEAD in remote sessions
  skills/<lib>-aql/SKILL.md   portable, auto-loaded agent skill (canonical copy)
.claude-plugin/
  marketplace.json            this repo is also a plugin marketplace
plugins/<lib>-aql/
  .claude-plugin/plugin.json  plugin manifest
  skills/<lib>-aql/SKILL.md   BUNDLED copy of the skill (must equal the canonical one)
proposals/
  README.md                   slot for upstream-language RFCs (see the file)
.github/workflows/
  test.yml                    GitHub Actions: build boru @ main HEAD, run every suite,
                              the divergence gate, and the consistency job
docs/                         Diátaxis docs: tutorial, how-to, reference, explanation
test/
  <lib>_unit_test.aql         example-based unit tests — imperative (Test.test)
  <lib>_unit_spec.aql         example-based unit tests — declarative spec
  <lib>_prop_test.aql         property tests — imperative (Test.check-prop)
  <lib>_prop_spec.aql         property tests — declarative spec
  <lib>_smoke_test.aql        end-to-end smoke run over every public word
  divergence/run.sh           single-path gate: every suite runs (compiled) + checks clean
  divergence/README.md        what the gate asserts and why
```

---

## Conventions this template encodes

- **Test naming:** `<subject>_<unit|prop>_<test|spec>.aql`, plus one
  `<project>_smoke_test.aql`. `unit` vs `prop` is the *what*; `test` =
  imperative surface (`Test.test` / `Test.check-prop`), `spec` = declarative
  data surface (`Test.run-spec` / `Test.run-property`). `<subject>` is the
  library name for a single-module library, or the variant name for a
  multi-module one (e.g. `radix_unit_test.aql`). Every assertion-bearing suite
  ends with the same tail and prints `all green`; smoke suites carry no
  assertion (pass = no error).
- **boru version: track `main`, record what you verified.** There is no pinned
  commit. CI (`.github/workflows/test.yml`), the SessionStart hook and
  `test/divergence/run.sh` each resolve boru-lang/boru **main HEAD** at run
  time and build it (`cmd/go` → `./boru`, cached by SHA), and a daily CI
  schedule surfaces upstream breaks while the repo is idle. `api.json` keeps
  `aql_ref: "main"` and records the last commit the docs were re-verified on
  in `verified_against`; AGENTS.md, SKILL.md and CLAUDE.md name the same
  commit.
- **One execution path.** `boru X` compiles the program to bytecode and runs it
  on the VM, after a static pre-flight check whose errors block the run; there
  is no interpreter fallback and the `--compile` / `--force-compile` /
  `--no-compile` flags are retired (since boru 2026-09-19). "The suite runs"
  therefore means "the suite fully compiles". Never reach for `-no-check` to
  get a suite green.
- **The gate:** `test/divergence/run.sh` — every suite exits 0 under `boru X`
  (and prints `all green` if it asserts) and `boru check X` reports 0 errors,
  and every library module checks clean standalone. `BORU=/path/to/boru`
  skips the build.
- **Imports resolve against the importing file’s directory**, for run and
  check alike: suites in `test/` import `"../<lib>.aql"`; consumers write the
  path relative to their own script.
- **Calling convention:** every public word takes its receiver LAST, so the
  forward form `<Ns>.verb …args receiver` is canonical and piping
  `receiver <Ns>.verb …args` binds identically. Export functions in the
  namespace map with `/v` (`make: make-thing/v`) — a bare name that holds a
  function CALLS it (ADR-011; `/r` is the pre-2026-08-19 spelling and no
  longer parses). The same goes for any function passed as data (a comparator,
  a callback): pass `f/v`.
- **Agent docs, layered (kept self-contained, guarded against drift):**
  `AGENTS.md` is the canonical prose guide; `CLAUDE.md` `@`-imports it;
  `.claude/skills/<lib>-aql/SKILL.md` is a strict condensation that auto-loads;
  `api.json` is the machine-readable signature source; `docs/reference.md` is
  the prose signature source. The bundled plugin SKILL.md must stay byte-equal
  to the canonical one (CI checks this).
- **Docs follow Diátaxis** (tutorial / how-to / reference / explanation), with
  `docs/how-to.md#install-and-run-boru` as the canonical install anchor (the
  page also keeps a legacy `install-and-run-aql` anchor for old links).
- **Known upstream defect to expect (boru main @ 64c5ab2):** if your library
  defines a type (`class`, `refine …`) and a suite imports `boru:test`, a word
  that declares that type as its return can fail with
  `expected <T>, got <T>` — `boru:test` mints its record types from a fresh
  type-ID counter. Workaround: **export the type** from the library's
  namespace (as `bloom.aql` exports `BloomFilter`) and **import the library
  before `boru:test`** in every suite — the first exported type to claim an
  ID wins. (If the type is not exported, import order does not help.) See
  this repo’s `dx-report.md` §M1; do not weaken the library’s return types
  to dodge it.
- **`.aql` module header** opens with: one-line summary, the exported
  namespace(s), a `# --- representation ---` block, a `Calling convention:`
  paragraph, and the `# Imported via …` line.

---

## Instantiation checklist

Replace `<lib>` with your library name (kebab-case, e.g. `skip-list`) and
`<Ns>` with your namespace (PascalCase, e.g. `SkipList`).

1. **Rename the module.** `git mv bloom.aql <lib>.aql`; rewrite it for your
   data structure, exporting one `<Ns>` namespace. Keep the header shape.
2. **`boru.jsonic`** — set `name`, `main: <lib>.aql`, `files: [<lib>.aql]`.
3. **Tests.** `git mv` the five `bloom_*` files to `<lib>_*`; rewrite their
   bodies and point their import at `"../<lib>.aql"`. Keep the standard tail —
   ``print (`fail count: ${(Test.fail-count)}`)``, then
   `Assert.equal 0 (Test.fail-count)`, then `print "all green"` (one grouped
   `print (…)` per statement; postfix `x print` chains reorder).
4. **`api.json`** — set `name`, `description`, the `Bloom` → `<Ns>` namespace,
   and `word_specs` with your exact call shapes (forward form), arg order
   (signature order, receiver last), and return types. Keep `aql_ref: "main"`
   and set `verified_against` to the boru commit you verified on.
5. **`AGENTS.md`** — rewrite the calling convention, API table, idioms, and
   common mistakes for `<Ns>`. This is the single source agents read.
6. **`CLAUDE.md`** — update the one-line description; it `@`-imports `AGENTS.md`
   so it needs no API content of its own.
7. **Skill + plugin.** `git mv .claude/skills/bloom-filter-aql
   .claude/skills/<lib>-aql` and `plugins/bloom-filter-aql plugins/<lib>-aql`;
   rewrite both `SKILL.md` copies (keep them identical) and update
   `marketplace.json` + `plugin.json` (name, source, description,
   homepage/repository).
8. **SessionStart hook** — in `.claude/hooks/session-start.sh`, set the smoke
   path to `test/<lib>_smoke_test.aql` (the hook builds boru @ main HEAD; no
   ref to set).
9. **Gate + CI** — in `test/divergence/run.sh`, set `SUITES` and `MODULES` to
   your files. In `.github/workflows/test.yml`, list your suites with clear step
   labels, point the static check at `<lib>.aql`, and update the `consistency`
   job’s plugin paths (editing a workflow file needs a token with `workflow`
   scope).
10. **Docs** — rewrite `docs/*` for your domain; keep the four-mode structure
    and the install anchor.
11. **`dx-report.md`** — clear it and record the boru-runtime gotchas *you* hit;
    they are project-specific.
12. **`README.md`** — rewrite for your library (drop the “Using this as a
    template” pointer).
13. **`proposals/`** — leave empty apart from its `README.md` until you have an
    upstream-language RFC to file.
14. **Delete `TEMPLATE.md`** (this file).
15. **CI is already live** at `.github/workflows/test.yml` — a repo created with
    “Use this template” inherits it and runs it on the first push/PR (just enable
    Actions for the new repo).

When the rename is done, `for f in test/*.aql; do boru "$f"; done` should end
every assertion-bearing suite with `all green`, and
`BORU=$(command -v boru) test/divergence/run.sh` should report PASS.
