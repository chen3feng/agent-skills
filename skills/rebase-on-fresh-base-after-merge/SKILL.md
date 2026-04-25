---
name: rebase-on-fresh-base-after-merge
description: After a PR is merged, cut the next branch from fresh origin/master — don't keep committing on the old feature branch.
tags: [git, github, workflow]
---

# Start the next change from a fresh base

## When to use

You just landed a PR (or the user says "I merged it"), and you're
about to make the *next* change in the same repo. The working tree
is still on the feature branch that was merged, or on a stale local
`master` / `main`.

## Problem

Three related failure modes:

1. **Branching off a stale base.** Local `master` hasn't been fetched
   since the merge, so a new branch cut from it is missing the commits
   that were just merged upstream. The next PR will show a confusing
   diff (including files you didn't touch) or fail to merge cleanly.
2. **Reusing the old feature branch.** Appending new commits to the
   already-merged branch and pushing again creates a "PR #2 on the
   same branch" situation — GitHub may refuse to open it, or will
   show the old merged commits in the diff.
3. **Uncommitted edits trapped on the wrong branch.** You already
   made edits for the next task while still on the old branch; a
   naive `git checkout master` either fails or silently drags the
   edits along to a base that doesn't match them.

Symptom example:

```
$ git log --oneline origin/master..HEAD
fac933f Seed agent-skills with format spec and initial 7 skills   # already merged!
```

## Solution

Standard recipe, safe with or without uncommitted edits:

```bash
# 0. (If you have unstaged edits for the next task) stash them
git -C <repo> stash push -u -m "wip-<next-task>"

# 1. Sync local refs with the remote
git -C <repo> fetch origin --prune

# 2. Find the real default branch — don't assume main or master
default=$(git -C <repo> symbolic-ref refs/remotes/origin/HEAD | sed 's@^.*/@@')

# 3. Fast-forward local default to origin
git -C <repo> checkout "$default"
git -C <repo> pull --ff-only origin "$default"

# 4. Cut the new branch from the fresh base
git -C <repo> checkout -b <new-slug>

# 5. Reapply the stashed edits (if step 0 happened)
git -C <repo> stash pop

# 6. Verify you're actually ahead of origin/$default by 0 commits
git -C <repo> log --oneline "origin/$default..HEAD"   # should be empty
```

Then commit, push, open the PR as usual
([github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md)).

## Example

Wrong — reusing the merged branch:

```bash
# still on seed-initial-skills, which was merged in PR #1
git add README.md
git commit -m "tweak"
git push                 # pushes to the old, already-merged branch
gh pr create ...         # "No commits between master and seed-initial-skills"
```

Right — fresh branch off updated master:

```bash
git stash push -u -m "wip-update-readme"
git fetch origin --prune
git checkout master
git pull --ff-only origin master
git checkout -b update-readme
git stash pop
# edit / add / commit / push / gh pr create
```

## Pitfalls

- `git pull` without `--ff-only` can create an unexpected merge
  commit on local `master` if you ever committed there directly.
  Always use `--ff-only`.
- `git checkout master` without stashing first will fail when the
  edits conflict with tracked files on `master`; the stash dance
  above avoids the ambiguity.
- If the PR was **squashed** on merge, your local feature branch's
  commits won't appear in `origin/master` by hash — that's expected,
  don't try to "rescue" them.
- Delete the merged local branch once the new one is working:
  `git branch -D <old-slug>` (the remote copy is usually auto-deleted
  by GitHub on merge).

## See also

- [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md)
- [git-commit-author-identity](../git-commit-author-identity/SKILL.md)
