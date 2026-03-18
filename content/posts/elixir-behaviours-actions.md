+++
title = "Designing Extensible Action Pipelines in Elixir with Behaviours and Dynamic Dispatch"
date = "2026-02-21"
draft = false
tags = ["elixir", "architecture", "behaviours", "design-patterns"]
categories = ["engineering"]
description = "How to build a polymorphic, extensible action system in Elixir using Behaviours and dynamic module dispatch, without coupling your orchestrator to every action it runs."
[cover]
  hidden = true
+++

When you're building a system that needs to execute many different "actions" (think AI-powered pipelines, data processing jobs, or any multi-step business logic) you quickly face a design question: how do you keep things extensible without coupling everything together?

In Elixir, the answer often comes down to two powerful primitives: **Behaviours** and **dynamic dispatch via atoms**. Let me walk you through an architecture that elegantly solves this problem.

---

## The Problem: Many Actions, One Orchestrator

Imagine you have an application that runs multiple independent actions, and that each one takes some input, does some processing, and produces a formatted output saved to a database. You could write each action as a standalone module with its own logic, but then whoever orchestrates them needs to know about every single one. That's tight coupling, and it makes adding new actions painful.

The classic solution in OOP would be an interface. In Elixir, we have **Behaviours**.

---

## Behaviours = Interfaces in Elixir

A Behaviour defines a *contract*: a set of callbacks that any module implementing it must provide. The key insight is that Elixir will warn you at compile time if you forget to implement one.

```elixir
defmodule MyApp.ActionRunner do
  @callback get_action_id() :: atom()
  @callback run(data :: map(), config :: map(), opts :: keyword()) :: {:ok, map()} | {:error, term()}
end
```

**BEHAVIOUR = INTERFACE**: it specifies *what*, not *how*.

Any module that declares `@behaviour MyApp.ActionRunner` must implement all these callbacks. The compiler enforces it, you can't ship broken code silently.

---

## Dynamic Dispatch: Modules Are Atoms

Here's where Elixir gets interesting. Modules are *first-class atoms*, meaning you can store a module name in a variable, pass it around, and call functions on it dynamically:

```elixir
action_module = MyApp.Actions.ProcessData
action_module.run(data, config, opts)
```

This enables a powerful pattern: an **orchestrator** that doesn't need to know which action it's executing. Instead, it looks up the right module at runtime via a registry:

```elixir
defmodule MyApp.Orchestrator do
  def run(action_name, data, config, opts) do
    action_module = MyApp.ActionRegistry.get(action_name)
    # action_module is an atom -> e.g. MyApp.Actions.ProcessData
    action_module.run(data, config, opts)
  end
end
```

This is **dynamic dispatch**. The orchestrator calls `run/3` without knowing which action it is. The registry resolves the atom, and Elixir calls the right module. Polymorphism without inheritance.

---

## The Full Architecture

```
API Request
      │
      ▼
  Orchestrator
  (loads context: data, config, user preferences)
      │
      ▼
  ActionRegistry
  (resolves action_name → module atom)
      │
      ▼
  Action Module (implements @behaviour ActionRunner)
  ┌─────────────────────────────────┐
  │ get_action_id/0                 │
  │ run/3             → core logic  │
  └─────────────────────────────────┘
```

The orchestrator loads all context (data, config, preferences), then delegates the actual work to whichever action module the registry returns. Adding a new action is a self-contained change: one new module, no changes to the orchestrator.

---

## Why Behaviours Over Plain Functions?

You could just define a map of `action_name => function` and call it a day. But Behaviours give you three things functions alone don't:

- **Compile-time safety**: missing a callback is a compiler warning, not a runtime surprise
- **Polymorphism**: the orchestrator can call any action the same way, regardless of what it does internally
- **Discoverability**: any developer reading the Behaviour immediately knows the full contract an action must fulfill

```elixir
defmodule MyApp.Actions.ProcessData do
  @behaviour MyApp.ActionRunner

  @impl true
  def get_action_id(), do: :process_data

  @impl true
  def run(data, config, _opts) do
    # some logic
    {:ok, %{result: "processed"}}
  end

  # ... other callbacks
end
```

The `@impl true` annotation tells the compiler (and future readers) that this function is explicitly implementing a Behaviour callback, not just a coincidentally named function.

---

## Summary

| Concept | Elixir Primitive | What it gives you |
|---|---|---|
| Interface / contract | `@behaviour` | Compile-time safety, polymorphism |
| Dynamic dispatch | Module atoms | Decouple orchestrator from actions |
| Registry | Map/ETS | Resolve action name → module |

The beauty of this pattern is that complexity scales horizontally: adding a new action is entirely self-contained. The orchestrator, registry, and DB layer don't need to change. That's the kind of extensibility worth designing for upfront.

In the next post, I'll cover a tricky problem that comes up once you start running these actions over large datasets: **why wrapping a stream in a transaction will come back to bite you**, and what to do instead.