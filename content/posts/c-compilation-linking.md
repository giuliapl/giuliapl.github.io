+++
title = "What Actually Happens When You Compile a C Program"
date = "2026-03-08"
draft = false
tags = ["c", "compilers", "linker", "systems-programming"]
categories = ["engineering"]
description = "Most developers treat compilation as a black box. Here's what's actually happening in three distinct stages, and why understanding it matters."
[cover]
  hidden = true
+++

Most developers treat compilation as a black box: you put source code in, you get a binary out. But there are actually three distinct stages happening, each with its own job and its own output. Understanding them has made me significantly better at debugging linker errors, understanding dependencies, and reasoning about what ends up in a binary.

Let's walk through what happens when you run `gcc hello.c -o hello`.

---

## Stage 1: The Preprocessor

Before the compiler sees a single line of your code, the **preprocessor** runs. Its job is mechanical: it handles all the `#` directives.

When you write:

```c
#include <stdio.h>
```

The preprocessor literally copy-pastes the entire contents of `stdio.h` into your source file. That's it. No magic: it's text substitution.

`stdio.h` is a **header file** (`.h`). Header files contain *prototypes*: they tell the compiler "this function exists, it takes these arguments, it returns this type." They don't contain implementations. The actual implementation of `printf` lives somewhere else entirely: inside `libc.so`.

This distinction matters:
- **Header files** (`.h`) → prototypes and declarations
- **Source files** (`.c`) → implementations
- **Libraries** (`.so`, `.a`) → compiled implementations, linked later

---

## Stage 2: The Compiler

After preprocessing, the compiler (e.g. `gcc`) translates your C code into **machine code**, but stops short of a complete executable. The output is an **object file** (`.o`).

```bash
gcc -c hello.c -o hello.o   # compile only, no linking
```

An object file is machine code that's *incomplete*. It knows what `printf` should be called, but it doesn't know *where* `printf` lives in memory yet. That's the linker's job.

You can think of object files as puzzle pieces: all the shapes are there, but nothing is connected.

---

## Stage 3: The Linker

The linker (`ld`, invoked automatically by `gcc`) takes your object files and connects them to the library implementations they depend on, producing a runnable executable.

```bash
gcc hello.o -o hello         # link only
```

Or in one step:

```bash
gcc hello.c -o hello         # compile + link
```

For `printf`, the linker connects your call to the implementation inside `libc.so.6`, the C standard library. On a typical Linux system:

```
libc.so.6 → /lib/x86_64-linux-gnu/libc.so.6
```

There are two kinds of linking:

- **Static linking**: the library code is copied directly into your binary. The binary is self-contained but larger.
- **Dynamic linking**: the binary just records *which* library it needs. The library is loaded at runtime. Smaller binaries, but requires the library to be present on the target system.

Most programs use dynamic linking by default.

---

## Inspecting Dependencies with `ldd`

You can see which shared libraries a binary depends on with `ldd`:

```bash
ldd hello
```

Output on Linux:
```
linux-vdso.so.1 (0x...)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x...)
/lib64/ld-linux-x86-64.so.2 (0x...)
```

Three entries:
- `linux-vdso.so.1` — a virtual library injected by the kernel for fast syscalls
- `libc.so.6` — the C standard library (where `printf` etc. live)
- `ld-linux-x86-64.so.2` — the **dynamic linker/loader** itself (more on this in the next post)

`ldd` is actually a bash script that prints what `ld.so` would load, without executing the program. Useful for debugging "library not found" errors on deployment.

---

## The ELF Format

The output of all this (whether it's an object file, a shared library, or an executable) is stored in **ELF format** (Executable and Linkable Format).

ELF is the standard binary format on Linux. It encodes metadata the OS and linker need:

- Target architecture (x86-64, ARM, ...)
- File type (executable, shared library, object file, ...)
- Entry point address (where execution starts)
- Where to load code and data in memory
- Which dynamic linker to use (stored in a special section called `.interp`)
- Which shared libraries are required

When you run a program, the kernel reads this metadata before doing anything else. The `.interp` section in particular tells it: *"before you run me, load this other program first"*, pointing to `ld.so`, the dynamic linker.

Which is where things get really interesting, and what I'll cover in the next post.

---

## Quick Reference

```bash
gcc hello.c              # compile + link → a.out (default name)
gcc hello.c -o hi        # compile + link → hi
gcc -c hello.c           # compile only   → hello.o (not runnable)
gcc hello.o -o hi        # link only      → hi (runnable)
ldd hi                   # show shared library dependencies
```

The mental model: **preprocessor** handles text substitution, **compiler** produces incomplete machine code, **linker** connects the pieces into something runnable. Each stage has a distinct job, and knowing where one ends and the next begins makes a whole class of build errors suddenly make sense.
