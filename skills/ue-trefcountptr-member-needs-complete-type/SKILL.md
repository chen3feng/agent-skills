---
name: ue-trefcountptr-member-needs-complete-type
description: A TRefCountPtr/TSharedPtr member in a UE class needs the pointee's full definition, not a forward decl.
tags: [unreal, cpp, smart-pointer, forward-declaration, pitfall]
---

# `TRefCountPtr<T>` / `TSharedPtr<T>` members need `T`'s complete type

## When to use

You added a data member like this to a UE module header:

```cpp
// Pb4ueTrpcServerConnection.h
class FPb4ueTrpcServerStream;           // forward decl

class FPb4ueTrpcServerConnection : public TSharedFromThis<...>
{
    ...
    TMap<uint32, TRefCountPtr<FPb4ueTrpcServerStream>> ServerStreams;
};
```

The header itself parses. Then the first TU that constructs the
enclosing class — typically via `MakeShared<FPb4ueTrpcServerConnection>()`
or a `TUniquePtr<FPb4ueTrpcServerConnection>` reset — fails to compile
with errors pointing at the smart pointer's destructor:

```
error: invalid application of 'sizeof' to an incomplete type 'FPb4ueTrpcServerStream'
note: in instantiation of member function 'TRefCountPtr<FPb4ueTrpcServerStream>::~TRefCountPtr' requested here
note: in instantiation of member function 'TMap<unsigned int, TRefCountPtr<FPb4ueTrpcServerStream>>::~TMap' requested here
note: in instantiation of member function 'FPb4ueTrpcServerConnection::~FPb4ueTrpcServerConnection' requested here
```

Same pattern with `TSharedPtr<T>`, `TUniquePtr<T>`, and `TArray<TRefCountPtr<T>>`.

## Problem

`TRefCountPtr<T>`, `TSharedPtr<T>`, `TUniquePtr<T>` all call into `T`'s
destructor from their own destructor. That requires `T` to be a
**complete type at the point the smart pointer's destructor is
instantiated**. When the smart pointer is a class member, the enclosing
class's implicit destructor instantiates the member destructors, so the
completeness requirement propagates out to **every TU that destroys the
enclosing class** — including the one that calls `MakeShared<...>()`.

A forward declaration in the header is enough to *parse* the member
declaration (you're only taking a pointer-sized field), but not enough
to *destroy* it. The compiler doesn't fire at the header; it fires at
the first caller that forces instantiation.

This is the same root cause as the classic "pimpl needs out-of-line
destructor" rule, but UE's `TRefCountPtr` has no `std::unique_ptr`-style
defaulted-dtor escape hatch, so the workaround is narrower.

## Solution

Pick one, in order of preference:

1. **Include the pointee's header in the enclosing class's header.**
   Simplest and correct when the pointee is already an internal type
   of the same module:

   ```cpp
   // Pb4ueTrpcServerConnection.h
   #include "Pb4ueTrpcServerStream.h"   // not just a forward decl

   class FPb4ueTrpcServerConnection : ...
   {
       TMap<uint32, TRefCountPtr<FPb4ueTrpcServerStream>> ServerStreams;
   };
   ```

2. **Move the member into a PImpl.** If you genuinely want to keep the
   pointee out of the public header (e.g. it drags in a heavy vendor
   header), hide the whole state behind an opaque `FImpl` forward-declared
   in the header and defined in the `.cpp`, and give the enclosing class
   an **out-of-line destructor** in that `.cpp`:

   ```cpp
   // Foo.h
   class FFoo { TUniquePtr<struct FImpl> Impl; public: ~FFoo(); };

   // Foo.cpp
   struct FImpl { TRefCountPtr<FHeavy> Heavy; };
   FFoo::~FFoo() = default;     // instantiated here, where FImpl is complete
   ```

3. **Store a raw `FHeavy*` and manage lifetime by hand.** Only if (1)
   and (2) don't fit. You lose the ref-count/ownership guarantees that
   made you reach for `TRefCountPtr` in the first place, so this is
   rarely the right answer in UE code.

Do **not** try to fix it by adding `= default` to the enclosing class's
destructor in the header — that's still an inline definition and still
requires the complete type at every use site. The fix is either
"include the header" or "move the dtor out of line".

## Example

Wrong:

```cpp
// Pb4ueTrpcServerConnection.h
class FPb4ueTrpcServerStream;   // forward-declared only

class FPb4ueTrpcServerConnection : public TSharedFromThis<FPb4ueTrpcServerConnection, ESPMode::ThreadSafe>
{
    TMap<uint32, TRefCountPtr<FPb4ueTrpcServerStream>> ServerStreams;
};

// SomeOther.cpp
auto C = MakeShared<FPb4ueTrpcServerConnection>();  // <-- fails here
```

Right:

```cpp
// Pb4ueTrpcServerConnection.h
#include "Pb4ueTrpcServerStream.h"

class FPb4ueTrpcServerConnection : public TSharedFromThis<FPb4ueTrpcServerConnection, ESPMode::ThreadSafe>
{
    TMap<uint32, TRefCountPtr<FPb4ueTrpcServerStream>> ServerStreams;
};
```

## Pitfalls

- The error points at the smart pointer's destructor, not at your
  member declaration. It's easy to misread it as "something wrong with
  `TRefCountPtr`" and start swapping smart pointer types. Swap the
  include, not the smart pointer.
- Unity builds mask this locally: if the TU that includes the pointee
  header happens to be glued next to the TU that constructs the
  enclosing class, the build passes. It breaks the moment someone turns
  unity off, or reorders the unity buckets. Fix the include even if the
  build is currently green.
- Same rule applies to `TArray<TRefCountPtr<T>>`, `TSet<TSharedPtr<T>>`,
  `TMap<K, TUniquePtr<T>>`, etc. The container's destructor instantiates
  the element's destructor, which instantiates the smart pointer's.
- `TWeakPtr<T>` and raw `T*` do **not** require a complete type, because
  they don't call `T`'s destructor. If you really only need an observer
  handle in the header, downgrade to one of those.

## See also

- The classic `std::unique_ptr` pimpl-dtor rule
  (<https://en.cppreference.com/w/cpp/memory/unique_ptr>) — same root
  cause, UE-flavored.
