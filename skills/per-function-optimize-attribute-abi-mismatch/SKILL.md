---
name: per-function-optimize-attribute-abi-mismatch
description: `__attribute__((optimize("O0")))` on one function inside an `-O2` TU on GCC can crash on first call.
tags: [gcc, cpp, optimizer, abi, pitfall]
---

# `__attribute__((optimize("O0")))` silently breaks the ABI on GCC

## When to use

You're trying to isolate a miscompile in a single helper function by
forcing it to lower optimization, using one of these forms inside a
translation unit that is otherwise built at `-O2` or higher:

```cpp
__attribute__((optimize("O0")))
static void Helper(uint8_t* Dst, const uint8_t* Src, int N) { ... }
```

or equivalently:

```cpp
#pragma GCC push_options
#pragma GCC optimize("O0")
static void Helper(...) { ... }
#pragma GCC pop_options
```

and the program now crashes — usually SIGSEGV — on the *first* call
into `Helper`, before any of its body runs, even though the same code
worked at `-O2`.

## Problem

GCC's per-function `optimize` attribute changes the **optimization
level** of the callee, but it does **not** re-align the ABI between
that callee and its `-O2` caller. Concretely:

- `-O0` GCC emits a full stack frame with a frame pointer, spills all
  arguments to memory, and does not use the x86-64 SysV red zone.
- `-O2` GCC elides the frame pointer, keeps arguments in registers,
  and may use the red zone.
- The caller, built at `-O2`, passes arguments and return address
  under `-O2` assumptions. The callee reads them under `-O0`
  assumptions. The mismatch shows up as reading `0` or garbage from a
  pointer argument, or a segfault on return (frame pointer mismatch).

Crucially, there is **no compiler warning**. The attribute silently
produces a working-looking binary that crashes at runtime.

On Clang, `__attribute__((optnone))` exists and is better-behaved
about this, but it's also more restrictive (disables *all* opts, not
just some) and is not a fix-by-dropping-level tool.

## Solution

If you really need one function at a lower optimization level, pick
one of these, in order of preference:

1. **Don't.** If you reached for `optimize("O0")` to diagnose a
   miscompile, see
   [stop-chasing-the-optimizer-reduce-instead](../stop-chasing-the-optimizer-reduce-instead/SKILL.md)
   — reduce the repro and fix the real bug (usually UB) instead.
2. **Isolate the function in its own TU, compiled at `-O1`.** Move
   `Helper` into `helper.cpp` and add a per-file flag in the build
   system:

   ```cmake
   # CMake
   set_source_files_properties(helper.cpp PROPERTIES COMPILE_FLAGS "-O1")
   ```

   ```python
   # Unreal Build Tool, in a Module's Build.cs
   // equivalent: exclude just this file from unity builds, then use a
   // per-file override. UBT doesn't expose per-file -O1 directly, so
   // the usual escape is a separate Module or a .cpp-level #pragma.
   ```

   `-O1` is enough to keep the ABI in sync with `-O2` callers while
   disabling the handful of aggressive passes that tend to miscompile.
3. **Whole-TU pragma, not per-function.** If you can't split the TU,
   at least widen the scope:

   ```cpp
   // Top of the file, before any function definition.
   #pragma GCC optimize("O1")
   ```

   This degrades every function in the file equally, so the callees
   within the file are ABI-consistent with each other. Callers in
   *other* TUs still see the file's functions at `-O1`, which is fine
   — `-O1` is ABI-compatible with `-O2` in all the ways that matter.
4. **Never mix `optimize("O0")` with a helper that's inlined.** Even
   without ABI issues, the inlining decision can flip and silently
   undo your intent. Pair the attribute with `noinline`:

   ```cpp
   __attribute__((noinline, optimize("O1")))  // O1, not O0
   ```

## Example

Minimal repro of the crash:

```cpp
// build with: g++ -std=c++17 -O2 repro.cpp -o repro
#include <cstdio>
#include <cstdint>

__attribute__((noinline, optimize("O0")))
static void Store(uint8_t* Dst, uint64_t V)
{
    Dst[0] = (uint8_t)(V >> 0);
    Dst[1] = (uint8_t)(V >> 8);
}

int main()
{
    uint8_t Buf[16] = {};
    Store(Buf, 0x11223344);      // segfault here on some GCC versions
    std::printf("%02X %02X\n", Buf[0], Buf[1]);
}
```

Replace `optimize("O0")` with `optimize("O1")` and the crash goes
away. That's the smallest fix; the real fix is to remove the attribute
entirely and address whatever miscompile made you add it.

Diagnostic tells:

- Crash is on the **first** call into the annotated function, not
  deep inside it. If you `printf` as the first line of the function,
  the message never appears.
- `gdb` shows a corrupt return address or a bogus `rbp` immediately
  on function entry.
- `g++ -O2 -S repro.cpp` shows the caller storing args in registers
  only; the annotated callee's assembly spills them to stack offsets
  the caller never wrote to.

## Pitfalls

- **`#pragma GCC optimize("O0")` has the same problem** as the
  attribute form. The "pragma vs attribute" choice doesn't rescue
  you; the ABI mismatch is about the optimization *level*, not the
  *spelling*.
- **This is not symmetric across compilers.** Clang's `optnone` does
  not crash in the same way — but assuming Clang behavior on GCC is
  how you get here in the first place.
- **MSVC has no equivalent attribute.** `#pragma optimize("", off)`
  is whole-file, applied between function definitions, not per
  function. Code that relies on per-function `optimize("O0")` is
  automatically non-portable.
- **LTO can make it worse.** With `-flto`, the "caller" and "callee"
  may be re-optimized together post-attribute, and the level you
  asked for silently doesn't stick.
- If you see this pattern in a third-party codebase, it's almost
  always a stale workaround for a compiler bug that has long been
  fixed. Try just removing the attribute against a current GCC before
  porting it forward.

## See also

- [stop-chasing-the-optimizer-reduce-instead](../stop-chasing-the-optimizer-reduce-instead/SKILL.md)
  — the meta-skill; this pitfall is most often hit while violating it.
- GCC manual, "Common Function Attributes → optimize":
  <https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html>
- GCC bugzilla search for "optimize attribute ABI":
  <https://gcc.gnu.org/bugzilla/buglist.cgi?quicksearch=optimize+attribute+abi>
