---
name: test-layout-evolution
description: When adding unit tests to a repo whose `test/` dir is actually integration, create a parallel `tests/unit/` instead of mixing them.
tags: [testing, project-layout, refactor, python]
---

# Test layout evolution: unit tests vs. legacy `test/`

## When to use

You are about to add the *first* real unit tests to a repository
that already has a directory called `test/` (or `tests/`), but on
inspection that directory is **not** unit tests — it drives the
real binary end-to-end against fixture projects, boots a server,
shells out, or otherwise takes seconds per case.

Signal phrases: "there's already a `test/` folder, I'll add mine
in there", "let's just add a `test_xxx.py` next to the existing
integration script", or a PR diff where a 10-ms mocked test
suddenly depends on `testdata/`, `PATH`, or network.

## Problem

Conflating the two kinds of tests causes recurring pain:

- **Speed and feedback loop.** Integration suites are slow (real
  subprocess, real filesystem, real network) and can't run on
  every save. Unit suites are ~10ms and should run on every
  save. Put them in one directory and CI either pays the
  integration cost every time or never runs the units.
- **Import / discovery conflicts.** The legacy `test/` is often
  not structured as a Python package at all (it's shell scripts
  + a driver). Dropping `test_*.py` next to `run_tests.sh`
  confuses pytest's collection and custom runners.
- **Fixtures mismatch.** Integration fixtures tend to live in
  `testdata/` with real files; unit tests prefer in-memory
  mocks. Mixing them invites a unit test to accidentally open
  a fixture file and become a slow integration test with a
  unit test's name.
- **Dependency creep.** Unit tests mock `run_command`, network,
  clocks. Integration tests need those for real. One
  `conftest.py` trying to serve both grows monkey-patches that
  silently break the integration side.
- **"Just refactor the old tests"** is a trap. The integration
  suite is usually load-bearing for release gating and has
  accumulated non-obvious dependencies (env vars, working-dir
  assumptions, testdata). Moving it is a multi-PR project that
  should not block shipping the first unit test.

## Solution

Introduce a **parallel directory**, leave the legacy suite alone,
defer the merge:

1. Pick a layout that makes the two roles obvious and can later
   be unified without another rename. Common choices:

   ```
   src/
     test/            ← legacy, integration, runs real binary
     tests/unit/      ← new, pure unit, mocked
   ```

   or, if you prefer the final shape now:

   ```
   src/test/unit/          ← new
   src/test/integration/   ← move legacy here later
   ```

   Prefer the first form if the legacy suite is large / risky to
   touch; prefer the second if the legacy suite is small enough
   to move in the same PR.

2. **Document the split** in one line in the repo's top-level
   README or `CONTRIBUTING.md`: "`src/test/` = integration, runs
   real <tool>. `src/tests/unit/` = unit, mocked, <10ms."

3. **Give each suite its own runner.** A `runall.sh` (or a
   pytest config with a marker) per directory keeps local
   iteration fast. In CI, run them as separate jobs so a broken
   integration test doesn't block a unit-only PR and vice versa.

4. **Ban cross-imports.** Unit tests must not import from the
   integration suite, and integration tests must not import
   from unit helpers. If a helper is useful to both, promote it
   to the product code (or a shared `tests/common/`).

5. **Record the future merge as a TODO**, not a blocker. A one-
   line note ("will fold into `src/test/{unit,integration}/`
   once the legacy driver is ported") is enough. Do it in a
   separate PR when you have cycles.

## Example

Real case from blade-build (v3, PR #1093):

- `src/test/` had long existed: a bash-driven suite that runs
  the real `blade` against `src/test/testdata/` subprojects and
  diffs the output. Slow, load-bearing, not a Python package.
- The v3 bugfix PR needed to add the first mocked unit tests
  for `ToolChain.cc_is` (6 tests, all mock `run_command`,
  milliseconds total).
- Instead of dropping `toolchain_test.py` into `src/test/` (and
  fighting the existing driver), the PR created:

  ```
  src/tests/unit/
    toolchain_test.py    # mocks run_command, pure unittest
    runall.sh            # `python -m unittest discover -s .`
  ```

- Legacy `src/test/` was untouched.
- A note in the PR body and the commit message spelled out that
  a future refactor may collapse both into
  `src/test/{unit,integration}/`; the CI wiring for the new
  directory was **deferred to a follow-up PR** (pyright + unit
  job together), because that PR was already doing enough.

Net result: the new suite ran and guarded the bug fix without
destabilizing the existing gate.

## Pitfalls

- **Don't silently shadow.** `tests/` vs. `test/` at the same
  level is fine on Unix but can look like a typo to new
  contributors. Call it out in the README or rename one side
  (e.g. `integration_test/`) to make the split unmissable.
- **Don't rely on pytest's rootdir auto-magic** to keep the two
  suites separate. If both directories end up collected,
  integration tests will run in your "fast" loop. Either set
  `testpaths` in the unit suite's config or give the suites
  distinct markers.
- **Don't promise the future merge in the same PR.** Adding the
  unit directory and moving/renaming the integration suite in
  one PR produces a diff that's impossible to review: the
  reviewer can't tell whether a failing integration test broke
  because of the move or because of the new code.
- **Don't let "unit" tests reach for `testdata/`**. The
  temptation is real ("I already have a fixture here…"). As soon
  as they do, they're integration tests misfiled, and your
  "<10ms" promise is gone.
- **CI wiring is a separate concern.** Adding the directory in
  one PR and adding the CI job in the next is not only OK, it's
  often *better* — it keeps the bug-fix PR small and lets the
  CI-plumbing PR focus on caching, matrix, and runtime
  tradeoffs without also arguing about test content.

## See also

- [detect-tool-vendor-by-query](../detect-tool-vendor-by-query/SKILL.md)
- [agent-work-artifacts-layout](../agent-work-artifacts-layout/SKILL.md)
- [python-code-audit-sweep](../python-code-audit-sweep/SKILL.md)
