+++
title = "What Runs Before Your Program Does: The Dynamic Linker, PLT, and GOT"
date = "2026-03-14"
draft = false
tags = ["c", "linker", "systems-programming", "linux", "elf"]
categories = ["engineering"]
description = "When you run a binary, something else runs first. Here's what ld.so actually does, why shared libraries load at different addresses every time, and how PLT and GOT make function calls work anyway."
[cover]
  hidden = true
+++

In the [previous post](/posts/c-compilation-linking/), we saw that dynamically linked binaries don't include library code, they just reference it. Which raises an obvious question: *who actually loads that code at runtime, and how does the program find it?*

The answer is `ld.so`, the dynamic linker. And it runs before your `main()` does.

---

## The Bootstrap Problem

When you execute a binary, the kernel reads its ELF header. One of the fields in that header (stored in the `.interp` section) contains a path like:

```
/lib64/ld-linux-x86-64.so.2
```

This is the dynamic linker. The kernel loads it into memory and hands control to it *before* your program starts. `ld.so` is the program that runs before your program.

Its job: set everything up so that by the time your `main()` is called, all the shared libraries are loaded, all the symbols are resolved, and all the memory mappings are in place.

---

## What `ld.so` Actually Does

When `ld.so` takes control, it does three things:

**1. Opens and maps your ELF file and all required libraries**

It uses `mmap`, a system call that maps a file directly into the process's virtual address space. This is not a copy of the file's bytes from disk to RAM. Instead, the OS creates a *mapping*: "pages at these addresses correspond to this region of this file." The actual bytes are loaded lazily, only when accessed.

This is why:
- Programs start fast even with large libraries
- Multiple processes using the same library share the same physical memory pages
- The OS can swap library pages out and back in without your program noticing

**2. Resolves relocations**

Your object code has "holes" where addresses should be: placeholders for functions like `printf` whose location wasn't known at compile time. `ld.so` fills in these holes with the real runtime addresses.

**3. Starts your program**

Once everything is mapped and resolved, `ld.so` jumps to your entry point and your program begins.

---

## The ASLR Problem

Here's a complication: on modern Linux systems, **ASLR** (Address Space Layout Randomization) is enabled by default. Every time a shared library is loaded, it lands at a different base address in memory. This is a security feature: it makes it much harder for exploits to predict where code lives.

But it creates a problem. If `libc.so` loads at a different address every run, how does your code know where `printf` is?

The answer is: it doesn't hardcode the address. Instead, it goes through two indirection tables: the **PLT** and the **GOT**.

---

## PLT and GOT: Lazy Binding

When the compiler generates a call to `printf`, it doesn't emit:

```asm
call 0x7f3a42b1c000   ; direct address, impossible, changes every run
```

It emits:

```asm
call puts@PLT         ; jump to the PLT stub for puts
```

**PLT** (Procedure Linkage Table) is a set of small stubs, one per external function. Each stub is a trampoline that jumps through the **GOT** (Global Offset Table).

**GOT** is a table of addresses, filled in at runtime. Initially, GOT entries point back into the PLT (not yet resolved). On the first call to `printf`:

1. Your code jumps to `printf@PLT`
2. The PLT stub checks the GOT entry for `printf` (it's not resolved yet)
3. The PLT calls `ld.so` to resolve the address
4. `ld.so` finds the real `printf` in memory and writes its address into the GOT
5. Execution jumps to the real `printf`

On every subsequent call:

1. Your code jumps to `printf@PLT`
2. The PLT stub checks the GOT (now it has the real address)
3. Direct jump to `printf`

This is **lazy binding**: symbols are resolved on first use, not at startup. It keeps startup time fast when a binary has hundreds of library dependencies.

```
First call:
  call puts@PLT → PLT stub → GOT (unresolved) → ld.so → real puts
                                                    ↓
                                             writes address to GOT

Subsequent calls:
  call puts@PLT → PLT stub → GOT (resolved) → real puts
```

---

## Why This Matters Beyond Theory

Understanding the PLT/GOT trampoline explains a few things you might have wondered about:

**Why does `strace` show library calls being resolved at startup?**  
With `LD_BIND_NOW=1` or `-z now` at link time, you can disable lazy binding and force all symbols to resolve upfront. Some security-hardened builds do this.

**What is a GOT overwrite exploit?**  
If an attacker can write an arbitrary address into a GOT entry, they can redirect any subsequent call to that function to arbitrary code. This is why modern hardened binaries mark the GOT as read-only after startup (`RELRO`).

**Why do the same libraries appear at different addresses in `cat /proc/PID/maps`?**  
ASLR. Every process gets a different layout. The PLT/GOT mechanism is precisely what makes this work transparently.

---

## The Full Picture

```
$ ./hello

  Kernel reads ELF header
       │
       ▼
  Loads ld.so (from .interp section)
       │
       ▼
  ld.so opens ELF + all required .so files
  ld.so mmaps them into process memory
  ld.so sets up PLT/GOT tables
       │
       ▼
  ld.so jumps to _start → main()
       │
       ▼
  printf("hello\n")
  → call puts@PLT
  → PLT checks GOT
  → first call: ld.so resolves, writes GOT
  → subsequent calls: direct via GOT
```

The dynamic linker is the program that runs before your program. It's doing a surprising amount of work in the milliseconds before `main()`: mapping files, resolving symbols, setting up the indirection tables that make ASLR and shared libraries coexist. Once you see it, you can't unsee it every time you run a binary.
