# bloom-filter

A small, dependency-light **bloom filter** implemented in
[boru](https://github.com/boru-lang/boru) — a probabilistic set that
answers *"have I seen this item?"* in far less memory than storing the
items, with no false negatives and a false-positive rate you choose up
front.

```boru
import "./bloom.aql"

def seen (Bloom.make {n: 10000, p: 0.01})
def _ (Bloom.add "ada" seen)

print (Bloom.contains "ada" seen)     # => true
print (Bloom.contains "linus" seen)   # => false
```

> **Calling convention — forward args, receiver last:**
> `Bloom.verb …args bf`. Piping `bf Bloom.verb …args` also works;
> receiver-first `Bloom.verb bf …args` misbinds (`boru check` rejects it,
> except for `merge`, where it silently merges the other way). Details in
> **[AGENTS.md](AGENTS.md)**.

> **Status (2026-10-01, boru main @ `64c5ab2`):** the library and all five
> suites compile, run green and `boru check` clean. If your program imports
> `boru:test`, import `./bloom.aql` **before** it: an upstream boru defect
> (colliding type IDs) otherwise makes `Bloom.add` / `Bloom.merge` fail with
> `expected BloomFilter, got BloomFilter`. See [`dx-report.md`](dx-report.md) §M1.

> **Forking this to build a new boru library?** This repo is a GitHub
> template — read **[TEMPLATE.md](TEMPLATE.md)** for the instantiation
> checklist, then delete it.

> **Calling this library from an AI coding agent?** Read
> **[AGENTS.md](AGENTS.md)** first — the exact boru calling convention,
> verified idioms, and common mistakes. (Claude Code auto-loads it via
> `CLAUDE.md`; a portable skill lives in
> [`.claude/skills/bloom-filter-aql`](.claude/skills/bloom-filter-aql/SKILL.md).)

## Documentation

The docs follow the [Diátaxis](https://diataxis.fr) framework — four
modes, each serving a different need. Start wherever your need is:

| | Mode | Read this when you want to… |
|--|------|----------------------------|
| 🎓 | **[Tutorial](docs/tutorial.md)** | learn by building your first filter step by step |
| 🔧 | **[How-to guides](docs/how-to.md)** | accomplish a specific task (size, merge, persist, test…) |
| 📖 | **[Reference](docs/reference.md)** | look up exact words, signatures, and return types |
| 💡 | **[Explanation](docs/explanation.md)** | understand how it works and why it's built this way |

New here? Read the [Tutorial](docs/tutorial.md). Already know bloom
filters and just want the API? Jump to the [Reference](docs/reference.md).

## The `Bloom` API at a glance

| Word | Purpose |
|------|---------|
| `Bloom.make {n, p}`      | build a filter sized for capacity `n` at false-positive rate `p` |
| `Bloom.add item bf`      | insert an item (mutates `bf`) |
| `Bloom.contains item bf` | test membership → Boolean |
| `Bloom.count bf`         | estimate distinct items added |
| `Bloom.params bf`        | report `{n, p, m, k}` |
| `Bloom.merge b a`        | union `b` into `a` (matching `(m, k)`; mutates `a`) |
| `Bloom.encode bf`        | serialize to a snapshot string |
| `Bloom.decode text`      | rebuild a filter from a snapshot string |

Full details, including the calling convention (forward args, receiver
last), are in the [Reference](docs/reference.md) and [AGENTS.md](AGENTS.md).

## For AI coding agents

If an agent will call this library, point it at **[AGENTS.md](AGENTS.md)**
— the exact boru calling convention, verified idioms, and the common
mistakes to avoid.

To make that guidance available in *another* project that uses this
library, install the bundled skill either way:

- **Copy the skill** — drop
  [`.claude/skills/bloom-filter-aql/`](.claude/skills/bloom-filter-aql/SKILL.md)
  into that project's `.claude/skills/` (or your `~/.claude/skills/`). It
  loads on demand whenever Bloom calls appear.
- **Install the plugin** — this repo is also a plugin marketplace:

  ```
  /plugin marketplace add voxgig-boru/bloom-filter
  /plugin install bloom-filter-aql@voxgig-boru
  ```

Working inside *this* repo, Claude Code picks the guidance up
automatically via `CLAUDE.md` (which imports `AGENTS.md`) and the bundled
skill.

## Project layout

```
bloom.aql                  the library (the Bloom namespace)
AGENTS.md                  agent guide: how to call this library correctly
test/bloom_unit_test.aql   example-based unit tests — direct (Test.test)
test/bloom_unit_spec.aql   example-based unit tests — declarative spec format
test/bloom_prop_test.aql   property-based tests — direct (Test.check-prop)
test/bloom_prop_spec.aql   property-based tests — declarative spec format
test/bloom_smoke_test.aql  end-to-end smoke run over every public word
docs/                      Diátaxis documentation (above)
dx-report.md               developer-experience notes (last verified: boru main @ 64c5ab2)
test/divergence/run.sh     single-path gate: every suite runs (compiled) and checks clean
proposals/                 language proposals raised from this module's DX
```

Test files follow a consistent naming convention: `_test.aql` for
direct tests (unit or property), `_spec.aql` for declarative specs (unit
or property). The suites import the library as `"../bloom.aql"` — boru
resolves a relative import against the importing file's directory.

## Running it

Build the `boru` binary, then run any script or test — see
[How-to → Install and run](docs/how-to.md#install-and-run-boru) and
[Run the tests](docs/how-to.md#run-the-tests):

```bash
boru test/bloom_unit_test.aql   # unit tests — direct
boru test/bloom_unit_spec.aql   # unit tests — declarative spec format
boru test/bloom_prop_test.aql   # property tests — direct
boru test/bloom_prop_spec.aql   # property tests — declarative spec format
boru test/bloom_smoke_test.aql  # end-to-end smoke run
```

`boru X` compiles the script to bytecode and runs it (the only execution
path since boru 2026-09-19), after a static pre-flight check. To gate every
suite (run + `boru check`, 0 errors) in one go:

```bash
test/divergence/run.sh                              # builds boru @ main HEAD
BORU=$HOME/.local/bin/boru test/divergence/run.sh   # or use an existing binary
```

A GitHub Actions workflow
([`.github/workflows/test.yml`](.github/workflows/test.yml)) builds boru at
the current `main` HEAD (daily, and on each push and pull request) and runs
every suite, the `test/divergence/run.sh` gate, and a `consistency` job
(agent-skill drift and JSON manifests).

## License

See [LICENSE](LICENSE).
