# Developer-experience report: bloom-filter on boru

## Migration to boru main @ 64c5ab2 (2026-10-01)

**Build under test:** `boru-lang/boru` main @ `64c5ab2` (2026-09-30) —
1,587 commits past `6185620` (2026-07-21), the build this library was last
verified on. **Branch:** `claude/boru-main-migration`.

**Result.** `bloom.aql` and all five suites **compile** (no
`compile_failed` anywhere) and `boru check` reports **0 errors, 0
warnings** on `bloom.aql` and on every suite (one info each:
`module_body_executed_in_check`, emitted for every program that imports a
source module). **All five suites run green.** The four suites that import
`boru:test` hit one upstream runtime defect (§M1 below) and work around it
by importing `../bloom.aql` **before** `boru:test`; a scratch boru build
carrying the one-line upstream fix runs them green in either order.

| suite | `boru X` (compiled — the only path) | `boru check` |
|---|---|---|
| `bloom_unit_test.aql`  | ✓ all green (library imported before `boru:test`, §M1) | 0 errors |
| `bloom_unit_spec.aql`  | ✓ all green (library imported before `boru:test`, §M1) | 0 errors |
| `bloom_prop_test.aql`  | ✓ all green, 8 of 8 properties (library imported first, §M1) | 0 errors |
| `bloom_prop_spec.aql`  | ✓ all green, 5 of 5 properties (library imported first, §M1) | 0 errors |
| `bloom_smoke_test.aql` | ✓ (no assertions; does not import `boru:test`) | 0 errors |
| `bloom.aql` (module)   | — | 0 errors |

With `boru:test` imported first (the order before this migration) the same
four suites fail: `bloom_unit_test` and `bloom_unit_spec` abort on
`type_error: bloom-add: return value 1: expected BloomFilter, got
BloomFilter`, and 6 of 8 / 2 of 5 properties fail on it in the two property
suites.

### Breaking changes hit, and what changed here

1. **`/r` → `/v`** (ADR-011, 2026-08-19). The export map's
   `make: make-bloom/r` … no longer parsed (`f/r` is now just an unbound
   slash-bearing name): `boru check bloom.aql` reported 8 ×
   `undefined_word: undefined word: make-bloom/r`. Now `make-bloom/v` etc.
2. **Relative imports resolve against the importing file's directory**
   (run and check alike), not the working directory. Every suite's
   `import "./bloom.aql"` found nothing, so `Bloom` was undefined
   (e.g. 36 check errors in the smoke suite). Suites now
   `import "../bloom.aql"`; docs no longer say "relative to the working
   directory".
3. **One execution path.** Since 2026-09-19 a program compiles to bytecode
   and runs on the VM or fails with `[boru/compile_failed]`; there is no
   interpreter, and `--compile` / `--force-compile` / `--no-compile` are
   usage errors. `boru X` also runs the static check first and a check error
   blocks the run. `test/divergence/run.sh` was rewritten from an
   interpreter/check/bytecode agreement matrix into a run + check gate.
4. **`get` evaluates its key.** The documented handler idiom
   `do […] error [ get message ]` and `e get code` now fail the check with
   `undefined word: message` / `code`. Docs now use `e.code` / `e.message`
   on a bound error and `get "message"` (quoted) in a handler.
5. **Receiver-first misbinds are now loud.** `Bloom.add bf "x"` /
   `Bloom.contains bf "x"` used to return a plausible wrong answer; now
   `boru check` reports `uncalled_function: call to 'bloom-add' matched no
   signature` and the run is blocked. `Bloom.merge a b` (two
   `BloomFilter`s) is still accepted and merges `a` into `b` — the docs now
   spell out the direction. The old advice about `boru check`'s
   `mixed_form_call` nudge is gone: that advisory now fires only for 3+-arg
   calls whose deepest stack slot is `Any`, never for a `Bloom.*` word.
6. **Output formats.** `print` of a Map renders JSON-style
   (`{"n": 1000, "p": 0.01, "m": 9586, "k": 7}`) while template
   interpolation renders jsonic (`{n:1000 p:0.01 m:9586 k:7}`), and Map
   keys keep insertion order (the docs showed them sorted). Encode
   snapshots read `{n:… p:… m:… k:… added:… set:[…]}`; the set-bit indices
   are unchanged, so snapshots from the old build still decode.
7. **Suite tails.** `"---" print` followed by
   `"fail count: " print Test.fail-count end print` collected forward and
   left `"---"` on the stack (`Assert.equal: expected 2, got ---`). Tails are
   now one grouped `print (…)` per statement and the forward
   `Assert.equal 0 (Test.fail-count)`. `Assert.equal`'s forward form is
   `expected actual`; the old stack spelling `expected actual Assert.equal`
   bound them the other way round, so failures read
   `expected 6, got 0`. Only the failure message changes; equality is
   symmetric. Unit assertions now use the forward form too.
8. **`Test.check-prop` returns its `PropertyResult` Map.** The suite left
   eight of them as end-of-run stack residue (printed after `all green`);
   each is now bound with `def _pN (…)`.
9. **Import order of the four `boru:test` suites** (works around §M1, not a
   language change): `import "../bloom.aql"` now comes before
   `import "boru:test"`, with a comment naming the defect.

No test case, expected value or tolerance was changed. A mutation check
confirms the suites still bite: changing one expected value, one spec `out`
or one property per suite makes each fail with the matching fail count.

### Open upstream defects

**M1. 🔴 `boru:test` mints its types with a colliding type-ID counter**
(runtime answer bug; not recorded in boru's NUR.md).

Minimal repro (`lib.boru` + `main.boru`):

```boru
# lib.boru
def Box class { v: 0 }
def mk fn [ [n:Integer] [Box] [ make Box {v: n} ] ]
export "L" { mk: mk/v }
```

```boru
# main.boru
import "boru:test"
import "./lib.boru"
print (L.mk 1)
# => error: [boru/type_error]: mk: return value 1: expected Box, got Box
# (without the boru:test import: Class/Box{v:1})
```

Cause: `BuildTestModule` (`lang/go/modules/test.go`) builds its module
sub-registry with `newDefaultRegistry()` and never calls
`modReg.Types.AdoptSeqFrom(parent.Types)`, so the record types its preamble
mints draw IDs from a fresh counter and collide with the first three types
minted anywhere else in the program. Commit `e69b9ac35` (2026-07-02, "Fix
minted-type ID collisions across sibling registries") added that call to
the module-body path and to `parse`, `model`, `matrix-util`, `time-util`,
`io`, `net` and `minilang`, but not to `boru:test`. The VM's return check
compares against `core.CanonicalType(r, exp)` — a lookup by ID — and so
finds `boru:test`'s type instead of `BloomFilter` (`eng/go/vm.go`,
`checkReturnContract`); the retired interpreter compared `got.Is(exp)` directly,
which is why the defect stayed invisible until the VM became the only path.
The same lookup makes `bf is Bloom.BloomFilter` answer `false`. Per word,
with `boru:test` imported first on stock main @ `64c5ab2`: `Bloom.add` and
`Bloom.merge` (which return the filter they were handed) fail the return
check; `Bloom.make` and `Bloom.decode` (which construct the value through the
same mis-resolved type) return without error, and `contains` / `count` /
`params` / `encode` run. So a `boru:test` program that only builds filters
and asserts a raised error (e.g. the `incompatible_merge` example in
`AGENTS.md`) passes in either order; anything that adds or merges does not.

**Import order matters when the library exports its type.** The collision
itself happens in either order (`boru:test`'s counter always restarts), but
the importing registry's ID index keeps the **first** type adopted under an
ID (`TypeTable.Adopt`, `core/go/typetable.go`), and a module's types are
adopted only when it *exports* them as bare type literals
(`adoptEscapedTypes`, `lang/go/native/native_module_module.go`). `bloom.aql`
exports `BloomFilter` in the `Bloom` map, so importing it **before**
`boru:test` lets `BloomFilter` claim the ID, and `boru:test`'s colliding
type is skipped. The minimal repro above does not export `Box`, so there the
order makes no difference; add `Box` to its export map
(`export "L" { Box mk: mk/v }`) and `import "./lib.boru"` before
`import "boru:test"` prints `Class/Box{v:1}`. (An earlier draft of this
section concluded from the non-exporting repro that order never matters;
that was wrong for this library.)

Verified fix: the same tree with that one line added after
`modReg.BaseDir = parent.BaseDir` in `BuildTestModule`, built in a scratch
directory, runs all five suites green, unchanged, and makes the repro print
`Class/Box{v:1}`.

Workaround: **applied** — each of the four `boru:test` suites imports
`../bloom.aql` before `boru:test` (a natural, semantics-preserving reorder,
commented in place). All five suites then run green on stock main @
`64c5ab2`, and the behaviour matches the patched build: a probe exercising
`Test.case` / `Test.spec` / `Test.run-spec` / `Test.prop` /
`Test.run-property` / `Test.check-prop` plus `Bloom.make` / `Bloom.add`
gives the same output with the library first on stock main as with
`boru:test` first on the patched build. Consumers that import `boru:test`
must do the same; `AGENTS.md`, the skill and `api.json` say so. Dropping
`BloomFilter` from the return contracts was rejected: it would weaken the
API to dodge a harness bug. (A fragile alternative for a library that does
*not* export its type — defining throwaway classes after `boru:test` so the
program's own mints absorb the colliding IDs — depends on how many types
`boru:test` mints internally and is not used here.) Remove the reorder
comments once the upstream fix lands.

Side effect seen in the same report: the diagnostic's `-->` header names
`bloom.aql` (the callee's file) but prints the **caller's** source lines
under it (e.g. `--> …/bloom.aql:56:42` above test-file line 56).

**M2. 🟡 A caught, statically certain error fails compilation.**

```boru
def C class { x: 0  ys: FlexList }
def e (do [make C {x: 1}])
print e.code
# boru check  => 0 errors (1 info: the same type_error, downgraded because `do` traps it)
# boru X      => [boru/compile_failed]: … the check pass stopped at [type_error]
#                make: missing field "ys" for class Class/C — this is a compiler defect
```

The compile gate stops on a check finding even when the surrounding
`do […]` traps it at run time (`compileFailedError` in `lang/go/boru.go`
selects `CaughtAtRuntime` findings deliberately). Not hit by the suites: it
only affects the documented anti-pattern `make BloomFilter {…}`, which now
fails at compile time instead of raising a catchable error. Relatedly, the
runtime `make: missing field` error (reached through a computed map) has no
`code` field (`e.code` is `None`).

**M3. 🟡 `print` forward collection (§1 below) is still open.** Two postfix
prints on consecutive lines run out of order: `"a" print` then `"b" print`
prints `b`, then `a` (the first `print` collects `"b"` forward), and
`(1 add 1) print (2 add 2) print` prints `4`, then `2`. The verb-first
`print (value)` idiom remains the reliable one, and every suite and doc uses
it.

**Property generators now compile too (2026-10-02, boru-lang/boru#528).**
Every suite compiled as a program, but 11 runtime callbacks still declined
their compile stamp and ran on the interpreter — the `Test.check-prop` /
`Test.prop` generator bodies (`boru -compile-report`: "closure
storedfn$body: unapplied fn-value in body residual (dynamic apply not
lowered)"; boru COMPILABLE-SUBSET §5). `bloom_prop_test.aql` declined 8
(50:3, 67:3, 90:3, 122:3, 138:3, 161:3, 189:3, 212:3), `bloom_prop_spec.aql`
3 (74:5, 83:5, 112:5); both now decline **0**. The rewrite: a direct member
draw is grouped — `[(r.int 1 50)]`, `[(r.string charset 8)]`,
`[ (r.string charset (r.int 1 16)) ]` — and each NESTED generator
(`r.list-of` over an inner `[r.string …]` body) moved into a named fn whose
body groups the call (`gen-key-pair` in the test suite, `gen-keys` in the
spec), called as `[(gen-key-pair r)]` / `[ (gen-keys r) ]`. The two other
spellings of the nested generator hit known upstream compiler divergences,
both reproduced on this library's shape: grouped inline,
`[(r.list-of [r.string charset 6] 2)]` fails every run with `undefined word:
r`; and with the call left bare as the fn's result it compiles but repeats
the first draw — the merge property would have been handed `["z2nxjz",
"z2nxjz"]` (the same key into both filters) instead of `["z2nxjz",
"ebhy3a"]`, silently weakening it. No site was left. Value identity was
proved with a scratch harness that runs the old and new bodies through
`Test.check-prop` with a value-printing property (seeds 4, 250, 99999 × 25
runs), the real property at the suite's own runs/seed/shrinks and at those
seeds, and — for the spec — the `Test.prop` / `Test.run-property` path at
its 100 runs / seed 1 / 200 shrinks: the old (interpreted) and new
(compiled) outputs are byte-identical (1,218 lines), as are failing-property
shrink reports and the suites' own output. No runs, seed, max-shrinks
argument or property body changed.

Fixed and re-verified on this build: the §3 bytecode block-local `each`
binding defect (a block-local filter filled from an `each` body inside a
`Test.test` block counts 50 of 50), and the `boru check` unused_def false
positive for defs read only inside code bodies.

Re-verification of the docs: every `boru` code block in `AGENTS.md`,
`SKILL.md` (both copies, byte-identical), `README.md`, and
`docs/{tutorial,how-to,reference}.md` was extracted and run against this
build from a scratch directory; outputs in the `# =>` comments are the
observed ones.

---

**Date:** 2026-06-11 (second round)
**boru build under test:** `boru-lang/boru` @ `7193a7d3`
(`7193a7d3c69857207e44b4bd53541b9b0d4348aa`, main as of 2026-06-11;
39 commits past `958c379b`, which this report previously covered;
built locally with `GOFLAGS=-mod=mod`; version string now reports
`boru 0.1.0-dev (git 7193a7d3c698)`).
**Context:** re-verification round. The first 2026-06-11 report (at
`958c379b`) filed eight issues after migrating this module to the
class/Array/raise surface. Six of the eight — including all three
🔴 — were fixed upstream within the same day's 39 commits, several
visibly in direct response to the DX reports. Every verdict below was
re-reproduced first-hand against the build above using the original
minimal repros; the module's five test suites pass on this build
unmodified.

Severity: **🔴 high** (silent wrong results / crash / blocks a use case) ·
**🟡 medium** (friction, clear workaround) · **🟢 low** (papercut).

---

## Update (DX-driven boru fixes)

A later boru HEAD split the accessor family: `get`/`getr` now **evaluate**
their key (so `lst get i` reads the bound variable `i`), while literal
bare-word field access moved to the new `.field` / `!.field` sugar
(lowering to `dot`/`dotr`), with the quoted-atom `get field/q` form kept
for the receiver-less stack-value case. Migrating this module to that
surface was a test-only change: `bloom.aql` already read fields with
`.field` / `!.` and only used `get` with String or computed keys, so it
needed no edits; the six literal-field error reads in
`test/bloom_unit_test.aql` (`e get code`, etc., where the caught error is
a bound receiver and `code` is a literal field) became `e.code`. No
`comp/r` box-pattern workaround applies here — this library does not use
`comp/r`, so nothing of that kind was removed (that cleanup is specific to
the sort/stats modules). Verification additionally depends on three
upstream fixes carried by the local boru build under which this was checked
— the `comp/r` frame over-pop fix, `StructUtil.parse` float fidelity, and
the checker `no_signature` fix; the migrated suites and `boru check` are
green only on a build carrying them.

---

## Fixed since the `958c379b` report

- **🔴→✅ Guard `if` + following `def`: guards fire first now**
  (boru `00cb7a79`, "guards fire before the next statement"). The
  defining repro — an else-less validation `if` whose `raise` was
  pre-empted by eager evaluation of the next `def` statement — now
  raises the guard's own error:

  ```boru
  def t fn [ [x:Any] [Integer] [
    if ((x is Float) not) [
      def m "not a float"
      raise bad_input m
    ]
    def y (x gt 0.0)
    7
  ] ]
  do [t none] error [ get code ]    # => bad_input  (was: incomparable)
  ```

  `bloom.aql` keeps the explicit empty else `[]` on its guards anyway —
  it costs nothing, reads as intent, and stays correct on older builds.

- **🔴→✅ Class-field defaults are per-instance** (boru `607cd1b9`).
  A mutable schema default (`store:(flex {})`) is no longer one shared
  value: writing through one instance is invisible to another. The
  Python-style mutable-default trap is gone. (`BloomFilter` still
  declares `bits` as a required typed field and passes a fresh Array
  per `make` — that remains the clearer design.)

- **🔴→✅ `Object` instances format** (same commit, "open objects
  render"). `print (object {a:1}) end` prints `Object{a:1}`; a bare
  `make Object {}` on the final stack prints `Object{}` instead of
  SIGSEGV-ing the interpreter.

- **🟡→✅ `raise` accepts template-string messages** (boru `00cb7a79`,
  "templates fill typed slots"). Both the bare and parenthesised forms
  now work, with the code and interpolated message intact:

  ```boru
  raise bad_input `got ${t}`        # => bad_input, message "got x"
  ```

  The bind-first idiom (`def msg …` then `raise code msg`) is no longer
  required; this module keeps it for back-compat and readability.

- **🟢→✅ `getr` raises the documented `not_found`** (boru `93ebcd40`;
  was `getr_error`, contradicting REFERENCE.md).

- **🟢→✅ `StructUtil.jsonify` emits Floats as JSON numbers** (boru
  `862546fd`); a `jsonify` → `parse` round trip preserves the Float
  type now. (`Bloom.encode` continues to use canon — unchanged, just
  no longer the only type-preserving option.)

Also fixed without having been formally filed: `boru -version` now
stamps the git commit (`1981f601`), so "which build am I on?" — a
recurring nuisance across these reports — answers itself.

---

## Still open

### 1. 🟡 `print` forward-arg collection reverses/breaks chained prints

Unchanged through three builds:

```boru
(1 add 1) print (2 add 2) print     # prints 4 then 2 — the first
                                    # print collects (2 add 2)
```

The reliable idiom remains one fully-grouped value per statement —
`print (`label: ${value}`) end` — with which output appears strictly
in source order. Every print in this module's tests and docs uses it.

### 2. ✅ `boru check` is now gating-ready (resolved on the pinned build)

The check-mode false positives are **gone** on the current pin. Two upstream
re-pins cleared them: `2342477` ("checker-accuracy fixes: 0 check
warnings/info") and `7b1a4fb` ("full check cleanliness: 0 errors/warnings/
info"). Both blockers this section tracked are fixed:

- the false `no_signature: no matching signature for mul` in
  `derive-m`/`derive-k` (arithmetic through `convert Float`) and the
  consequent `fn_body_error` — gone (the `convert` return-type fix);
- the `unused_def` cascade on the words reachable only through the
  `export "Bloom" {…}` map — the checker now traces those exports as uses.

`boru check bloom.aql` reports **0 errors** (and exits `0`), so it is safe to
gate. **Follow-up:** the CI static-check step in `.github/workflows/test.yml`
still runs `boru check --soft bloom.aql` with `continue-on-error: true` (an
advisory carried over from when the false positives were real). Dropping
`--soft` and `continue-on-error` — `run: boru check bloom.aql` — turns it into
a real gate. That edit needs a token with `workflow` scope (as the workflow
promotion did), so it is left for a maintainer.

### 3. ✅ Bytecode (`--compile`) each-body block-local divergence — fixed upstream (`407feda`)

> **Resolved 2026-06-24.** The divergence below is **fixed** on boru
> `407feda` (the reduced repro is byte-identical between interpreter and
> `--compile`), along with two short-lived `main` regressions that broke
> the library on the 2026-06-23 tips — a `None`-in-template interpolation
> bug and `convert`/fold `no_signature` check false positives (all in
> `f247557` / `fc47452`; see `aql-backend-report.md` and upstream
> `design/CLIENT-FIXES-2026-06-24.md`). `test/divergence/run.sh` now pins
> `407feda` and every suite is clean across interpreter, `boru check` (0
> errors), and `boru --compile`. The original finding is kept below as the
> record; the unit suite's top-level `_seen` fixture is retained (harmless,
> and keeps the suite robust on older builds).

Newer boru can run a program through a bytecode backend instead of the
interpreter, selectable at the CLI: `boru --compile X` (bytecode when
compilable, else a *silent* fallback to the interpreter — documented to be
identical, "opt-in performance, never semantics") and `boru --force-compile X`
(require the bytecode path, or abort with a refusal reason). A differential
test (`test/divergence/`, run with `test/divergence/run.sh`) checks the
contract `boru --compile X == boru X` across this library's suites.

Most of it holds — and that is real progress: the loop-free core
(`make`/`add`/`contains`/`merge`/`encode`/`decode`) now **fully compiles**
under `--force-compile` and returns byte-identical results, where at this
module's pin (`7193a7d3`) the bytecode path couldn't run the library at
all. The one sharp edge: a compiled `each` body **drops a block-local
binding** from the enclosing block. Reduced repro (passes on the
interpreter, wrong under `--compile`):

```boru
import "boru:test" end
import "./bloom.aql" end
[ def bf ({n: 1000, p: 0.01} Bloom.make end)
  def _ (iota 50 each [ var [[i] bf Bloom.add (convert String i) end 0 ] ])
  def cnt (bf Bloom.count end)
  true (45 lte cnt) Assert.equal end
] "count-within-tolerance" Test.test end
# interpreter => passes
# --compile   => each: element 0: [aql/undefined_word]: undefined word: bf
```

Inside the `each` the compiled path can't see the block-local `bf`, so
`bf Bloom.add …` raises `undefined word: bf`. The damage is that this leaks
through `--compile` (TRY): the emitter thinks it can lower the body, so it
does *not* fall back to the interpreter, and the wrong result escapes —
breaking the "identical, never semantics" guarantee. Trigger is narrow: a
*block-local* `def` referenced from an `each` body. A **top-level** binding
survives; a single-expression top-level loop is instead *refused* (`each`
Stage 2/3) and falls back cleanly. Upstream boru bug, not a bloom defect.

The fix on our side is one structural choice: `test/bloom_unit_test.aql`
builds its bulk fixture (`_seen`) at **top level** rather than inside the
`Test.test` block — keeping it in scope for the compiler, and (the leading
underscore) skipping `boru check`'s unused_def false positive for body-only
defs. With that, every suite is clean across all three surfaces
(interpreter, `boru check` with 0 errors, and `boru --compile` identical to
the interpreter); `test/divergence/run.sh` enforces it. Tested against boru
`c44d994` (the harness builds a newer boru than this module's pin, since the
bytecode CLI postdates `7193a7d3`). See `test/divergence/README.md`.

---

## Observations on the new build

- **The DX feedback loop works.** Six issues filed against `958c379b`
  were fixed within 39 commits, with commit messages that read
  straight off the report ("guards fire before the next statement",
  "per-instance mutable class defaults; open objects render"). A
  parallel report from the `boru:decision` module got the same
  treatment (`1981f601`), and that module moved out of core
  (`a7882da9`).
- **New language surface since `958c379b`** (not yet exercised by this
  module): lambda arrows (`(x:Integer => body)`, `ec35e87a`/
  `dfe262d6`), map overloads for `each`/`fold`/`filter` plus `keys`/
  `vals` and a `KeyVal` entry type (`c6ed6e1a`), a `canon` word for
  round-trippable source (`c0b727bf`), type-valued params
  (`ce9914a3`), and a categorised `describe` with guaranteed-complete
  word docs (`ce133d6c`/`fd82aee9`). The `keys`/`vals` words would
  have simplified the sparse-map bit store this module used two
  designs ago; the packed-Array design doesn't need them.
- **Stability:** all five suites, the AGENTS.md verification script,
  and both tutorial scripts produce byte-identical results on
  `958c379b` → `7193a7d3`. Hashing, sizing, encode payloads, and the
  measured tutorial false-positive rate (97/1000 at p = 0.1) are
  unchanged.

---

## Upgrade notes: `db828ec` → current main

Carried forward for anyone jumping from the older pin (all migrated in
this module's history):

| Change | Before | After |
|--------|--------|-------|
| `refine Object` removed | `def T (refine Object {…})` | `def T class {…}` (subclass: `refine <Class> {…}`) |
| `StringUtil.indexof` argument order | haystack-first (`indexof <haystack> <needle>`) | **haystack-last** (`indexof <needle> <haystack>`); whole string module is subject-last |
| Integer overflow | silent 64-bit wrap | hard `integer_overflow` error — mask (`BinUtil.band`) before multiplying if you relied on wrap |
| `set` on a mutable container | returned values varied | Store / Object / Array / class: writes in place, **returns nothing**; FlexMap/FlexList: returns the node; Map: returns a new map |
| `import` terminator | `import "x" end` required | `end` optional (structure-first); bare `import "x"` is the idiomatic form again |
| Custom errors | only the undefined-word idiom | `raise` (code, message — template literals fine, payload map form) |

---

## Summary

| # | Severity | Issue | Status vs `958c379b` |
|---|----------|-------|----------------------|
| — | — | guard `if` + following `def` pre-empted (was §1 🔴) | **fixed** (`00cb7a79`) |
| — | — | mutable class default shared across instances (was §2 🔴) | **fixed** (`607cd1b9`) |
| — | — | formatting an `Object` crashes (was §3 🔴) | **fixed** (`607cd1b9`) |
| — | — | `raise` rejects template messages (was §4 🟡) | **fixed** (`00cb7a79`) |
| — | — | `getr` code ≠ docs (was §6 🟢) | **fixed** (`93ebcd40`) |
| — | — | `jsonify` stringifies Floats (was §7 🟢) | **fixed** (`862546fd`) |
| 1 | 🟡 | `print` forward-collection reverses/breaks | unchanged (3rd report) |
| 2 | ✅ | `boru check`: false `mul` no_signature + export-map `unused_def` | **resolved** on `7b1a4fb` (0 errors; gating-ready) |
| 3 | ✅ | bytecode `--compile` block-local `each`-body divergence (+ two 2026-06-23 `main` regressions) | **fixed** upstream `f247557`/`fc47452`; harness pin moved to boru `407feda` |
