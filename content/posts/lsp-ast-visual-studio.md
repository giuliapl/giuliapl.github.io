+++
title = "How Visual Studio Understands Your Code: LSP, Syntax Trees, and Language Services Behind the Curtain"
date = "2026-08-05"
draft = false
tags = ["lsp", "editors", "compilers", "ast", "developer-tools"]
categories = ["engineering", "systems"]
description = "What happens behind autocomplete, diagnostics, and go-to-definition: editors as clients, LSP as a communication contract, language services as the intelligence, and ctags as the older navigation index idea."
enableEmoji = true
[cover]
  hidden = true
+++

Autocomplete feels like the editor understands the code.🙂

Visual Studio, VS Code, Neovim, and Emacs know about files, tabs, selections, cursor positions, and how to draw the UI. The deeper language knowledge lives in a separate language layer: sometimes an integrated IDE service, sometimes an external language server process.

That language layer is the part that parses code, tracks names, builds semantic information, and answers questions like:

- what is under the cursor?
- where is this function defined?
- which completions are valid here?
- is this variable unused?
- what edit would safely rename this symbol?

The editor asks, and the language layer answers: when that layer is an external process speaking a standard protocol, the protocol is often the **Language Server Protocol**, or **LSP**.
So LSP is the message format that lets an editor talk to the language-specific system that has the intelligence.

---

## Start with ctags

Before LSP, a lot of editor navigation was built around tools like `ctags`. It solves a narrower problem, but it makes the broader idea easier to see.

`ctags` scans source files and writes a `tags` file: an index of named elements and their locations, especially definitions. A simplified entry looks like this:

```text
parse_config    src/config.py    line 42    function
```

An editor can load that file and jump from a name under your cursor to the line where that name was introduced.

That is already useful. It is also intentionally limited.

`ctags` is good at:

- finding definitions
- listing named elements in a project
- giving an editor a fast name-to-location index

It is not a semantic analysis system in the compiler sense. It usually cannot tell you, with complete accuracy, which overloaded method a call resolves to, what type an expression has, or whether a rename is safe across a project. Universal Ctags has grown more capable than the old versions, including richer fields and some reference tagging, but the core idea is still source navigation through a named-element/location index, not a live semantic model of your unsaved program.

That also makes `ctags` different from broader search/indexing systems like Solr or Lucene. Those systems are built for retrieving documents or records from indexed text and fields. `ctags` is narrower: it gives code editors a quick map from names to source locations.

That distinction is the bridge to LSP.

## Text is not code yet

When you write code, the editor sees text:

```python
total = price * (1 + tax)
```

For a language tool, that line needs to become structure.

First the text is split into tokens: names, numbers, operators, parentheses. Then the parser turns those tokens into a tree.

There are two related terms here:

- a **concrete syntax tree** represents the source syntax more directly; a lossless syntax tree preserves tokens, whitespace, comments, and other trivia that matter for formatting, refactors, and precise editor ranges
- an **abstract syntax tree** simplifies the source into the program structure the analyzer cares about

Different tools keep different trees. Some keep a lossless tree for editor features and a separate abstract representation for analysis. Some use one structure for both. Either way, the important step is the same: flat text becomes structured code.

The tree says something more precise than "there is a `*` and a `+` in this string." It says:

- this is an assignment
- the left side is the name `total`
- the right side is a multiplication
- the multiplication uses `price` and a parenthesized addition
- the addition uses `1` and `tax`

Once the code has that shape, semantic analysis can add more meaning:

- which identifier declaration does this use refer to?
- which names are visible in this scope?
- what type does this expression have, if the language can know that?
- which diagnostics should be reported?

From there, the tool can answer editor questions:

- which syntax node is under the cursor?
- is this name being defined or used?
- what function or class contains this expression?
- which part of the tree changed after the last edit?

So a more accurate pipeline is:

```text
text -> tokens -> parser -> syntax tree -> semantic analysis -> symbol/type model
```

The tree alone is not enough for every feature. The language layer also needs symbol tables, scopes, imports, package/module information, compiler data, caches, and sometimes type inference. But parsing is the first major step from "characters in a file" to "program with meaning."

## What LSP actually standardizes

LSP does not standardize ASTs, type systems, symbol tables, or compiler internals.

That is an important detail. There is no universal tree shape that works cleanly for Python, C++, Go, TypeScript, C#, and every other language. The protocol avoids that problem.

Instead, LSP standardizes editor-level messages and JSON-RPC shapes:

- `textDocument/didOpen`: a document was opened
- `textDocument/didChange`: the document changed
- `textDocument/definition`: find the definition at this cursor position
- `textDocument/hover`: return hover information
- `textDocument/completion`: return completion items
- `textDocument/publishDiagnostics`: show errors or warnings for this file

The important thing is not the JSON itself, but the boundary it creates.

The editor sends a request like:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "textDocument/definition",
  "params": {
    "textDocument": {
      "uri": "file:///project/main.py"
    },
    "position": {
      "line": 10,
      "character": 8
    }
  }
}
```

The server replies with editor-friendly locations. For definition requests, the LSP result can be a single `Location`, a list of `Location`s, a list of `LocationLink`s, or `null`. A simple single-location response looks like this:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "uri": "file:///project/shapes.py",
    "range": {
      "start": { "line": 3, "character": 4 },
      "end": { "line": 3, "character": 8 }
    }
  }
}
```

Notice what is missing from the response: the syntax tree, the type graph, the parser state, the internal symbol object. All of that stays inside the language server. The editor gets locations it knows how to open and display.

That is the whole design.

## Visual Studio's role

In an LSP setup, Visual Studio is the **client**.

When a relevant file opens, Visual Studio can start or connect to a language server. Early in that connection, the client and server exchange capabilities through `initialize`.

The server response to `initialize` includes capabilities, roughly like:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "capabilities": {
      "textDocumentSync": 2,
      "definitionProvider": true,
      "hoverProvider": true,
      "completionProvider": {
        "triggerCharacters": ["."]
      }
    }
  }
}
```

That means, roughly:

- I can keep documents synchronized
- I can answer go-to-definition
- I can answer hover requests
- I can provide completions, especially after `.`

For LSP-backed features, Visual Studio owns the visible experience:

- completion dropdowns
- hover popups
- red and yellow squiggles
- peek definition windows
- code action menus
- rename UI

The language server owns the meaning behind those LSP-backed features.

One accuracy note: Visual Studio also has native language services, especially for Microsoft-backed languages. C# and Visual Basic, for example, are built around Roslyn rather than being merely generic LSP integrations. Not every IntelliSense, diagnostic, navigation, or refactoring feature in Visual Studio is LSP-powered. Microsoft describes LSP support as another integration path, especially useful for language services that are not already part of Visual Studio.

## The editor buffer is the truth

A subtle part of LSP is that the server cannot rely only on files saved to disk.

When you are typing, your code may be:

- unsaved
- syntactically broken
- halfway through a rename
- missing an import you are about to add
- different from the file currently on disk

So the editor keeps the server updated.

When a file opens, the editor sends `textDocument/didOpen`, usually with the full document text. When you type, it sends `textDocument/didChange`. Depending on the server and client capabilities, that change can be the whole document or an incremental edit. Incremental sync is better for performance because the server does not need to receive the entire file after every keystroke.

The server then updates its in-memory copy and re-runs only the analysis it needs. A naive server could reprocess a lot of state. A serious one usually tries not to. It may use incremental parsing, dependency tracking, cached semantic models, background indexing, compiler graphs, or persisted symbol databases. The exact strategy depends heavily on the language and implementation.

This is why editor tooling is harder than batch compilation. A compiler can reject invalid code and stop. An editor-facing language service has to be useful while the code is invalid.

## Go to definition, step by step

Here is the flow when you Ctrl+click a name in an LSP-backed editor feature:

1. The editor records the current document URI and cursor position.
2. It sends `textDocument/definition` to the language server.
3. The server finds the syntax node at that position.
4. It decides which symbol that node refers to.
5. It finds the declaration using whatever model it has: compiler data, scopes, graphs, caches, indexes, or a symbol database.
6. It returns one or more locations.
7. The editor opens the returned location, or asks you to choose if there is more than one.

The hard part is step 4.

For a local variable, the server might only need lexical scope:

```python
def total_with_tax(price, tax):
    total = price * (1 + tax)
    return total
```

The `return total` reference points to the `total` assigned one line above.

For a method call, the server may need type information:

```python
shape.area()
```

The name `area` alone is not enough. The server needs to know what `shape` probably is. In a static language, that may come from declared types. In a dynamic language like Python, it may come from inference, type hints, imports, or best-effort analysis.

That is why go-to-definition sometimes feels perfect and sometimes feels approximate. The protocol is the same, but the language model behind it is different.

## Diagnostics are usually pushed

Diagnostics are the squiggles and warnings.

In LSP, diagnostics are commonly sent from the server to the editor with `textDocument/publishDiagnostics`. The editor does not need to understand the error. It needs the range, severity, message, and source.

Example:

```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/publishDiagnostics",
  "params": {
    "uri": "file:///project/main.py",
    "diagnostics": [
      {
        "range": {
          "start": { "line": 3, "character": 6 },
          "end": { "line": 3, "character": 11 }
        },
        "severity": 1,
        "message": "undefined name: total",
        "source": "example-lsp"
      }
    ]
  }
}
```

Visual Studio renders the squiggle. The language service decided there was a problem.

This explains a lot of small editor behaviors:

- a squiggle can appear late because analysis is still running
- a warning can disappear after analysis catches up
- rename can be shallow in one language and precise in another
- go-to-definition can land on a stub, an interface, a generated file, or nothing

The UI is only as accurate as the language service behind it.

## Why this goes beyond the curtain

The practical mental model is:

```text
Editor
  UI, files, buffers, cursor positions

Language layer
  path 1: integrated IDE language service
  path 2: external language server reached through LSP

Inside that layer
  parser, syntax trees, semantic model, diagnostics, caches/indexes

LSP
  JSON-RPC contract for path 2

ctags
  simpler named-element/location index
```

That is the part I like. The editor is not a magical code mind. It is a coordinator.

In the LSP path, your keystrokes become document-change messages and your Ctrl+click becomes a definition request. Somewhere behind the UI, a parser turns text into a tree, semantic analysis connects identifiers to symbols, and cached compiler state makes the answer fast enough to feel instant.

Once you see that, autocomplete and go-to-definition stop being black boxes. They become normal systems: a protocol boundary, parsers, semantic models, caches, symbol data, and a UI that knows how to make the result feel natural.

## Further reading

- [How VS Code Understands Your Code: Inside the Language Server Protocol](https://dev.to/archycode/how-vs-code-understands-your-code-inside-the-language-server-protocol-2gop)
- [How LSP works: Building an LSP Server from Scratch with Rust](https://www.aroy.sh/posts/lsp-deep-dive/) (useful for the protocol flow even if you skip the Rust code)
- [Universal Ctags](https://ctags.io/)
- [Visual Studio Language Server Protocol overview](https://learn.microsoft.com/en-us/visualstudio/extensibility/language-server-protocol)
- [Adding a Language Server Protocol extension to Visual Studio](https://learn.microsoft.com/en-us/visualstudio/extensibility/adding-an-lsp-extension)
- [Official Language Server Protocol site](https://microsoft.github.io/language-server-protocol/)
