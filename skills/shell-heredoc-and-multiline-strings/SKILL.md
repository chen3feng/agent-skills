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
- In zsh, wrapping a heredoc inside a double-quoted command-substitution —
  `gh pr create --body "$(cat <<'EOF' … EOF)"` — hangs at
  `cmdand dquote>` / `cmdand quote>` prompts as soon as the heredoc body
  contains triple-backtick fences, embedded `$(…)`, or backslashes.
  zsh's parser can no longer tell whether quotes are balanced and waits
  forever for input. Common symptom: the `terminal` tool reports
  "command was running, but the user actively interrupted" with a
  transcript full of `cmdand dquote>` lines.
- Quoted multi-line `-m` strings being split at newlines, so only the
  first line becomes the commit title and the rest is interpreted as
  extra git arguments — sometimes parsed as `-c` config keys (see
  `error: key does not contain a section`).
- **Multiple `-m` flags where one flag's value contains embedded
  newlines** (e.g. `git commit -m 'title' -m 'para1' -m 'line1\nline2\nline3'`).
  The agent terminal → shell → git chain re-interprets the real newlines
  inside the quoted argument: the shell sees an unclosed quote and
  drops into a `quote>` / `dquote>` continuation prompt, waiting
  forever for input. Symptom is identical to the heredoc hang: the
  `terminal` tool reports the command as running but the user has to
  interrupt. This failure mode persists even when each individual `-m`
  value looks "safe" in isolation — it only takes one multi-line value
  anywhere in the argv to wedge the parse.

## Solution

**Iron rule: any `git` / `gh` argument whose value contains a real
newline must be passed via `-F <file>` / `--body-file <file>`, never
via `-m` / `--body`.** No exceptions, no "just this once". Even a
single multi-line value anywhere in argv can wedge the shell.

**General rule: write the string to a file first, then point the
tool at the file.** The file can be produced two ways, both reliable:

### Option A — editor tool writes to a temp path

Preferred when the content is long or contains heavy markdown.
Use the agent's `edit_file` / `write_file` targeting `mktemp`'s
output path (system temp, not the repo):

```bash
MSG=$(mktemp -t commit_msg.XXXXXX.txt)
# Then use the agent's editor tool to write to "$MSG".
git commit -F "$MSG" && rm -f "$MSG"
```

### Option B — `printf '%s\n'` line-by-line

Also reliable in an agent terminal because each line is a separate
*argument* (not an embedded newline inside one argument), so the
shell never sees an unclosed quote:

```bash
MSG=$(mktemp -t commit_msg.XXXXXX.txt)
printf '%s\n' \
  'Subject line' \
  '' \
  'First paragraph of the body.' \
  '' \
  '- bullet one' \
  '- bullet two' \
  > "$MSG"
git commit -F "$MSG" && rm -f "$MSG"
```

The critical detail: every line is its own single-quoted argument.
`printf` emits the real newlines on its own, so the shell parser
never has to reason about balance across lines.

### Commit message

```bash
# 1. Write the message to a file in the system temp dir
#    (use editor tool for rich content, or printf '%s\n' … for short msgs).
MSG=$(mktemp -t commit_msg.XXXXXX.txt)
#    … populate "$MSG" via editor tool or printf …

# 2. Reference it by path
git commit -F "$MSG"

# 3. Clean up immediately
rm -f "$MSG"
```

### PR body

```bash
BODY=$(mktemp -t pr_body.XXXXXX.md)
#   agent edit tool -> "$BODY"

gh pr create --title "…" --body-file "$BODY"
rm -f "$BODY"
```

### Any other long string

Use `python3 -c 'open("out.txt","w").write(r"<content>")'` or an
editor tool — never a fragile heredoc through the shell.

## Example

Wrong — heredoc inside command substitution (hangs in `cmdsubst heredoc>`):

```bash
MSG=$(cat <<'EOF'
Subject line

Body with some backticks: `foo` and a quote: it's.
EOF
) && git commit -m "$MSG"
```

Wrong — multiple `-m` where one value is multi-line (hangs in `quote>`):

```bash
git commit -m 'Subject' -m 'Paragraph one' -m '- bullet one
- bullet two
- bullet three'      # ← embedded newlines wedge zsh's parser
```

Right — write to a temp file, commit with `-F`:

```bash
MSG=$(mktemp -t commit_msg.XXXXXX.txt)
printf '%s\n' 'Subject' '' 'Paragraph one' '' '- bullet one' '- bullet two' > "$MSG"
git commit -F "$MSG" && rm -f "$MSG"
```

## Pitfalls

- **Prefer system temp (`mktemp -t ...`) over in-repo temp files.**
  System temp can't accidentally be added by `git add .` and won't
  show up in `git status`. The older advice "put the temp file in
  the repo and gitignore it" still works but is strictly inferior.
- **Always clean up immediately.** `rm -f "$MSG"` on the same line
  as the consumer command (`git commit`, `gh pr create`). Defer it
  and a later session inherits orphaned temp files.
- **Don't try to be clever with `$'…\n…'` or `echo -e`.** ANSI-C
  quoting and `echo -e` don't fix the root cause — the shell still
  sees a multi-line value eventually, and the same hang recurs.
  Only the file-based path is robust.
- **Verify before pushing.** `git log -1 --format=%B | cat` to sanity-
  check the message actually landed with all paragraphs intact
  before you `git push`.

## See also

- [git-commit-author-identity](../git-commit-author-identity/SKILL.md)
- [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md)
- [workspace-path-constraints](../workspace-path-constraints/SKILL.md)
