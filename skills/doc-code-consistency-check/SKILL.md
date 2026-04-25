---
name: doc-code-consistency-check
description: Before "fixing" a README, verify the actual code behavior — don't trust either in isolation.
tags: [docs, review, workflow]
---

# Doc–code consistency check

## When to use

The user asks you to review or tidy up documentation (README,
tutorials, help text) for a project whose code you can also read.
Especially when they say "the docs look wrong" or "check whether the
docs match the code".

## Problem

Docs drift. Common observed drifts in real repos:

- A CLI flag was renamed in code but the README still shows the old
  one.
- A function described as "returns X" actually returns X-or-None.
- A "supported versions" list that nobody updated after a bump.
- Chinese and English versions of the same doc disagreeing with each
  other (and both possibly disagreeing with the code).

If you edit the docs to be self-consistent *without* re-reading the
code, you often cement the wrong behavior.

## Solution

Before changing a single word of a doc, do this loop:

1. **List the concrete claims** the doc makes — flags, options,
   return types, default values, error behavior, supported platforms.
2. **Grep the code** for each claim. Cross-reference against tests
   where possible.
3. **Categorize each discrepancy**:
   - (a) Doc wrong, code right → update the doc (this is most
     common).
   - (b) Code wrong, doc right → ask the user; do not silently
     "fix" the doc.
   - (c) Both wrong → surface it; don't guess.
4. **For multilingual docs** (`README.md` + `README-zh.md`), diff
   them after step 3 so both stay in sync.
5. **Write a short changelog** of what you changed in the doc and
   why, based on the code evidence. This lets the user audit quickly.

## Example

Symptom: README says `_check_python` accepts a version string like
`"3.11"`.

Bad fix: silently change the README to `"3.11.0"` because that's what
"looks right".

Good fix:

```bash
grep -n "_check_python" -r src/
# read the function, note it calls shutil.which("python3") and parses
# sys.version_info, no version string is accepted
```

Then update the README to describe *what the code actually does*, and
mention the mismatch in the PR description so the maintainer can
decide if the code should change instead.

## Pitfalls

- Don't fix both files in the same PR without flagging it. Separate
  "doc matches code" from "change behavior" into different commits /
  PRs.
- Tests are a better source of truth than the code's docstring.
- Translated docs often lag the primary one; when in doubt, treat the
  language the maintainer writes most often as source-of-truth.

## See also

- [chinese-markdown-style](../chinese-markdown-style/SKILL.md)
- [safe-markdown-auto-fix](../safe-markdown-auto-fix/SKILL.md)
