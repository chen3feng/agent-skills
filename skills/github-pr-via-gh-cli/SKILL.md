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
- **`--body "…\n…"` does NOT give you line breaks.** `gh` takes the
  value verbatim; `\n` stays as a two-character literal and the PR
  body on GitHub will show `\n` in the middle of sentences. Always
  use `--body-file <path>` for anything longer than one line (see
  [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md)).
- **Very long `gh pr create` invocations get backgrounded** by some
  agent shells — the command appears to "hang" and the PR URL never
  comes back. Keep the command short (use `--body-file` instead of a
  giant inline `--body`), and verify with
  `gh pr list --repo <owner>/<repo> --state open` afterwards.
- If you already created the PR and the body is malformed (e.g.
  literal `\n` showing up), fix it without a new PR:

  ```bash
  gh pr edit <N> --repo <owner>/<repo> --body-file pr_body.md
  ```

- If `gh pr create` complains about "no commits between base and head",
  you forgot to push the branch first, or you branched off the wrong
  base — see
  [rebase-on-fresh-base-after-merge](../rebase-on-fresh-base-after-merge/SKILL.md).
- If the target repo has branch protection or required checks, the
  command still succeeds — the PR is created but will sit in a
  pending state. Mention this to the user if relevant.

## See also

- [git-commit-author-identity](../git-commit-author-identity/SKILL.md)
- [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md)
- [rebase-on-fresh-base-after-merge](../rebase-on-fresh-base-after-merge/SKILL.md)
