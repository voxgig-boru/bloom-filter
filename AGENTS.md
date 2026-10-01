# AGENTS.md — using the `Bloom` library

Guidance for an AI coding agent calling this bloom-filter library from a
boru project. Every code block below was re-run against `boru-lang/boru`
**main @ `64c5ab2`** (2026-10-01). If you read nothing else, read
[The one calling rule](#the-one-calling-rule) and
[Common mistakes](#common-mistakes).

> **Calling convention — forward args, receiver last:** `Bloom.verb …args
> bf`. Piping `bf Bloom.verb …args` also works. Receiver-first
> `Bloom.verb bf …args` misbinds: `boru check` (which `boru X` runs first)
> rejects it for `add`/`contains`, and for `merge` it silently picks the
> other filter as the target.

> **Known upstream defect (boru main @ 64c5ab2):** in a program that
> imports **`boru:test`** as well as this library, the words that return a
> filter (`Bloom.add`, `Bloom.make`, `Bloom.merge`, `Bloom.decode`) can fail
> with `type_error: …: return value 1: expected BloomFilter, got
> BloomFilter` — `boru:test`'s record types are minted with colliding type
> IDs. Programs that do not import `boru:test` are unaffected. See
> `dx-report.md` → "Migration to boru main @ 64c5ab2".

## What it is

A probabilistic set: "have I seen this item?" in little memory, with **no
false negatives** and a tunable false-positive rate. The public surface
is the `Bloom` namespace plus the `BloomFilter` type.

## Import

```boru
import "./bloom.aql"
```

- The path is resolved **relative to the directory of the file that
  contains the `import`** (not the working directory), for both `boru X`
  and `boru check X`. A test in `test/` therefore writes
  `import "../bloom.aql"`.
- No `end` is needed after `import`; a trailing `end` still works and is
  harmless.
- Do **not** import `boru:math-util`, `boru:array-util`, `boru:bin-util`, or
  `boru:struct-util` yourself — `bloom.aql` imports its own dependencies.

## The one calling rule

boru is not C/Python/JS: there is no `f(a, b)` and no `obj.method(a)`.
Every public `Bloom.*` word takes the **receiver — the `BloomFilter` — as
its LAST argument**. A call binds its arguments in signature order: the
tokens written after the word fill the leading positions, and whatever is
left comes off the stack. Because the receiver is last, two orders bind
correctly:

```
Bloom.verb arg1 arg2 receiver     # forward form (canonical)
receiver Bloom.verb arg1 arg2     # piping form (also fine)
```

- **Forward form (preferred/canonical):** verb first, every argument
  forward, receiver last — `Bloom.add "x" bf`.
- **Piping form (equally correct):** the receiver flows in from the left /
  off the stack — `bf Bloom.add "x"`.

Group a call in parens to use its result as a value:

```boru
def bf (Bloom.make {n: 1000, p: 0.01})
def _ (Bloom.add "alice" bf)
print (Bloom.contains "alice" bf)    # => true
print (bf Bloom.contains "alice")    # => true  (piping — same result)
```

**The wrong order is receiver-*first*, all-forward.** `Bloom.add bf "x"`
puts `bf` in the *item* slot and leaves the receiver slot to a String, so
no signature matches: `boru check` reports
`uncalled_function: call to 'bloom-add' matched no signature`, and because
`boru X` runs that check first, the program does not run. The one case the
checker cannot catch is `merge`, whose two arguments are both
`BloomFilter`s: `Bloom.merge a b` is legal and merges **`a` into `b`**
(the last argument is the receiver). Write `Bloom.merge b a` (or
`a Bloom.merge b`) to merge `b` into `a`.

**Avoid unnecessary `end`.** A call is already terminated by the parens
around it — or by being the complete forward argument of `print` / `def` /
another verb — so a trailing `end` *there* is redundant noise. Reserve `end`
(or its preferred spelling `;`) for a **bare, ungrouped** call at statement
level that is followed by more tokens. A stray `end` is harmless.

## API reference (exact call shapes)

Call shapes below use the canonical forward form. The piping form
`receiver Bloom.verb args` binds identically (e.g. `bf Bloom.add item`).
Group each in parens to use its result as a value.

| Call | Returns | Notes |
|------|---------|-------|
| `Bloom.make {n: Integer, p: Float}` | `BloomFilter` | `n` = expected distinct items; `p` = target false-positive rate in `(0, 0.5]`. Derives `m`, `k`. Bad arguments raise `bad_input`. (`{…} Bloom.make` is the same call.) |
| `Bloom.add item bf` | the **same** `bf` (mutated) | Any value; stringified internally. Sets `k` bits, increments `added`. |
| `Bloom.contains item bf` | `Boolean` | `false` = **definitely never added**. `true` = *probably* added (may be a false positive). |
| `Bloom.count bf` | `Integer` | **Estimate** of distinct items, not an exact tally. Empty filter ⇒ `0`. |
| `Bloom.params bf` | `Map` | `{n, p, m, k}`. |
| `Bloom.merge b a` | the **same** `a` (mutated) | Union of `b` into the receiver `a`. Requires identical `m` and `k`; else raises `incompatible_merge`. Piping: `a Bloom.merge b`. |
| `Bloom.encode bf` | `String` | jsonic snapshot: params + set-bit indices. Round-trips through `Bloom.decode`. |
| `Bloom.decode text` | `BloomFilter` | Rebuild a filter from an `encode` snapshot. Malformed text raises `bad_payload`. |

Construct filters **only** through `Bloom.make`. Treat `BloomFilter`
fields as read-only; mutate through the namespace words.

Errors carry a code and message. Catch with `do […] error […]`. Bound to a
name, read them with the field sugar — `e.code` / `e.message`; inside the
handler the error is on the stack, so read it with a **quoted** key —
`get "code"` / `get "message"` (`get` evaluates its key, so a bare
`get code` is `undefined word: code`). Dispatch on the code with `case` if
you handle several.

## Copy-paste idioms (all verified)

Create, add, query:

```boru
import "./bloom.aql"
def seen (Bloom.make {n: 10000, p: 0.01})
def _ (Bloom.add "ada" seen)
print (Bloom.contains "ada" seen)     # => true
print (Bloom.contains "linus" seen)   # => false
```

Add many in a loop (`each` body must yield a value — group the call in parens
and push a `0`):

```boru
def bf (Bloom.make {n: 1000, p: 0.01})
def _ (iota 50 each [
  var [[i] (Bloom.add (convert String i) bf) 0 ]
])
print (Bloom.count bf)          # => 50 (an estimate — expect within a few)
```

Merge two filters built with the **same `(n, p)`**:

```boru
def a (Bloom.make {n: 1000, p: 0.01})
def b (Bloom.make {n: 1000, p: 0.01})
def _a (Bloom.add "from-a" a)
def _b (Bloom.add "from-b" b)
def merged (Bloom.merge b a)             # b into a; a is the receiver
print (Bloom.contains "from-a" merged)   # => true
print (Bloom.contains "from-b" merged)   # => true
```

Guard an incompatible merge (mismatched `(n, p)` raises
`incompatible_merge`):

```boru
def a (Bloom.make {n: 1000, p: 0.01})
def b (Bloom.make {n:  500, p: 0.01})    # different n ⇒ different m
def result (do [Bloom.merge b a] error [
  get "message"                          # or: get "code", case […]
])
print (result)
# => Bloom.merge: filters disagree on m (9586 vs 4793); build both with the same (n, p)
```

In a test, assert the failure (or the specific code). This block runs
as-is even with the defect above, because the error is raised before any
`BloomFilter` is returned:

```boru
import "boru:test"
Assert.throws [Bloom.merge b a]
def e (do [Bloom.merge b a])
Assert.equal incompatible_merge/q e.code
```

Persist and reload through the snapshot string:

```boru
def snap (Bloom.encode bf)
def back (Bloom.decode snap)
print (Bloom.contains "7" back)          # => true
```

## Common mistakes

| ✗ Don't write | ✓ Write | Why |
|---------------|---------|-----|
| `Bloom.contains(bf, "x")` | `(Bloom.contains "x" bf)` | No `f(a,b)` syntax in boru. |
| `bf.contains("x")` | `(Bloom.contains "x" bf)` | No method-call syntax. |
| `Bloom.add bf "x"` (receiver *first*, all-forward) | `Bloom.add "x" bf` (forward, receiver last) or `bf Bloom.add "x"` (piping) | The receiver is the **last** param. Receiver-first matches no signature; `boru check` reports `uncalled_function` and the run is blocked. |
| `Bloom.merge a b` meaning "merge b into a" | `Bloom.merge b a` or `a Bloom.merge b` | Both args are `BloomFilter`, so nothing rejects it: the **last** argument is the receiver, and `Bloom.merge a b` mutates `b`. |
| `(Bloom.contains "x" bf end)` everywhere | `(Bloom.contains "x" bf)` | Parens already terminate the call — the `end` is redundant. Reserve `end`/`;` for a bare statement-level call followed by more tokens. |
| `def bf2 (Bloom.add "x" bf)` then use `bf` as "before" | `add` mutates in place | `bf` and the returned value are the **same** object; there is no immutable copy. |
| treat `contains ⇒ true` as certain | verify against source of truth | `true` is probabilistic (≈ rate `p`); only `false` is certain. |
| `Bloom.merge b a` with different `(n, p)` | build both with identical `(n, p)` | Mismatched `m`/`k` raises `incompatible_merge` (read `e.message` for which). |
| `e get code` / handler `[ get message ]` | `e.code`, or `get "code"` in a handler | `get` evaluates its key; a bare word key is looked up as a variable. |
| `make BloomFilter {…}` | `Bloom.make {n, p}` | Construct only via `Bloom.make` (the class has a required internal `bits` field). |
| `(Bloom.count bf)` for an exact count | read `bf.added` (or `added:` in `Bloom.encode`) | `count` is an estimate; `added` is the exact insert count. |
| `import "boru:math-util"` in your script | nothing | `bloom.aql` imports its own deps. |
| `import "./bloom.aql"` from a file in a subdirectory | `import "../bloom.aql"` | Relative imports resolve against the importing file's directory. |

A note on `print` while debugging: `print` collects its argument *forward*,
so write `print (value)` — verb first, one value per statement — and output
appears in source order. The postfix spellings reorder: `"a" print` on one
line followed by `"b" print` on the next prints `b` first (the first `print`
collects `"b"` forward), and `(a) print (b) print` prints `b` then `a`.

## Where to look next

- `docs/reference.md` — full signatures, stack-in columns, complexity.
- `api.json` — the same API as a machine-readable manifest (exact call
  shapes, argument order, return types).
- `docs/how-to.md` — task recipes (sizing, merge, persist, test).
- `test/bloom_smoke_test.aql` — a complete, runnable worked example.
- `dx-report.md` — known boru-runtime gotchas observed with this build.
