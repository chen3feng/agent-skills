---
name: python-modernization-sweep
description: Plan a Python 3 modernization sweep (f-strings, super(), type hints) as a series of mechanical PRs, not one mega-PR.
tags: [python, refactor, workflow, ruff, pyupgrade]
---

# Python modernization sweep

## When to use

You own a Python codebase that has just dropped support for an old
interpreter (Py2, or <3.10), and you want to **actually use** the
newer language features that dropping support unlocked: f-strings,
`super()`, PEP 585 generics, `match` statements, type hints, etc.

Signals this skill applies:

- The user says "全面拥抱 Py3 新特性", "modernize this codebase",
  "clean up the Py2 shim residue", or similar.
- The repo still has `super(Cls, self)`, `class X(object):`,
  `'%s' % x`, `u'...'` literals, `IOError`/`EnvironmentError`, etc.
- There is no (or patchy) type-hint coverage, and the user wants
  to start adding hints.

This is **different from [python-code-audit-sweep]** (which finds
latent bugs with pyflakes + grep and splits them into bug /
dead-code / style PRs). A modernization sweep introduces no bug
fixes; every change is either a pure AST-level rewrite or a new
type annotation.

## Problem

The naive approach is one giant "modernize.py" PR that does
everything at once:

- f-string rewrites for every `%` and `.format()` call (often
  hundreds of sites).
- `super()` call modernization.
- Dropping `(object)` base class.
- Adding type hints to every public function.
- Killing `deps=[]` mutable-default args.
- Turning `except IOError` into `except OSError`.

Reviewers cannot tell what is mechanical and what is semantic. The
type-hint portion alone needs slow review; the f-string portion
is 300 identical rewrites that no human should read line by line.
They merge into one blob, one reviewer blocks on the hardest
single line, and the whole modernization stalls for weeks.

A second failure mode: **guessing** what's in the repo. Without a
tool-assisted census, you don't know whether f-string rewrites are
20 sites or 400, whether there are 3 mutable-default anti-patterns
or 167, whether `super()` rewrites are AST-safe or whether
something is calling a sibling class's `__init__` manually. Picking
tools before counting is how you end up writing manual sed scripts
for something `pyupgrade --py310-plus` would do in one command.

## Solution

Three-phase plan: **census → mechanical PRs → semantic PRs**.
Each phase's output tells you what the next phase should do, so
you can stop at any point without leaving the tree in a weird
intermediate state.

### Phase 0 — Census (one command, no commits)

Install ruff and pyupgrade into a throwaway venv (so the repo's
own CI toolchain isn't touched). Run both in **preview mode**:

```bash
# ruff — broad-spectrum static findings, grouped by rule code.
ruff check <src_dir> --select=UP,SIM,PERF,PL,RUF,B,E722 --statistics

# pyupgrade — what a --py310-plus rewrite would change, in a
# throwaway copy so you can diff without touching the real tree.
cp -r <src_dir> /tmp/pyup-preview
pyupgrade --py310-plus /tmp/pyup-preview/*.py
diff -r <src_dir> /tmp/pyup-preview | grep -c '^>'   # changed-line count
diff -u  <src_dir>/<sample_file>.py /tmp/pyup-preview/<sample_file>.py | head -40
```

The ruff `--statistics` table is the modernization inventory. The
rule prefixes each map to a different PR class:

| Ruff prefix | What it finds                                      | Phase |
|-------------|----------------------------------------------------|-------|
| `UP`        | Py2/old-Py3 → modern Py3 rewrites                  | 1 (mechanical) |
| `SIM`       | `if/else` → ternary, `open` without context mgr    | 1 (most are mechanical, some are noisy) |
| `PERF`      | Micro-perf hints                                   | **skip by default** |
| `PL` (PLW/PLR) | Pylint-derived — mixed quality                 | review individually |
| `RUF`       | Ruff's own checks (e.g. RUF005, RUF012)            | 1 for `[*]`-fixable, case-by-case otherwise |
| `B`         | flake8-bugbear — real anti-patterns (B006 etc.)    | 2 (semi-automated) |

The `[*]` marker in ruff's output means "auto-fixable"; that's your
automation boundary. Anything non-`[*]` needs a human.

### Phase 1 — Mechanical PRs (one tool == one PR)

One PR per tool, in this order:

```bash
# PR 1: pyupgrade — UP001..UP038 family.
pyupgrade --py310-plus $(git ls-files 'src/**/*.py')
git commit -am 'chore: pyupgrade --py310-plus'
# Title: "chore(<branch>): pyupgrade --py310-plus <src_dir>"

# PR 2: ruff --fix, only the [*]-safe rules.
ruff check <src_dir> --fix \
    --select=UP,RUF005,SIM103,SIM110,PLR5501,PERF102,RUF015,PLR1714
git commit -am 'chore: ruff --fix safe rules'

# PR 3 (optional): noqa-annotate the intentional-but-flagged sites.
#   e.g. SIM115 on a DSL-exposed open() wrapper,
#        PLW1510 on a subprocess.run that returns returncode.
#   These are *not* bugs; add `# noqa: <CODE>  <reason>` and then
#   enable the rule as a hard error in ruff config.
```

**Why separate the PRs?** Because a `pyupgrade` PR is 100% AST-safe
and reviewable by scrolling, while a ruff-fix PR touches different
kinds of sites and wants spot checking. Bundling them forces every
line into the stricter review.

### Phase 2 — Semi-automated PR (`--unsafe-fixes`)

Some ruff rules have fixes that are correct **in your codebase**
but can't be proven correct in the general case, so ruff tags them
`--unsafe-fixes`. The canonical example is **B006 (mutable default
argument)**:

```python
def cc_library(name, deps=[], srcs=[]):   # 167 sites in blade-build
    ...
```

Ruff's auto-rewrite turns every `deps=[]` into
`deps=None` + `if deps is None: deps = []` inside the body. That
changes the *observable* default from `[]` to `None` if any caller
introspects the signature, which is why it's "unsafe". In a DSL /
build-system / config-schema codebase, no caller ever does that
— every call is `cc_library(name='x', deps=['//y:y'])` — so the
rewrite is safe here, just ruff can't prove it.

Run this as its own dedicated PR so the diff is reviewable in
isolation:

```bash
ruff check <src_dir> --fix --unsafe-fixes --select=B006
git commit -am 'refactor: kill B006 mutable-default arguments'
```

After it lands, enable `B006` as a hard error in `pyproject.toml`
so regressions can't sneak back in.

### Phase 3 — Semantic PRs (type hints, one module per PR)

Type hints are **not** auto-generatable. Tools like `MonkeyType`
and `pytype infer` observe runtime types, which means:

- They need your test suite to exercise every code path (cold
  paths get skipped).
- They annotate with concrete runtime types (`List[str]`) instead
  of intent types (`Iterable[str]`).
- They are wrong about `Optional` most of the time.
- They can't annotate functions that recurse or always raise.

So: add hints by hand, one PR per one-or-two modules, 200–500
lines each. Order by **dependency depth**: leaves first (`util`,
`constants`, `console`), then the modules that import them, so
each PR can see the signatures of what it depends on.

Each type-hint PR should also:

1. Remove the corresponding `# pyright: ignore[...]` pragmas
   in the files it touches.
2. Turn on one more pyright / ruff rule from **warning** to
   **error** in the config file, so the net is strictly
   tightening.

The final PR of the sweep flips the remaining pyright warnings
to errors and promotes any remaining ruff rules from the
`extend-select` list to the enforced list.

## Example

Real numbers from a single-day census of **blade-build v3**
(`src/blade/`, 47 modules, ~225 KB Python):

```
ruff check src/blade --select=UP --statistics
392     UP031  [*] printf-string-formatting     ('%s' % x   → f'{x}')
 43     UP008  [*] super-call-with-parameters   (super(C,s) → super())
 23     UP004  [*] useless-object-inheritance   (class X(object):)
  7     UP024  [*] os-error-alias               (IOError → OSError)
  3     UP032  [*] f-string                     ('{}'.format(x) → f'{x}')
  2     UP009  [*] utf8-encoding-declaration    (coding: utf-8 shebang)
  2     UP015  [*] redundant-open-modes         (open(p, 'r') → open(p))
  1     UP021  [*] replace-universal-newlines   (universal_newlines= → text=)

pyupgrade --py310-plus preview
194 changed lines across 40+ files, all AST-safe.

ruff check src/blade --select=B,SIM,RUF012 --statistics
167     B006   [*] mutable-argument-default     ← one dedicated PR
  6     SIM115     open-without-context         ← 6/6 intentional, noqa
  3     RUF012     mutable-class-default        ← 3/3 are const tables, ClassVar
  1     PLW1510    subprocess.run without check ← intentional, add check=False
```

Plan derived from this census:

- **PR 1** (mechanical): `chore: pyupgrade --py310-plus src/blade`
  — 194 lines, 40 files, reviewable by scrolling.
- **PR 2** (mechanical): `chore: ruff --fix` safe rules + `noqa`
  the 10 intentional sites — ~50 lines.
- **PR 3** (semi-auto): `refactor: kill B006 mutable defaults`
  — 167 sites, dedicated PR with `--unsafe-fixes` justified in
  the PR body.
- **PR 4+**: type hints, one module per PR, starting from
  `util.py` and `config.py`.

Total estimated cost: 3 mechanical PRs (a few hours), then
10–15 semantic PRs (one or two per day, for two weeks) instead
of a single unmergeable "modernize everything" blob.

## When f-strings show up (and when they don't)

A reviewer reading the `pyupgrade --py310-plus` PR will almost
always ask: "why did `'%s' % x` become `'{}'.format(x)` instead of
`f'{x}'`? f-strings are nicer." The answer is that pyupgrade has
**two independent rules** with very different safety guarantees:

| Rule  | Rewrite                                  | Safety |
|-------|------------------------------------------|--------|
| UP031 | `'%s' % x` → `'{}'.format(x)`            | Always safe — same evaluation model. |
| UP032 | `'{}'.format(x)` → `f'{x}'`              | Only safe under strict preconditions. |

UP032 (the f-string promotion) is deliberately conservative and
**skips** any of the following:

1. **Repeated positional / named arguments.**
   `'{0} {0}'.format(expensive())` → `f'{expensive()} {expensive()}'`
   would evaluate `expensive()` twice. pyupgrade never introduces
   extra evaluations, even if your CI happens to pass.
2. **Backslashes inside the replacement field.** Python ≤3.11
   disallows `\n`, `\\`, etc. inside f-string `{...}`.
3. **Nested quotes that would collide.** `"{}".format(d["key"])`
   becomes `f"{d["key"]}"`, which is a syntax error before 3.12.
4. **`#` inside the expression.** f-string expressions cannot
   contain `#` (the parser treats it as a comment).
5. **Complex conversion/format specs with nested `{}`.**
6. **Multi-line `.format(` calls.** The fixer bails rather than
   try to rejoin lines safely.

So pyupgrade's output looking "half-modernized" (`.format` instead
of f-string) is **not a bug or a missing pass**; it is the tool
being honest about the cases it can't prove safe mechanically.
Don't hand-write a follow-up pass that rewrites every remaining
`.format`.

**Policy for this sweep:**

1. **Phase 1 accepts `.format()` as a terminal state.** The PR
   that runs `pyupgrade --py310-plus` stops where pyupgrade stops.
   Don't second-guess it by hand.
2. **Phase 1b (optional, cheap) picks up the safe subset.**
   Add `UP032` to the ruff `--fix` select list if you want to
   harvest the f-string rewrites that *are* safe (ruff and
   pyupgrade implement UP032 with the same preconditions):
   ```bash
   ruff check <src_dir> --fix --select=UP,UP032,RUF005,SIM103,...
   ```
   This is free and reviewable, but will only catch the easy
   `'{}'.format(single_name)` cases — in a real codebase that's
   usually a single-digit count, not hundreds.
3. **Phase 3 picks up the rest incidentally.** Each module's
   type-hint PR naturally re-reads every line in the file; that's
   the right moment to upgrade the remaining `.format` (and any
   lingering `%`) to f-string **in the files that PR already
   touches**. Zero extra cognitive cost, and the change is local
   to a PR the reviewer is already reading carefully.
4. **Do not open a standalone "repo-wide f-string sweep" PR.**
   It would either (a) be mechanical-only and miss the unsafe
   cases, or (b) touch unsafe cases and require line-by-line
   review of hundreds of diffs — defeating the whole phase split.
   Also: a giant f-string commit pollutes `git blame` for every
   line of every touched string, making future archaeology
   harder for zero behavioral benefit.

## Pitfalls

- **Don't skip the census.** Picking tools before counting hits
  is how you hand-write a regex for something pyupgrade does in
  one flag. Ten minutes of `ruff --statistics` up front saves
  hours of shoveling.
- **Don't enable `--select=ALL`.** Ruff has hundreds of rules and
  many are stylistic/opinionated (`ANN`, `D`, `COM`, `ERA`).
  Start with the prefix list in Phase 0 and add rules
  deliberately, one at a time, with a rationale in the PR body.
- **Noisy-by-default rules exist.** `PLW2901` (loop variable
  reassignment) almost always matches `for line in f: line =
  line.rstrip()`, which is idiomatic normalization, not a bug.
  Add it to `lint.ignore` with a comment instead of "fixing"
  22 false positives.
- **`--unsafe-fixes` is a knife, not a button.** Read every
  `--unsafe-fixes` patch before committing. If you can't explain
  in one sentence why ruff flagged it unsafe *and* why it's safe
  here, don't merge.
- **Don't mix phases.** A mechanical PR (Phase 1) must contain
  zero behavior changes and zero type hints. A semantic PR
  (Phase 3) must contain zero mechanical rewrites. If you mix
  them, reviewers have to read the mechanical diff line by line
  anyway, defeating the whole split.
- **Type-hint tools overpromise.** `MonkeyType` / `pytype infer`
  will happily generate annotations; they will also be wrong
  about `Optional`, `Iterable` vs `list`, and anything recursive.
  Use them for inspiration, not as a source of truth.
- **Pin tool versions.** Different versions of `pyupgrade` and
  `ruff` produce subtly different outputs. Pin them in a dev
  requirements file (`pyupgrade==3.17.0`, `ruff==0.6.9`) so
  re-running the sweep months later gives the same answer.
- **Install tools in a throwaway venv.** If the repo's own CI
  doesn't use ruff/pyupgrade yet, don't drag them into
  `requirements.txt` just to run the census. Use `python -m
  venv .agent/venv-modernize` (already gitignored per
  [agent-work-artifacts-layout]) and install there.

## See also

- [python-code-audit-sweep](../python-code-audit-sweep/SKILL.md)
  — the complementary "find latent bugs / dead code / typos"
  sweep. Run it before or after this one, but don't mix them
  into the same PRs.
- [python-indent-aware-edits](../python-indent-aware-edits/SKILL.md)
  — why `compileall` isn't enough verification after a
  mechanical rewrite; actually run the touched code paths.
- [agent-work-artifacts-layout](../agent-work-artifacts-layout/SKILL.md)
  — where to keep the census report (`.agent/scratchpad/...`)
  so it doesn't get committed.
- [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md)
  — standard branch-push-PR recipe for each of the phase PRs.
- [rebase-on-fresh-base-after-merge](../rebase-on-fresh-base-after-merge/SKILL.md)
  — always cut phase-N+1 from freshly-merged `master`, not
  from the phase-N branch.

External references:

- [ruff rules index](https://docs.astral.sh/ruff/rules/)
  — the canonical list of `UP`/`SIM`/`B`/`PL`/`RUF` codes.
- [pyupgrade changelog](https://github.com/asottile/pyupgrade)
  — which UP rewrites each `--pyXY-plus` level adds.
