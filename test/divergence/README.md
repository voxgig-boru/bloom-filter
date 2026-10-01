# Single-path gate: run · check

This library's suites are written once and must run clean on boru **main**.
`run.sh` holds every suite to two conditions:

```bash
boru X         # compile to bytecode + run on the VM — must exit 0, and an
               #   assertion-bearing suite must print `all green`
boru check X   # static check — must report 0 errors
```

and also checks the library module (`bloom.aql`) standalone for 0 errors.

## Why it is no longer a three-way comparison

Until 2026-09 this harness compared three execution surfaces — the
interpreter (`boru --no-compile X`), `boru check X`, and the byte compiler
(`boru --compile X`, plus an informational `--force-compile` coverage line) —
and failed on any disagreement. That is gone upstream:

- **There is one execution path.** Since boru 2026-09-19 a program is
  compiled to bytecode and run on the VM, or it fails with
  `[boru/compile_failed] … this is a compiler defect`. There is no
  interpreter fallback.
- **The flags are retired.** `--compile`, `--force-compile`, `--no-compile`
  and the `BORU_COMPILE` / `BORU_FORCE_COMPILE` / `BORU_NO_COMPILE` env vars
  are usage errors now.
- **`boru X` runs the check first.** A pre-flight `boru check` error blocks
  the run (`-no-check` skips it; this harness never uses it).

So "the suite runs" now means "the suite fully compiles", and the only two
surfaces left to gate are the run and the standalone check. The directory
keeps its old name so the CI job and the docs that call
`test/divergence/run.sh` keep working.

## Running it

```bash
test/divergence/run.sh                          # build boru @ main HEAD (cached), then gate
BORU=$HOME/.local/bin/boru test/divergence/run.sh   # use an existing binary, no build
BORU_REF=64c5ab2 test/divergence/run.sh             # build a specific ref
```

Without `BORU`, the script builds its own boru so it never depends on what is
on `PATH`: it resolves `boru-lang/boru` main HEAD, fetches the source as a
codeload tarball (works where a raw `git clone` is blocked), builds
`cmd/go` → `./boru`, and caches the binary under `~/.cache/boru-divergence`
keyed by the SHA (it rebuilds only when main advances). Needs `go` + network
for that one-time build. `BORU_TIMEOUT` (default 600) caps each invocation.

Sample output (boru main @ 64c5ab2, 2026-10-01):

```
[divergence] suites — run (boru X) must exit 0 [+ print 'all green'], check must report 0 errors:
  SUITE                       RUN                     CHECK         SECONDS
  bloom_unit_test.aql         ok                      ok            1
  bloom_unit_spec.aql         ok                      ok            1
  bloom_prop_test.aql         ok                      ok            1
  bloom_prop_spec.aql         ok                      ok            3
  bloom_smoke_test.aql        ok                      ok            1

[divergence] modules — boru check must report 0 errors:
  bloom.aql                   ok

[divergence] PASS — every suite compiles, runs green, and checks clean; every module checks clean.
```

Every suite **compiles** (none reports `compile_failed`), runs green and
checks with 0 errors. One upstream runtime defect is worked around in the
suites rather than fixed: `boru:test`'s module sub-registry mints its record
types from a fresh type-ID counter instead of adopting the importing
program's (`lang/go/modules/test.go` `BuildTestModule` lacks
`modReg.Types.AdoptSeqFrom(parent.Types)`, which every other type-minting
native module has), so its types collide with `BloomFilter`'s ID. With
`import "boru:test"` first, `BloomFilter` fails its own declared return-type
check (`expected BloomFilter, got BloomFilter`) and the four `boru:test`
suites go red; so each of them imports `../bloom.aql` **before**
`boru:test`, which lets the exported `BloomFilter` claim the ID first. A
scratch build with the one-line upstream fix runs all five green in either
import order. Details and the minimal repro are in `../../dx-report.md` §M1
("Migration to boru main @ 64c5ab2").

## Background: the divergence this harness used to catch

On boru `c44d994` a compiled `each` body **dropped a block-local binding**
from the enclosing block, so `bf Bloom.add …` inside an `each` raised
`undefined word: bf` under `--compile` while the interpreter passed — and,
because the emitter believed it could lower the body, `--compile` did not fall
back. `test/bloom_unit_test.aql` builds its bulk fixture (`_seen`) at top level
for that reason. The defect was fixed upstream (`407feda`) and does not
reproduce on main @ 64c5ab2 (a block-local filter filled from an `each` body
inside a `Test.test` block counts 50 of 50). The fixture stays at top level
because two tests share it.

### Wiring it into CI

`.github/workflows/test.yml`'s `divergence` job already runs
`test/divergence/run.sh` (it sets up Go; the script builds boru itself).
