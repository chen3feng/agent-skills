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

2. Repo-scoped helper, human-authored, part of project lifecycle?
   → scripts/  (or  tools/  if the repo already uses that)
     with argparse CLI, docstring, usage example.

3. Repo-scoped helper, agent-authored, reviewed and reusable?
   → .agent/tools/  (tracked subdirectory of the agent working area,
     see the four-cell layout below)

4. General-purpose, useful across projects?
   → offer to publish as a GitHub Gist.

5. Agent working notes / drafts / caches NOT meant for sharing?
   → .agent/scratchpad/ | .agent/artifacts/ | .agent/context/
     (gitignored, local-only)
```

If none apply, **delete the file** rather than leaving it ambiguous.

### `.agent/` four-cell layout

`.agent/` is not a single bucket — split it by lifetime and
tracking intent:

| Subdirectory | Purpose | Tracked? |
| --- | --- | --- |
| `.agent/scratchpad/` | Free-form notes, pseudo-code, drafts. | No |
| `.agent/artifacts/` | Generated deliverables (exported docs, images, one-off reports). | No |
| `.agent/tools/` | Reviewed agent-authored helpers for this repo. | **Yes** |
| `.agent/context/` | Project index / knowledge-base caches / embeddings. | No |

The tracked subset is surfaced via a **whitelist-style** `.gitignore`
(see Example). Keeping `.agent/tools/` in Git lets genuinely reusable
agent-authored helpers survive across sessions without mixing with
the project's human-authored `scripts/`.

### `scripts/` vs `.agent/tools/`

They **coexist**, with clear provenance separation:

- `scripts/` (if/when the repo has one) = human-authored core
  tooling that's part of the project's normal lifecycle
  (release, lint, bootstrap).
- `.agent/tools/` = helpers written by an agent, reviewed, and
  kept because they proved useful. Provenance is tagged at the
  directory level — no silent promotion to `scripts/`, only via
  a dedicated "graduation" PR.

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

### Right way — agent working area with whitelist gitignore

```
<repo>/.agent/
  README.md            # tracked: layout convention for contributors
  scratchpad/          # ignored: notes, drafts
  artifacts/           # ignored: generated deliverables
  context/             # ignored: indexes, caches
  tools/
    README.md          # tracked: intake standard
    <helper>.py        # tracked: reviewed agent-authored helpers
    <helper>.sh        # tracked
```

Corresponding `.gitignore` block (whitelist pattern — blanket ignore
first, then un-ignore each tracked path):

```gitignore
/.agent/*
!/.agent/README.md
!/.agent/tools/
/.agent/tools/*
!/.agent/tools/*.py
!/.agent/tools/*.sh
!/.agent/tools/README.md
```

Verify with `git check-ignore -v` after editing — it's the only
way to be sure reopens line up with blanket ignores.

### Intake standard for `.agent/tools/`

Because helpers here are tracked, they need a quality bar. Every
committed script should have:

1. Provenance header naming the agent session.
2. `argparse` CLI with `--help`.
3. Dry-run default; `--apply` / `--write` to mutate.
4. Module-level docstring with a real usage example.
5. Stdlib-only (or deps the project already has).
6. UTF-8 safe (`encoding="utf-8"` on file I/O).
7. Introduced in its own dedicated PR, not smuggled in.

Document this in a `.agent/tools/README.md` at repo setup time.

## Pitfalls

- **`.agent/` is convention, not project API.** A tracked
  `.agent/README.md` explaining the layout is fine (similar in
  spirit to `.github/`'s own README), but nothing in the build,
  tests, or CI should read from `.agent/`. If a file needs to run
  on fresh clones or other contributors' machines, it doesn't
  belong here — promote it to `scripts/` with a dedicated PR.
- **Whitelist gitignore is order-sensitive.** The blanket
  `/.agent/*` must come *before* the `!`-prefixed reopens,
  otherwise the reopens have no effect. Always verify with
  `git check-ignore -v <path>` for both a tracked path and an
  ignored path before committing.
- **Do not default to `.agent/scratchpad/` for PR bodies.** Once filed,
  GitHub is the source of truth. Keep a draft only if the user
  explicitly asked you to iterate on wording locally.
- **Do not rely on `.agent/scratchpad/`, `/artifacts/`, or
  `/context/` for anything needed on another machine** — they are
  gitignored. Only `.agent/tools/` survives cloning.
- **Windows:** `/tmp` and `mktemp` are not portable. Prefer
  `tempfile.gettempdir()` from Python when the caller might be on
  Windows.
- **Do not confuse `scripts/` with `contrib/`.** In PostgreSQL /
  Git, `contrib/` means "distributed alongside core but not part of
  it". Unless the target repo already uses that convention, stick
  with whichever of `scripts/` or `tools/` the repo already has.
- **Do not silently "graduate" `.agent/tools/` scripts to
  `scripts/`.** Promotion is a conscious decision that warrants a
  PR of its own (with a new provenance line: "originally authored
  by agent, audited and graduated"). Silent moves destroy the
  provenance signal the `.agent/tools/` location exists to preserve.
- **Industry precedent:** CPython, Django, Kubernetes, Rust, LLVM
  all ship a top-level `scripts/` or `tools/`. No mainstream project
  formalises an "AI agent working area" yet; `.agent/` is the
  convention this skill proposes, sitting alongside existing local
  caches like `.pytest_cache/`, `.mypy_cache/`, `.idea/`.

## See also

- [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md)
- [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md)
- [workspace-path-constraints](../workspace-path-constraints/SKILL.md)
