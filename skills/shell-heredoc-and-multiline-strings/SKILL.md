---
name: shell-heredoc-and-multiline-strings
description: Pass long/multi-line strings (commit messages, PR bodies) through an agent shell without corruption.
tags: [shell, git, gh, quoting]
---

# Multi-line strings through an agent shell

## When to use

You need to supply a multi-line string to a command — a commit
message, a `gh pr create --body`, a SQL snippet — from an AI agent's
terminal tool.

## Problem

Agent terminals often paste multi-line commands with subtle
whitespace/newline mangling. Observed failures:

- Heredocs that silently truncate at the first embedded `'` or `` ` ``.
- `$(cat <<'EOF' … EOF)` producing a stray `cmdsubst heredoc>` prompt
  the agent never escapes, leaving a process wedged in the background.
- Quoted multi-line `-m` strings being split at newlines, so only the
  first line becomes the commit title and the rest is interpreted as
  extra git arguments — sometimes parsed as `-c` config keys (see
  `error: key does not contain a section`).

## Solution

**Write the string to a file first, then point the tool at the file.**

### Commit message

```bash
# 1. Write the message to a file using a normal editor tool
#    (agent's edit_file / write_file works perfectly here).
cat > .commit_msg.tmp <<PLACEHOLDER
(created via the edit tool, not via the shell)
PLACEHOLDER

# 2. Reference it by path
git commit -F .commit_msg.tmp

# 3. Clean up
rm -f .commit_msg.tmp
```

### PR body

```bash
# Put the body in a file
#   agent edit tool -> pr_body.md

gh pr create --title "…" --body-file pr_body.md
```

### Any other long string

Use `python3 -c 'open("out.txt","w").write(r"<content>")'` or an
editor tool — never a fragile heredoc through the shell.

## Example

Wrong (hangs the agent in `cmdsubst heredoc>`):

```bash
MSG=$(cat <<'EOF'
Subject line

Body with some backticks: `foo` and a quote: it's.
EOF
) && git commit -m "$MSG"
```

Right:

```bash
# Write .commit_msg.tmp via the editor tool, then:
git commit -F .commit_msg.tmp && rm -f .commit_msg.tmp
```

## Pitfalls

- If the editor tool is restricted to indexed workspace paths, write
  the temp file into the *current* repo (e.g. `./.commit_msg.tmp`)
  rather than `/tmp`, and `.gitignore` it or delete it right after.
- Don't forget to delete the temp file — otherwise it shows up in
  `git status` and may sneak into the next `git add .`.

## See also

- [git-commit-author-identity](../git-commit-author-identity/SKILL.md)
- [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md)
- [workspace-path-constraints](../workspace-path-constraints/SKILL.md)
