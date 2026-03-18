+++
title = "Why You Shouldn't Wrap a Stream in a Transaction in Elixir"
date = "2026-03-01"
draft = false
tags = ["elixir", "ecto", "postgresql", "streams", "performance"]
categories = ["engineering"]
description = "Wrapping a database stream in a transaction seems safe. It isn't. Here's what actually happens and how to build resumable batch processing that doesn't lock your tables."
[cover]
  hidden = true
+++

When you need to process a large number of records in Elixir, the instinct is often to wrap everything in a transaction. All-or-nothing semantics, right? Safe by default. 

Turns out, this is one of those cases where the "safe" choice creates a bigger problem than the one you were trying to avoid. Here's what's actually happening under the hood, and a better approach that's both safer and more efficient.

---

## The Naive Approach

Let's say you have a batch job that needs to process thousands of files, calling an external API for each one and saving the result to the database.

```elixir
Repo.transaction(fn ->
  files
  |> Stream.each(&process_file/1)
  |> Stream.run()
end)
```

This feels right. If something fails halfway through, the transaction rolls back and nothing is committed. Clean.

But there's a problem.

---

## Transactions Hold a Single DB Connection

A PostgreSQL transaction runs on a **single connection** for its entire duration. While that connection is open, it holds locks. When you run a DB-backed stream inside that transaction, you're:

- Holding one connection open for potentially minutes (or hours)
- Blocking that connection from being used by anything else
- In many cases, locking the rows (or entire table) you're reading from

If your connection pool has 10 connections and this job occupies one for 20 minutes, that's 10% of your capacity gone. Run a few of these jobs in parallel and you start seeing timeouts and queued requests across your whole application.

The stream itself is lazy, as it fetches data in chunks, but the transaction wrapping it is not. It stays open until the stream is exhausted.

---

## The Better Approach: Paginated Streams with Per-Record Commits

Instead of one long transaction wrapping everything, process records in small batches and commit each one independently:

```elixir
files
|> Stream.chunk_every(@batch_size)
|> Stream.each(fn batch ->
  Enum.each(batch, fn file ->
    case process_file(file) do
      {:ok, result}    -> mark_as_done(file, result)
      {:error, reason} -> mark_as_error(file, reason)
    end
  end)
end)
|> Stream.run()
```

Each call to `mark_as_done/2` or `mark_as_error/2` is its own small, fast transaction. No long-lived connections, no table locks.

---

## Making It Resumable

The other advantage of this approach is **resumability**. If the job crashes at record 5,000 out of 10,000, you don't want to start over from record 1.

The key is marking each record's status as soon as it's processed:

```elixir
defp mark_as_done(file, result) do
  file
  |> Ecto.Changeset.change(%{status: "done", result: result})
  |> Repo.update!()
end

defp mark_as_error(file, reason) do
  file
  |> Ecto.Changeset.change(%{status: "error", error_message: inspect(reason)})
  |> Repo.update!()
end
```

Then when you build your stream, filter out already-processed records:

```elixir
from(f in File, where: f.status == "pending")
|> Repo.stream()
```

If the job crashes and restarts, it picks up exactly where it left off. Records marked `"done"` or `"error"` are skipped automatically. No duplicate processing.

---

## Handling Errors Explicitly

Marking errors explicitly (rather than letting them crash) is important for two reasons:

**1. It prevents infinite retry loops.** A file that fails due to a malformed payload will fail every time. If you don't mark it as an error, every job run will attempt it again, waste API calls, and potentially trigger rate limits.

**2. It gives you visibility.** You can query `where: f.status == "error"` and inspect what went wrong, rather than digging through logs.

```elixir
# Check how many files are stuck in error
Repo.aggregate(from(f in File, where: f.status == "error"), :count)
```

---

## The Schema Side: Ecto Migrations

To support this pattern, you need a `status` field (and probably a `result` field) on your table. That means a migration:

```elixir
defmodule MyApp.Repo.Migrations.AddStatusToFiles do
  use Ecto.Migration

  def change do
    alter table(:files) do
      add :status, :string, default: "pending", null: false
      add :result, :text
      add :error_message, :text
    end

    create index(:files, [:status])
  end
end
```

A few things worth noting here:

- `default: "pending"` means existing rows get a sensible starting state automatically
- The index on `:status` makes your `where: f.status == "pending"` query fast even with millions of rows
- `result` and `error_message` are nullable: only one will be populated per record

Run `mix ecto.migrate` and your Ecto schema will reflect the new structure. The **migration** changes the database, the **schema** tells your Elixir code how to interact with it: they're complementary, not redundant.

---

## Summary

| Approach | Connection usage | Resumable | Table locks |
|---|---|---|---|
| `Repo.transaction` wrapping stream | One long-held connection | No | Yes |
| Per-record commits + status field | Short, fast transactions | Yes | No |

The core insight is simple: **transactions are for atomicity, not for iteration**. When you need to process thousands of records, what you actually want is idempotency: the guarantee that re-running the job produces the same result. That comes from tracking state per record, not from wrapping everything in a single transaction.

It's a small shift in thinking, but it makes your batch jobs significantly more robust in production.
