+++
title = "C: The Memory Model and Stack Frames"
date = "2026-03-28"
description = "No garbage collector, no runtime. Local variables are register allocations or offsets relative to EBP, and they're gone when the function returns."
tags = ["c", "systems-programming", "memory", "stack"]
series = ["C internals"]

[cover]
hidden = true
+++

This is part two of two. [Part one](/posts/c-variables-and-symbolic-constants/) covers variables, symbolic constants, and the preprocessor.

---

## C always evaluates expressions before executing

C evaluates every expression before running it. If `main` is missing, the linker signals an error, that's the entry point contract. Parameters to `main` (`argc`, `argv`) behave like local variables.

---

## No garbage collector

In C there's no garbage collector. Local variables are not "objects" managed by a runtime; they're temporary allocations on registers or the stack. When the function returns, they're gone. There's no tracing, no finalization, no safety net. This is by design.

---

## Two kinds of storage: registers and RAM

Every microprocessor has two fundamentally different places to put data:

- **Registers**: storage internal to the core, extremely fast, limited capacity.
- **RAM**: addressable memory, much larger, much slower.

Instructions operate directly on registers. To work with data in RAM, the CPU loads it into a register first, operates, then optionally stores it back.

---

## What a local variable actually becomes

At the machine level, a local variable in C becomes one of two things:

**A register**: if the compiler can fit it in one of the available general-purpose registers, it lives there for its lifetime and never touches RAM. This is the fast path; the compiler tries hard to make this happen.

**A memory cell on the stack**: specifically, an offset relative to EBP (the base pointer of the current stack frame). EBP marks the base of the frame; ESP is the stack pointer. Locals live at negative offsets from EBP:

```asm
mov  DWORD PTR [ebp-4], 10   ; int a = 10
mov  DWORD PTR [ebp-8], 20   ; int b = 20
```

The compiler knows these offsets at compile time and hardcodes them. There's no map, no name lookup, just fixed arithmetic relative to EBP.

---

## Stack frame lifecycle

When a function is called:

1. The caller pushes arguments (or puts them in registers per the calling convention).
2. `call` pushes the return address and jumps to the function.
3. The callee saves EBP, sets `EBP = ESP`, then decrements ESP to carve out space for locals.

When the function returns:

1. ESP is restored from EBP.
2. EBP is restored (popped).
3. `ret` pops the return address and jumps back to the caller.

Local variables that lived in that frame are gone. The stack pointer has moved back; the memory is immediately available for the next function call to overwrite. This is why returning a pointer to a local is undefined behavior: by the time the caller dereferences it, the frame has been torn down.

---

## So local variables:

- Are allocated at call time, valid only during the function's execution
- Are overwritten by subsequent calls, they don't persist past `ret`
- Have no "active" lifetime at the memory level once the frame is gone

There's no "destruction" event, no finalizer. The stack pointer moves.

---

## Why the stack is a convention, not a hardware primitive

The stack frame layout is a *calling convention*: an agreement between compiler, OS, and toolchain on how function calls work. It exists for three reasons:

- **Interoperability** between separately compiled modules
- **Binary compatibility** so object files compiled at different times can still link and call each other
- **Correct return handling**: `ret` needs the return address exactly where it was pushed; if the callee corrupts the stack, the program flies off into undefined behavior

After `ret`, the stack pointer is repositioned, the cells used for parameters and locals are effectively invalidated, and any registers saved by the callee are restored. Clean slate for the caller to resume.

---

## Back to part one

→ [C: Variables, Symbolic Constants, and the Preprocessor](../2026-03-18-c-variables-and-symbolic-constants/)
