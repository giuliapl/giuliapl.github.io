+++
title = "How AI Coding Assistants Use Code Context: LSP, ASTs, Search, and Retrieval"
date = "2026-08-11"
draft = false
tags = ["ai", "llms", "lsp", "ast", "developer-tools", "retrieval"]
categories = ["engineering", "systems"]
description = "How AI coding assistants combine language models with code search, syntax trees, language servers, diagnostics, tests, and retrieval systems instead of relying only on autocomplete."
enableEmoji = true
[cover]
  hidden = true
+++

In the previous post, the important idea was that an editor is not a magical code mind...

Visual Studio, VS Code, Neovim, and Emacs know how to display files and react to cursor positions. The deeper code knowledge usually lives in language services: parsers, syntax trees, symbol tables, type checkers, diagnostics, caches, and sometimes language servers speaking LSP.

That explains how an editor can answer precise questions:

- what symbol is under the cursor?
- where is it defined?
- what type does this expression have?
- which references should change during a rename?
- where should this diagnostic squiggle appear?

AI coding assistants start from a different place.

A model like Claude, ChatGPT, Codex, or the model behind an editor assistant is not a language server. It does not automatically have the compiler's full internal state just because you ask it a question. A plain language model receives text as input and produces text as output.

But serious coding assistants are not just a model sitting in an empty chat box. They become useful when they are surrounded by tools that can gather code context, search the repository, inspect files, run tests, read diagnostics, and sometimes ask language services for exact answers.

So the more interesting mental model is:

```text
AI coding assistant
  language model
  + repository search
  + file reads
  + syntax-aware parsing
  + LSP or IDE signals
  + diagnostics and tests
  + edit application
```

The model is still probabilistic. The surrounding tools are what keep it attached to the real codebase.

---

## Start with the difference

An LSP server is built to answer narrow, structured questions.

If the editor asks:

```text
definition at file:///project/app.py line 27 character 14
```

the language server can answer with a location:

```text
file:///project/models.py line 8 character 6
```

That is a precise editor operation. The result is not prose. It is a location the editor can open.

An AI assistant is usually asked a messier question:

```text
Can you add pagination to this endpoint and update the tests?
```

That request contains intent, not a protocol method. To do it well, the assistant needs to discover several things:

- where the endpoint lives
- what framework the app uses
- how request parameters are parsed
- how responses are shaped
- where tests are located
- which test style the project already uses
- whether pagination helpers already exist

Some of those answers may come from text search. Some may come from reading files. Some may come from an AST or language server. Some may come from running tests and seeing what breaks.

That is the core difference:

```text
LSP:
  exact request -> structured answer

AI coding assistant:
  human intent -> context gathering -> probable plan -> edits -> verification
```

The AI layer is broader, but less inherently exact.

## Text search is still powerful

The first tool an AI coding assistant needs is usually not an AST. It is search.

If the user asks:

```text
Where is the billing webhook handled?
```

a simple search for `webhook`, `billing`, `stripe`, `invoice`, or `checkout.session.completed` may find the right files faster than any complex semantic analysis.

This is where tools like `grep`, `ripgrep`, `ctags`, Solr, Lucene, Elasticsearch, or custom code indexes enter the picture. They are not all the same kind of tool, but they share one important idea: they make a large body of text searchable.

There are different levels of indexing:

- `grep` and `ripgrep` scan text directly
- `ctags` indexes named code elements and their locations
- Solr and Lucene-style systems index documents, fields, terms, and metadata
- vector databases index embeddings for semantic similarity
- language services build semantic models of code

Each index answers a different kind of question.

If the assistant needs the literal string `DATABASE_URL`, full-text search is excellent. If it needs every function named `connect`, a symbol index can help. If it needs "the code that handles failed payments," embedding search may surface relevant files even when those exact words are not present. If it needs to know whether `user.id` is an integer or a UUID, a type-aware language service is a better source.

This is why "AI code search" is not one thing. A useful assistant often needs hybrid retrieval: exact text search, semantic search, and code-aware navigation.

## Why ASTs matter

An abstract syntax tree gives the assistant structure.

Without a tree, code is just text:

```python
def total_with_tax(price, tax):
    total = price * (1 + tax)
    return total
```

With a tree, a tool can say:

- this file contains a function definition
- the function is named `total_with_tax`
- it has parameters `price` and `tax`
- the `return` statement returns the local name `total`
- the assignment creates or updates that local name

That structure matters for AI in a few practical ways.

First, ASTs make chunking better.

A retrieval system that splits code every 1,000 characters can cut a function in half. That is bad context. A syntax-aware chunker can keep functions, classes, methods, imports, and docstrings together. The model receives coherent units instead of arbitrary slices.

Second, ASTs make edits easier to localize.

If the assistant is changing one method, a parser can help identify the exact method body rather than asking the model to reason from line numbers alone.

Third, ASTs make refactors less reckless.

Renaming every occurrence of the text `user` is dangerous. A syntax-aware tool can distinguish a variable name from a string literal, comment, property name, import alias, or unrelated symbol in another scope.

Fourth, ASTs help with summaries.

Instead of summarizing a whole file as one blob, a tool can extract:

```text
class InvoiceService
  method create_invoice
  method mark_paid
  method retry_failed_payment
```

That becomes compact context for the model.

The tree does not make the model "understand" the project by itself. It gives the assistant a better map.

## Where LSP fits

LSP is useful because it exposes language-service intelligence through a standard editor-facing protocol.

An AI coding assistant can benefit from the same kinds of answers an editor uses:

- `textDocument/definition`: where is this symbol defined?
- `textDocument/references`: where is this symbol used?
- `textDocument/hover`: what type or documentation appears here?
- `textDocument/documentSymbol`: what symbols are in this file?
- `textDocument/rename`: what edit set would safely rename this symbol?
- `textDocument/publishDiagnostics`: what errors or warnings does the language service report?

The key point is that LSP is not the intelligence itself. It is the contract for asking the language service questions.

For an AI assistant, that contract can reduce guessing.

Suppose the model sees:

```typescript
const result = await client.search(query)
```

The word `search` is not enough. There may be many `search` methods in the repository. A language server may be able to resolve the exact method based on imports and types. That is a better answer than asking the model to infer from nearby text.

The assistant can then use the result as context:

```text
The call client.search(query) resolves to SearchClient.search in src/search/client.ts.
```

Now the model can reason over a fact produced by code tooling instead of inventing one.

## The model is the planner and writer

If deterministic tools are so useful, why involve a language model at all?

Because most programming requests are not just protocol calls.

This request:

```text
Add pagination to the customer list endpoint.
```

does not map cleanly to one compiler operation. It involves product intent and code judgment:

- Which endpoint is "the customer list endpoint"?
- Does the API already have a pagination convention?
- Should the parameters be `page` and `per_page`, or `limit` and `offset`?
- What should the default limit be?
- Should the response include total count?
- Which tests should change?
- Does documentation need updating?

An LSP server does not decide those things. It can provide facts, but it does not understand the user's goal as a software change.

The language model is good at the flexible part:

- interpreting vague intent
- comparing nearby patterns
- drafting code in the project's style
- explaining tradeoffs
- producing patches
- updating related tests and docs

The deterministic tools are good at grounding:

- find the files
- resolve the symbols
- report type errors
- run the tests
- show the diff
- verify the behavior

The strongest assistant is not "LLM instead of compiler." It is closer to:

```text
LLM as planner and editor
compiler/language server/search/tests as reality checks
```

## Retrieval is context engineering

Language models have context windows, not permanent automatic access to a whole repository.

Even when a model can accept a lot of text, feeding it an entire codebase is usually wasteful. Most of the code is irrelevant to the current task. Some of it may distract the model. Some of it may be generated or vendored code that should not influence the edit.

So the assistant needs retrieval.

For code, retrieval is harder than document Q&A because relevance is not only semantic. Exact names matter.

If the user asks about:

```text
JWT_REFRESH_EXPIRY_SECONDS
```

embedding search might understand the general topic of authentication, but exact text search is the safer first move. If the user asks:

```text
Where do we expire old login sessions?
```

semantic search may help even if the code uses names like `prune_tokens`, `session_ttl`, or `delete_stale_auth_records`.

This is where Solr/Lucene-style indexing fits the story. These systems are very good at scalable text retrieval: terms, fields, filters, ranking, metadata, and query syntax. In a code assistant, that style of indexing can complement vector search and language-server navigation.

A practical retrieval stack might combine:

- exact search for names, errors, flags, and constants
- symbol search for functions, classes, and methods
- AST-aware chunking so retrieved code is not torn apart
- vector search for concept-level similarity
- dependency graphs for related files
- diagnostics and test output for current failures

None of these replaces the model. They decide what the model gets to see.

Bad context produces bad edits. Good context gives the model a chance.

## A search app example

I worked on a search-heavy document application where this distinction became very concrete.

The app was not an AI coding assistant. It was a normal web application with users, permissions, document upload, metadata filters, autocomplete, and search results. But the retrieval problems were the same kind of problems an AI coding assistant has to solve before it can answer anything useful.

There were two different search paths.

One path used Solr for document retrieval. User-facing filters became indexed fields: document type, title, category, year range, location, visibility, and sort order. The app did not fetch the entire document corpus. It asked Solr for specific fields, a limited number of rows, a sort order, and a cursor for pagination. For detail pages, it also asked Solr for highlighted snippets around matching terms.

That is a very different shape from "load all documents and let the model figure it out."

The other path used Postgres for keyword suggestions. Exact substring matches were useful, but not enough, because users mistype things and search terms vary. So the app also used trigram similarity through `pg_trgm`, merged exact and fuzzy matches, removed duplicates, and returned a small ranked list for autocomplete.

The pattern looked roughly like this:

```text
document search:
  Solr query
  + structured filters
  + selected fields
  + sorting
  + cursor pagination
  + highlighted snippets

keyword suggestions:
  exact database match
  + trigram similarity
  + threshold
  + dedupe
  + limit
```

That kind of system is a good analogy for AI coding tools. Before a model writes an edit, something has to retrieve the right context. Sometimes the right context is a document chunk. Sometimes it is a symbol definition. Sometimes it is an exact string match. Sometimes it is a fuzzy match that catches the thing the user meant, not the exact thing they typed.

The retrieval layer is already making product decisions before the model sees anything.

## Diagnostics are feedback

Part one described diagnostics as squiggles pushed by the language server to the editor.

For AI coding assistants, diagnostics are also feedback.

The assistant might make an edit, then ask the project what it thinks:

```text
run typecheck
run tests
read diagnostics
inspect failing files
patch again
```

That loop matters because generated code often looks plausible before it is correct.

A model may write:

```typescript
return customers.map(CustomerPresenter.render)
```

but the project may expect:

```typescript
return customers.map((customer) => CustomerPresenter.fromModel(customer))
```

The difference may be obvious to the type checker. It may be obvious to a failing test. It may not be obvious from the prompt alone.

This is another reason AI coding tools should not be judged only by the first answer they produce. The more important question is whether they can observe the result, use deterministic feedback, and correct the edit.

## Why AI still gets code wrong

Even with ASTs, LSP, search, and tests, AI assistants still make mistakes.

There are several reasons.

The retrieved context may be incomplete. The relevant file might not have been opened or indexed. A generated file might hide the real source of a type. A framework convention may live outside the repository.

The language service may be approximate. Dynamic languages, macro systems, generated code, conditional imports, plugins, and partial type annotations can make exact analysis hard.

The tests may be thin. Passing tests do not prove the change is correct. They only prove the checked behavior still passes.

The model may overgeneralize. It may see one pattern and apply it where another local convention would be more appropriate.

The user request may be underspecified. "Make this faster" or "clean this up" can imply many different acceptable edits.

So the right expectation is not perfection. The right expectation is a workflow:

```text
gather context
make a bounded change
run checks
inspect failures
adjust
show the diff
```

That is much closer to how a careful human works too.

## A concrete example

Imagine asking an assistant:

```text
Rename AccountManager to CustomerAccountService.
```

A weak assistant might search for the text `AccountManager` and replace it everywhere.

That can break things:

- strings used in logs
- database migration names
- serialized class names
- documentation examples
- unrelated comments
- generated files
- imports that need path changes

A better assistant asks the codebase for structure.

It may use a language service rename operation if the language supports it. That returns an edit set based on symbol identity, not raw text. Then it may run tests and type checks. Then it may separately decide whether human-facing strings or docs should change.

The distinction matters:

```text
symbol rename:
  change references to this exact code symbol

text replacement:
  change matching characters wherever they appear
```

AI does not remove that distinction. It makes the distinction more important, because the assistant can act across many files quickly.

## The practical mental model

Part one ended with this shape:

```text
Editor
  UI, files, buffers, cursor positions

Language layer
  parser, syntax trees, semantic model, diagnostics, caches/indexes

LSP
  JSON-RPC contract between editor and language server

ctags
  simpler named-element/location index
```

For AI coding assistants, I would extend it like this:

```text
AI coding assistant
  model
    interprets intent
    proposes changes
    writes explanations and patches

  retrieval layer
    exact search
    symbol indexes
    Solr/Lucene-style text indexes
    vector search
    AST-aware chunking

  code intelligence layer
    parser
    syntax trees
    semantic model
    language server / IDE APIs
    diagnostics

  verification layer
    formatter
    linter
    type checker
    tests
    build
    git diff
```

The model is the flexible part. The rest of the system is how it stays connected to the actual program.

That is the important shift from autocomplete to agents. Autocomplete predicts the next piece of code from local context. An agent can gather context, choose tools, edit files, run checks, and iterate.

But the old machinery did not disappear. Parsers, ASTs, symbol tables, language servers, text indexes, diagnostics, and tests became more valuable, not less.

The better AI coding tools get, the more they look like orchestration layers over the systems developers already trust.

## Further reading

- [How Visual Studio Understands Your Code: LSP, Syntax Trees, and Language Services Behind the Curtain](/posts/lsp-ast-visual-studio/)
- [Official Language Server Protocol site](https://microsoft.github.io/language-server-protocol/)
- [Universal Ctags](https://ctags.io/)
- [Apache Solr](https://solr.apache.org/)
- [Apache Lucene](https://lucene.apache.org/)
