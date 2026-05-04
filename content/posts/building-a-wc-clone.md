+++
title = "Reimplementing wc: Where Text Abstractions Leak"
date = "2026-05-04"
draft = false
tags = ["python", "unix", "cli", "encoding", "systems-programming", "unicode", "posix"]
categories = ["engineering", "systems"]
description = "A minimal wc clone used to examine byte streams, Unicode code points, and the Unix I/O contract—focusing on where common abstractions break down."
[cover]
  hidden = true
+++

Reimplementing `wc` is less about counting and more about confronting the places where “text” stops being a clean abstraction. The interface is trivial; the semantics are not. Most of the interesting behavior lives at the boundary between bytes, encodings, and Unix I/O conventions.

Below is a minimal clone. It’s deliberately scoped: correct along a few dimensions, incomplete along others.

```python
#!/usr/bin/env python3

import argparse, sys

def get_bytes(raw): return len(raw)
def get_lines(text): return text.count('\n')
def get_words(text): return len(text.split())
def get_chars(text): return len(text)

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('file_path', nargs='?')
    parser.add_argument('-c', action="store_true")
    parser.add_argument('-l', action="store_true")
    parser.add_argument('-w', action="store_true")
    parser.add_argument('-m', action="store_true")
    args = parser.parse_args()

    if args.file_path:
        with open(args.file_path, "rb") as f:
            raw = f.read()
        label = args.file_path
    elif not sys.stdin.isatty():
        raw = sys.stdin.buffer.read()
        label = None
    else:
        return

    text = raw.decode('utf-8')

    if not any([args.c, args.l, args.w, args.m]):
        args.c = args.l = args.w = True

    results = []
    if args.l: results.append(str(get_lines(text)))
    if args.w: results.append(str(get_words(text)))
    if args.c: results.append(str(get_bytes(raw)))
    if args.m: results.append(str(get_chars(text)))

    parts = ' '.join(results)
    print(f"{parts} {label}" if label else parts)

if __name__ == "__main__":
    main()
```

---

## Bytes vs characters is an interface boundary, not trivia

The `-c` / `-m` distinction is where most implementations quietly diverge from spec.

* `-c`: byte count (size of the underlying stream)
* `-m`: character count (number of Unicode code points)

ASCII collapses these into the same number, which is why many implementations get away with treating them as interchangeable. UTF-8 does not.

The consequence is structural: you need two representations of the same input.

```python
raw = f.read()               # bytes
text = raw.decode('utf-8')  # Unicode string
```

These are not interchangeable views. They answer different questions:

* `len(raw)` → storage size
* `len(text)` → code point count

Any attempt to derive one from the other is wrong once you leave single-byte encodings.

Example:

```
"café"
```

* bytes: 5
* code points: 4

Even this is still a compromise. Code points are not grapheme clusters; `"🇮🇹"` is two code points but one user-perceived character. GNU `wc` counts code points for `-m`, so that’s the contract we follow. You could choose to count graphemes instead, but that would make your tool more intuitive, and less compatible.

That trade-off (correctness vs. compatibility) is the real constraint when cloning Unix tools.

---

## Encoding is a policy decision

```python
text = raw.decode('utf-8')
```

This line bakes in a decision that the real `wc` defers.

GNU `wc` operates primarily on bytes and only interprets characters when necessary, using the current locale. This implementation hardcodes UTF-8, which means:

* invalid byte sequences will raise
* behavior diverges under non-UTF-8 locales
* “character” semantics are no longer environment-dependent

For an exercise, that’s acceptable. For parity, it isn’t.

A stricter clone would:

* consult `locale.getpreferredencoding(False)`, or
* treat decoding as a best-effort operation (`errors="replace"` or `"ignore"`), or
* avoid decoding entirely unless `-m` is explicitly requested

Each option encodes a different failure mode. There’s no neutral choice here.

---

## stdin vs arguments is part of the contract

The input model is where CLI tools either compose cleanly or become annoying.

```python
if args.file_path:
    raw = open(args.file_path, "rb").read()
elif not sys.stdin.isatty():
    raw = sys.stdin.buffer.read()
else:
    return
```

This implements the expected Unix behavior:

* explicit file → read the file
* piped input → read stdin
* interactive terminal with no input → exit

The key is `isatty()`. Without it, your program will block waiting for stdin even when the user clearly didn’t intend to pipe anything.

Also note:

```python
sys.stdin.buffer.read()
```

`sys.stdin` is a text wrapper that has already applied decoding using Python’s startup locale. That’s unacceptable if you care about byte-accurate behavior. `.buffer` exposes the raw stream.

This distinction matters anywhere you cross the text/binary boundary, not just here.

---

## Word counting is underspecified (and that’s fine)

```python
def get_words(text): return len(text.split())
```

This is intentionally naive. It treats “words” as whitespace-delimited tokens, which is not how GNU `wc` defines them in all cases, and definitely not how natural language works.

`str.split()` with no argument already splits on any Unicode whitespace, so it’s closer to right than it looks. The gaps are elsewhere:

* punctuation boundaries (`hello,world` is one token here, two to a human)
* languages without explicit word separators (CJK)
* GNU `wc`’s actual definition is locale-dependent and doesn’t map cleanly onto `split()`

A faithful implementation would need to mirror `wc`’s definition, not invent a better one. That’s a recurring theme: correctness here is conformance, not ideal semantics.

---

## Output format is part of the API

```python
if not any([args.c, args.l, args.w, args.m]):
    args.c = args.l = args.w = True
```

and:

```python
if args.l: results.append(...)
if args.w: results.append(...)
if args.c: results.append(...)
if args.m: results.append(...)
```

Two constraints are being enforced:

1. Default output includes lines, words, and bytes
2. Output order is fixed, regardless of flag order

This isn’t aesthetic, it’s compatibility. Downstream tools parse `wc` output positionally. Changing column order because it feels “more flexible” breaks pipelines.

When you clone a Unix tool, you’re not just reproducing behavior, you’re reproducing a contract that other tools depend on.

---

## Where it actually diverges

Theory is cheap. Here’s `mywc` against the real thing on a few targeted inputs.

**One character, three answers.** The string `café` is 4 user-perceived characters, 4 code points, and 5 bytes. Both tools agree, but the agreement only proves the point — `-c` and `-m` are not the same number, and treating them as interchangeable is wrong by 25% on a four-letter word.

```
$ wc -c cafe.txt && wc -m cafe.txt
5 cafe.txt
4 cafe.txt
$ mywc -c cafe.txt && mywc -m cafe.txt
5 cafe.txt
4 cafe.txt
```

**Code points are not graphemes.** The Italian flag `🇮🇹` is one thing to a human, two regional-indicator code points to Unicode, and eight UTF-8 bytes on disk. `wc -m` reports the code-point count. So does `mywc`. Neither tool reports what a user would call "one character."

```
$ wc -c flag.txt && wc -m flag.txt
8 flag.txt
2 flag.txt
```

**The strict-decode policy bites.** A Latin-1 file with three accented bytes is something real `wc` handles without comment — it operates on bytes by default and only invokes the locale when asked. `mywc` crashes:

```
$ wc latin1.txt
0 0 3 latin1.txt
$ mywc latin1.txt
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xe9 in position 0: invalid continuation byte
```

This is the actual leak. The earlier sections describe it in the abstract; this is what it looks like in practice. The hardcoded `raw.decode('utf-8')` isn’t a small simplification — it’s a policy that turns any non-UTF-8 input into a fatal error, where the reference tool would have shrugged and counted bytes.

---

## What this still gets wrong

Even within its scope, this implementation diverges from real `wc` in meaningful ways:

* Memory usage: reads the entire input into memory instead of streaming
* Encoding behavior: assumes UTF-8 instead of respecting locale
* Error handling: the strict-decode policy is a strategy, but a brittle one; any non-UTF-8 input crashes
* Multi-file support: missing aggregation and totals
* Performance: Python string operations vs byte-wise scanning

A production-quality clone would stream, avoid unnecessary decoding, and operate primarily on bytes, only lifting into text when required.

---

## Why this exercise is still useful

The code itself is trivial. The value is in the constraints it forces you to surface:

* bytes vs code points is not an academic distinction, it affects correctness
* text vs binary I/O in Python is an implicit policy decision
* CLI behavior is defined as much by ecosystem expectations as by logic
* “correct” often means “compatible”, not “ideal”

Small reimplementations like this are useful precisely because they’re constrained. You don’t get to hand-wave the boundary conditions: you have to pick a side.