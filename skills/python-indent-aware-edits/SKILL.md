---
name: python-indent-aware-edits
description: Over-indented suites are valid Python; compileall won't catch a dropped with/try scope. Verify end-to-end.
tags: [python, refactor, edit-safety, review]
---

# Python indent-aware edits: compileall won't catch a dropped `with` / `try` scope

## When to use

You are making a multi-line edit **inside** a block-introducing
statement (`with`, `try`, `for`, `if`, `def`, `class`) using an
editor tool that does literal search-and-replace (e.g. `replace_in_file`,
`multi_replace`, `sed -i` with a multi-line pattern). The edit rewrites
or replaces a nested suite — not just a single token.

If the edit is purely **mechanical in-line rename** (e.g. `iteritems(d)`
→ `d.items()`, `basestring` → `str`) done by `sed` that never touches
the leading whitespace of any line, this skill does not apply.

## Problem

Python's grammar treats **any** indentation greater than the enclosing
block as a valid new suite. That means an accidentally over-indented
line is still syntactically legal — `compileall`, `ast.parse`,
`python -c 'import mymodule'` all pass — but the **semantic scope**
has silently changed.

Concretely: an `old_string` that captured only the body lines of a
`with` block (not the `with` statement itself) can be replaced with
a `new_string` whose first line is indented to the `with` level. The
body lifts out of `with`, the file `f` closes, and the line below it
(still indented deeper) becomes the body of the *outer* statement
instead. The module still imports. The bug only fires when the
specific function is *called*.

Real symptom caught during a `blade-build` py2-shim cleanup:

```python
# before (correct)
def dump(self, output_file_name):
    with open(output_file_name, 'w') as f:
        print('# header', file=f)
        for name, value in sorted(iteritems(self.configs)):
            self._dump_section(name, value, f)
```

A `replace_in_file` meant to rewrite only the `for` line used
`old_string` starting *after* the `with`, and the tool re-emitted the
`for` with the method-body indent (8 spaces) instead of the
with-body indent (12 spaces):

```python
# after (broken but syntactically valid)
def dump(self, output_file_name):
    with open(output_file_name, 'w') as f:
        print('# header', file=f)
    for name, value in sorted(self.configs.items()):   # ← escaped the `with`
            self._dump_section(name, value, f)         # ← over-indented, still legal
```

- `python -m compileall` exits 0.
- `import config` succeeds.
- 12-item import-level smoke test passes.
- The bug fires only when someone calls `BladeConfig().dump(path)`
  and gets `ValueError: I/O operation on closed file` — because `f`
  closed at the end of `with`, and the `for` is now outside it.

## Solution

Three rules, in order of importance.

### 1. Anchor `old_string` on the block header, not inside the suite

When rewriting any line that lives inside a `with` / `try` /
nested `for` / `if` block, include the **block-introducing line**
in `old_string`. The tool then has an unambiguous indentation anchor
and cannot accidentally shift the replacement out.

```text
# BAD — old_string starts mid-suite, leading whitespace is ambiguous
old_string:
            for name, value in sorted(iteritems(self.configs)):
                self._dump_section(name, value, f)

# GOOD — include the `with` header so the indentation ladder is fixed
old_string:
    def dump(self, output_file_name):
        with open(output_file_name, 'w') as f:
            print('# header', file=f)
            for name, value in sorted(iteritems(self.configs)):
                self._dump_section(name, value, f)
```

### 2. After the edit, run the **callee**, not just the importer

`compileall` and `import` prove the module parses. They do not prove
that a specific function still works. If you edited a function body,
call that function at least once:

```python
# Smoke the exact function you edited.
python3 - <<'PY'
import sys; sys.path.insert(0, 'src')
from blade.config import BladeConfig
import tempfile, os
out = os.path.join(tempfile.mkdtemp(), 'x')
BladeConfig().dump(out)
assert os.path.getsize(out) > 0
print('dump OK')
PY
```

### 3. Diff-audit indentation changes

Before committing, grep the diff for suspicious indent shifts on any
line you didn't intend to re-indent:

```bash
# Any line where only leading whitespace changed between - and +?
git --no-pager diff -U0 -- path/to/file.py \
  | awk '/^-[^-]/{getline n; if (n ~ /^\+[^+]/ && \
         substr($0,2) !~ /^[[:space:]]/ "" substr(n,2)) print}'
```

Or simpler: just scan `git diff` output for any `+` line whose leading
whitespace doesn't match the `-` line it replaces. Manual eyeballing
of the hunks around `with` / `try` / nested `for` is usually enough.

## Example

Full fix from the real incident above:

```diff
     def dump(self, output_file_name):
         with open(output_file_name, 'w') as f:
             print('# header', file=f)
-        for name, value in sorted(self.configs.items()):
-                self._dump_section(name, value, f)
+            for name, value in sorted(self.configs.items()):
+                self._dump_section(name, value, f)
```

Verification after the fix:

```bash
python3.10 -m compileall -q src/        # still passes (as before)
python3.12 -m compileall -q src/        # still passes (as before)
python3 -c "from blade.config import BladeConfig; \
            import tempfile, os; \
            out=os.path.join(tempfile.mkdtemp(),'x'); \
            BladeConfig().dump(out); \
            print(os.path.getsize(out))"
# -> prints 5572; function works end-to-end
```

## Pitfalls

- **"It compiles" is not verification.** Over-indented suites,
  disconnected `else` clauses (attached to the wrong `for`/`while`),
  and `return` moved one level too shallow are all legal Python.
  Your CI and your `import` smoke cannot see them.
- **Mixing mechanical sed with hand-written `replace_in_file` is the
  highest-risk combo.** `sed 's/iteritems(\([^)]*\))/\1.items()/g'`
  preserves leading whitespace by construction; `replace_in_file`
  does not. After a mixed batch, run the end-to-end call for every
  file that was touched by the *hand-written* edits, not every file
  in the diff.
- **`ast.parse` and `py_compile` agree with the compiler.** They will
  not help. Tools that *do* help: `pylint` (W0101 unreachable,
  W0104 statement-with-no-effect sometimes catches dangling suite
  bodies), `pyright --outputjson` (flow-based, catches "variable
  possibly unbound after with"), or — most reliably — actually
  invoking the function.
- **Diff-stat can hide it.** A 1-file / +1 -1 diff that shifts a
  suite out of a `with` looks trivial in `--stat`. Review the diff,
  not the stats.

## See also

- [python-code-audit-sweep](../python-code-audit-sweep/SKILL.md) —
  parent skill; this one is the "editing hygiene" half of an audit PR.
- [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md) —
  when you need to pipe a verification script through the terminal
  tool without the shell eating it.
- [agent-work-artifacts-layout](../agent-work-artifacts-layout/SKILL.md) —
  where to dump the temporary verification script.
