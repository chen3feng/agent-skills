# agent-skills

A personal, living collection of lessons I've taught (or been taught by)
AI coding agents. Each "skill" is a short, self-contained Markdown note
that captures *one* recurring problem and the fix, so it can be fed back
into future agent sessions.

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
- [workspace-path-constraints](skills/workspace-path-constraints/SKILL.md) — which tools only accept indexed workspace paths, and how to fall back for sibling repos.

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
