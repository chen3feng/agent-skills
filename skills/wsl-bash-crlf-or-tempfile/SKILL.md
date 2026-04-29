---
name: wsl-bash-crlf-or-tempfile
description: When a Windows shell (PowerShell/cmd) feeds a bash script into WSL, CRLF line endings can corrupt the first shell builtin; force LF or pipe via a temp file.
tags: [wsl, powershell, bash, windows, shell, quoting]
---

# WSL bash scripts: kill CRLF, or route through a temp file

## When to use

You're driving a Linux toolchain from Windows: a PowerShell or `.bat`
script builds a multi-line bash body and runs it in WSL via
`wsl.exe -d <distro> -- bash -c "<body>"`. Symptoms that land you
here:

- `bash: line 1: set: pipefail : invalid option name`
  (note the phantom space before the `:` — that's a literal `\r`).
- `bash: line N: $'<something>\r': command not found`.
- Heredoc bodies that silently truncate at the wrong line.
- A bash `if`/`then` block reports a syntax error at a line that
  looks perfectly fine in your editor.

It's not "my bash is wrong"; it's **CRLF contamination** crossing
the Windows → WSL boundary.

## Problem

PowerShell's here-strings (`@' … '@`, `@" … "@`) and `.bat` `echo`
lines emit **CRLF** by default. When you hand the resulting string
to `wsl.exe bash -c "$body"`, Linux `bash` reads the `\r` as a
normal character that is part of the previous token:

- `set -euo pipefail\r` becomes the command
  `set` with options `-e`, `-u`, `-o`, `pipefail\r`. bash prints
  `set: pipefail : invalid option name` — the "space" before the
  colon in the error message is actually the carriage return.
- `FOO=bar\r` sets `FOO` to `bar\r`, and the later comparison
  `[ "$FOO" = "bar" ]` silently fails.
- `fi\r` / `done\r` are accepted as words, so the `if` / `for` is
  never closed and the parser reports an error many lines below
  the actual cause.

Why it's easy to miss:

- Your editor (VS Code, Notepad++, etc.) happily hides `\r`.
- The CRLF shows up *only* when the script crosses into WSL. If
  you run the same body via `Invoke-Expression` on Windows, or
  dump it to disk and `type` it, everything *looks* fine.
- The usual quoting-skills reflex (escape the dollar signs, quote
  the body, etc.) doesn't help — this is a line-ending issue, not
  a quoting issue.

The related `\0` NUL trap at the API boundary is a *different*
problem (see *Pitfalls* below).

## Solution

Two reliable fixes. Prefer #2 for anything beyond a one-liner —
it composes better and gives you a debuggable artifact.

### Fix 1 — Normalize to LF before handing to `bash -c`

Strip `\r` from the body immediately before invocation:

```powershell
$bash = $bashTemplate `
    -replace '__PLACEHOLDER__', $value

# Force LF endings. Order matters: CRLF first, then any lone CR.
$bash = $bash -replace "`r`n", "`n" -replace "`r", "`n"

& wsl.exe -d $Distro -- bash -c $bash
```

Works, but still has two weaknesses:

- Long bodies are passed as a single command-line argument; quoting
  and escaping interact with `cmd` → `wsl.exe` → `bash -c` across
  three layers.
- There's nothing on disk to inspect when it fails.

### Fix 2 — Write to a temp file (UTF-8 no BOM, LF), run `bash <path>`

Reliable for any script size:

```powershell
# Normalize endings.
$bash = $bash -replace "`r`n", "`n" -replace "`r", "`n"

# UTF-8 *without* BOM — Out-File / Set-Content may add a BOM,
# which bash interprets as an unrecognised character on line 1.
$tmp       = Join-Path $env:TEMP ("fbc-sweep-{0}.sh" -f ([IO.Path]::GetRandomFileName()))
$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
[IO.File]::WriteAllText($tmp, $bash, $utf8NoBom)

# Translate C:\Users\foo\... -> /mnt/c/Users/foo/...
$drive    = $tmp.Substring(0,1).ToLowerInvariant()
$tail     = $tmp.Substring(2).Replace('\','/')
$wslPath  = "/mnt/$drive$tail"

try {
    & wsl.exe -d $Distro -- bash $wslPath
    $exit = $LASTEXITCODE
} finally {
    Remove-Item -LiteralPath $tmp -ErrorAction SilentlyContinue
}
```

Why this one works:

- A real file with LF endings is what `bash` expects. No CRLF
  handling to get wrong.
- `bash <path>` means no `-c "<huge string>"`: no quoting /
  escaping issues, no command-line-length limit, no weird
  interactions with `cmd.exe`'s percent expansion.
- The temp file is your debuggable artifact. If WSL prints an
  error, just `wsl bash -x <path>` (or `cat -A <path>`) to inspect
  exactly what bash saw.
- `try / finally` ensures cleanup even on Ctrl-C or exceptions.

### Verification one-liner

Before you trust the pipeline, verify with `cat -A`:

```powershell
& wsl.exe -d $Distro -- bash -c "cat -A '$wslPath' | head -n 3"
```

Good output (LF everywhere):

```text
set -euo pipefail$
echo ok$
```

Bad output (CRLF — the `^M` is the `\r`):

```text
set -euo pipefail^M$
echo ok^M$
```

If you see `^M$`, the LF normalization didn't run or wasn't
applied to the content you actually wrote.

## Example

Broken — heredoc straight into `bash -c`:

```powershell
$bash = @'
set -euo pipefail
echo hello
'@
& wsl.exe -d Ubuntu-24.04 -- bash -c $bash
# bash: line 1: set: pipefail : invalid option name
```

Fixed — normalize + temp file:

```powershell
$bash = @'
set -euo pipefail
echo hello
'@
$bash = $bash -replace "`r`n","`n" -replace "`r","`n"

$tmp = Join-Path $env:TEMP ("wsl-run-{0}.sh" -f ([IO.Path]::GetRandomFileName()))
[IO.File]::WriteAllText($tmp, $bash, (New-Object System.Text.UTF8Encoding($false)))

$wslPath = "/mnt/" + $tmp.Substring(0,1).ToLowerInvariant() + $tmp.Substring(2).Replace('\','/')
try   { & wsl.exe -d Ubuntu-24.04 -- bash $wslPath }
finally { Remove-Item -LiteralPath $tmp -ErrorAction SilentlyContinue }
# hello
```

## Pitfalls

- **Don't use `Out-File` / `Set-Content` without specifying
  encoding.** Windows PowerShell 5.1 defaults to UTF-16 LE with
  BOM; PowerShell 7 defaults to UTF-8 *no* BOM — but relying on
  that difference is fragile. Always pass `UTF8Encoding($false)`
  explicitly via `[IO.File]::WriteAllText`, or
  `Set-Content -Encoding utf8NoBOM` on PowerShell 7+. A BOM on
  line 1 makes bash report an error on the `#!` / first command.
- **The `wsl.exe --list` NUL trap is a separate bug, same family.**
  `wsl.exe --list --verbose` emits UTF-16 LE; PowerShell decodes
  it into a string, but leaves a `\0` byte between each character.
  Exact-string matching against a distro name fails for a reason
  totally invisible in the printout. Cure: pre-strip NULs with
  a `-replace` pass (replace the literal `` `0 `` escape with the
  empty string) before any regex / `-contains` check.
  Symptomatically similar ("my match is right but the code says it
  isn't"), mechanistically different (`\0` at the API boundary vs
  `\r` inside the payload) — fix both.
- **Don't try to pipe via `echo | wsl bash`.** `echo` in `cmd.exe`
  emits CRLF, and the pipe inside `cmd` doesn't translate. You end
  up back at the original bug. Either `wsl.exe bash <file>` or
  `wsl.exe bash -c "<LF-only body>"`.
- **`-lc` vs `-c` won't save you.** Login shells don't strip `\r`;
  they just source more rc files. Fix the content, not the shell
  invocation flags.
- **Mixed CRLF in existing repo scripts.** If the `.sh` file lives
  in the repo and was edited on Windows, `git config core.autocrlf`
  may have rewritten it to CRLF on checkout. Either set
  `* text eol=lf` in `.gitattributes` for `*.sh` (persistent cure)
  or normalize once with `dos2unix` before invoking. This is the
  same family of bug as above but the source is the checkout, not
  the PowerShell heredoc.
- **WSL networkingMode fallbacks are unrelated noise.** Messages
  like `wsl: Failed to configure network (networkingMode Mirrored),
  falling back to networkingMode None` often appear alongside the
  `set: pipefail` error and look causal, but they're not. Fix the
  CRLF first; if your bash script then fails at `apt-get update`,
  *that* is when to investigate the network config.
- **Don't trust your editor.** "But I can see `set -euo pipefail`
  is on one line!" — yes, because VS Code hides `\r`. Use
  `cat -A` (or `od -c`) inside WSL on the exact file the agent
  produced, not on a copy you opened in your editor.

## See also

- [shell-heredoc-and-multiline-strings](../shell-heredoc-and-multiline-strings/SKILL.md)
  — sibling skill about passing multi-line strings *inside* a single
  shell (bash → git / gh), not across the Windows → WSL boundary.
- [workspace-path-constraints](../workspace-path-constraints/SKILL.md)
  — why the temp file must end up reachable from both sides.
