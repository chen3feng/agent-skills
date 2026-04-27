---
name: detect-tool-vendor-by-query
description: Identify a tool's real vendor/version by asking it (`--version`), not by sniffing its filename or PATH entry.
tags: [build-systems, cross-platform, toolchain, python, macos]
---

# Detect tool vendor by query, not by name

## When to use

You are writing code that branches on *which* compiler, linker,
interpreter, or other build tool is in use — GCC vs. Clang, CPython
vs. PyPy, BSD vs. GNU coreutils, system Python vs. a pyenv shim — and
your current approach is to look at the executable's **name or path**
(`"gcc" in argv0`, `which python3`, `basename $CC`).

Signal phrases: "we assume `/usr/bin/gcc` is GCC", "the driver is
called `cc` so it must be…", "`python3` on PATH is 3.x", or a bug
report that reproduces only on macOS / only on a system with a
vendor-renamed toolchain.

## Problem

Tool names lie. Common traps:

- **macOS ships `/usr/bin/gcc`** but it's Apple Clang under the
  hood. Any substring check like `'gcc' in cc_path` misclassifies
  it as GNU and then tries to pass GNU-only flags
  (`-static-libgcc`, `-flto=auto`, `-fno-fat-lto-objects`, …),
  which Clang rejects at link time.
- **`cc` is a symlink** whose target varies by distro: GCC on
  most Linux, Clang on FreeBSD / macOS, tcc on some minimal
  installs. The name `cc` tells you nothing.
- **`python3` on PATH** may be the system's frozen 3.9, a pyenv
  shim, a Homebrew keg, or a virtualenv. Checking the path
  (`/usr/bin/python3` → "system Python") gives you the *source*,
  not the *version*.
- **GNU vs BSD coreutils** — `sed`, `awk`, `tar`, `date` on
  macOS are BSD variants with different flag sets; they're
  spelled exactly the same as their GNU cousins.
- **Cross-compilers and wrappers** (`ccache gcc`, `distcc clang`,
  `arm-linux-gnueabihf-gcc`) contain the vendor substring but
  may forward to a different backend than the name implies.

The fix is always the same: **ask the tool**.

## Solution

1. **Run the tool's identity query once, at init time**, and cache
   the result. Don't re-probe on every call site.
2. **Parse a known-stable field**, not the whole banner:
   - C/C++ drivers: first line of `cc --version`. Look for
     `'Apple clang'` / `'clang'` / `'gcc'` / `'Free Software
     Foundation'`. Normalize to a closed enum
     (`{'clang', 'gcc', 'unknown'}`).
   - Python: `python -c 'import sys; print(sys.version_info[:2])'`
     or `python -V` and parse `"Python X.Y.Z"`. Compare as a
     tuple, never as a float (`3.10 < 3.9` if you use floats).
   - coreutils: `sed --version 2>&1 | head -1` → `"GNU sed"` vs.
     "invalid option" on BSD.
3. **Have a defined `unknown` bucket.** Exotic toolchains exist;
   don't let the detector crash on them. Emit a warning and fall
   back to the most conservative flag set.
4. **Gate feature flags on the cached vendor**, not on the path:

   ```python
   # Wrong
   if 'gcc' in self.cc:
       flags.append('-static-libgcc')

   # Right
   if self.cc_vendor == 'gcc':
       flags.append('-static-libgcc')
   ```

5. **Let the user override**, because detection has its limits.
   Honor an env var (`$CC_VENDOR`, `$BLADE_PYTHON_INTERPRETER`,
   `$FORCE_GNU_SED`) before auto-detection, so users on exotic
   toolchains can opt out without patching the detector.

## Example

Real case from blade-build (commit `b5e09dd`). The old detector:

```python
# Misclassifies macOS /usr/bin/gcc (= Apple Clang) as GCC.
def cc_is(self, vendor: str) -> bool:
    return vendor in self.cc  # substring match on the binary path
```

On macOS, `self.cc == '/usr/bin/gcc'`, so `cc_is('gcc')` returns
`True`, so the link step adds `-static-libgcc`, so the linker
errors out (Apple Clang doesn't know that flag).

The fix — probe once, cache, compare exactly:

```python
class ToolChain:
    def __init__(self, ...):
        self._cc_vendor = self._probe_cc_vendor(self.cc)

    @staticmethod
    def _probe_cc_vendor(cc: str) -> str:
        out, _ = run_command([cc, '--version'])
        first = out.splitlines()[0].lower() if out else ''
        if 'clang' in first:        # covers 'Apple clang' too
            return 'clang'
        if 'gcc' in first or 'free software foundation' in first:
            return 'gcc'
        return 'unknown'

    def cc_is(self, vendor: str) -> bool:
        return self._cc_vendor == vendor  # exact equality
```

Analogous launcher fix for Python interpreter selection: instead of
"`python3` is on PATH, done", `_probe_python()` runs
`<candidate> -V`, parses the version tuple, and accepts only
`>= 3.10` — and it consults `$BLADE_PYTHON_INTERPRETER` first so
users whose system `python3` is 3.9 can override with `3.12`.

## Pitfalls

- **Don't parse localized output.** Some tools localize `--version`
  (`Paquet GNU sed version ...`). Set `LC_ALL=C` in the probe
  command, or match on the program name token that does not get
  translated (`sed`, `gcc`).
- **Don't probe inside hot paths.** `--version` forks a process;
  calling it from every compile rule will dominate build time.
  Probe once at toolchain construction and cache.
- **Don't rely on exit code alone.** Some tools exit non-zero for
  `--version` (very rare, but `bc` and a few BSD tools do). Also
  accept stderr; some write the banner there.
- **Watch out for "wrapper noise"**: `ccache gcc --version` prints
  the real compiler's banner, which is what you want; but a
  user-written wrapper script might swallow `--version`. Document
  that the env-var override is the escape hatch.
- **A closed enum beats free-form strings.** Returning the raw
  banner to call sites invites every caller to re-parse it and
  disagree about edge cases. Collapse to
  `{'gcc', 'clang', 'msvc', 'unknown'}` at the boundary.
- **Path-based checks are still OK for *locating* the tool** (did
  the user install it? which one is first on PATH?), but never
  for deciding *what it is*.

## See also

- [workspace-path-constraints](../workspace-path-constraints/SKILL.md)
- [python-code-audit-sweep](../python-code-audit-sweep/SKILL.md)
- [test-layout-evolution](../test-layout-evolution/SKILL.md)
