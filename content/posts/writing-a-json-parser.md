+++
title = "Writing a JSON Parser From Scratch: Lexer, Recursive Descent, and the Cases You Forget"
date = "2026-05-10"
draft = false
tags = ["python", "parsers", "compilers", "systems-programming"]
categories = ["engineering"]
description = "A hand-written JSON parser in Python. Two stages, a tiny token model, and a tour of the spec corners that bite you when you stop reaching for the standard library."
[cover]
  hidden = true
+++

I've been reading JSON for years without ever writing the code that parses it. That's a strange gap: it's a format we touch most days, and the one whose internals I'd thought about least. So I sat down one evening and wrote one from scratch in Python, no `json.loads`, no regex shortcuts, just the spec and a blank file.

The structure is the textbook one: a lexer that turns characters into tokens, and a recursive-descent parser that turns tokens into a tree. What I want to write about is less the structure itself (that part is well-trodden) and more the places where the spec is sharper than it looks, and where a reasonable-seeming shortcut quietly produces a parser that accepts invalid input.

---

## Stage 1: the lexer

A lexer (or "tokenizer") turns a flat stream of characters into a stream of *tokens*: atomic units like "left brace", "string literal", "number". The parser then operates on tokens instead of raw characters, which is a huge simplification: the parser never has to think about whitespace or about how `123.45e-7` is spelled.

The token model is dead simple:

```python
class Token:
    def __init__(self, type_, value=None):
        self.type = type_
        self.value = value
```

A type tag plus an optional value. Structural tokens like `LBRACE` or `COMMA` don't need a value, their type *is* the information. Tokens with content (`STRING`, `NUMBER`, `TRUE`, `FALSE`) carry their decoded value. Note that the boolean and null literals are recognized at the *lexer* level here, not the parser. They could go either way; baking them in early means the parser only sees `TRUE`/`FALSE`/`NULL` tokens and never has to do string comparisons.

The lexer's entry point takes the source text and a position, returns the next token and the updated position:

```python
def process_token(text, pos):
    pos = skip_whitespace(text, pos)
    if pos >= len(text):
        return Token("EOF"), pos

    char = text[pos]
    if char == "{": return Token("LBRACE"), pos + 1
    elif char == '"':            return read_string(text, pos)
    elif char.isdigit() or char == '-': return read_number(text, pos)
    elif text.startswith("true", pos):  return Token("TRUE", True), pos + 4
    # ... and so on
```

One subtlety: `startswith` alone isn't sufficient. `truex` will match `"true"` at position 0 and the lexer will happily emit a `TRUE` token followed by an `x` it can't classify, but the real bug is that `truex` should have been rejected outright as an invalid literal, not split into a valid token and a stray character. The fix is to also check that the character immediately after the literal is a valid terminator (whitespace, structural punctuation, or EOF), or to look up identifier-like runs and only then check whether the run spells a known literal.

The "return position alongside the value" pattern is everywhere in this code. It's the functional equivalent of a stateful cursor: instead of mutating `self.pos`, every function takes the current position and returns the new one. This makes recursion trivial (no shared state to worry about) and means each function's effect on the cursor is visible at the call site. I'd reach for a class with a `pos` field if this were a longer-running parser, but for a few hundred lines the threading style is cleaner.

### Reading strings

Strings are the largest piece of the lexer because of escape sequences:

```python
escapes = {'"': '"', '\\': '\\', '/': '/',
           'b': '\b', 'f': '\f', 'n': '\n',
           'r': '\r', 't': '\t'}
```

Plus `\uXXXX` for arbitrary code points, which has to be parsed as exactly four hex digits and converted with `chr(int(hex_digits, 16))`. Four hex digits only addresses the Basic Multilingual Plane, though. Anything above `U+FFFF` (most emoji, historical scripts, a lot of CJK extensions) is encoded as a UTF-16 surrogate pair: two consecutive `\uXXXX` escapes, a high surrogate in `D800–DBFF` followed by a low surrogate in `DC00–DFFF`, which together decode to a single code point. A fully spec-correct parser detects this case and combines them; a naive one will hand you back a string containing lone surrogates, which most downstream consumers will choke on. And, this is the part the spec is explicit about and that "good enough" parsers often miss: raw control characters (anything below `U+0020`) are *not* allowed unescaped inside a string. A literal newline character sitting between two quotes is an error; it must be written as `\n`.

```python
if ord(char) < 0x20:
    raise ValueError(f"Invalid unescaped control character U+{ord(char):04X}")
```

### Reading numbers

Numbers look easy and aren't. The grammar is precise:

- Optional leading minus.
- Either `0` or a non-zero digit followed by more digits. **No leading zeros allowed.** `01` is invalid JSON. This is the part I'd never enforced before.
- Optional fractional part: `.` followed by at least one digit.
- Optional exponent: `e` or `E`, optional sign, at least one digit.

I track an `is_float` flag while scanning so I can decide at the end whether to call `int()` or `float()` on the captured substring. JSON itself has only one number type, but Python doesn't: `1` and `1.0` compare equal but have different runtime types, and consumers downstream (a `type()` check, a serializer, a numpy cast) can absolutely tell them apart. Preserving the distinction at parse time is cheaper than reconstructing it later.

## Stage 2: the parser

With the lexer in place, the parser is a recursive descent over the token stream. A JSON text is exactly one value, and that value can take one of three shapes: a scalar, an array, or an object. Top-level dispatch in `main` is just:

```python
token, pos = process_token(text, pos)
if token.type == "LBRACE":      result, pos = parse_object(text, pos)
elif token.type == "LBRACKET":  result, pos = parse_array(text, pos)
elif token.type in SIMPLE_TYPES: result = token.value
else: raise ValueError(...)
```

After parsing the top-level value, I pull one more token and assert it's `EOF`. This matters: without it, garbage like `{"a":1}garbage` would parse the object successfully and silently drop the trailing junk.

### The array and object loops

Both `parse_array` and `parse_object` follow the same shape, which is essentially a tiny state machine:

- After consuming a value, the next token must be either a comma or the closing bracket.
- After consuming a comma, the next token must be a value (not another comma, not the closing bracket).
- The container can also be empty, closing immediately after the opener.

I track this with a flag I called `next_should_be_comma_or_ending`. When it's `True`, only `,` or `]`/`}` are valid. When it's `False`, only a value (or, on the very first iteration, the closing bracket for an empty container) is valid. This is what rules out trailing commas: `[1, 2,]` fails, because the loop sees `]` while expecting a value.

```python
if (is_first_cycle or next_should_be_comma_or_ending) and token.type == "RBRACKET":
    return arr, pos
if next_should_be_comma_or_ending and token.type == "COMMA":
    next_should_be_comma_or_ending = False
    continue
elif not next_should_be_comma_or_ending and token.type == "COMMA":
    raise ValueError("Expected value but got comma")
```

It's a little verbose. The cleaner version is probably to peek instead of consume, but threading positions makes peeking awkward without extra plumbing... You'd want a `peek_token(text, pos)` that returns the token without advancing, or a small wrapper that buffers one token. For the size of this project the flag is fine.

Recursion on nested containers is direct: a value that turns out to be `LBRACKET` or `LBRACE` calls back into `parse_array` or `parse_object`, threading position through:

```python
elif token.type == "LBRACKET":
    data, pos = parse_array(text, pos)
    arr.append(data)
```

That's the entire structural part of the parser. Recursive descent really is this clean when the grammar is well-behaved.

## The edge cases worth memorizing

If you build one of these, this is the short list of things the spec actually cares about that you might not test for by default:

1. **No leading zeros in numbers.** `01` is invalid.
2. **No trailing commas.** `[1, 2,]` is invalid. (JavaScript allows them, JSON does not.)
3. **Unescaped control characters in strings are forbidden.** A literal newline character inside `"…"` is an error; it must be written as `\n`.
4. **`\uXXXX` requires exactly four hex digits.** Three is an error; five just leaves the fifth as a normal character of the string.
5. **After the top-level value, only whitespace is allowed before EOF.** A stray comma, another value, or any other junk is an error. `{"a":1}{"b":2}` is two JSON texts, not one, and a single-document parser must reject it.
6. **Duplicate object keys.** The grammar doesn't forbid them, and most parsers accept them; but the spec leaves the resulting behavior undefined. Python's `dict` silently overwrites earlier keys with later ones, which is almost never what the producer intended. If correctness matters, detect duplicates explicitly and raise rather than relying on dict semantics.
7. **Distinguishing integers from floats.** Not a strict spec requirement (JSON has only one number type), but most consumers care.

## Why this is worth doing

Writing a JSON parser by hand has the same shape as a lot of real systems work: there's a precise external specification, the easy 80% lulls you into thinking you're done, and the remaining 20% is where the bugs live. Doing it once on something this small builds the muscle for when you have to do it on something bigger (a config-file format, a wire protocol, a query language), where you don't get to reach for a standard-library function and have to actually think about what "valid input" means.

It's also a good antidote to the temptation to reach for parser combinators or generator libraries on every problem. For a grammar this regular, plain recursive descent with a hand-rolled lexer is often shorter, faster to iterate on, and easier to debug than pulling in a generator.

## The real payoff: constrained decoding

The other reason this pays off, especially if you spend time around LLMs: the constrained-decoding systems you’ve likely used (tool calling, structured output, JSON mode, libraries like `outlines`, `llguidance`, `xgrammar`) are running a grammar-driven parser at sample time. It's the same machinery we just built, run forward alongside generation: the decoder maintains a parse state and consumes each emitted token to advance it, exactly like our parser does, but it additionally uses that state at every step to mask the model's logits so only continuations that keep the output in a valid parse state get any probability mass.

A note on the word "token," which is now doing double duty. Everything above this section was about *grammar* tokens, the `LBRACE`/`STRING`/`NUMBER` units the lexer emits. A constrained decoder operates on *model* tokens, the BPE pieces the model actually samples, and a single model token can straddle multiple grammar tokens (`,"` is often one piece, as is `":`). Real implementations therefore usually compile the grammar down to a byte-, character-, or token-aware recognizer and ask, for each candidate model token, "does the string this expands to keep us inside the grammar?" That mismatch is most of the engineering difficulty in these systems and the reason naive token-level masks don't work.

With that caveat, every edge case in the list above is a state the grammar has to track. "No trailing commas" decomposes into two distinct grammar states: after a value inside an array, both `,` and `]` are legal continuations and the model is free to pick either; but immediately after a comma, only the start of another value is legal, so `]` must be masked. The illegality of `[1, 2,]` lives entirely in that second state. "No leading zeros" means: after a leading `0` in a number context, further digits are masked, and only `.`, `e`, `E`, `,`, `}`, `]`, or whitespace can follow. "`\uXXXX` requires four hex digits" means: once `\u` is emitted, the decoder has to allow only model tokens whose expansion is consistent with the remaining hex-digit positions, re-checking after each step until all four positions are filled, at which point it returns to the ordinary string state. The grammar isn't a check that runs after generation, it's a constraint that shapes which tokens are even reachable.

Building one of these by hand is the cheapest way to develop intuition for what those systems can and can't enforce, and why they sometimes behave oddly. The failure mode to internalize: when the model puts most of its probability mass on continuations the grammar forbids, the sampler falls back to whatever legal continuations remain. The model may have had a strong opinion about the illegal path and only a weak, poorly calibrated opinion about the legal leftovers. That's how schema-constrained output ends up picking a weird-but-valid enum value instead of the obviously-intended one, or a structurally fine but semantically off field name: not because the model "changed its mind," but because the decoder forced it onto a path it barely preferred among the valid options. You read those failure modes very differently after you've written the state machine yourself.
