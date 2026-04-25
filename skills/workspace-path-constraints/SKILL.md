---
name: workspace-path-constraints
description: Work around editor/search tools that only accept indexed-workspace paths.
tags: [agent, tooling, workspace]
---

# Working across indexed and non-indexed workspaces

## When to use

You need to create or edit files in a sibling directory the user just
cloned (e.g. `../agent-skills`) but the agent's `edit_file`,
`list_dir`, `codebase_search`, or `view_code_item` refuses it with
`路径不在workspace下,拒绝请求` / "path is not in workspace".

## Problem

Several tools are scoped to the *indexed* workspaces only (the ones
listed in the session's workspace configuration). They won't touch
files outside, even if the process has filesystem permission.

Trying to keep inventing ad-hoc `cat > file <<EOF` heredocs in the
shell is how you get the problems documented in
[shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md).

## Solution

Pick one based on what the target needs:

1. **One or two short text files** → use the agent's `terminal` tool
   with `printf` / `tee`, quoting carefully.

2. **Multiple files, or files with backticks / quotes / multi-line
   code** → write a *generator script* inside an indexed workspace
   (e.g. `blade-build/tmp_build_skills.py`) that builds a
   `{path: content}` dict and writes every file with standard
   `Path.write_text`. The generator is plain Python source, so
   `edit_file` is happy to create it. Run it once with `python3`,
   then delete.

3. **Read-only exploration** (listing, grepping) of the non-indexed
   directory → use `terminal` with `ls`, `find`, `cat`, `rg`.

4. **Large existing file you need to modify** → `cat` it through the
   terminal to inspect, then apply a small `sed -i` or re-emit via
   the generator-script trick.

## Example

Generator-script pattern:

```python
# blade-build/tmp_build_skills.py
from pathlib import Path
ROOT = Path("/Volumes/SanDiskSSD/code/agent-skills")
FILES = {
    "README.md": "…",
    "skills/foo/SKILL.md": "…",
}
for rel, body in FILES.items():
    p = ROOT / rel
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_text(body)
print(f"wrote {len(FILES)} files")
```

Then:

```bash
python3 /Volumes/SanDiskSSD/code/blade-build/tmp_build_skills.py
rm /Volumes/SanDiskSSD/code/blade-build/tmp_build_skills.py
```

## Pitfalls

- Don't accidentally commit the generator script to the indexed repo.
  Either delete it immediately or stage selectively.
- `codebase_search` / `view_code_item` will *not* see the non-indexed
  repo even after files are written; fall back to `grep` + `cat`.
- If the user later adds the sibling repo to the workspace, rerun any
  cached searches — results from before won't include it.

## See also

- [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md)
