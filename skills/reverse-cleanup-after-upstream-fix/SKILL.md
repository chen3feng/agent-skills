---
name: reverse-cleanup-after-upstream-fix
description: When a sidecar or downstream repo papers over an upstream bug, track the workarounds so they can be reversed the moment upstream releases the fix.
tags: [git, workflow, cross-repo, sidecar]
---

# Reverse-clean downstream workarounds after the upstream fix lands

## When to use

You are setting up a **sidecar** or **downstream consumer** of another
repo (integration-test workspace, example repo, plugin host, ...) and
the upstream repo currently has a bug you need to paper over to get
anything running. You plan to file or are about to land a fix upstream.

The skill applies from the moment you first type the workaround, not
just after upstream merges — the bookkeeping you do *now* is what makes
the reversal painless *later*.

Typical triggers:

- Adding a shim script (`tools/python-shim/`, `tools/gcc-wrapper/`, a
  `bin/` directory prepended to `PATH`) to dodge an upstream detection
  bug.
- Exporting an env var (`CC=clang`, `PIP_NO_BUILD_ISOLATION=1`) in a
  wrapper because the upstream default is broken on your platform.
- Pinning an upstream dep to a stale version because the latest has a
  regression you have filed.
- Duplicating an upstream file locally to patch one line.

## Problem

Workarounds in a sidecar repo look harmless when you add them, but
without discipline they turn into permanent cruft:

1. **The fix lands upstream and nobody notices.** Six months later the
   shim is still there, new contributors assume it's load-bearing, and
   removing it becomes a scary multi-PR archeology exercise.
2. **The commit message describes the wrong reality.** You wrote "add
   `CC=clang` to dodge blade toolchain detection" on day 1. Upstream
   merges a real fix on day 3. Now the sidecar's git history tells a
   story that stopped being true — but the code still runs, so nobody
   flags it.
3. **No anchor back to upstream.** You remember filing the upstream
   issue, but the sidecar doesn't cite it. When you later try to
   remove the workaround, you have to re-derive *which* upstream
   commit authorized the removal.
4. **The removal PR is messier than it needs to be.** Without a
   dedicated "drop workarounds now fixed upstream" commit, the cleanup
   gets tangled with unrelated feature work and the reviewer has to
   untangle "what was scaffolding" from "what is new behavior".

Symptom example from a real sidecar:

```
$ grep -rn "# TODO" blade-test/
# (nothing — no anchors)
$ git log --oneline blade.sh
1dbf879 chore: consolidate legacy demo and add v3 toolchain entry
# message mentions CC=clang hack and python-shim but upstream #1093
# already fixed both — the script is now lying about why it exists.
```

## Solution

Four steps, the first two happen when you *add* the workaround:

### 1. Anchor the workaround at introduction time

Put a `TODO` next to every workaround with a specific upstream link.
Use a consistent, greppable token so future-you can find them all:

```bash
# blade.sh — downstream wrapper
#
# TODO(upstream:blade-build/blade-build#1093): drop CC=clang once the
# toolchain-detection fix is released. Until then Apple clang
# mis-detects as gcc and gets -static-libgcc.
export CC=clang
```

```python
# conftest.py
# TODO(upstream:psf/requests#6789): remove this monkey-patch when
# requests>=2.33 is released with the connection-pool fix.
requests.adapters.HTTPAdapter._close_pool = _patched_close_pool
```

Conventions:

- `TODO(upstream:<org>/<repo>#<num>)` is the anchor form. A single
  `grep -rn 'TODO(upstream:' .` must enumerate every tracked workaround.
- The note says **what to remove**, not just **why it exists**.
- Never add a workaround without the anchor; if there's no upstream
  issue/PR yet, file it first.

### 2. Record the workaround in the commit message too

The `TODO` inline anchor is the machine-readable index; the commit
message explains the rationale for reviewers of the sidecar:

```
chore: bootstrap v3 regression workspace

blade.sh: wrapper that forwards to ../blade-build/blade. The initial
version also carries two bootstrap workarounds for issues being fixed
upstream in blade-build#1093 and removed in the next commit:
  * a PATH-prepended python3 shim ...
  * CC=clang / CXX=clang++ on macOS ...
```

### 3. When upstream merges, open a dedicated reverse-cleanup PR

Don't bundle the cleanup with unrelated feature work. One PR, narrow
scope, explicit title:

- Branch: `chore/drop-<N>-workarounds-fixed-upstream`
- Title: `chore: drop v3 bootstrap workarounds now fixed upstream`
- Commit message body: cite the upstream fix by number, list each
  workaround being dropped.

Process:

```bash
# a. Enumerate what's about to be removed
grep -rn 'TODO(upstream:' . \
  | grep -v '^\./\.git/'

# b. For each one, confirm upstream has actually released (merged
#    isn't always enough — a tag or a consumer-visible version may
#    be required).

# c. Clean slate: remove the whole thing, not just comment it out.
git rm tools/python-shim/python3
# edit blade.sh to drop the CC=clang block
# edit blade.sh to drop the python-shim PATH prepend

# d. Re-run the sidecar end-to-end on a clean tree to prove the
#    workaround was only protecting against the now-fixed bug.
/bin/rm -rf build64_release blade-bin
./blade.sh test //suites/cc_basic/...
```

### 4. Commit message must cite the upstream fix by identifier

```
chore: drop v3 bootstrap workarounds now fixed upstream

Upstream blade-build/blade-build#1093 fixed two detection bugs that
this workspace was papering over:

- toolchain.cc_is no longer substring-matches $CC, so
  `export CC=clang` in blade.sh is no longer needed.
- launcher._check_python now honors BLADE_PYTHON_INTERPRETER
  directly, so tools/python-shim/ and the PATH prepend are no
  longer needed.

Both are removed here. Verified with a clean-slate
./blade.sh test //suites/cc_basic/... on macOS 14 / Apple clang /
python 3.12.
```

## Example

Wrong — workaround with no anchor, generic commit message:

```bash
# blade.sh
export CC=clang  # needed on mac
```
```
chore: tweak blade.sh
```

Two months later nobody knows *why* `CC=clang` is there, whether it's
safe to remove, or which upstream bug it corresponds to. The cleanup
becomes an archeology task.

Right — anchored at birth, reversible by grep:

```bash
# blade.sh
# TODO(upstream:blade-build/blade-build#1093): drop once released.
export CC=clang
```
```
chore: bootstrap v3 regression workspace

Adds two workarounds tracked under TODO(upstream:...):
  * CC=clang (blade-build#1093, toolchain detection)
  * PATH-prepended python3 shim (blade-build#1093, _check_python)
Both are anchored in blade.sh and will be reversed as soon as #1093
is released.
```

Cleanup two weeks later is then a one-liner search:

```bash
$ grep -rn 'TODO(upstream:blade-build/blade-build#1093)' .
./blade.sh:6: # TODO(upstream:blade-build/blade-build#1093) ...
./blade.sh:9: # TODO(upstream:blade-build/blade-build#1093) ...
```

…followed by a surgical cleanup PR that the reviewer can approve in
30 seconds because the diff is purely deletions.

## Pitfalls

- **"Merged" ≠ "released".** A workaround against a released library
  (pypi, crates.io, maven) can't be removed until a new version is
  *published* and your dep pin allows it. Against a sibling repo on
  the same machine (like blade-build ↔ blade-test) merged-to-branch
  is usually enough, but double-check which ref the sidecar consumes.
- **Don't batch reverse-cleanups with feature work.** The temptation
  is "I'm touching `blade.sh` anyway, I'll add this new flag too".
  Resist — keep the reverse-cleanup PR pure deletions so it can be
  reviewed on diff shape alone.
- **The `TODO(upstream:...)` grep is only as good as its uniformity.**
  Enforce the exact token in code review of the *introducing* PR, not
  later. A project-wide `grep` that finds 6 of 7 anchors is worse
  than useless, because it gives false confidence that cleanup is
  complete.
- **Don't "soften" the workaround over time.** If the upstream fix is
  delayed, the temptation is to dress up the shim, add fallbacks,
  document it in README as "how it works". Every layer you add is
  something you'll have to peel back. Keep workarounds ugly and
  conspicuous so their removal stays obvious.
- **Reword-or-merge the old commit carefully.** If you later
  `git rebase -i` the introducing commit to tidy its message, keep it
  honest about the tree it actually introduced — the message should
  describe the workaround that *did* exist at that SHA, not pretend
  it wasn't there. The reversal belongs in the follow-up commit, not
  in rewriting history.

## See also

- [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md)
- [rebase-on-fresh-base-after-merge](../rebase-on-fresh-base-after-merge/SKILL.md)
- [git-commit-author-identity](../git-commit-author-identity/SKILL.md)
