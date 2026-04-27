# agent-skills

A personal, living collection of lessons I've taught (or been taught by)
AI coding agents. Each "skill" is a short, self-contained Markdown note
that captures *one* recurring problem and the fix, so it can be fed back
into future agent sessions.

## Usage

### For human users

Just cooy the follow description to your agent:

```
This is a vibe coding skills library that documents various common problems to help you avoid pitfalls.

https://github.com/chen3feng/agent-skills

Usage: Clone it to the same directory as your current project.
```

Welcome to fork this repository and accumulate your own coding skills.

## For Agents

If agents encounter common insights not unique to their current project while solving problems, they can add them using the format provided in this repository.
Note that this repository may be modified by other agents, so you should pull the latest code and submit a pull request (PR) for each modification.

## Repository layout

```
agent-skills/
├── README.md         # this file
├── SKILL_FORMAT.md   # the authoring conventions every skill follows
└── skills/
    └── <skill-slug>/
        └── SKILL.md  # required; plus optional examples/, assets/
```

Every skill lives in its own directory under `skills/`. The directory
name is the skill's slug (lowercase, hyphen-separated). The main file
is always `SKILL.md`. See [SKILL_FORMAT.md](SKILL_FORMAT.md) for the
exact template.

## Index

General engineering workflow:

- [git-commit-author-identity](skills/git-commit-author-identity/SKILL.md) — set the right `user.name` / `user.email` per repo, and avoid the `git -c user.name='First Last'` space-splitting trap.
- [shell-heredoc-and-multiline-strings](skills/shell-heredoc-and-multiline-strings/SKILL.md) — passing multi-line commit messages and long strings through an agent terminal without getting eaten by the shell.
- [github-pr-via-gh-cli](skills/github-pr-via-gh-cli/SKILL.md) — standard "branch → push → `gh pr create`" recipe, including when fork vs. direct branch is appropriate.
- [rebase-on-fresh-base-after-merge](skills/rebase-on-fresh-base-after-merge/SKILL.md) — after a PR lands, cut the next branch from a freshly-fetched `origin/<default>` instead of reusing the merged feature branch.
- [git-rev-parse-multi-ref-short](skills/git-rev-parse-multi-ref-short/SKILL.md) — `git rev-parse --short ref1 ref2 ...` fails with "Needed a single revision"; use a `for` loop or drop `--short`.
- [workspace-path-constraints](skills/workspace-path-constraints/SKILL.md) — which tools only accept indexed workspace paths, and how to fall back for sibling repos.
- [detect-tool-vendor-by-query](skills/detect-tool-vendor-by-query/SKILL.md) — identify a compiler / interpreter by running it (`--version`) and caching the answer, not by sniffing its filename — macOS `/usr/bin/gcc` is Apple Clang.
- [python-code-audit-sweep](skills/python-code-audit-sweep/SKILL.md) — run a quick non-behavioral audit of a Python repo and split findings into bug / dead-code / style PRs.
- [python-indent-aware-edits](skills/python-indent-aware-edits/SKILL.md) — over-indented suites are still legal Python; `compileall` won't catch a dropped `with` / `try` scope, so verify by actually calling the edited function.
- [test-layout-evolution](skills/test-layout-evolution/SKILL.md) — when the existing `test/` dir is really integration, add unit tests in a parallel `tests/unit/` instead of mixing them; defer the merge.
- [sidecar-smoke-suite-reveals-upstream-bugs](skills/sidecar-smoke-suite-reveals-upstream-bugs/SKILL.md) — a downstream sidecar smoke suite can expose upstream bugs unit tests miss; fix upstream first and close the coverage gap there, then land the sidecar suite with no knobs.
- [agent-work-artifacts-layout](skills/agent-work-artifacts-layout/SKILL.md) — where to put PR bodies, throwaway scripts, and audit reports so they don't pollute the repo or get lost.
- [repo-org-migration-url-cleanup](skills/repo-org-migration-url-cleanup/SKILL.md) — after a GitHub repo is transferred to a new owner/org, sweep stale URLs everywhere but preserve historical narrative.
- [stop-chasing-the-optimizer-reduce-instead](skills/stop-chasing-the-optimizer-reduce-instead/SKILL.md) — after two failed anti-optimization patches, stop adding `volatile` / `noinline` and reduce the repro instead.
- [per-function-optimize-attribute-abi-mismatch](skills/per-function-optimize-attribute-abi-mismatch/SKILL.md) — `__attribute__((optimize("O0")))` on one function inside an `-O2` TU on GCC silently breaks the ABI and crashes on first call.

Documentation work:

- [chinese-markdown-style](skills/chinese-markdown-style/SKILL.md) — the Chinese-doc style rules I follow, plus how to run the `cndocstyle` package from `cn-doc-style-guide` to enforce them.
- [safe-markdown-auto-fix](skills/safe-markdown-auto-fix/SKILL.md) — how to auto-fix Markdown without corrupting code blocks, inline code, URLs, or HTML.
- [doc-code-consistency-check](skills/doc-code-consistency-check/SKILL.md) — before editing a README, verify the actual code behavior; don't just "correct" the doc in isolation.

## Adding a new skill

1. Pick a short slug, e.g. `skills/awesome-thing/`.
2. Copy the frontmatter + headings from [SKILL_FORMAT.md](SKILL_FORMAT.md)
   into a new `SKILL.md`.
3. Keep it *small* — one problem, one fix, one or two minimal examples.
4. Link it from the relevant section of this README.
