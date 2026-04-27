---
name: sidecar-smoke-suite-reveals-upstream-bugs
description: A downstream sidecar smoke suite can expose upstream bugs that unit tests miss; fix upstream first, then land the suite.
tags: [testing, cross-repo, sidecar, ci, regression]
---

# Sidecar smoke suites reveal upstream bugs — handshake order matters

## When to use

You maintain an **upstream** library / build tool / framework and a
separate **sidecar** repository that consumes it end-to-end against
real fixture projects (an integration workspace, example repo,
regression harness, plugin host, ...).

You are either:

1. About to add a new suite to the sidecar and it fails on a real
   host in a way that looks like an upstream bug — not a bug in
   the fixture; or
2. Already triaging such a failure and asking yourself "do I work
   around this in the sidecar, or fix upstream?".

Signal phrases: "it works in upstream's own tests but breaks here",
"the unit tests never hit this path", "the failure only shows up
on macOS / on ubuntu-22.04 / without `python` on `PATH`".

This skill is about **what to do with the discovery**, not about how
to paper over it. For the paper-over-then-clean-up flow, see
[reverse-cleanup-after-upstream-fix](../reverse-cleanup-after-upstream-fix/SKILL.md).

## Problem

Unit tests in the upstream repo protect the functions they cover,
but they rarely exercise:

- The **bootstrap shell** a tool emits (a `#!/bin/sh` wrapper, an
  `exec python -m …` launcher) — because the unit test process
  already has `python` / `sh` / the right `PATH`.
- **Environment assumptions** of the deployed artifact — `python`
  vs `python3`, GNU `sed` vs BSD `sed`, `/usr/bin/gcc` being Apple
  Clang.
- **Cross-platform skew** between the CI matrix and real user
  hosts — the CI runner has `python` on `PATH`, macOS 14 does not.
- The **composition** of features — unit tests cover `py_library`
  alone and `py_test` alone; neither catches a launcher bug that
  only manifests when a `py_test` runs a real subprocess that
  itself runs a shell.

A sidecar repo that builds real fixtures with the real tool on a
real host finds these bugs *by existing*. The moment you add a
second language / a second platform / a second Python minor, the
uncovered path light up.

Symptom pattern from a real session (blade-build ↔ blade-test,
2026-04-27):

- blade-build v3 had 11 green CI jobs including a Python unit job
  at 3.10–3.14 and a pyright job — all green.
- Adding a 3-file `py_basic` suite (py_library + py_test, 24 lines
  of Python) to the blade-test sidecar failed on macOS host with
  exit 127 at `greeter_test`'s launcher.
- Root cause: `src/blade/builtin_tools.py` emitted
  `exec python -m …` into the `.pex`-style launcher shell. macOS
  14 has no `python`, only `python3`. No upstream unit test
  covered the emitted shell text.

If you catch this kind of discovery and **just work around it in
the sidecar** (hardcode `BLADE_PYTHON_INTERPRETER=python3` in the
suite's BUILD file, or skip `py_test` on macOS), you lose twice:

1. The upstream bug stays latent and hurts every user.
2. The sidecar accumulates platform-specific knobs it does not
   actually want to own — those knobs become load-bearing.

## Solution

Four steps, in strict order:

### 1. Stop before writing any workaround; reproduce from upstream unit test land

The first question is not "how do I make this green". It is
"**is this reachable as an upstream unit test?**" If yes, the
bug has a permanent home in the upstream suite and every future
consumer is protected; if no, you're about to accept a regression
gap.

```bash
# upstream unit tests should be able to assert on the artifact the
# downstream is tripping over — the emitted shell, the generated
# Makefile, the rendered config file. If they can't, that's a
# coverage gap the fix PR should close.
cd upstream/
grep -rn 'def test_.*launcher\|bootstrap\|shebang' src/tests/
```

If there's no existing unit test for the emitted artifact, **the
upstream fix PR adds one** — that's the permanent plug. Without it
the same class of bug will bite the next sidecar.

### 2. Fix upstream first, land it, then add the sidecar suite

The cross-repo PR order is:

```
upstream fix PR  ── merge ──▶ sidecar suite PR
     (#1096)                       (#2)
```

**Never the reverse**. If you open the sidecar PR first:

- Its CI is red against released upstream, so reviewers can't tell
  whether the suite itself is correct.
- The sidecar PR has to either pin a nonexistent upstream SHA or
  carry a workaround that you'll have to revert in a follow-up —
  both are worse than waiting.

If you *must* work in parallel (e.g. to prove the fix), keep the
sidecar PR in **draft** until the upstream fix is merged and
released (for sibling-repo sidecars, "merged to the consumed ref"
is usually enough — see pitfalls).

### 3. Close the coverage gap upstream with a targeted unit test

The unit test lives in upstream, not in the sidecar. It asserts
on the specific artifact that the downstream tripped over:

```python
# upstream: src/tests/unit/builtin_tools_test.py
def test_py_binary_launcher_uses_python3_not_python(self) -> None:
    pylib = _make_minimal_pylib(...)
    generate_python_binary(out_path, pylib, main='mod.main')
    bootstrap = out_path.read_text()
    # regression guard: the launcher must not hardcode `python`
    self.assertNotIn('exec python -m', bootstrap)
    self.assertIn('${BLADE_PYTHON_INTERPRETER:-python3}', bootstrap)
```

Two properties make this useful:

- It runs in milliseconds as part of every upstream push, not only
  when the sidecar happens to be built.
- It reads as a **behavioral contract** ("the emitted launcher
  honors `BLADE_PYTHON_INTERPRETER`, defaults to `python3`"), not
  a bisected symptom ("exit 127 on macOS").

### 4. Land the sidecar suite as a pure contract test, no knobs

Once upstream is fixed and released, the sidecar suite should be
the *simplest possible* reproduction: no env vars, no platform
`if`s, no comments that explain around the (now-fixed) bug. The
suite's value is that it stays boring forever:

```python
# sidecar/suites/py_basic/BUILD
py_library(name = 'greeter', srcs = 'greeter.py', visibility = 'PUBLIC')
py_test(name = 'greeter_test', srcs = 'greeter_test.py', deps = ':greeter')
```

A sidecar suite that carries `if platform == 'darwin': ...` or
`env = {'FOO_PY': 'python3'}` has been **captured** by the bug it
was supposed to expose, and future regressions in the same
neighborhood will be silently absorbed.

## Example

Real sequence from 2026-04-27, blade-build ↔ blade-test:

1. Sidecar PR opens `py_basic` suite; `blade test //suites/py_basic:greeter_test`
   fails with exit 127 on macOS 14 (no `python` on `PATH`).
2. Investigate: the failure is in the emitted launcher shell, not
   the test. Upstream source: `src/blade/builtin_tools.py` emits
   `exec python -m …` verbatim.
3. Check upstream unit coverage for the emitted launcher: **none**.
   `grep -rn builtin_tools_test src/tests/unit/` → 0 hits.
4. Open upstream PR blade-build#1096:
   - fix: change emitted shell to
     `exec "${BLADE_PYTHON_INTERPRETER:-python3}" -m …`
   - add: `src/tests/unit/builtin_tools_test.py`, 5 tests that
     build a minimal `.pylib` and assert the bootstrap text.
   - +169/-1, all upstream CI green.
5. Merge #1096 to v3. Only then reopen the sidecar PR. Its `BUILD`
   file stays the 20-line default; no `BLADE_PYTHON_INTERPRETER=…`
   escape hatch.
6. Rerun the sidecar e2e job against fresh upstream master; cc
   suite + py suite both green in ~4 s.

Diff shape of each side:

```
upstream  PR #1096:  src/blade/builtin_tools.py          (1 line fix)
                    src/tests/unit/builtin_tools_test.py (5 new tests)
sidecar   PR #2:    suites/py_basic/BUILD, greeter.py, greeter_test.py
                    (no env knobs, no platform conditionals)
```

Six months later a reader opening the sidecar sees a boring
contract suite, not a bug scar.

## Pitfalls

- **"Merged upstream" is not always enough.** For a sidecar
  consuming a released library (pypi, crates.io, maven), you must
  wait for a *release* and update the pin before the sidecar PR
  can go green. For a sibling repo on the same CI job (cloned
  side-by-side like blade-build ↔ blade-test), merged-to-branch is
  usually enough because the sidecar's CI clones the branch fresh.
  Confirm which case you're in before promising an order.
- **Don't let the sidecar grow a "helpful" workaround.** The first
  time a new suite surfaces an upstream bug, the temptation is to
  add `export FOO=bar` to the suite's runner "so the test still
  runs while we wait for upstream". This defeats the purpose — a
  sidecar's value is being a **faithful consumer**, not a
  hardened one.
- **Don't skip the upstream unit test.** The sidecar suite is not
  a substitute — it runs less often (heavier setup, matrix is
  smaller) and its failure mode ("some integration test on some
  host") is much harder to attribute than a 10-ms unit failure.
  Treat the sidecar discovery as a *coverage gap signal*; close
  the gap where it permanently belongs, in upstream.
- **The sidecar suite is not "owned" by the upstream fix PR.** Do
  not bundle them into one PR even if the same person writes both
  — they live in different repos, have different reviewers, and
  the sidecar is easier to revert in isolation if the upstream
  fix turns out to be incomplete.
- **Don't delete the sidecar suite after upstream is fixed.** The
  suite's job is not to reproduce *that* bug; it's to catch the
  *next* regression in the same area. Keep it; it's cheap (a few
  kB of fixture, seconds of CI) and becomes more valuable the
  more upstream changes.
- **Size the sidecar suite for discovery, not completeness.** The
  point is to run the real tool end-to-end against a minimal
  fixture in each supported language/toolchain, not to duplicate
  upstream's test matrix. A 20-line `py_basic` is worth more than
  a 2000-line port of upstream's own tests, because the former
  exercises the composition that upstream unit tests skip.

## See also

- [reverse-cleanup-after-upstream-fix](../reverse-cleanup-after-upstream-fix/SKILL.md) — the companion skill for the case where you *do* add a temporary workaround while waiting for upstream.
- [test-layout-evolution](../test-layout-evolution/SKILL.md) — where unit vs. integration tests live within a single repo.
- [detect-tool-vendor-by-query](../detect-tool-vendor-by-query/SKILL.md) — another class of "works in CI, fails on real host" bug (Apple Clang masquerading as `/usr/bin/gcc`).
- [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md) — the mechanics of opening the two PRs in order.
