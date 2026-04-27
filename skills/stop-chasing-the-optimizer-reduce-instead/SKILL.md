---
name: stop-chasing-the-optimizer-reduce-instead
description: After two failed anti-optimization patches, stop and reduce; don't keep bolting on `volatile` / `noinline`.
tags: [debugging-strategy, compilers, gcc, clang, optimizer, cpp]
---

# Stop chasing the optimizer; reduce the repro instead

## When to use

You have a hot function that passes on one toolchain (typically MSVC)
but miscompiles or crashes under GCC / Clang at `-O2` (or higher), and
you're on attempt three of sprinkling anti-optimization decorations:

- `volatile` on a local / a parameter
- `__attribute__((noinline))` on the callee
- `__attribute__((optimize("O0")))` on the callee
- `-fno-strict-aliasing` on the TU
- `asm volatile("" ::: "memory")` barriers

Each one "almost works" — the symptom moves by a few bytes or into a
different test case — but never fully goes away. That's the signal to
stop.

## Problem

Adding anti-optimization pragmas is a *local* fix applied to a *global*
belief: "the optimizer is wrong." Three things go wrong in practice:

1. **The decorations change the bug, not the cause.** `noinline` in one
   spot shifts the inlining decision elsewhere; the miscompile now
   lives in a different frame. You spend another iteration finding it.
2. **Some decorations are themselves traps.** `__attribute__((optimize("O0")))`
   on a single function inside an `-O2` TU is known to produce ABI
   mismatches between the `-O0` callee and the `-O2` caller on GCC
   (frame pointer, red-zone, stack alignment). The "fix" introduces a
   new SIGSEGV on the first call.
3. **You never learn whether it's a compiler bug, your UB, or your
   aliasing.** All three present identically — "works on MSVC, breaks
   on GCC `-O2`" — and have very different fixes. Patching blindly
   leaves the root cause unknown, so the same bug returns the next
   time someone touches the file.

The heuristic: **after two anti-optimization patches in a row have
failed to fully fix the symptom, the next step is not a third patch.
It's reduction.**

## Solution

Switch from "patch" mode to "reduce" mode:

1. **Extract a standalone repro.** Rip the suspect function out of the
   project into a single `.cpp` of ≤ 200 lines that links with nothing
   but libc. Must reproduce the divergence between MSVC and GCC/Clang
   `-O2`. If it doesn't reproduce standalone, the bug is in *how* the
   function is called, not the function itself — go up a frame.
2. **Diff the codegen.** `g++ -O2 -S -masm=intel repro.cpp` vs
   `clang++ -O2 -S -masm=intel repro.cpp` vs MSVC `/FAs`. Look for
   loads from offsets you never wrote, or stores that the compiler
   elided. This usually tells you within minutes whether it's UB
   (compiler is within its rights) or a real miscompile.
3. **Shrink with `creduce` / `cvise`.** Feed the standalone repro plus
   a predicate script (`g++ -O2 x.cpp && ./a.out; [ $? -ne 0 ]`) to
   `cvise`. 200 lines typically collapses to 20.
4. **Decide once, fix once.**
   - UB in your code → fix the UB (e.g. replace `x >> 64` with a
     branch, use `memcpy` instead of pointer-punning, add an
     explicit bounds check). No `volatile` needed.
   - Real compiler bug → file upstream with the reduced repro, then
     put *one* narrow workaround (guarded by compiler-version macros)
     with a link to the bug report.
   - Aliasing assumption → use `memcpy` or `__attribute__((may_alias))`
     at the type, not `-fno-strict-aliasing` on the whole TU.

The key is that step 4 is a **single, documented** change, not another
round of sprinkling.

## Example

Real case (the one this skill came from): a bit-blit routine
`appBitsCpyFast` passed all MSVC tests but produced zeros on Linux GCC
`-O2`. Sequence that *didn't* work, in order:

```cpp
// Attempt 1: mark the output volatile at call sites. Symptom moves.
// Attempt 2: __attribute__((noinline)) on the inner helper. Passes
//            aligned cases, still fails unaligned.
// Attempt 3: __attribute__((optimize("O0"))) on the helper.
//            Now SIGSEGVs on the first call (ABI mismatch). Worse.
```

At this point the right move was not attempt 4. It was:

```bash
# Predicate script: exits 0 iff the bug reproduces.
cat > check.sh <<'EOF'
#!/bin/sh
g++ -std=c++17 -O2 -o /tmp/a repro.cpp || exit 1
/tmp/a | grep -q 'tmp\[0\]=0x00'   # bug: should be 0x45, reads 0x00
EOF
chmod +x check.sh

cvise check.sh repro.cpp            # shrinks 180 lines → 22 lines
```

The 22-line reducer made it obvious the miscompile traced to a
`Word >> 64` on a 64-bit value — undefined behavior that MSVC happens
to fold to `0` and GCC is free to leave as whatever the shifter
produces. One `if (Shift == 64) Word = 0;` guard — no `volatile`, no
`noinline`, no `optimize` — fixed it permanently on all three
compilers.

## Pitfalls

- **`__attribute__((optimize("O0")))` on a single function inside an
  `-O2` TU on GCC can crash.** The `-O0` callee and `-O2` caller
  disagree on frame-pointer / red-zone / alignment convention. If you
  really need a function at lower optimization, put it in its own TU
  and compile that whole TU at `-O1` (not `-O0`) via the build system.
- **"Works on MSVC" is weak portability evidence.** MSVC tends to fold
  undefined shifts, undefined unions, and out-of-range enum values
  into "the obvious answer". GCC and Clang exploit them. Do not use
  MSVC green as a justification to ship.
- **`-fno-strict-aliasing` is a TU-wide sledgehammer.** If you reach
  for it as a fix, first check whether a targeted `memcpy` (or
  `std::bit_cast` in C++20) expresses what you meant. The sledgehammer
  also disables legitimate optimizations for every other function in
  the file.
- **The optimizer is almost never "wrong."** In ~20 years of shipping
  C++, roughly 1 in 50 "GCC miscompile" reports survives reduction.
  Assume UB until reduction says otherwise.
- If `cvise` is unavailable, a manual binary-chop of the function body
  (delete the second half, does it still fail?) gets you 80% of the
  way in 15 minutes.

## See also

- [detect-tool-vendor-by-query](../detect-tool-vendor-by-query/SKILL.md)
  — another "don't guess the compiler, query it" lesson.
- [doc-code-consistency-check](../doc-code-consistency-check/SKILL.md)
  — the mirror-image pitfall: patching the doc instead of the code.
- `cvise` project: <https://github.com/marxin/cvise>
- `creduce` project: <https://github.com/csmith-project/creduce>
