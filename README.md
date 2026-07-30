# mutabench

**Generate pytest suites and score them by mutation testing, not line coverage.**

Coverage tells you a line ran. It does not tell you that anything would notice if
the line were wrong. mutabench introduces one small bug at a time — flips a
comparison, shifts a constant, blanks a `return` — and reports how many slip past
your test suite unnoticed.

```
Suite                                Tests   Line coverage   Mutation score
Weak (calls everything, asserts nothing)  3            100%             0.0%
AST boundary prober (no model)           30            100%            79.5%
Human, boundary-focused                  17            100%            84.6%
```

Three suites, one module, identical coverage, and a 0-to-85 spread in what they
actually catch. That gap is the reason this tool exists.

---

## Contents

- [Why not coverage](#why-not-coverage)
- [Install](#install)
- [Usage](#usage)
- [How the feedback loop works](#how-the-feedback-loop-works)
- [The no-model baseline](#the-no-model-baseline)
- [Using it on your own code](#using-it-on-your-own-code)
- [Dashboard](#dashboard)
- [Design decisions](#design-decisions)
- [Performance](#performance)
- [Project layout](#project-layout)
- [Development](#development)
- [Limitations](#limitations)
- [Roadmap](#roadmap)
- [Licence](#licence)

---

## Why not coverage

A test that calls a function and asserts nothing scores 100% line coverage and
catches zero bugs. That is the first row of the table above, and it is not a
strawman — it is what a suite written to hit a coverage target looks like.

Mutation testing asks the question coverage cannot. For each small change it
makes to your source, it re-runs your tests:

- **Killed** — a test failed. Good: that behaviour is pinned down.
- **Survived** — every test still passed. That bug would ship.

The mutation score is the proportion caught. A survivor is not an abstract
metric; it is a specific missing assertion, on a specific line.

---

## Install

Requires **Python 3.11+**. On macOS, the system `python3` is often 3.9 and will
not work — install a newer one via [python.org](https://www.python.org/downloads/)
or Homebrew (`brew install python@3.12`).

```bash
git clone https://github.com/YOUR_USERNAME/mutabench.git
cd mutabench

python3.12 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

pip install -e ".[dev]"
```

Verify:

```bash
python -m pytest tests/ -q         # expect 47 passed, ~30s
```

Optional, only for the LLM providers:

```bash
pip install -e ".[models]"
```

---

## Usage

### Inspect what would be mutated

```bash
python -m mutabench.cli mutants demo/pricing.py --list
```

```
39 mutants in demo/pricing.py
  constant       19
  compare        12
  return_none     5
  arithmetic      3

  pricing:5:1    line    5  '<' became '<='
  pricing:5:2    line    5  '<' became '>'
  pricing:5:3    line    5  100 became 101
  ...
```

Runs nothing. Useful for sizing a job before committing to it.

### Score an existing suite

```bash
python -m mutabench.cli score --project demo --target pricing.py --tests test_strong.py
```

```
Mutation score   84.6%
  killed         33
  timed out      0     (counted as caught, listed separately)
  survived       6
  uncovered      0     (no test executes the line)
  errored        0     (excluded from the score)
  wall clock     9.6s

By operator (weakest first):
  compare         75.0%   9/12
  constant        84.2%   16/19
  arithmetic     100.0%   3/3
  return_none    100.0%   5/5

Survivors (6), first 6:
  survived  line 17  0.12 became 0
            if rate > 0.12:
  ...
```

This is the most useful command day to day. The survivor list tells you exactly
which assertions are missing.

### Generate a suite

```bash
# No API key required — deterministic AST analysis
python -m mutabench.cli generate --project demo --target pricing.py --provider heuristic

# With a model
export ANTHROPIC_API_KEY=sk-ant-...
python -m mutabench.cli generate --project demo --target pricing.py --provider claude --iterations 4
```

```
Iteration history:
  iter  tests    score  killed  survived  pruned
     1     30    79.5%      31         7       0
     2     30    79.5%      31         7       0

Wrote suite to demo/test_pricing_generated.py
```

Providers: `heuristic` (offline AST prober), `echo` (deliberately bad, for
testing the plumbing), `claude`, `gpt`, or any model name.

### Compare two suites

```bash
python -m mutabench.cli compare --project demo --target pricing.py \
    --human test_strong.py --generated test_pricing_heuristic.py
```

```
suite         tests    score  killed  missed  runtime
human            17    84.6%      33       6    0.22s
generated        30    79.5%      31       8    0.31s

Generated suite is worse than human (-5.1% mutation score).
```

Both numbers always print. Reporting only the wins would make the tool useless
as a measurement instrument.

### Build the dashboard dataset

```bash
python -m mutabench.cli dashboard --project demo --target pricing.py \
  --suites "test_weak.py=Weak (coverage-driven)" \
           "test_pricing_heuristic.py=AST boundary prober" \
           "test_strong.py=Human (boundary-focused)" \
  --out web/results.json --workers 6

python -m http.server --directory web    # http://localhost:8000
```

### Exit codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Real failure — baseline suite broken, no suite scoreable |
| `2` | Usage error — file not found, unparseable module, bad argument |

Suitable for CI: `mutabench score ... || exit 1`.

---

## How the feedback loop works

```
generate → discard tests that fail on correct code → score → collect survivors
         → "these bugs went unnoticed, write tests that catch them" → repeat
         → stop when the score stops moving
```

**Step 2 is the one people skip, and it is load-bearing.** A model writing tests
for unfamiliar code produces some tests that encode its *assumption* about
behaviour rather than the real behaviour. Those fail against the unmutated
module. Keeping them is actively harmful: they become false failures, so every
mutant looks killed and the mutation score climbs toward 100% while the suite is
lying. mutabench deletes them function by function and reports how many it pruned
in the `pruned` column.

**Step 4 is what coverage cannot do.** "Line 42 ran" is not actionable.
"Changing `<=` to `<` on line 42 went undetected" names a missing assertion, and
models are markedly better at writing a test for a described bug than at
inventing edge cases unprompted.

If a later iteration produces tests that break the suite, mutabench keeps the
last working version rather than returning something that does not run.

---

## The no-model baseline

Most work on LLM test generation compares against no tests, or against coverage.
That is a low bar. `--provider heuristic` is a harder one: a few hundred lines of
AST analysis, no model, no API key, no network. It finds every constant compared
against a parameter, probes the boundary values around it, executes the real
module to capture actual outputs, and emits characterization tests.

**It scores 79.5% against the human suite's 84.6%.**

This matters for interpreting any model result. An LLM scoring 80% has not beaten
"no tests" — it has roughly tied a deterministic program that understands
nothing. Reporting only the model number would make a null result look like a win.

The caveat cuts both ways, and is worth stating plainly: characterization tests
pin *current* behaviour, not correct behaviour. If the module has a bug, they
enshrine it, and they will fail when you fix it. They are worthless for finding
existing defects and excellent at detecting change — which is precisely what
mutation score measures. So the baseline is strong on this metric and weak at what
you actually want tests for. A model that merely matches its score may still be
more useful in practice.

---

## Using it on your own code

```bash
cd ~/projects/myapp
python -m mutabench.cli score --project . --target discounts.py --tests test_discounts.py
```

Three constraints to know:

**One file at a time.** `--target` takes a single module, not a package. Loop over
files for a whole codebase.

**The module must be importable by name from `--project`.** Generated tests do
`from discounts import ...`. Flat modules work out of the box; nested packages
with relative imports need `--project` pointed at a directory where the module is
importable, or a `conftest.py` that adjusts `sys.path`.

**Pure functions work far better than side-effecting ones.** The tool cannot mock
your database. A module of calculation logic is the ideal target; a module of
route handlers is not.

### On committing generated tests

**Read them before committing.** The AST prober writes characterization tests —
it executes your code and asserts whatever came out. Genuinely useful before a
refactor ("prove I did not change behaviour"), actively misleading if read as "my
code is correct."

The LLM path reasons about intent, so its tests can catch real bugs — but it also
writes tests encoding wrong assumptions, which is why the loop prunes any test
failing against your unmodified code. That protects the score from being a lie; it
does not guarantee the surviving tests assert *meaningful* things.

Honest positioning: `score` is production-ready. `generate` is a first draft you
review.

---

## Dashboard

The centrepiece is a **kill map** — your source annotated with one marker per
mutant: filled where a test caught it, hollow where it survived, dashed where no
test executed the line. It shows where your suite is blind, line by line. Hovering
a marker names the specific change.

Also shown: a per-operator breakdown (weakest first, so "strong overall but poor
on `compare`" reads as a specific fixable gap) and the full list of undetected
bugs.

`web/results.json` is committed so the dashboard works on a fresh clone without
running anything.

> Note: `compare --json` writes a different, smaller file that the dashboard
> cannot render. Use `dashboard` for the web view.

---

## Design decisions

**No hosted web service.** Running this as a SaaS means executing two kinds of
untrusted code as the core feature — the user's module, and model-generated tests
targeting it. Doing that safely needs gVisor or Firecracker, egress blocking and
resource caps: weeks of infrastructure demonstrating DevOps rather than program
analysis. CLI-first runs code where it is already trusted.

**Mutants are recipes, not file copies.** A mutant is an index into the ordered
list of mutation opportunities. Memory stays flat on large modules, and mutant IDs
stay stable across runs so survivors can be diffed between iterations.

**The tool never writes to your source.** Each worker gets a private copy of the
project tree; mutated modules exist only in temp directories.

**Uncovered mutants count as survivors.** Excluding them gives the flattering
"mutation score of covered code", under which a suite that tests one function
thoroughly and ignores the rest of the module looks excellent.

**Timeouts count as caught but are reported separately.** A mutant that hangs the
suite did change behaviour detectably, but folding it into "killed" hides
infinite-loop mutants that deserve a human look.

**A failing baseline is an error, not a zero.** If the suite fails on unmutated
code, scoring is meaningless. An early version returned an all-zero report that
the loop rendered as "0.0% mutation score" — a measurement bug dressed as a
result. There is now a test for exactly this.

**Errors are messages, not tracebacks.** Missing files, unparseable modules and
bad arguments produce one-line explanations and exit code 2. Inputs are validated
before any expensive work starts, so a typo in the third argument does not waste
a scoring run on the first two.

---

## Performance

Naive mutation testing is O(mutants × suite runtime): 300 mutants against a
2-second suite is ten minutes per iteration, and the loop needs several. Three
optimisations:

1. **Coverage pre-filter.** Mutants on lines no test executes cannot be killed.
   One coverage run marks them without executing anything — often 30-50% of all
   mutants on real modules.
2. **Fail fast.** `-x` stops at the first failure; we only need to know *whether*
   a mutant is caught, not by how many tests.
3. **Parallel workers**, each in its own project copy. Tune with `--workers`.

`--limit N` caps the mutant count for a quick look at a large module.

---

## Project layout

```
src/mutabench/
  mutate.py     AST mutation engine — operators, enumeration, application
  execute.py    Parallel runner, coverage pre-filter, scoring
  generate.py   Prompts, providers, test pruning, the feedback loop
  baseline.py   AST boundary prober (the no-model baseline)
  cli.py        mutants / score / generate / compare / dashboard
web/
  index.html    Dashboard with kill map
  results.json  Committed so the dashboard works on a fresh clone
tests/
  test_mutabench.py   47 tests
demo/
  pricing.py, test_weak.py, test_strong.py
```

### Mutation operators

| Operator | Example |
|---|---|
| `compare` | `<` ↔ `<=`, `==` → `!=`, `in` → `not in`, `is` → `is not` |
| `arithmetic` | `+` ↔ `-`, `*` ↔ `/`, `%` → `*`, shifts, bitwise |
| `boolean` | `and` ↔ `or` |
| `unary` | `not x` → `x` |
| `constant` | `n` → `n+1`, `n` → `0`, `True` ↔ `False`, `"s"` → `""` |
| `return_none` | `return x` → `return None` |
| `control` | `break` ↔ `continue` |

Docstrings are never mutated — changing them produces guaranteed survivors no
test could kill, which would make the score meaningless.

---

## Development

```bash
python -m pytest tests/ -q
python -c "from mutabench import cli; print('imports ok')"   # quick smoke test
ruff check src/ tests/
```

The engine's central invariant is that **enumeration and application agree**: the
mutant labelled `'<' became '<='` must, when applied, produce exactly that change.
An early version violated this because the collector recorded its site before
descending into child nodes while the applier descended first, so every mutant's
description was attached to the wrong edit. Nothing crashed — the tool silently
reported the wrong reason for every survivor, which would have made the feedback
prompts subtly wrong. `test_description_matches_the_actual_edit` guards it.

---

## Limitations

**Equivalent mutants inflate the denominator.** Some mutations produce code that
behaves identically to the original (`x * 1` → `x / 1`), making them impossible to
kill by definition. Obvious cases are flagged at generation time; the general
problem is undecidable. A perfect suite will not score 100%, and the residual is
not all your fault.

**One module at a time.** Cross-module mutants and integration behaviour are out
of scope.

**`ast.unparse` loses formatting.** Mutated modules are only executed, never shown
or written back, so this is acceptable — but the engine cannot produce a readable
patch of a mutation.

**Not validated against `mutmut`.** Operator sets should broadly agree, and
cross-checking is worth doing. Until then, treat absolute scores as this tool's own
metric rather than an industry-comparable number.

**No LLM provider has been run end to end.** Every number in this README comes
from the AST prober and hand-written suites. The model path is implemented and the
offline paths are tested, but there is no claim here about LLM test generation
quality yet.

**Timeout handling is coarse.** The per-mutant timeout is a multiple of baseline
suite runtime. A suite with high variance may see flaky timeout classifications.

---

## Roadmap

- [ ] Run a real model end to end and report the iteration curve
- [ ] Cross-validate the engine against `mutmut` on the same modules
- [ ] Evaluate across 10–15 real PyPI packages with existing test suites,
      reporting losses as prominently as wins
- [ ] Per-test coverage contexts, so each mutant runs only the tests touching its
      line rather than the whole suite
- [ ] Package-level targets rather than single modules

---

## Licence

MIT
