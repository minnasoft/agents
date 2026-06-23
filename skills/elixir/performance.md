# Performance Review

Use this file as a cross-cutting deep pass for performance-sensitive changes. Domain files also include local performance sections; this file gathers the shared lens.

## Core Performance Smells

Flag:

- N+1 queries;
- repeated DB calls that could be batched;
- per-node resolver computation;
- unbounded preloads;
- in-memory pagination over DB-backed data;
- multiple full passes where one pass or streaming would work;
- loading huge datasets into jobs or migrations;
- doing work synchronously that should be chunked/backgrounded;
- race-prone read-modify-write flows.

## Batching

Prefer APIs that can operate on many records at once.

Good shapes:

- `list_things(id: ids)`;
- `batch_things(args, parent_ids)` returning `%{parent_id => status}`;
- dataloader or Absinthe batch functions;
- `Repo.preload/2` on a collection when appropriate;
- one query to fetch referenced rows before a loop.

Avoid single-item functions inside loops unless the single-item function delegates to an efficient batch path.

## Memory

Flag:

- loading all historical associations to compute current state;
- `Repo.all` on unbounded queries;
- `Connection.from_list` for DB-backed data;
- list materialization where streams or DB-side filtering would be clearer;
- huge backfills in one process/transaction;
- passing huge data structures across process boundaries;
- repeatedly rebuilding deeply nested maps/structs when a single mutable scratchpad pass would be materially cheaper.

## Repo.stream

Use `Repo.stream` for large DB result sets where loading everything would be wasteful or dangerous. `Repo.stream` must run inside a transaction because it uses a database cursor.

Good fits:

- exports;
- backfills;
- ingestion/transformation pipelines;
- large report-ish jobs;
- one-pass processing where the full result does not need to be materialized.

Prefer shapes like:

```elixir
Repo.transaction(fn ->
  query
  |> Repo.stream()
  |> Stream.map(&...whatever transformation...)
  |> Stream.run()
end)
```

or, when accumulating is actually needed:

```elixir
Repo.transaction(fn ->
  query
  |> Repo.stream()
  |> Enum.reduce(initial, fn row, acc ->
    ...whatever accumulation...
  end)
end)
```

Do not stream just to `Enum.to_list()` a huge result anyway. That usually defeats the point unless the caller truly needs the full materialized result.

Joining/preloading while streaming can be awkward. Prefer expressing relationships in the query when it stays readable, but chunked preload can be a good pragmatic pattern:

```elixir
Repo.transaction(fn ->
  query
  |> Repo.stream()
  |> Stream.chunk_every(500)
  |> Stream.flat_map(&Repo.preload(&1, ...whatever associations...))
  |> Stream.map(&...whatever processing...)
  |> Stream.run()
end)
```

The point is to keep memory bounded while still using Ecto's preload machinery. Tune chunk size based on row size, association size, query cost, and DB pressure.

Be careful mixing `Repo.stream` with concurrent tasks. A stream transaction holds a DB connection open; spawned tasks that also need DB connections can create pool pressure or deadlock-shaped pain. Processing already-streamed data in tasks can be fine when the tasks do not touch the database and the transaction is not held open around slow external work. If DB parallelism is needed, usually shard first so each task/job owns its own short DB interaction.

## Task.async_stream

Use `Task.async_stream` for parallelism around independent work. It is useful, but review it in context rather than treating it as automatically good or bad.

Good fits:

- independent external API calls;
- file operations;
- independent CPU-ish transforms;
- work where order does not matter or is explicitly handled.

Flag:

- `Task.async_stream` inside a DB transaction when tasks touch the database, perform slow external work, or keep the transaction open longer than needed;
- unbounded/default concurrency when the work is heavy enough that scheduler, memory, API, or DB pressure matters;
- tasks doing DB reads/writes without considering pool pressure;
- parallelizing work that should be one query, `insert_all`, `update_all`, window query, or batch function;
- ignoring `{:exit, reason}` or `{:error, reason}` results;
- sending huge structs/maps into each task.

Prefer explicit options when concurrency, timeout, ordering, or failure behavior matters:

```elixir
items
|> Task.async_stream(
  fn item ->
    ...independent work...
  end,
  max_concurrency: 10,
  timeout: :timer.seconds(30),
  ordered: false
)
|> Enum.reduce(:ok, fn
  {:ok, _result}, :ok -> :ok
  {:exit, reason}, _ -> {:error, reason}
end)
```

Before suggesting tasks for DB-heavy work, ask whether the database can do it better with one query, a batch query, `insert_all`, `update_all`, a window function, or a correlated subquery.

## Process Copies And ETS Scratchpads

Cross-process work copies data. Passing large maps, deeply nested structs, or huge lists into tasks can erase the benefit of parallelism through memory copies and GC pressure.

Flag:

- `Task.async_stream` over huge full structs when each task only needs an ID or a few fields;
- capturing a huge parent structure in each task closure;
- passing large intermediate maps between processes;
- repeated immutable updates to deeply nested data structures in hot loops.

Prefer passing IDs or small payloads into tasks when the copied structures are large enough to matter. Do not contort simple code just to avoid passing normal-sized structs; convenience is often worth it until data size or hot-path behavior proves otherwise.

For very large nested transformations, consider ETS as a deliberate mutable scratchpad. This can turn repeated immutable map rebuilding into a single pass over the data structure while keeping memory churn lower.

ETS can fit when:

- the transformation is hot enough to justify complexity;
- the data structure is large/deeply nested;
- many updates target shared intermediate state;
- a single owner/process can control lifecycle clearly;
- the final result can be materialized once at the end.

Do not reach for ETS casually. It is a performance tool, not a readability tool. Require a clear reason, bounded lifecycle, and comments/tests around the tricky parts.

## Database Work

Prefer DB-side filtering, counting, ordering, and pagination when the data lives in the DB.

Do not pull data into Elixir just to filter/count/page it unless the dataset is proven small and bounded.

For reporting/history-style query domains, explicit filters can be useful even when they are not strict auth boundaries. Prefer passing the source, account, customer, timestamp, and other natural reporting dimensions through the query API when that makes the query easier to index and reason about. Frame this as reporting/indexability unless there is a concrete authorization bug.

## Background Work

For large jobs/imports/backfills:

- chunk work;
- make work idempotent;
- avoid giant transactions;
- persist enough progress to resume;
- avoid repeated lookups inside loops.

## Performance vs Abstraction

Do not write inefficient code because "it's just MVP" or "simpler." Small code can still be batch-aware. If a direct implementation would be N+1, stop and shape the API correctly.
