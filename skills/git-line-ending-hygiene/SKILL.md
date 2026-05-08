---
name: git-line-ending-hygiene
description: Set up .gitattributes so CRLF/LF mixing can't happen across Windows/Linux/macOS, and detect the mix-up before an agent edit turns a 5-line fix into a 700-line diff.
tags: [git, gitattributes, line-endings, crlf, cross-platform]
---

# Git line-ending hygiene for cross-platform repos

## When to use

A git repo that lives on more than one OS (most commonly: Unreal /
Unity / game-engine codebases edited on Windows + Linux + macOS).
Trigger signals that land you here:

- `git diff --stat` on a one-line edit shows `+720 / -720` on a file
  you barely touched.
- `git show HEAD -- path/to/file.cpp | cat -A` reveals `^M$` at EOL
  while its neighbours have just `$`.
- CI reports "unexpected changes" on a PR that has no real code
  change.
- A `.bat` / `.cmd` script with `if`/`for`/`goto` parses fine on one
  machine and dies on another.
- `git blame` is useless because a "renormalize" commit rewrote
  every line of a file.

This is **not** the "PowerShell heredoc into `wsl.exe bash -c`" CRLF
problem — see [wsl-bash-crlf-or-tempfile](../wsl-bash-crlf-or-tempfile/SKILL.md)
for that. This skill is about repo content crossing the Windows ↔
Unix Git boundary, and the agent-specific trap of *silently
normalizing* a CRLF file to LF during an edit.

## Problem

### 1. `core.autocrlf` is per-clone, not per-repo

Three settings govern line endings: `core.autocrlf`, `core.eol`,
and `.gitattributes`. Only `.gitattributes` is tracked in the repo.
The other two live in each collaborator's git config, so two
developers can check out the *same* commit and end up with *different*
working-tree bytes — then one of them commits the "fix" and the
other sees a giant diff on next pull.

### 2. `* text=auto` without `eol=lf` is not enough

Most "sane default" `.gitattributes` stop at:

```gitattributes
* text=auto
```

This tells Git the storage form is LF, **but the checkout form is
still governed by `core.autocrlf`**. On a Windows clone with
`autocrlf=true` (Git for Windows default), files come out as CRLF in
the working tree; on Linux/macOS they're LF. Editors happily save
whatever they got. Result: the same repo contains a mix.

The Pb4ueRpc symptom (2026-05-08): one `.cpp` file was stored with
CRLF inside the blob because it had been committed from Windows
years earlier; every other `.cpp` in the same tree was LF. The next
time an agent-editor touched that file on macOS the save went out
as LF, producing a 720-line diff for a 5-line real change.

### 3. Agent edit tools are a CRLF amplifier

`edit_file` / `replace_in_file` / `multi_replace` read the file,
apply an edit in memory, and write it back using the host OS's text
conventions. On macOS / Linux that means **LF**, regardless of the
file's previous EOL. There's no `-b` binary-mode flag to opt out.
Consequence: editing any CRLF-stored file from an agent session on a
Unix host silently renormalizes the whole file.

This is bad in three ways:

- The PR diff explodes from "5 lines" to "whole-file rewrite",
  swamping code review.
- `git blame` on the renormalized file attributes every line to the
  agent commit, erasing history.
- The edit may succeed on the target lines *and* simultaneously
  regress any in-flight feature branch that expected the original
  CRLF file to stay CRLF.

### 4. `.bat` / `.cmd` genuinely need CRLF

Not aesthetics — `cmd.exe` multi-line commands (`if`/`for`/`goto`
across lines) **silently misparse** with LF endings. So
"normalize everything to LF" is wrong too; the rule must be
per-extension.

## Solution

### Step 1 — Author a complete `.gitattributes`

Declare text-vs-binary explicitly, pick `eol=lf` globally, exempt
Windows shell scripts, and mark known binary formats. Template:

```gitattributes
# Default: text files stored as LF, checked out as LF everywhere.
# eol=lf is the key that defeats per-clone core.autocrlf.
* text=auto eol=lf

# Windows shell scripts must stay CRLF (cmd.exe parser requirement).
*.bat       text eol=crlf
*.cmd       text eol=crlf
*.ps1       text eol=crlf

# Explicit binary (prevents text=auto from misclassifying files
# that happen to contain no NUL bytes).
*.png       binary
*.jpg       binary
*.jpeg      binary
*.gif       binary
*.ico       binary
*.pdf       binary
*.zip       binary
*.gz        binary
*.7z        binary
*.exe       binary
*.dll       binary
*.dylib     binary
*.so        binary
*.a         binary
*.lib       binary
*.pdb       binary

# Unreal assets
*.uasset    binary
*.umap      binary
*.pak       binary

# Unity assets (if applicable)
*.unity     binary
*.prefab    binary
*.asset     binary
*.meta      text eol=lf
```

Rule of thumb: every `text` line carries `eol=…`. Leaving `eol`
unspecified re-opens the `core.autocrlf` escape hatch.

### Step 2 — Renormalize existing content (one-shot)

After landing the new `.gitattributes`, apply it to history-to-date:

```bash
git add .gitattributes
git commit -m "chore: normalize line endings via .gitattributes"

git add --renormalize .
git status         # every previously-CRLF file shows up as modified
git commit -m "chore: renormalize line endings (one-time CRLF -> LF)"
```

`--renormalize` only rewrites EOLs; no other bytes change. Run it
on a dedicated branch, not mixed with feature work.

### Step 3 — Protect `git blame` with `.git-blame-ignore-revs`

The renormalize commit touches every file. Point `git blame` (and
GitHub's blame UI) at a skip-list:

```bash
# .git-blame-ignore-revs at repo root
<sha-of-renormalize-commit>  # chore: renormalize line endings
```

```bash
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

GitHub auto-reads this file; no extra UI setting required.

### Step 4 — Agent-side pre-edit check

When an agent tool is about to edit a file in a repo that *may*
still contain CRLF residue, inspect EOL **before** the first write:

```bash
file path/to/target.cpp
# => "path/to/target.cpp: C++ source, ASCII text, with CRLF line terminators"
```

or

```bash
head -c 2048 path/to/target.cpp | cat -A | head
# => lines ending in `^M$` = CRLF;  `$` = LF
```

If the file is CRLF but the rest of the repo is LF, choose one of:

1. **Explicit renormalize first** (recommended): convert the file to
   LF, commit that as a separate "chore: normalize EOL" commit, then
   do the real edit on top. Keeps the meaningful diff small.
2. **Preserve CRLF through the edit**: not possible with generic
   agent editors on Unix hosts. Fall back to `perl -i -pe 's/\n/\r\n/'`
   or `unix2dos` *after* the edit to restore CRLF. Brittle; only do
   this if the file must stay CRLF for some external reason.

### Step 5 — Verify

```bash
# find remaining CRLF-stored files in the repo
git grep -I --cached -l $'\r' -- ':!*.bat' ':!*.cmd' ':!*.ps1'
```

Empty output = clean.

## Example

### Wrong: minimal `.gitattributes`

```gitattributes
* text=auto
```

Windows clone with `autocrlf=true` checks out CRLF. First commit from
that clone stores CRLF in the blob. Mac / Linux clones later see
"whole-file modified" after any editor opens it.

### Right: explicit eol + per-extension overrides

```gitattributes
* text=auto eol=lf

*.bat  text eol=crlf
*.cmd  text eol=crlf
*.ps1  text eol=crlf

*.png  binary
*.uasset binary
```

Now every clone, regardless of `core.autocrlf` value, materializes
the same bytes.

### Agent workflow: detect-then-edit

```bash
# Before editing an unfamiliar file:
$ file Plugins/Pb4ueRpc/Source/Pb4ueRpc/Private/TRPC/Pb4ueTrpcClientConnection.cpp
... C++ source, ASCII text, with CRLF line terminators

# Option A — normalize first, separate commit, then edit:
$ perl -i -pe 's/\r\n/\n/g' Plugins/.../Pb4ueTrpcClientConnection.cpp
$ git commit -am "chore: normalize Pb4ueTrpcClientConnection.cpp to LF"

# Now the agent edit produces a clean 5-line diff instead of 720.
```

### Rescue: diff-stat mismatch mid-PR

```bash
# Real change is a few lines, but git diff --stat shows +N/-N ≈ file length.
$ git diff --stat feature/my-branch HEAD -- path/to/file.cpp
 path/to/file.cpp | 1440 ++++++++++++++++++++++++++--------------
 1 file changed, 720 insertions(+), 720 deletions(-)

# Confirm CRLF was silently normalized:
$ git show HEAD:path/to/file.cpp | file -
/dev/stdin: C++ source, ASCII text, with CRLF line terminators
$ cat path/to/file.cpp | file -
/dev/stdin: C++ source, ASCII text

# Recover: convert back to CRLF in the working tree, re-apply the
# real edit, commit only the intended lines.
$ perl -i -pe 's/\n/\r\n/g' path/to/file.cpp
```

## Pitfalls

- **`git checkout -- .` after renormalize won't undo it** if the
  `.gitattributes` already declares `eol=lf`; the checkout re-materializes
  LF. That's the correct behaviour, but surprises people trying to
  "revert" a CRLF→LF accident. The fix is always at the file level,
  not the checkout.
- **Renormalize + feature branch = painful rebase.** Land the
  renormalize commit on `master` first and rebase feature branches
  onto it; trying to cherry-pick feature commits over a mid-branch
  renormalize produces conflicts on nearly every line.
- **Don't mix `core.autocrlf=true` with `eol=lf` attributes.** They
  technically coexist (attributes win), but if a collaborator's
  global config flips to `input` or `true` mid-project, pre-existing
  working trees develop phantom modifications. Document in
  `CONTRIBUTING.md`: "set `git config --local core.autocrlf false`;
  `.gitattributes` controls EOL."
- **`text=auto` still sniffs content.** Files that are actually
  binary but happen to lack `0x00` bytes (some thumbnails, small
  Lua bytecode files) can get misclassified. Adding an explicit
  `binary` rule per extension is cheap insurance.
- **Agent edit tools have no opt-out for LF-on-write on Unix hosts.**
  Don't assume you can "just edit carefully"; the write call itself
  normalizes. The only reliable defence is ensuring the file was
  already LF before the edit, which is why Step 1-2 above are
  mandatory prerequisites.
- **`git grep -l $'\r'` misses CRLF in newly-staged files.** Pass
  `--cached` explicitly (as shown in Step 5) or `-- :(attr:text)` to
  restrict to text files. On a fresh clone, also try
  `git ls-files -z | xargs -0 grep -lP '\r$'`.
- **GitHub's web editor always writes LF.** Edits made via the
  GitHub UI will silently renormalize CRLF files, same trap as
  agent editors. Either avoid the web editor on mixed-EOL repos,
  or fix the EOL first.
- **`.git-blame-ignore-revs` only hides; it doesn't heal.** A
  reviewer doing `git log -p <file>` still sees the renormalize
  commit's whole-file diff. That's why the commit should stand
  alone ("chore: renormalize …"), not be bundled with real changes.

## See also

- [wsl-bash-crlf-or-tempfile](../wsl-bash-crlf-or-tempfile/SKILL.md) —
  sibling CRLF pitfall on the PowerShell → WSL `bash -c` boundary
  (payload CRLF in a single-shell-invocation body, *not* repo
  content).
- [check-git-log-before-refix](../check-git-log-before-refix/SKILL.md) —
  when a large diff looks like a real regression, verify history
  before re-fixing; combined with this skill, catches the
  "renormalized file masquerading as a rewrite" case early.
- Git docs: [`gitattributes(5)` — End-of-line conversion](https://git-scm.com/docs/gitattributes#_end_of_line_conversion)
- Git docs: [`git config` — `core.autocrlf`](https://git-scm.com/docs/git-config#Documentation/git-config.txt-coreautocrlf)
