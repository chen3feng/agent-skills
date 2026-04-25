---
name: git-commit-author-identity
description: Commit with the right name/email per repo, and survive spaces in `git -c user.name=...`.
tags: [git, shell, identity]
---

# Git commit author identity across repos

## When to use

You're about to create a commit in a repo whose default identity may
not match the user's intent — a freshly cloned sibling repo, a
throwaway scratch clone, or a CI-like environment where `user.name` /
`user.email` are unset. Common trigger: the user says "use the same
identity as <other repo>".

## Problem

Two recurring failures:

1. Committing with the wrong identity (e.g. a work email into a
   personal OSS repo, or `root@hostname` from an unconfigured clone).
2. Trying to override per-command with `git -c user.name='First Last'`
   and having the shell split the quoted value, so git sees
   `user.name=First` and treats `Last` as the next config key:

   ```
   error: key does not contain a section: useAdd
   fatal: unable to parse command-line config
   ```

   Or, if it parses, the committed author ends up as `First_Last` /
   `First` because quoting was stripped somewhere along the pipeline.

## Solution

1. **Read the target identity from an existing repo** the user trusts:

   ```bash
   git -C /path/to/reference-repo config user.name
   git -C /path/to/reference-repo config user.email
   ```

2. **Commit using environment variables**, which survive any shell
   quoting shenanigans:

   ```bash
   GIT_AUTHOR_NAME="First Last" \
   GIT_AUTHOR_EMAIL="me@example.com" \
   GIT_COMMITTER_NAME="First Last" \
   GIT_COMMITTER_EMAIL="me@example.com" \
   git -C /path/to/repo commit -F .commit_msg.tmp
   ```

3. **Verify immediately** before pushing:

   ```bash
   git -C /path/to/repo log -n1 --format='author=%an <%ae>%n committer=%cn <%ce>'
   ```

4. If you got it wrong, fix in place with `--amend --reset-author`
   (only safe if you haven't pushed yet):

   ```bash
   GIT_AUTHOR_NAME="First Last" GIT_AUTHOR_EMAIL="me@example.com" \
   GIT_COMMITTER_NAME="First Last" GIT_COMMITTER_EMAIL="me@example.com" \
   git -C /path/to/repo commit --amend --no-edit --reset-author
   ```

## Example

Wrong (will sometimes give `First_Last` as the author, sometimes
error out with `key does not contain a section`):

```bash
git -c user.name='First Last' -c user.email=me@example.com commit -m "msg"
```

Right:

```bash
GIT_AUTHOR_NAME="First Last" GIT_AUTHOR_EMAIL="me@example.com" \
GIT_COMMITTER_NAME="First Last" GIT_COMMITTER_EMAIL="me@example.com" \
git commit -F .commit_msg.tmp
```

## Pitfalls

- `git -c` *does* work on the shell in a normal terminal, but through
  agent tools the quoting is lossy. Env vars are the safe default.
- `--reset-author` only rewrites the **author** by default; set both
  `GIT_AUTHOR_*` and `GIT_COMMITTER_*` or the committer will keep the
  old identity.
- Once pushed, rewriting author requires a force-push; prefer to get
  it right on the first commit.

## See also

- [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md)
- [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md)
