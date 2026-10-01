# CLAUDE.md

This repository is the `Bloom` bloom-filter library, written in boru.

## Using the library

See @AGENTS.md for how to call the `Bloom` API correctly from boru — the
calling convention, the full API, copy-paste idioms, and the common
mistakes to avoid. Every example there was re-run against boru main @
`64c5ab2` (2026-10-01).

## Working on this repository

- A SessionStart hook (`.claude/settings.json` →
  `.claude/hooks/session-start.sh`) builds `boru` from boru-lang/boru
  **main HEAD** in remote sessions, so a fresh session can run the suites.
  Locally, build it once from source (there is no tagged release and
  `go install …/cmd/go/boru@latest` is blocked by replace directives) — see
  [docs/how-to.md](docs/how-to.md#install-and-run-boru).
- `boru X` compiles the program to bytecode and runs it on the VM — the only
  execution path since boru 2026-09-19 (`--compile` / `--force-compile` /
  `--no-compile` are retired) — after a static pre-flight check whose errors
  block the run. Never use `-no-check` to get a suite green.
- Relative imports resolve against the **importing file's directory**: the
  suites in `test/` import `"../bloom.aql"`.
- Tests live in `test/`, named `<subject>_<unit|prop>_<test|spec>.aql` plus a
  `bloom_smoke_test.aql`: `_test` = imperative (`Test.test`/`Test.check-prop`),
  `_spec` = declarative spec; `unit` = example-based, `prop` = property-based.
  Each assertion-bearing suite ends by asserting `Test.fail-count` is `0` and
  prints `all green`.
- `test/divergence/run.sh` is the single-path gate: every suite must exit 0
  under `boru X` (and print `all green` if it asserts) and report 0 errors
  under `boru check X`, and `bloom.aql` must check clean. It builds its own
  boru at main HEAD (codeload tarball) unless given `BORU=/path/to/boru`. See
  its `README.md`.
- **Current status (main @ 64c5ab2):** all five suites compile, run green
  and check with 0 errors; `bloom.aql` checks clean.
- **Import `../bloom.aql` BEFORE `boru:test`** in every suite (and tell
  consumers to do the same). On boru main @ 64c5ab2 `boru:test` mints its
  record types from a fresh type-ID counter, so with `boru:test` imported
  first `Bloom.add` / `Bloom.merge` fail with
  `expected BloomFilter, got BloomFilter`. Importing the library first lets
  the exported `BloomFilter` claim its type ID first. Details, the repro and
  the one-line upstream fix are in `dx-report.md` §M1 ("Migration to boru
  main @ 64c5ab2").
- Known boru-runtime gotchas are in `dx-report.md`. CI
  (`.github/workflows/test.yml`), the hook, and the divergence harness all
  track boru **main** (no pinned commit); `api.json`'s `verified_against`
  records the last commit the docs were re-verified on.
- Forking this repo to start a new boru library? See `TEMPLATE.md`.
