# How-to guides

Task-oriented recipes. Each one assumes you already know roughly what a
bloom filter is; if not, start with the [Tutorial](tutorial.md). For the
*why* behind any of these, follow the links into the
[Explanation](explanation.md); for exact signatures, the
[Reference](reference.md).

- [Install and run boru](#install-and-run-boru)
- [Size a filter for a target false-positive rate](#size-a-filter-for-a-target-false-positive-rate)
- [Add and query items](#add-and-query-items)
- [Estimate how many distinct items you've added](#estimate-how-many-distinct-items-youve-added)
- [Merge two filters](#merge-two-filters)
- [Handle an incompatible merge](#handle-an-incompatible-merge)
- [Serialize a filter](#serialize-a-filter)
- [Reload a serialized filter](#reload-a-serialized-filter)
- [Use the filter from your own script](#use-the-filter-from-your-own-script)
- [Run the tests](#run-the-tests)

---

<a id="install-and-run-aql"></a>

## Install and run boru

The module is written in boru, which has no tagged release yet, so build
the `boru` binary from source (`go install …/cmd/go/boru@latest` fails on
the repo's `replace` directives). This library tracks boru **main**:

```bash
git clone https://github.com/boru-lang/boru /tmp/boru-source
cd /tmp/boru-source/cmd/go          # the CLI module; its main package is ./boru
go build -o "$HOME/.local/bin/boru" ./boru
```

(Inside a full checkout the repo's `go.work` wires the sibling modules
together. From a source tarball or a lone `cmd/go`, build with
`GOWORK=off GOFLAGS=-mod=mod go build …` instead — that is what CI, the
SessionStart hook and `test/divergence/run.sh` do.)

Make sure `$HOME/.local/bin` is on your `PATH`, then check it:

```bash
boru -version
```

Run any script in this repo by passing its path:

```bash
boru test/bloom_smoke_test.aql
```

`boru X` compiles the program to bytecode and runs it on the VM — since
boru 2026-09-19 that is the only execution path (the old `--compile` /
`--force-compile` / `--no-compile` flags are gone) — after a static
pre-flight check; a check error stops the run. `boru check X` runs the
check alone.

This module was last verified against boru main @ `64c5ab2`
(2026-10-01); the CI workflow (`.github/workflows/test.yml`) builds
whatever main is at run time.

---

## Size a filter for a target false-positive rate

Pick `n` (how many distinct items you expect) and `p` (the
false-positive rate you'll tolerate, in `(0, 0.5]`), and hand them to
`Bloom.make`:

```boru
import "./bloom.aql"
def bf (Bloom.make {n: 100000, p: 0.001})
print (Bloom.params bf)
# => {"n": 100000, "p": 0.001, "m": 1437759, "k": 10}
```

You do not choose the bit width or hash count — `m` and `k` are derived
to meet your `p` at load `n`. Smaller `p` costs more bits. Inspect the
result with `Bloom.params`. Out-of-range arguments (a non-integer or
non-positive `n`, a `p` outside `(0, 0.5]`, or a missing key) raise a
`bad_input` error rather than building a useless filter. (How the
numbers are derived: [Explanation → Sizing](explanation.md#sizing-the-filter).)

---

## Add and query items

`Bloom.add` records an item (any value — it is stringified internally);
`Bloom.contains` tests membership and returns a Boolean:

```boru
def _ (Bloom.add "user@example.com" bf)

print (Bloom.contains "user@example.com" bf)     # => true
print (Bloom.contains "nobody@example.com" bf)   # => false  (guaranteed correct)
```

The filter goes **last** — `Bloom.add item bf` — or pipes in from the
left, `bf Bloom.add item`. Writing it first (`Bloom.add bf item`) matches
no signature and `boru` refuses to run the program.

A `false` is always correct. A `true` means "probably present" — verify
against your real store if a false positive would be costly.

To add many items, loop with `each` (push a sentinel `0` so the loop
body yields a value):

```boru
def _ (iota 1000 each [
  var [[i] (Bloom.add `key-${i}` bf) 0 ]
])
```

---

## Estimate how many distinct items you've added

```boru
print (Bloom.count bf)
```

`count` returns an **estimate** derived from the bit pattern, not a
stored tally, so it drifts a little as the filter fills. If you need the
*exact* number of `add` calls instead, read the `added` field — it is
accessible directly as `bf.added` and is also carried in the
`Bloom.encode` snapshot. (Background:
[Explanation → Estimating cardinality](explanation.md#estimating-cardinality).)

---

## Merge two filters

Two filters built with the **same `(n, p)`** can be unioned. `merge`
folds its first argument into its last — the receiver — and returns the
receiver:

```boru
def a (Bloom.make {n: 1000, p: 0.01})
def b (Bloom.make {n: 1000, p: 0.01})
def _a (Bloom.add "from-a" a)
def _b (Bloom.add "from-b" b)

def merged (Bloom.merge b a)                 # b into a (piping: a Bloom.merge b)
print (Bloom.contains "from-a" merged)   # => true
print (Bloom.contains "from-b" merged)   # => true
```

`merge` mutates the receiver (`a`) in place, so `a` and `merged` are
the same object. `b` is left untouched. Mind the direction: both
arguments are filters, so `Bloom.merge a b` is accepted too — and merges
`a` into `b`. This is the basis for
distributed counting — build filters independently, then union them.

---

## Handle an incompatible merge

`merge` requires both filters to share `m` and `k`; otherwise it raises
an `incompatible_merge` error whose message names the mismatched
parameter. Wrap the call in `do … error …` to recover; inside the
handler the Error value is on the stack, with `code` and `message`
fields:

```boru
def a (Bloom.make {n: 1000, p: 0.01})
def b (Bloom.make {n:  500, p: 0.01})   # different n → different m

def result (do [Bloom.merge b a] error [
  get "message"
])
print (result)
# => Bloom.merge: filters disagree on m (9586 vs 4793); build both with the same (n, p)
```

The key is quoted because `get` evaluates its key (a bare `get message`
looks up a variable named `message`). On an error bound to a name, the
field sugar reads the same fields: `e.code`, `e.message`. To branch on
the code instead, dispatch with `case` on `get "code"`. In a test, assert
the failure (or its exact code). Import the library **before**
`boru:test` — on boru main @ 64c5ab2 the reverse order trips an upstream
type-ID collision (see the note at the top of [AGENTS.md](../AGENTS.md)):

```boru
import "./bloom.aql"
import "boru:test"
def a (Bloom.make {n: 1000, p: 0.01})
def b (Bloom.make {n:  500, p: 0.01})
Assert.throws [Bloom.merge b a]
def e (do [Bloom.merge b a])
Assert.equal incompatible_merge/q e.code
```

(Why the module raises coded errors:
[Explanation → Raising errors](explanation.md#raising-errors).)

---

## Serialize a filter

`Bloom.encode` produces a jsonic-style string snapshot — parameters plus
the set bit indices — suitable for logging or persistence:

```boru
def snap (Bloom.make {n: 1000, p: 0.01})
def _ (Bloom.add "x" snap)
print (Bloom.encode snap)
# => {n:1000 p:0.01 m:9586 k:7 added:1 set:[603 2193 2602 4192 4601 6191 8190]}
```

---

## Reload a serialized filter

`Bloom.decode` rebuilds a filter from an encode snapshot — the round
trip preserves the parameters, the exact `added` count, and every set
bit:

```boru
def text (Bloom.encode snap)
def back (Bloom.decode text)
print (Bloom.contains "x" back)   # => true
```

The rebuilt filter is independent of the original (mutating one does
not touch the other). Malformed input — unparseable text, or a payload
missing any of `n p m k added set` — raises a `bad_payload` error.

One caveat: the bit indices are produced by this module's hash
functions, so a snapshot is portable across processes running the
*same* module version, not across versions that changed the hashing.

---

## Use the filter from your own script

Import the library by a path relative to **your script's own
directory** (not the directory you run `boru` from); you do **not** need to import
`boru:math-util`, `boru:array-util`, `boru:bin-util`, or `boru:struct-util`
yourself — `bloom.aql` pulls in its own dependencies:

```boru
import "./bloom.aql"

def bf (Bloom.make {n: 1000, p: 0.01})
# … use the Bloom namespace …
```

A script in a subdirectory writes `import "../bloom.aql"` (as the test
suites in `test/` do). No `end` is needed after `import`. Wrap a
`Bloom.*` call in parens to use its value; a bare call followed by more
tokens on the same statement needs `;` (or `end`) so the word doesn't
collect them. `test/bloom_smoke_test.aql` is a complete worked example
you can copy from.

If your script also imports `boru:test`, import `./bloom.aql` **first**:
on boru main @ 64c5ab2, `boru:test` imported first makes `Bloom.add` and
`Bloom.merge` fail with `expected BloomFilter, got BloomFilter`
(an upstream type-ID collision — see the note at the top of
[AGENTS.md](../AGENTS.md)).

---

## Run the tests

Five suites ship with the module. Run them with `boru`:

```bash
boru test/bloom_unit_test.aql   # example-based unit tests — direct (boru:test)
boru test/bloom_unit_spec.aql   # example-based unit tests — declarative spec format
boru test/bloom_prop_test.aql   # property tests — direct Test.check-prop form
boru test/bloom_prop_spec.aql   # property tests — declarative spec format
boru test/bloom_smoke_test.aql  # end-to-end walk-through over every public word
```

The file names follow a consistent convention: `_test.aql` is a direct
suite (assertions or `Test.check-prop` calls written out in code), and
`_spec.aql` is a declarative suite (cases or properties built as data
and handed to a runner). Both the unit and property layers ship in both
forms.

The two unit suites express the same example checks two ways:
`bloom_unit_test.aql` asserts imperatively with `Test.test` /
`Assert.equal`, while `bloom_unit_spec.aql` builds each check as a
`TestSpec` (`Test.spec` / `Test.case`) that `Test.run-spec` dispatches.

The two property suites are likewise split: `bloom_prop_spec.aql` builds
each property as a declarative `PropertySpec` (`Test.prop`) and runs it
with `Test.run-property` at the default 100 iterations — clean, but the
run count is fixed. `bloom_prop_test.aql` calls the imperative
`Test.check-prop` driver directly, passing `runs`/`seed`/`max-shrinks`
explicitly, which is why it carries the expensive O(m) properties
(merge, encode, decode) at a smaller run budget.

Each test file ends by asserting `Test.fail-count` is `0` (and prints
`all green`), so a failure makes `boru` exit non-zero — which is exactly
what the [CI workflow](../.github/workflows/test.yml) checks on every
push and pull request.

> **Status on boru main @ 64c5ab2:** all five suites compile, run green and
> check with 0 errors. The four suites that import `boru:test` import
> `../bloom.aql` **first**, which works around an upstream defect
> (`boru:test`'s types collide with `BloomFilter`'s type ID, so with
> `boru:test` first `Bloom.add` / `Bloom.merge` fail `expected
> BloomFilter, got BloomFilter`). See `dx-report.md` §M1.

One more check sits outside this set. `test/divergence/run.sh` is the
single-path gate: every suite must exit 0 under `boru X` (compiled — the
only execution path now) **and** report 0 errors under `boru check X`, and
`bloom.aql` must check clean on its own:

```bash
test/divergence/run.sh                               # builds boru @ main HEAD (cached)
BORU=$HOME/.local/bin/boru test/divergence/run.sh    # or use an existing binary
```

It replaced the old interpreter / check / byte-compiler agreement matrix
when boru retired the interpreter fallback and the `--compile` flags. See
[`test/divergence/README.md`](../test/divergence/README.md).
