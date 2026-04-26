---
name: agent-work-artifacts-layout
description: Where to put transient files, one-off scripts, audit reports and PR bodies the agent creates while working.
tags: [workflow, hygiene, gitignore, scripts]
---

# Agent work artifacts: where do they go?

## When to use

Any task where the agent produces files that are **not** the user's
intended deliverable: PR body drafts for `gh pr create --body-file`,
throwaway `repro_*.py` / `check_*.sh` scripts, audit reports,
analysis dumps, temporary fixtures. Without a convention these end
up either polluting the working tree (triggering "do I need to
delete this?" round-trips) or getting lost across sessions even
when they had reuse value.

## Problem

Three recurring failure modes:

1. **PR body files left in the repo root** (e.g. `pr_body.md`),
   forcing the user to confirm deletion after every PR.
2. **One-off scripts committed to random places** — sometimes in
   the repo root, sometimes nowhere — making future contributors
   unsure whether they are project tooling or dead weight.
3. **Fixed temp paths** like `/tmp/pr_body.md` that collide between
   parallel agent sessions.

## Solution

Route every artifact through this four-way decision:

```
1. Single-use (PR body, gh intermediate, shell pipe)?
   → system temp via  mktemp -t <slug>.XXXXXX.<ext>
     (POSIX: $TMPDIR or /tmp; Windows: %TEMP%)

2. Specific to THIS repo AND plausibly useful to future contributors?
   → commit to  scripts/  (or  tools/  if the repo already uses that)
     with argparse CLI, docstring, usage example.

3. General-purpose, useful across projects?
   → offer to publish as a GitHub Gist.

4. Agent working notes you want across sessions but NOT shared?
   → .agent/  (gitignored, local-only)
```

If none apply, **delete the file** rather than leaving it ambiguous.

Key rule: `.agent/` is **not** a substitute for `scripts/`. Anything
in `.agent/` is lost on a fresh clone; if it truly has reuse value,
promote it to `scripts/` or a Gist instead.

## Example

### Right way — PR body in system temp

```bash
body=$(mktemp -t pr_body.XXXXXX.md)
# write body via the editor tool, targeting "$body"
gh pr create --repo <owner>/<repo> --base <default> --head <slug> \
  --title '<title>' --body-file "$body"
```

### Wrong way — PR body in repo root

```bash
# Leaves pr_body.md in the working tree after the PR is filed.
# The user then has to approve its deletion in a follow-up turn.
vi pr_body.md
gh pr create --body-file pr_body.md
```

### Right way — repo-scoped helper that others might reuse

```
<repo>/scripts/check_doc_consistency.py     # committed, with argparse + docstring
```

### Right way — agent-only working area

```
<repo>/.agent/
  tmp/        # repro scripts, scratch Python
  reports/    # audit output, pyflakes dumps
  drafts/     # PR body drafts kept across sessions (rare)

<repo>/.gitignore:
  .agent/
```

## Pitfalls

- **Do not promote `.agent/` to a tracked README feature.** It is
  local convention, not project API. One line in `.gitignore` is
  enough.
- **Do not default to `.agent/drafts/` for PR bodies.** Once filed,
  GitHub is the source of truth. Keep a draft only if the user
  explicitly asked you to iterate on wording locally.
- **Do not rely on `.agent/` for anything needed on another
  machine** — it is gitignored.
- **Windows:** `/tmp` and `mktemp` are not portable. Prefer
  `tempfile.gettempdir()` from Python when the caller might be on
  Windows.
- **Do not confuse `scripts/` with `contrib/`.** In PostgreSQL /
  Git, `contrib/` means "distributed alongside core but not part of
  it". Unless the target repo already uses that convention, stick
  with whichever of `scripts/` or `tools/` the repo already has.
- **Industry precedent:** CPython, Django, Kubernetes, Rust, LLVM
  all ship a top-level `scripts/` or `tools/`. No mainstream project
  formalises an "AI agent working area" yet; `.agent/` is the
  convention this skill proposes, sitting alongside existing local
  caches like `.pytest_cache/`, `.mypy_cache/`, `.idea/`.

## See also

- [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md)
- [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md)
- [workspace-path-constraints](../workspace-path-constraints/SKILL.md)
