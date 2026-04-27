---
name: python-code-audit-sweep
description: Run a quick non-behavioral audit of a Python repo and split findings into bug / dead-code / style PRs.
tags: [python, review, refactor, workflow]
---

# Python code-audit sweep

## When to use

The user says something like "check the code for improvements, no
feature changes" or "do a general cleanup pass". You have a Python
codebase, no specific bug report, and need to surface high-signal,
low-risk findings without accidentally changing behavior.

## Problem

A naive "cleanup" PR usually fails review because it mixes three
very different kinds of change into one blob:

- A real latent bug (semantics change when fixed).
- Dead code / unused locals (safe, mechanical).
- Spelling, wording, anti-patterns in comments (pure style).

Reviewers want to merge (2) and (3) fast and scrutinize (1) carefully.
If you stuff them into one PR, the whole thing blocks on the hardest
item, and the diff hides the real bug behind dozens of typo fixes.

## Solution

Do the audit in two phases: **find**, then **partition into separate
PRs**.

### 1. Find (cheap, parallel grep + pyflakes)

```bash
# Static analysis: unused imports/vars, undefined names, simple smells.
pyflakes src/ | tee /tmp/pyflakes.txt

# Common anti-patterns (extend as needed).
grep -rnE 'not [a-zA-Z_][a-zA-Z_0-9.()\[\]]* is None' src/   # should be "is not None"
grep -rnE '== None|!= None' src/                             # should be "is (not) None"
grep -rn  'type(str)\|type(int)\|type(list)\|type(dict)' src/ # type() applied to a builtin type

# Typo shortlist — curated, not exhaustive; avoids false positives.
grep -rniE '\b(seperat|writen|recieve|occured|begining|lenght|tranform|initialis|standardalone|dependancy|mutiple|visibilty|accomodate|seperately)\w*' \
    src/ doc/ tool/

# Filenames with unusual punctuation (real bugs on GitHub's Markdown renderer).
find . -name '*,md' -o -name '*.m d' -o -name '*.mdd'
```

### 2. Partition into three PRs

Label every finding as **A**, **B**, or **C**. One PR per label, in
this order:

| Tag | Contents | Risk | Review burden |
|-----|----------|------|---------------|
| A   | Real bugs: broken error messages, dead-assignment-that-hides-a-branch, wrong file extensions. | Behavior may change (even if tiny). | Highest — maintainer must read carefully. |
| B   | Dead locals, unused imports, tuple-unpacking where half is unused. | None. | Low. |
| C   | Typos in comments/docs, `not x is None` → `x is not None`, file renames with no in-repo references. | None. | Lowest. |

Branch / commit layout:

```bash
git checkout master && git pull --ff-only

git checkout -b fix/real-bugs-<slug>            # PR A
# …apply A…
git commit -m "fix: <describe each bug, one bullet each>"
git push -u origin HEAD

git checkout master
git checkout -b chore/remove-dead-code          # PR B
# …apply B…

git checkout master
git checkout -b chore/style-and-typos           # PR C
# …apply C…
```

Each PR body should list *every* change as a bullet, quote the exact
before/after line, and end with a `diff --stat` block so the reviewer
can eyeball the blast radius in one screen.

## Example

Real symptom found during a blade-build audit:

```python
# src/blade/util.py
raise TypeError('Invalid type %s' % type(str))
```

`str` here is the builtin type object, so every failed call rendered
`Invalid type <class 'type'>` — zero diagnostic value. This is
**category A** (the error message is observably wrong).

Contrast with:

```python
# src/blade/backend.py
jar = self.get_java_command(java_config, 'jar')
args = '%s ${out} ${in}' % jar
# …neither `jar` nor `args` is ever read again in this function.
```

That's **category B** — pyflakes flagged it, removing is mechanical.

And:

```python
# src/blade/backend.py
if hasattr(self.options, 'profile-generate') and \
   not getattr(self.options, 'profile-generate') is None:
```

`not x is None` parses as `not (x is None)` but trips readers (and
pylint C0113). Rewrite to `x is not None`. **Category C.**

Three PRs, merged in order, each with a self-contained changelog.

## Pitfalls

- **Never mix categories.** The moment you add "oh, and also" to a
  style PR, it becomes unreviewable.
- **Read the real file before editing.** Summaries in your context
  may be stale; `read_file` the current content first so your
  `old_string` matches byte-for-byte. See
  [search-replace-old-string-must-match](#) discipline.
- **Verify "safe renames" are safe.** `grep -rn` the old filename
  across the entire repo (code *and* docs *and* CI configs) before
  `git mv`. One surviving reference turns a category-C rename into
  a category-A bug.
- **Typo lists produce false positives.** `seperat*` also matches
  non-English identifiers in third-party vendored code; scope the
  grep to your own source tree.
- **Don't rely on the remote URL shown by `git push`.** If GitHub
  says "repository moved", `gh pr create` may still need the *old*
  `--repo` slug to find your branch. See
  [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md).

## See also

- [python-modernization-sweep](../python-modernization-sweep/SKILL.md) — the complementary sweep that introduces Py3 features (f-strings, `super()`, type hints) as mechanical PRs; run it before or after this audit, but don't mix them into the same PRs.
- [github-pr-via-gh-cli](../github-pr-via-gh-cli/SKILL.md)
- [rebase-on-fresh-base-after-merge](../rebase-on-fresh-base-after-merge/SKILL.md)
- [doc-code-consistency-check](../doc-code-consistency-check/SKILL.md)
- [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md)
