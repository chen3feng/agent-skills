---
name: repo-org-migration-url-cleanup
description: After a GitHub repo moves to a new owner/org, sweep stale URLs everywhere — but preserve historical narrative.
tags: [git, github, docs, migration, refactor]
---

# Repo-org-migration URL cleanup

## When to use

The user tells you a repo has just been transferred on GitHub — from
a personal account to an organization, from one org to another, or
from Google Code / Bitbucket to GitHub. GitHub's redirect keeps the
old URLs "mostly" working, so it's tempting to do nothing, but the
redirect is unreliable for things like coveralls project-IDs, README
badges fetched by third parties, and anything cached upstream.

Signal phrases: "we moved the repo", "new org", "rename the owner",
"dead links after the move", or a link checker flagging old-URL
badges.

## Problem

Two things go wrong, and they pull in opposite directions:

1. **Missed URLs** — the owner string shows up in a lot more places
   than just the README badge you remember:
   - Coveralls / Codecov project URLs (GitHub's redirect does not
     help the third-party service, which keys by `owner/repo`).
   - Shields.io badges (`img.shields.io/github/downloads/<owner>/<repo>/...`).
   - `git remote -v` example output embedded in code comments and
     docstrings.
   - Issue / PR links in source comments (`# see
     https://github.com/<old-owner>/<repo>/issues/123`).
   - CI config that hardcodes the clone URL (mostly only self-hosted
     setups; GitHub Actions usually references the repo implicitly).
   - `pyproject.toml` / `setup.py` `project-urls`, `homepage`,
     `repository` fields.
   - Contrib-rocks / starchart-cc image URLs keyed by `owner/repo`.
2. **Over-eager rewriting** — it is wrong to blindly `sed` every
   `<old-owner>/<repo>` to `<new-owner>/<repo>`. Some hits are
   **historical narrative**, not live URLs. For example a README
   paragraph saying "the project was migrated from Google Code to
   <old-owner>'s personal repository" describes a real event and
   must not be rewritten to say it was migrated to the new org —
   that would be fabricating history.

## Solution

Do it as a dedicated sweep, separate from any functional change:

1. **Enumerate categories** first (don't just grep blindly). Use
   the list above as a checklist.
2. **Run the grep** with the exact `owner/repo` pattern (not just
   the owner slug, which will match user-names in author fields):

   ```bash
   grep -rn '<old-owner>/<repo>' .
   ```

3. **Classify every hit into one of three buckets**:
   - **(a) Live URL** (badge, link, fetch target, issue link) →
     rewrite to new owner. For GitHub issue links, numbers are
     preserved across ownership transfers, so the path after the
     repo name is safe to keep.
   - **(b) `git remote -v` / clone-command examples** in docs or
     comments → rewrite; the example is meant to match a fresh
     clone *today*.
   - **(c) Historical narrative** ("originally hosted at …",
     "migrated from … to …", author-credit lines, commit-message
     quotations) → **leave alone**. If in doubt, ask the user.
4. **Keep the sweep in its own commit / PR**, separate from any
   behavior change. Subject line like `docs: update stale repo
   URLs after org migration`. Reviewers can then eyeball the diff
   as pure find-and-replace.
5. **Re-run a link checker** after the sweep to catch genuinely
   dead links (old badges that no longer resolve at all, not just
   moved). Dead-but-not-moved links — e.g. a retired Codebeat
   badge — should be **deleted**, not rewritten.
6. **Fix other dead links in the same pass** if they are trivially
   co-located (e.g. `/image/foo.png` links that GitHub's web UI
   resolves from repo root instead of the file's directory →
   switch to a relative path). These are the same kind of "docs
   hygiene" change and belong in the same commit.

## Example

Repo `foo/bar` was transferred to org `bar-project`. Search:

```bash
grep -rn 'foo/bar' .
```

Sample output classified:

```
README.md:17:  [![Coverage](https://coveralls.io/repos/foo/bar/badge.svg?branch=master)]
   → (a) live URL, rewrite to bar-project/bar

README.md:52:  The project was later migrated to foo's personal repository …
   → (c) historical narrative, KEEP

src/util.py:54:  # origin  https://github.com/foo/bar.git (fetch)
   → (b) git remote example, rewrite

src/net.py:711:  # See https://github.com/foo/bar/issues/1034
   → (a) issue link, rewrite (issue numbers preserved by GitHub transfer)

pyproject.toml:12:  repository = "https://github.com/foo/bar"
   → (a) live URL, rewrite

AUTHORS:3:  Jane Doe <jane@foo.example> (originally from foo)
   → (c) author credit, KEEP
```

Commit 1: rewrite all (a) and (b). Commit 2 (optional): in the same
PR, delete retired badges and fix relative-path image links.

## Pitfalls

- **Don't rewrite inside strings that end up in user-visible output
  at runtime** unless you also understand the runtime contract.
  (Rare, but e.g. a migration tool that detects the old URL and
  does a rewrite on the user's behalf — changing its input pattern
  silently breaks that tool.)
- **Don't match just the owner slug** (`grep -rn foo`); it will
  match author emails, sentences that happen to contain the word,
  and file names. Match the full `<owner>/<repo>` token.
- **Coveralls project IDs are not forwarded** by GitHub's transfer
  redirect. The old badge URL may render a stale "unknown" badge
  instead of 404, which is easy to miss.
- **Don't force-push the sweep into an existing unrelated PR.**
  Keep it as its own commit (or PR) so future `git blame` still
  points at a meaningful "drop py2" / "refactor X" commit rather
  than a pile of URL edits.
- **Dead ≠ moved.** Link-checker 4xx on a retired third-party
  service (Codebeat, a defunct Travis host) means *delete the
  badge*, not rewrite it.
- **History files** (`CHANGELOG.md`, release notes, release
  commit messages, tags) are usually historical narrative — leave
  them alone, especially when they quote old PR / issue URLs as
  they existed at the time.

## See also

- [doc-code-consistency-check](../doc-code-consistency-check/SKILL.md)
- [safe-markdown-auto-fix](../safe-markdown-auto-fix/SKILL.md)
- [git-commit-author-identity](../git-commit-author-identity/SKILL.md)
