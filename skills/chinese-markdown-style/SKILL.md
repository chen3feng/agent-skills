---
name: chinese-markdown-style
description: House style for Chinese Markdown docs, and how to enforce it with the cndocstyle package from cn-doc-style-guide.
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
and lean on its tooling. The checker / previewer / formatter now ship as a
stdlib-only Python package `cndocstyle` (src-layout under `src/`), so invoke
them with `python3 -m cndocstyle.<module>`:

```bash
# 1. Clone the guide as a sibling of your doc repo
git clone https://github.com/chen3feng/cn-doc-style-guide.git

# 2. Make the package importable (src-layout). Pick ONE:
export PYTHONPATH=../cn-doc-style-guide/src      # for this shell, or
cd ../cn-doc-style-guide && pip install -e .     # editable install

# 3. Check — reports every violation, exits non-zero if any
python3 -m cndocstyle.check path/to/docs

# 4. Preview what an auto-fix would change (dry-run diff)
python3 -m cndocstyle.preview path/to/docs

# 5. Dry-run the formatter (default) — lists files that would change
python3 -m cndocstyle.formatter path/to/docs

# 6. Apply the safe subset of fixes
python3 -m cndocstyle.formatter --apply path/to/docs
```

> The old `tools/check.py` / `tools/preview.py` / `tools/format.py` scripts
> have been removed — use the module form above.

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
