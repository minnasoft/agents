# Queryable Review

Use this file only for projects that implement a Queryable-style pattern. Before applying these rules, check the codebase for modules, behaviours, macros, or conventions named things like `Queryable`, `QueryHelpers`, `Schema.query/1`, `query/1`, or schema-owned filter callbacks.

If the project does not have a Queryable pattern, do not invent one just because this file exists. Fall back to `database.md` and local query conventions.

## Core Rule

When a project has Queryable-style infrastructure, boring exact filters should live in that infrastructure and near the schema/query layer.

Default to the project's Queryable pattern for:

- exact schema-field filters;
- simple `id`, `org_id`, `location_id`, `status`, `deleted`, and similar predicates;
- reusable base queries;
- query composition that belongs close to schema fields and associations.

Do not hand-roll schema-specific query helpers in contexts when the Queryable pattern already handles the case.

## Flag

- custom `query/2` callbacks that only implement exact schema-field filters already handled by Queryable;
- schema-specific helpers that duplicate generic filter behavior;
- nested `filters: [...]` APIs when flat keyword filters work;
- nested one-off filter namespaces like `history: filters` when the same behavior can be expressed as normal flat `query/2` filters;
- `Repo.get(Schema, id)` when the local pattern is `Schema.query(id: id) |> Repo.one()`;
- context code that hand-builds predicates that belong in schema/query modules;
- context functions that hand-roll exact filters instead of delegating to `Schema.query/1` or the local Queryable convention;
- query helpers that only rename obvious Queryable calls;
- new filter syntaxes that do not match the existing project convention.

## Preferred API Shape

Prefer clean context APIs that delegate to schema-owned query/filter behavior.

```elixir
def get_item(id) when is_integer(id) do
  get_item(id: id)
end

def get_item(filters) when is_list(filters) do
  filters
  |> Item.query()
  |> Repo.one()
end
```

Prefer flat keyword filters:

```elixir
Catalog.list_items(location_id: location.id, available: true)
```

over nested wrapper shapes:

```elixir
Catalog.list_items(filters: [location_id: location.id, available: true])
```

Be picky about Queryable flattening. If a context API accepts filters, prefer flat keyword filters over wrapper shapes inherited from GraphQL/frontend input. This is an API-shape finding when callers are forced to preserve nested shapes or when schema-owned filters are hidden behind one-off wrapper code. Do not frame it as a correctness bug unless the nesting actually breaks behavior.

When a filter needs a join, keep it as a normal `query/2` clause and join the named binding it needs, ideally through the project's idempotent join helper if one exists.

```elixir
{:customer_id, customer_id}, query ->
  from([order: order] in join_assoc(query, :order, as: :order),
    where: order.customer_id == ^customer_id
  )
```

## Custom Predicates

Custom query helpers are fine when they express real semantics that generic exact filters cannot.

Good reasons:

- non-trivial joins;
- named bindings to avoid duplicate joins;
- domain predicates like `active_for_billing`, `visible_to_provider`, or `current_assignment`;
- window functions, correlated subqueries, or existence checks with real semantics;
- security/org scoping that is a project convention.

Bad reasons:

- wrapping `where([x], x.id == ^id)`;
- preserving old context helper names;
- hiding one obvious query call behind a private helper;
- adding a parallel filter system;
- adding a nested mini-DSL when flat Queryable filters with source-specific clauses would work.

## Nil And Empty Filters

Filter semantics should be deliberate and consistent with the project convention.

Flag:

- empty lists where it is unclear whether they mean match none, ignore the filter, or match everything;
- `nil` values changing semantics without an explicit convention;
- `IN []` behavior that only works by accident;
- source-specific filters that cannot apply to one source but lack tests for match-none behavior.

Prefer one explicit convention per query API: ignore nils, reject nils, or treat nil as a real value when the domain needs it.

## Scope Structs

For more complex domain scoping, a typed scope struct can be useful only when the grouped values are a real domain concept. Do not create a scope struct just because passing a few keyword filters feels boring.
