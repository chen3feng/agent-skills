---
name: chinese-markdown-style
description: House style for Chinese Markdown docs, and how to enforce it with cn-doc-style-guide/tools.
tags: [markdown, chinese, style, i18n]
---

# Chinese Markdown style

## When to use

Any time you edit, review, or generate Chinese documentation — READMEs,
design docs, tutorials. Especially relevant when the repo has mixed
Chinese + English + code.

## Problem

Chinese Markdown drifts in predictable ways:

- No space between CJK characters and adjacent ASCII letters / digits
  (`使用Python3` instead of `使用 Python 3`).
- Half-width punctuation after a Han character (`你好,世界` instead of
  `你好，世界`).
- English proper nouns lowercased or re-cased (`github`, `Macos`,
  `scons`).
- Unnecessary full-width or double spaces around inline code and links.

These accumulate quietly and make diffs noisy later.

## Solution

Follow [chen3feng/cn-doc-style-guide](https://github.com/chen3feng/cn-doc-style-guide)
and lean on its tooling:

```bash
# 1. Clone the guide as a sibling of your doc repo
git clone https://github.com/chen3feng/cn-doc-style-guide.git

# 2. Check — reports every violation, exits non-zero if any
python3 ../cn-doc-style-guide/tools/check.py path/to/docs

# 3. Preview what an auto-fix would change (dry-run diff)
python3 ../cn-doc-style-guide/tools/preview.py path/to/docs

# 4. Apply the safe subset of fixes
python3 ../cn-doc-style-guide/tools/format.py --apply path/to/docs
```

Core rules you should apply even without the tool:

1. One half-width space between CJK and ASCII (letters, digits),
   **except** around Chinese punctuation.
2. Use full-width punctuation in Chinese prose: `，。：；？！、""''
   （）《》` and `……` for ellipsis.
3. Keep English proper nouns in their canonical casing: GitHub,
   macOS, Python, JSON, Scons.
4. Units follow English convention with a space: `10 MB`, `3.6+`.
5. Code, filenames, commands, paths go in backticks and are not
   translated.

## Example

```text
Wrong:  使用Python3运行blade build,依赖scons.
Right:  使用 Python 3 运行 `blade build`，依赖 Scons。
```

## Pitfalls

- The auto-fixer deliberately leaves some things alone: half-width
  quotes (often URLs / code), half-width periods (version numbers,
  domains), and `C++` next to a Han character. Don't "finish the job"
  by hand without checking — see
  [safe-markdown-auto-fix](../safe-markdown-auto-fix/SKILL.md).
- Never insert spaces *inside* code spans or fenced code blocks, even
  when they contain Chinese characters.
- Don't add spaces on either side of Chinese punctuation — that's
  what the guide explicitly forbids.

## See also

- [safe-markdown-auto-fix](../safe-markdown-auto-fix/SKILL.md)
- [doc-code-consistency-check](../doc-code-consistency-check/SKILL.md)
- Upstream guide: <https://github.com/chen3feng/cn-doc-style-guide>
