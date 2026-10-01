---
name: bloom-filter-aql
description: Use when writing or editing boru code that calls the Bloom bloom-filter library — Bloom.make / Bloom.add / Bloom.contains / Bloom.count / Bloom.params / Bloom.merge / Bloom.encode / Bloom.decode, or any file that does `import "./bloom.aql"`. Provides the exact boru calling convention (which is not C/Python/JS), the API with mutation and probabilistic semantics, verified copy-paste idioms, and fixes for the mistakes agents most often make (foreign call syntax like `bf.contains(x)`, putting the receiver *first* in an all-forward `Bloom.add bf x` order — the receiver goes LAST — merging in the wrong direction, `e get code` instead of `e.code`, over-using `end`, assuming `add` returns a new filter).
---

# Calling the Bloom bloom-filter library (boru)

A probabilistic set: "have I seen this item?" in little memory, with **no
false negatives** and a tunable false-positive rate. Public surface = the
`Bloom` namespace. Everything below is verified against boru main @
`64c5ab2` (2026-10-01).

## Import

```boru
import "./bloom.aql"
```

- Path resolves relative to the **directory of the importing file** (not
  the working directory), for `boru X` and `boru check X` alike — a test in
  `test/` writes `import "../bloom.aql"`.
- No `end` is needed after `import` (a trailing `end` is harmless).
- Do **not** import `boru:math-util` / `boru:array-util` / `boru:bin-util` /
  `boru:struct-util` — the library does it.
- **Known upstream defect:** if the same program also imports `boru:test`,
  words that return a filter (`add`/`make`/`merge`/`decode`) can fail with
  `expected BloomFilter, got BloomFilter` (colliding type IDs in boru's
  `boru:test` module; see `dx-report.md`). Without `boru:test` they are fine.

## The one calling rule

boru has no `f(a, b)` and no `obj.method(a)`. Every public `Bloom.*` word
takes the **receiver — the `BloomFilter` — as its LAST argument**. Because
the receiver is last, two orders bind correctly:

```
Bloom.verb arg1 arg2 receiver     # forward form (canonical)
receiver Bloom.verb arg1 arg2     # piping form (also fine)
```

- **Forward form (preferred/canonical):** verb first, every argument
  forward, receiver last — `Bloom.add "x" bf`.
- **Piping form (equally correct):** the receiver flows in from the left /
  off the stack — `bf Bloom.add "x"`.

Group the call in parens to use its result as a value:
`(Bloom.contains "x" bf)` or `(bf Bloom.contains "x")`.

- **Receiver-*first*, all-forward is wrong.** `Bloom.add bf "x"` matches no
  signature; `boru check` reports `uncalled_function` and `boru X` (which
  checks first) refuses to run it.
- **`merge` is the silent case.** Both arguments are `BloomFilter`s, so
  `Bloom.merge a b` is legal and merges **`a` into `b`**. To merge `b` into
  `a`, write `Bloom.merge b a` or `a Bloom.merge b`.
- **Avoid unnecessary `end`.** The parens around a call already terminate
  it — as does being the complete forward argument of `print` / `def` /
  another verb — so a trailing `end` there is redundant. Reserve `end` (or
  `;`) for a bare statement-level call followed by more tokens.

## API

| Call | Returns | Notes |
|------|---------|-------|
| `Bloom.make {n: Integer, p: Float}` | `BloomFilter` | `n` = expected distinct items; `p` = target false-positive rate in `(0, 0.5]`. Bad arguments raise `bad_input`. |
| `Bloom.add item bf` | the **same** `bf` (mutated in place) | Any value, stringified internally. |
| `Bloom.contains item bf` | `Boolean` | `false` = **definitely never added**; `true` = *probably* added (false-positive rate ≈ `p`). |
| `Bloom.count bf` | `Integer` | **Estimate** of distinct items, not a tally. Empty ⇒ `0`. |
| `Bloom.params bf` | `Map` | `{n, p, m, k}`. |
| `Bloom.merge b a` | the **same** `a` (mutated) | Union of `b` into the receiver `a` (piping: `a Bloom.merge b`). Requires identical `m`/`k` (same `(n, p)`); else raises `incompatible_merge`. |
| `Bloom.encode bf` | `String` | jsonic snapshot; round-trips through `Bloom.decode`. |
| `Bloom.decode text` | `BloomFilter` | Rebuild from a snapshot; malformed text raises `bad_payload`. |

Construct filters only via `Bloom.make`; treat `BloomFilter` fields as
read-only. Catch errors with `do […] error […]`: a bound error reads as
`e.code` / `e.message`; in the handler (error on the stack) use a quoted
key — `get "code"` / `get "message"` (`get` evaluates its key, so a bare
`get code` is `undefined word: code`).

By-design notes (boru semantics that bite here):

- **`eq` is identity, not structural equality.** `Bloom.params` returns a
  fresh `Map`, so comparing two params maps with `eq` is `false` even when
  equal — use `deq` for structural comparison of Maps/Lists.
- **A `BloomFilter` mutates in place** (`add`/`merge`), unlike Maps/Lists,
  which are immutable and copy-return from `set`/`get`. There is no immutable
  filter copy; keep no "before" alias.
- **Integer overflow fails loud.** boru ints are 63-bit and raise
  `integer_overflow` rather than wrapping — an absurd `n` to `Bloom.make`
  errors, it does not silently truncate.

## Idioms (verified)

```boru
import "./bloom.aql"
def seen (Bloom.make {n: 10000, p: 0.01})
def _ (Bloom.add "ada" seen)
print (Bloom.contains "ada" seen)     # => true
print (Bloom.contains "linus" seen)   # => false
```

Add many (each body must yield a value — group the call in parens, push `0`):

```boru
def bf (Bloom.make {n: 1000, p: 0.01})
def _ (iota 50 each [
  var [[i] (Bloom.add (convert String i) bf) 0 ]
])
```

Merge (both built with the same `(n, p)`); guard the incompatible case:

```boru
def a (Bloom.make {n: 1000, p: 0.01})
def b (Bloom.make {n: 1000, p: 0.01})
def merged (Bloom.merge b a)                              # b into a
def safe (do [Bloom.merge b a] error [ get "message" ])   # a on success, the message on failure
```

Persist and reload:

```boru
def snap (Bloom.encode bf)
def back (Bloom.decode snap)
```

## Common mistakes

| ✗ Don't | ✓ Do | Why |
|---------|------|-----|
| `Bloom.contains(bf, "x")` / `bf.contains("x")` | `(Bloom.contains "x" bf)` | boru has no call/method syntax. |
| `Bloom.add bf "x"` (receiver *first*, all-forward) | `Bloom.add "x" bf` (forward, receiver last) or `bf Bloom.add "x"` (piping) | The receiver is the **last** param; receiver-first matches no signature (`uncalled_function` from `boru check`, run blocked). |
| `Bloom.merge a b` meaning "b into a" | `Bloom.merge b a` / `a Bloom.merge b` | The last argument is the receiver; `Bloom.merge a b` mutates `b`, silently. |
| `e get code`, handler `[ get message ]` | `e.code`, handler `[ get "message" ]` | `get` evaluates its key. |
| `(Bloom.contains "x" bf end)` everywhere | `(Bloom.contains "x" bf)` | Parens already terminate — the `end` is redundant. |
| keep a pre-`add` copy of `bf` | none — `add` mutates in place | The argument and the return value are the same object. |
| trust `contains ⇒ true` | verify against the real store | `true` is probabilistic; only `false` is certain. |
| `Bloom.merge b a` with different `(n, p)` | same `(n, p)` for both | Mismatch raises `incompatible_merge`. |
| `make BloomFilter {…}` | `Bloom.make {n, p}` | Construct only via `Bloom.make`. |
| `(Bloom.count bf)` for an exact count | read `bf.added` / `Bloom.encode` | `count` is an estimate; `added` is exact. |
| `import "./bloom.aql"` from `test/` | `import "../bloom.aql"` | Imports resolve against the importing file's directory. |
| `(v) print (w) print`, or `"a" print` then `"b" print` | `print (v)`, one per statement | `print` collects forward; the postfix spellings print out of order. |

If the full repo is available, `AGENTS.md`, `api.json` (machine-readable
signatures), and `docs/reference.md` have the complete guide;
`test/bloom_smoke_test.aql` is a runnable example.
