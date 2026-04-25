---
name: safe-markdown-auto-fix
description: Auto-fix Markdown prose without corrupting code blocks, inline code, URLs, or HTML.
tags: [markdown, regex, tooling]
---

# Safe Markdown auto-fix

## When to use

You're writing (or reviewing) a regex-based fixer for Markdown —
spacing rules, typography, terminology normalization. Applies to any
lint-and-fix tool, including the `cn-doc-style-guide/tools` family.

## Problem

A naive `re.sub` over the whole file will cheerfully mangle:

- Fenced code blocks (``` ``` ``` ```): inserting "helpful" spaces in
  the middle of a Python string breaks the program.
- Inline code spans (`` `foo_bar` ``): adding space around `_` ruins
  identifiers.
- Link URLs (`[text](https://…)`): even *reading* URLs as prose
  produces false positives like "space between CJK and ASCII" inside
  a percent-encoded path.
- HTML tags / comments: `<img alt="你好World">` is prose inside HTML,
  but the tag itself must stay intact.

## Solution

Always tokenize the file into "protected" and "prose" regions before
applying any substitution. A minimal protection list:

```python
PROTECT = [
    # fenced code blocks, greedy across lines
    re.compile(r"```[\s\S]*?```", re.MULTILINE),
    # indented code blocks (4+ leading spaces at start of line)
    re.compile(r"(?:^|\n)(?: {4,}|\t).+", re.MULTILINE),
    # inline code
    re.compile(r"`[^`\n]+`"),
    # link / image URLs — keep the URL, allow fixing the text
    re.compile(r"\]\([^)\n]+\)"),
    # raw HTML tags and comments
    re.compile(r"<!--[\s\S]*?-->"),
    re.compile(r"<[a-zA-Z/][^>]*>"),
]
```

Then:

1. Walk the text, carving out every match into a `placeholder`
   (e.g. `\x00PROTECT_N\x00`) and storing the original.
2. Run your prose-level substitutions on the placeholder-laced text.
3. Restore placeholders.

For a fixer it's also important to:

- **Default to dry-run.** Print a unified diff; require `--apply` to
  write.
- **Be idempotent.** Running the tool twice must not change output.
- **Fail closed.** If in doubt (ambiguous context, mixed-script
  identifiers), leave the text untouched and log a warning rather
  than silently "correcting" it.

## Example

Wrong — direct substitution corrupts code:

```python
# inserts a space between CJK and ASCII everywhere
new = re.sub(r"([\u4e00-\u9fff])([A-Za-z0-9])", r"\1 \2", text)
# => a Python string literal "你好World" inside a ``` block
#    becomes "你好 World", breaking the code
```

Right — protect first, then substitute:

```python
protected, restore = protect(text, PROTECT)
protected = re.sub(r"([\u4e00-\u9fff])([A-Za-z0-9])", r"\1 \2", protected)
new = restore(protected)
```

## Pitfalls

- Fenced code blocks can be indented inside list items; anchor your
  regex to ```` ``` ```` rather than "start of line".
- Backslash-escaped backticks inside inline code (`` \` ``) are rare
  but real — include a unit test.
- Don't touch line endings or trailing whitespace as part of the same
  pass; that's a separate tool (or a separate regex with clear
  opt-in).

## See also

- [chinese-markdown-style](../chinese-markdown-style/SKILL.md)
- [doc-code-consistency-check](../doc-code-consistency-check/SKILL.md)
