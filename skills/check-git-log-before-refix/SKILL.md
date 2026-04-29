---
name: check-git-log-before-refix
description: Before "fixing" a recurring error, check git log to see if it was already fixed upstream — the working tree may just be stale.
tags: [git, workflow, debugging]
---

# Check git log before re-fixing

## When to use

The user reports an error that *looks identical* to something you (or
another agent) recently fixed. Typical signals:

- The same error message you just worked on yesterday.
- The user says "it still fails" after you thought you landed a fix.
- You're on a feature branch that was cut before the fix merged.
- A teammate / another agent has been editing the same repo in
  parallel.

Before you write a single new line of code, stop and check whether the
fix is already on the default branch and you just aren't looking at it.

## Problem

Agents — and humans in a hurry — default to "I see an error, I write a
patch." When the fix already exists on `main`/`master` but you're on a
stale branch or a dirty working tree, that default makes things worse:

- You re-invent a worse version of the existing fix.
- You layer a *wrong* patch on top of a *right* one that's hidden by
  an uncommitted edit or an un-pulled commit.
- You open a PR that either duplicates an already-merged PR or, more
  embarrassingly, conflicts with it.

The concrete failure mode that motivated this skill:

1. PR-A lands a correct fix for error E on `main`.
2. An agent is still checked out on a feature branch cut *before* PR-A.
3. User re-runs the scenario, still sees E (because the feature branch
   predates the fix).
4. Agent "diagnoses" E again, invents a new — and wrong — patch, and
   sometimes even leaves it as uncommitted changes that silently
   revert the real fix on the next merge.

## Solution

Before modifying anything, run this short check:

1. **Check working tree and branch first.**

   ```bash
   git status
   git branch --show-current
   git log --oneline -5
   ```

   If there are uncommitted changes in the file that "still has the
   bug", that is a huge red flag — previous agent output may have
   reverted a real fix locally.

2. **Grep the history for the symptom.** Use a keyword from the error
   message or the affected file:

   ```bash
   git log --oneline --all --grep="AutomationCommandlet"
   git log --oneline --all -- path/to/affected_file
   ```

   If something matches, read that commit: `git show <sha> -- <file>`.

3. **Compare the current branch to the default branch.**

   ```bash
   git fetch origin
   git log --oneline HEAD..origin/main -- path/to/affected_file
   ```

   Any output here means "`main` has commits touching this file that
   your branch doesn't" — the fix is very likely among them.

4. **If the fix is already on `main`:**
   - Discard any local edits that look like a "re-fix":
     `git checkout -- path/to/file`.
   - Either rebase the feature branch on `origin/main`, or just ask
     the user to pull `main` and re-run. Do *not* write a new patch.

5. **Only if the history shows nothing** — then you have a real new
   bug and can start diagnosing.

## Example

Symptom (reported by user):

```
LogInit: Error: AutomationCommandlet looked like a commandlet, but we
could not find the class.
[FAIL] Editor automation tests failed
```

Bad flow:

```text
agent: sees error → edits run_testhost.bat → invents a
       PROCESSOR_ARCHITECTURE guard that doesn't make sense → leaves
       dirty working tree → user re-runs → still broken
```

Good flow:

```bash
$ git status
# On branch feat/old-branch
# Changes not staged for commit:
#   modified:   run_testhost.bat      <-- red flag: previous agent edited it

$ git checkout -- run_testhost.bat    # drop the bogus edit

$ git log --oneline --all --grep="ExecCmds\|AutomationCommandlet"
4f0e923 fix(testhost/Windows): use -ExecCmds instead of -Run=Automation (#20)
a44b370 fix(testhost): run automation via -Run=Automation commandlet ...

$ git show 4f0e923 -- run_testhost.bat
# ...confirms the fix exists on main...

$ git checkout main && git pull
# working tree now has the correct run_testhost.bat
```

Result: zero new code written, problem resolved, user just needed to
pull `main`.

## Pitfalls

- **Don't trust your own memory of "I just fixed this".** The fix may
  have landed on `main` but never been merged back into the feature
  branch you're on.
- **Don't trust `git status` being clean either.** The stale *commit*
  on the branch can still be older than the fix on `main`. Always
  compare against `origin/<default>` after `git fetch`.
- **Watch for previous-agent residue.** If `run_testhost.bat` (or any
  file you're about to touch) shows up as modified in `git status` and
  you didn't edit it, read the diff before doing anything else — a
  previous agent run may have half-reverted a real fix.
- **Don't grep only `main`.** Use `--all` so you also catch fixes on
  release branches, other feature branches, or still-open PRs.
- **A matching commit message isn't proof.** Open the diff
  (`git show <sha>`) and confirm the change actually addresses the
  current symptom — don't stop at the subject line.

## See also

- [rebase-on-fresh-base-after-merge](../rebase-on-fresh-base-after-merge/SKILL.md)
- [doc-code-consistency-check](../doc-code-consistency-check/SKILL.md)
