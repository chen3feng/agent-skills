# Skill format

Every skill is a single Markdown file named `SKILL.md`, living in its
own directory under `skills/`. It starts with YAML frontmatter and then
follows a fixed set of H2 headings, in order.

## Template

```markdown
---
name: <slug>                 # must match the directory name
description: <one line>      # ≤ 120 chars, imperative mood
tags: [<tag1>, <tag2>, ...]  # lowercase, hyphen-separated
---

# <Human-readable title>

## When to use

One short paragraph. What situation triggers this skill? What signal
tells the agent "this applies to me"?

## Problem

The concrete failure mode, ideally with a real symptom (error message,
wrong output, wasted tool calls). Keep it specific.

## Solution

The fix, as a short recipe. Prefer ordered steps or a small code block
over prose.

## Example

One minimal, copy-pasteable example. If the skill has a "wrong way" and
a "right way", show both side by side.

## Pitfalls

Edge cases, things that look like they'd work but don't, and related
skills to cross-reference.

## See also

Bullet list of related skills or external links.
```

## Conventions

- **One problem per skill.** If you find yourself writing "and also…",
  split it into a second skill and cross-link.
- **Show commands, not narratives.** Agents reuse commands; they skim
  prose. Put the command in a fenced block, then annotate.
- **Prefer absolute paths** in examples so the recipe survives being
  copied into an unrelated working directory.
- **Dry-run by default.** When a skill mutates files or history, show
  the dry-run / preview form first, then the `--apply` form.
- **No secrets.** Redact tokens, private URLs, and internal hostnames.
