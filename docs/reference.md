# Reference

Technical description of the `bloom-filter` module's public surface.
This page is information-oriented: it states what each word is, its
stack signature, and what it returns. For *why* the filter behaves the
way it does, see [Explanation](explanation.md); for goal-directed
recipes, see the [How-to guides](how-to.md).

> **AI agents:** [AGENTS.md](../AGENTS.md) condenses the calling
> convention, idioms, and common mistakes for machine use.

The module exports a single namespace, `Bloom`, plus the `BloomFilter`
type. Import it with:

```boru
import "./bloom.aql"
```

The path resolves against the **directory of the importing file**, not
the working directory (a script in `test/` writes `import "../bloom.aql"`).
No `end` is required after `import`; a trailing `end` is harmless. A
consuming script does **not** need to import
`boru:math-util`, `boru:array-util`, `boru:bin-util`, or `boru:struct-util`
itself — `bloom.aql` imports them internally.

---

## Calling convention

Every word takes the **receiver** (the `BloomFilter` it reads or
mutates) as its **last** parameter. A call binds its arguments in
signature order: the tokens written after the word fill positions from
the first, and every position still unfilled is taken from the value
stack, top first. So the canonical **forward** form writes the verb,
then the arguments, then the receiver — `Bloom.add "x" bf` — and the
**piping** form `bf Bloom.add "x"` (receiver flowing in from the left)
binds identically.

Group a call in parentheses to use its result as a value —
`(Bloom.contains "x" bf)`. The parens terminate the call; a trailing
`end` (or `;`) is only needed after a bare statement-level call that is
followed by more tokens.

The **receiver-first** all-forward order (`Bloom.add bf "x"`) matches no
signature: `boru check` reports `uncalled_function`, and `boru X` runs
that check before it compiles, so the program does not run. `Bloom.merge`
is the exception the checker cannot see — both of its arguments are
`BloomFilter`s, so `Bloom.merge a b` is legal and merges `a` into `b`.

The **Call** rows below give the forward form; the **Stack in** rows
give the signature (parameter order), receiver last.

---

## Types

### `BloomFilter`

A sealed `class` instance — the filter. Fields:

| Field   | Type     | Meaning                                            |
|---------|----------|----------------------------------------------------|
| `n`     | Integer  | Target capacity (expected number of distinct items)|
| `p`     | Float    | Target false-positive probability                  |
| `m`     | Integer  | Derived bit-array width                             |
| `k`     | Integer  | Derived number of hash functions                   |
| `added` | Integer  | Count of `add` calls made against this filter      |
| `bits`  | FlexList | Packed bit storage — 63 bits per integer word      |

Instances are created only through `Bloom.make`. Treat the fields as
read-only; mutate exclusively through the namespace words. (The class
is sealed and strictly typed, so writing an unknown field or a
mis-typed value is a loud error.)

`bits` is internal: a `FlexList` of `ceil(m / 63)` integer words, bit
`i` living at bit `i mod 63` of word `i div 63`. Bit 63 (the sign
bit) is never used, so every word stays a plain non-negative Integer.

---

## Words

### `Bloom.make`

Construct a filter sized for a target capacity and false-positive rate.

| | |
|--|--|
| **Call**    | `Bloom.make {n: Integer, p: Float}` (or `{n, p} Bloom.make`) |
| **Stack in**| `opts:Options` — a Map with keys `n` and `p` |
| **Returns** | `BloomFilter` |
| **Errors**  | raises `bad_input` when `n` is not an Integer ≥ 1 or `p` is not a Float in `(0, 0.5]` |

`m` and `k` are derived from `n` and `p` (see
[Explanation §Sizing](explanation.md#sizing-the-filter)). The bounds
are enforced: a `p` above `0.5` would round `k` toward `0`, so it is
rejected rather than accepted uselessly.

```boru
def bf (Bloom.make {n: 1000, p: 0.01})
print (Bloom.params bf)
# => {"n": 1000, "p": 0.01, "m": 9586, "k": 7}
```

### `Bloom.add`

Insert an item. Any value is accepted; it is stringified internally
before hashing.

| | |
|--|--|
| **Call**    | `Bloom.add item bf` (piping: `bf Bloom.add item`) |
| **Stack in**| `item:Any`, then the receiver `bf:BloomFilter` |
| **Returns** | the same `BloomFilter`, mutated in place |
| **Effect**  | sets `k` bits; increments `added` by 1 |

`add` mutates the filter it is given and also returns it, so the
return value and the argument are the same object. Adding the same
item twice sets no new bits but still increments `added`.

### `Bloom.contains`

Test membership.

| | |
|--|--|
| **Call**    | `Bloom.contains item bf` (piping: `bf Bloom.contains item`) |
| **Stack in**| `item:Any`, then the receiver `bf:BloomFilter` |
| **Returns** | `Boolean` |

`false` means the item was **definitely never added**. `true` means
the item was **probably added** — it may be a false positive at
approximately rate `p`. There are no false negatives. See
[Explanation §No false negatives](explanation.md#why-there-are-no-false-negatives).

```boru
def _ (Bloom.add "alice" bf)
print (Bloom.contains "alice" bf)   # => true
print (Bloom.contains "carol" bf)   # => false
```

### `Bloom.count`

Estimate the number of distinct items added.

| | |
|--|--|
| **Call**    | `Bloom.count bf` |
| **Stack in**| `bf:BloomFilter` |
| **Returns** | `Integer` (estimate) |

Uses the Swamidass–Baldi estimator over the set-bit population, with a
guard that returns the exact `added` count when every bit is set. The
result is an **approximation** and typically drifts below the true
insert count as the filter fills. An empty filter counts `0`. Cost is
one native popcount per 63-bit word — `O(m/63)`.

### `Bloom.params`

Return the filter's parameters as a Map.

| | |
|--|--|
| **Call**    | `Bloom.params bf` |
| **Stack in**| `bf:BloomFilter` |
| **Returns** | `Map` with keys `n`, `p`, `m`, `k` |

```boru
def ps (Bloom.params bf)
print (ps.m)   # => 9586
```

### `Bloom.merge`

Union the source filter into the receiver — the **last** argument:
`Bloom.merge b a` merges `b` into `a`.

| | |
|--|--|
| **Call**    | `Bloom.merge b a` (piping: `a Bloom.merge b`) |
| **Stack in**| source `b:BloomFilter`, then the receiver (target) `a:BloomFilter` |
| **Returns** | `a`, now containing every bit that was set in `a` or `b` |
| **Effect**  | mutates `a` in place; `b` is unchanged; `a.added` becomes `a.added + b.added` |
| **Errors**  | raises `incompatible_merge` if `a` and `b` differ on `m` or `k` |

Both filters must have identical `m` and `k`, which happens
automatically when both were built with the same `(n, p)`. After a
merge, every item present in `a` or `b` reads as contained. The union
itself is one bitwise OR per 63-bit word.

The error message names the mismatched parameter and both values, e.g.
`Bloom.merge: filters disagree on m (9586 vs 4793); build both with
the same (n, p)`. Trap it with `do […] error […]` (read `e.code` /
`e.message` on a bound error, or `get "code"` / `get "message"` in the
handler) or assert it with `Assert.throws`.

Because both parameters are `BloomFilter`s, a reversed order is not a
type error: `Bloom.merge a b` merges `a` into `b` and mutates `b`. The
receiver — the filter that changes — is always the last argument.

### `Bloom.encode`

Serialize the filter to a jsonic-style string snapshot.

| | |
|--|--|
| **Call**    | `Bloom.encode bf` |
| **Stack in**| `bf:BloomFilter` |
| **Returns** | `String` |

The string carries `n`, `p`, `m`, `k`, `added`, and the sorted list of
set bit indices. Cost is `O(m)`.

```boru
print (Bloom.encode bf)
# => {n:1000 p:0.01 m:9586 k:7 added:1 set:[1377 1551 3904 6257 6431 8610 8784]}
```

The snapshot round-trips through `Bloom.decode`. (Exact bit indices
depend on the module's hash functions, so snapshots are portable
across processes running the *same* module version, not across
versions that changed the hashing.)

### `Bloom.decode`

Rebuild a filter from a `Bloom.encode` snapshot.

| | |
|--|--|
| **Call**    | `Bloom.decode text` |
| **Stack in**| `text:String` — the snapshot |
| **Returns** | a fresh `BloomFilter` |
| **Errors**  | raises `bad_payload` when the text is not parseable jsonic or lacks the required fields |

The payload's own `m` and `k` are trusted (not re-derived from `n` and
`p`), so a snapshot survives changes to the sizing formulas. The
rebuilt filter is independent of the original — mutating one does not
affect the other.

```boru
def snap (Bloom.encode bf)
def back (Bloom.decode snap)
print (Bloom.contains "alice" back)   # => true
```

---

## Errors at a glance

All failures raise coded errors; catch with `do […] error […]` and
read `e.code` / `e.message` on a bound error, or `get "code"` /
`get "message"` on the error the handler receives (`get` evaluates its
key, so the key must be quoted). Dispatch on several codes with `case`.

| Code | Raised by | Situation |
|------|-----------|-----------|
| `bad_input` | `make` | `n` not an Integer ≥ 1, or `p` not a Float in `(0, 0.5]` |
| `incompatible_merge` | `merge` | the filters disagree on `m` or `k` |
| `bad_payload` | `decode` | text is not parseable jsonic, or is missing/mis-typing `n p m k added set` |

A call written receiver-first (`Bloom.add bf "x"`) is not a module
error but a binding error: `boru check` reports `uncalled_function` and
the program does not run. A bare call followed by more tokens on the
same statement can collect them as arguments — group it in parens or
end it with `;`/`end`.

## Complexity

| Word       | Cost      |
|------------|-----------|
| `make`     | `O(m/63)` (allocates the word FlexList) |
| `add`      | `O(k)`    |
| `contains` | `O(k)`    |
| `count`    | `O(m/63)` |
| `params`   | `O(1)`    |
| `merge`    | `O(m/63)` |
| `encode`   | `O(m)`    |
| `decode`   | `O(m/63 + s)` for `s` set bits |
