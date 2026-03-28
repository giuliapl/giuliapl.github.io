+++
title = "C: Variables, Symbolic Constants, and the Preprocessor"
date = "2026-03-22"
description = "Notes on -Wall, printf variadic semantics, and why #define is a preprocessor thing, not a C thing."
tags = ["c", "systems-programming", "preprocessor", "compilers"]

series = ["C internals"]
[cover]
hidden = true
+++
This is part one of two. [Part two](/posts/c-memory-model-and-stack/) covers the C memory model and stack frames.

---

## `-Wall`

If I use the `-Wall` flag, the compiler emits warnings. Simple rule: always use it.

---

## `0` means more than zero

The value `0` is equivalent to `false` in C boolean expressions. It also shows up in shell short-circuit evaluation. In an expression like `./a.out && ls`, `ls` only executes if `./a.out` returns `0`. That's because `0` is the success exit code, and `&&` short-circuits on failure. So `./a.out` returning `0` makes `ls` run. The `0` in the exit code sense is "success / true" from the shell's perspective, opposite of the C boolean convention, which trips people up.

---

## `printf` is variadic

`printf` is a *variadic function*: it accepts a variable number of arguments. The format string drives how the remaining arguments are interpreted. No runtime type checking; you lie in the format string, you get undefined behavior.

`printf` also doesn't write to the terminal immediately. Data goes into a buffer first. The buffer is flushed when:

- a `\n` is encountered
- `fflush(stdout)` is called explicitly

This is why output can seem to disappear when a program crashes mid-execution: the buffer never got flushed. If I need output to appear immediately regardless of newlines, `fflush(stdout)` after the `printf` is the fix. The buffering is intentional: it makes I/O significantly faster by batching writes instead of issuing a syscall on every character.

---

## Symbolic constants: `#define`

To avoid magic numbers, C gives you symbolic constants:

```c
#define NAME replacement_text
```

Convention: names are always **uppercase**. This signals to every reader that the identifier is a compile-time constant, not a variable.

`#define` is also the right tool when I need conditional compilation (`#if`, `#ifdef`) or when I'm writing macros. It's also what I reach for when writing header guards or shared header files (`#define FILE_H`).

---

## What `#define` actually is

`#define` belongs to the *preprocessor*, which runs before the compiler. It rewrites your source: textual substitution, nothing more. It has no type, no memory address, no scope in the C sense.

The compilation pipeline in C goes:

1. **Preprocessor**: macro expansion, `#include` resolution, conditional compilation. Stop here with `gcc -E main.c` to see what the compiler actually receives.
2. **Compiler**: source to assembly
3. **Assembler**: assembly to object code
4. **Linker**: object files → executable

The preprocessor doesn't know C. It doesn't know about types, variables, or functions. It does text substitution and moves on. So `#define MAX 100` means: wherever the token `MAX` appears, replace it with `100` before compilation begins. The compiler never sees `MAX`.

Same for function-like macros:

```c
#define SQUARE(x) ((x) * (x))
```

Parentheses around `x` and the whole expression are not optional. Without them:

```c
#define SQUARE(x) x * x
// SQUARE(1 + 1) → 1 + 1 * 1 + 1 = 3, not 4
```

The preprocessor is doing *instruction rewriting*, not function calls. It runs before the CPU gets involved, has no type information, and no scope.

---

## Up next

Part two: what happens at the machine level when you declare a variable — registers, RAM, EBP, ESP, and why locals don't survive past `return`.

→ [C: The Memory Model and Stack Frames](./c-memory-model-and-stack/)
