---
name: git-rev-parse-multi-ref-short
description: Don't combine `git rev-parse --short` with multiple refs; it demands a single revision.
tags: [git, cli, pitfall]
---

# `git rev-parse --short` rejects multiple refs

## When to use

You want to verify that several branches / tags / commits all point
to the *same* object (e.g. right after cutting a release branch and
tag from the same `master` HEAD) and you reach for the one-liner

```bash
git rev-parse --short v2.0.1 2.x v3 master
```

…only to get `fatal: Needed a single revision`. Same if you drop the
shortening length (`--short=7 …`).

## Problem

`git rev-parse --short` (or `--short=<n>`) is documented as an
*abbreviation* flag: it takes **exactly one** object and prints its
shortened hash. Passing more than one ref flips git into a different
code path that aborts with:

```
fatal: Needed a single revision
```

exit status `128`. This is independent of whether the refs resolve to
the same commit — git rejects the invocation *before* resolving.

Without `--short`, the very same argument list works fine:

```console
$ git rev-parse v2.0.1 2.x v3 master
84f89b4e30ebbf44caef67103b546f4990e94c83
84f89b4e30ebbf44caef67103b546f4990e94c83
84f89b4e30ebbf44caef67103b546f4990e94c83
84f89b4e30ebbf44caef67103b546f4990e94c83
```

The trap is that the error message (`Needed a single revision`) reads
like *"you gave me zero"* or *"I couldn't parse one"*, not *"you gave
me too many combined with `--short`"*. Agents often respond by
tweaking the refs rather than dropping the flag.

## Solution

Pick the shape that matches your intent:

| Goal                               | Recipe                                                  |
|------------------------------------|---------------------------------------------------------|
| Multiple refs, full SHA            | `git rev-parse ref1 ref2 ref3`                          |
| Multiple refs, short SHA, one line | `for r in ref1 ref2 ref3; do git rev-parse --short "$r"; done` |
| Multiple refs, short SHA, labelled | `for r in ref1 ref2 ref3; do echo "$r -> $(git rev-parse --short "$r")"; done` |
| Single ref, short SHA              | `git rev-parse --short ref1` (the canonical use)        |

The loop variants also produce clearer output when you want to see
which SHA belongs to which ref — which is usually the real goal when
you reach for this command.

## Example

Verifying a freshly cut LTS branch + release tag + next-major branch
all pivot on the same `master` commit:

```bash
# WRONG — fatal: Needed a single revision
git rev-parse --short v2.0.1 2.x v3 master

# RIGHT — labelled, short SHA each
for r in v2.0.1 2.x v3 master; do
    echo "$r -> $(git rev-parse --short "$r")"
done
# v2.0.1 -> 84f89b4
# 2.x    -> 84f89b4
# v3     -> 84f89b4
# master -> 84f89b4

# Also RIGHT — full SHA, one per line, order matches input
git rev-parse v2.0.1 2.x v3 master
```

## Pitfalls

- `git rev-parse --verify --short ref` still only takes one ref.
  `--verify` doesn't loosen the single-revision constraint.
- `git log --oneline -1 ref1 ref2` is *not* an equivalent workaround:
  `git log` with multiple positional refs means "history reachable
  from any of them", not "show each one".
- Don't fall back to parsing `git show-ref` output to "solve" this —
  it mixes local/remote refs and invites grep-of-output bugs. The
  loop is shorter and safer.
- Tab-completion sometimes auto-adds `--short` because it's a common
  flag; double-check when you're batching refs.

## See also

- [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md)
  — another "the error message lies about the root cause" shell trap.
- `git help rev-parse` — `--short[=<length>]` section.
