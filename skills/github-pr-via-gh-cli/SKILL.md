---
name: github-pr-via-gh-cli
description: Standard branch → push → `gh pr create` workflow, including when to fork.
tags: [git, github, gh, pr]
---

# Opening a GitHub PR via `gh`

## When to use

The user says "open a PR" / "send a pull request" and `gh` is
available (`which gh` succeeds). Works equally for first-party repos
and forks.

## Problem

Getting all of these right on the first try:

- Do I need to fork, or can I push a branch directly?
- What's the `base` branch (`main` vs `master`)?
- Body with multiple paragraphs, backticks, and lists — how to pass?

## Solution

1. **Determine push access and the fork question.**

   ```bash
   gh auth status                 # who am I?
   git -C <repo> remote -v        # what does origin point to?
   ```

   - If `origin` is owned by the authenticated user → push directly
     to a branch, no fork needed.
   - Otherwise → fork first: `gh repo fork --remote=true`.

2. **Create a descriptive branch, commit, push.**

   ```bash
   git -C <repo> checkout -b <slug>
   git -C <repo> add <paths>
   git -C <repo> commit -F .commit_msg.tmp    # see related skill
   git -C <repo> push -u origin <slug>
   ```

3. **Check the default branch**, don't assume `main`:

   ```bash
   git -C <repo> symbolic-ref refs/remotes/origin/HEAD | sed 's@^.*/@@'
   ```

4. **Create the PR with a body file** (see related skill):

   ```bash
   gh pr create \
     --repo <owner>/<repo> \
     --base <default-branch> \
     --head <slug> \
     --title "<one-line title>" \
     --body-file pr_body.md
   ```

   `gh` prints the PR URL on success. Capture it.

## Example

```bash
gh pr create --repo chen3feng/cn-doc-style-guide \
  --base master --head add-md-style-tools \
  --title 'Add Python tools for checking and auto-fixing Chinese Markdown docs' \
  --body-file pr_body.md
# → https://github.com/chen3feng/cn-doc-style-guide/pull/1
```

## Pitfalls

- `--title` with backticks or `$` through the shell is still fragile;
  prefer `--body-file` *and* keep the title short enough to put in a
  single-quoted string.
- If `gh pr create` complains about "no commits between base and head",
  you forgot to push the branch first, or you branched off the wrong
  base.
- If the target repo has branch protection or required checks, the
  command still succeeds — the PR is created but will sit in a
  pending state. Mention this to the user if relevant.

## See also

- [git-commit-author-identity](../git-commit-author-identity/SKILL.md)
- [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md)
