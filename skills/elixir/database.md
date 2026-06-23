# Database Review

Use this file for Ecto, schemas, query APIs, changesets, constraints, transactions, migrations, backfills, and persistence correctness.

## API Shape

Prefer small, pleasant APIs with one obvious path.

Flag:

- parallel APIs where one filter/options API would work;
- binary/string IDs in domain APIs only because GraphQL sends strings;
- nested `filters: [...]` APIs when flat keyword filters are enough;
- compatibility branches without proven callers or data;
- return shapes that serve implementation convenience instead of caller needs.

Prefer:

```elixir
def get_thing(id) when is_integer(id) do
  get_thing(id: id)
end

def get_thing(filters) when is_list(filters) do
  filters
  |> Thing.query()
  |> Repo.one()
end
```

Use typed scope structs when grouped values are a real domain concept, not just because passing several args feels boring.

## Query Composition

Schema modules should own query composition closely tied to fields and associations.

Flag:

- exact field filters implemented by hand in contexts;
- schema-specific query callbacks that duplicate generic filtering infrastructure;
- contexts reimplementing predicates that belong near schemas;
- query helpers that are only wrappers around obvious one-line queries.

Prefer:

- schema-owned query entry points when available;
- named bindings where they make queries composable and/or to avoid joining the same thing multiple times;
- semantic predicates only when behavior differs from exact filtering;
- flat keyword filters for simple queries.

## Window Functions And Correlated Subqueries

Use the database for relational problems it is good at. Ecto can express window functions, `exists`, correlated subqueries, lateral-style patterns, and ranked/partitioned queries; prefer those over pulling rows into Elixir and grouping/filtering manually.

Reach for window functions when the query needs:

- latest row per parent;
- first/last item within a group;
- rank, row number, dense rank, or partitioned ordering;
- per-group aggregates while still returning rows;
- deterministic tie-breaking inside grouped data.

Reach for correlated subqueries or `exists` when the query needs:

- checking whether related rows exist;
- filtering parents based on child rows;
- selecting a scalar derived from a related table;
- avoiding joins that multiply rows;
- keeping the parent query shape intact.

Flag:

- `Repo.all()` followed by `Enum.group_by`, `Enum.sort_by`, `List.first`, or `List.last` to find per-parent records;
- preloading a large association just to compute existence/count/latest state;
- joins that duplicate parent rows and then deduplicate in Elixir;
- repeated per-parent queries that could be a single window/subquery query;
- complex Elixir post-processing that is really a database ranking/filtering problem.

Prefer small, named schema query helpers around these patterns so callers do not need to understand the full window/subquery shape. Keep the helper near the schema/query layer unless the query is truly context-specific domain behavior.

When suggesting these patterns, keep snippets syntactically plausible but small. It is fine to sketch the shape with placeholders for unrelated predicates.

## Changesets And Constraints

Changesets should own persistence validation.

Prefer:

- `validate_required` and constraints over duplicating required-key checks in function heads;
- changeset errors for expected invalid data;
- database constraints for invariants the DB can enforce;
- boundary validation for argument conflicts or user-facing API misuse.

Flag:

- defensive shape clauses that restate changeset validation;
- resolver/controller code masking changeset errors with generic strings;
- validations in contexts that duplicate persistence constraints without adding domain meaning;
- missing unique/foreign-key/check constraints for DB-owned invariants.

## Repo.preload Pragmatism

`Repo.preload/2` is fine. Use it where the function needs loaded associations.

Prefer direct `Repo.preload/2` when:

- it is local and obvious;
- preloading is already idempotent/no-op when loaded;
- a wrapper would only rename `Repo.preload(order, account: [:owner])`.

Try not to use a named preload helper unless:

- the preload has real domain meaning;
- multiple call sites benefit;
- it hides complexity rather than hiding a single obvious call.

Functions should generally own their own preload needs. If a function needs an association to behave correctly, preload it locally instead of making every caller know that internal requirement.

It is fine for multiple clauses to call `Repo.preload/2`. Preloads are idempotent, so repeated local preloads are usually better than coupling callers to a function's internal data needs.

Caller-side preloading can still be a useful performance optimization, especially when preloading an entire collection before passing records into a helper. Treat that as an optimization, not a contract. The helper should still preload what it needs internally so single-record callers remain easy and correct.

## Replicas And Read Freshness

Prefer read replicas for queries when the project has a replica convention and the caller does not need hot/read-after-write data.

Use the project's established API, such as `Repo.replica()` or the local equivalent. Do not invent a new replica abstraction in a review suggestion unless the project lacks one and the diff is already about that infrastructure.

Prefer replicas for:

- read-only list/count/report/export queries;
- background reads where slight replica lag is acceptable;
- expensive dashboard or analytics reads;
- validations or lookups that do not depend on a just-written value.

Use the primary repo for:

- reads inside transactions;
- read-after-write flows that need fresh data;
- locks or queries that coordinate writes;
- user-facing flows where stale data would be confusing or incorrect;
- constraint-sensitive checks that must reflect the current primary state.

Flag read-heavy code that defaults to primary when a replica is available and freshness is not required. Also flag code that uses a replica for hot data where lag can create correctness bugs.

## Transactions

Transactions are database correctness, not just performance.

Use transactions when multiple operations must commit or fail together. Do not use them as decoration around one DB statement.

Prefer a plain `Repo.transaction(fn -> ... end)` for transactional workflows. Treat `Ecto.Multi` as banned in this review style; it usually adds ceremony and indirection instead of clarity. If a workflow feels like it needs named steps, first try a plain transaction with direct helper calls and explicit rollback/error handling.

Flag:

- `Repo.transaction(fn -> {:error, reason} end)` when the author expects rollback;
- ignored `Repo.update/insert/delete` results inside a transaction;
- `Enum.each` when individual failures must abort the workflow;
- partial updates around multi-step operations;
- race-prone read-modify-write flows that could be atomic updates/upserts.

Inside `Repo.transaction/1`, rollback requires one of:

- `Repo.rollback(reason)`;
- raising/crashing.

Returning `{:error, reason}` from inside a transaction is a successful transaction returning an error tuple. That distinction matters.

Example:

```elixir
Repo.transaction(fn ->
  case Repo.update(changeset) do
    {:ok, record} -> record
    {:error, reason} -> Repo.rollback(reason)
  end
end)
```

If the project has a transaction wrapper, prefer that when it preserves rollback semantics clearly.

## Migrations And Backfills

Migrations need to be deploy-safe and future-proof.

Prefer:

- raw table/column operations over current schema modules;
- small schema migrations separated from large data backfills;
- chunked jobs/backfill frameworks for large or long-running data updates;
- idempotent data operations when re-runs are possible;
- explicit rollback/irreversibility decisions.

Flag:

- schema modules referenced in migrations when future schema changes can break old migrations;
- long-running data updates in deployment-blocking migrations;
- backfills that load too much data into memory;
- migrations that assume current app code will always match old database shape;
- migration changes unrelated to the current move/refactor.

## Database Performance Lens

Flag:

- repeated queries that could be one query;
- unbounded preloads;
- loading all historical associations to compute one current value;
- in-memory pagination over DB-backed data;
- missing batch APIs for list-of-ID lookups;
- multiple full passes over data when one pass or streaming would work.

For reporting or large-table changes, also check:

- new filters, joins, and orderings have index-aware query shapes;
- common foreign-key joins have supporting indexes when the project expects them;
- large `update_all`, `delete_all`, or `insert_all` calls check returned counts when correctness depends on affected rows;
- large-table indexes use the project's concurrent/index-lock-safe migration pattern where available;
- raw SQL and fragments are parameterized rather than interpolating external strings.

Prefer batch functions, list filters, `Repo.preload/2` where appropriate, and DB-side filtering/counting/pagination.
