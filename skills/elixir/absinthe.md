# Absinthe Review

Use this file for GraphQL schemas, resolvers, dataloader, batch resolvers, connections, mutation return shapes, and GraphQL boundary errors.

## Resolver Boundary

Resolvers are user-facing orchestration boundaries.

They should:

- parse/coerce GraphQL-specific shapes;
- translate string/global IDs into domain-native values;
- enforce or initiate auth/org scoping where the boundary owns it;
- return useful user-facing errors in the project's normal error shape;
- delegate business behavior to contexts;
- stay thin enough that most behavior is tested lower down.

They should not:

- implement domain logic;
- perform schema/database query composition;
- mask context errors with vague strings;
- pass GraphQL wrapper shapes like `input: %{...}` or `filters: %{...}` unchanged into contexts when the context wants flat attrs/filters;
- duplicate resolver wrappers that only call another resolver.

Passing top-level args directly into a context can be fine when those args already match the context API. The smell is preserving GraphQL-specific nesting or frontend-shaped arguments past the resolver boundary.

Prefer:

- flattening `filters: %{location_id: id}` into `location_id: id` when the context API is filter-based;
- binding `input` to `attrs` for create/update functions when the context naturally accepts attrs;
- parsing nested/global IDs before calling the context;
- keeping UI-only or GraphQL-only wrapper fields out of context APIs.

For GraphQL connections, keep connection/pagination concerns at the GraphQL boundary unless the context already exposes a domain-native paginated API. Prefer resolver-owned coercion from GraphQL args into flat domain filters, then delegate to a context/query API.

## ID Coercion

GraphQL sends strings. Contexts should not care.

Flag:

- context functions accepting binary IDs only because GraphQL needs them;
- `String.to_integer` or global-ID parsing inside contexts;
- permissive overloads that accept both strings and integers without a non-GraphQL caller.

Prefer parsing at the resolver boundary and passing integer IDs or clean filters into contexts.

## Schema Shape

GraphQL should feel like a graph, not a bag of RPC endpoints.

Flag:

- field names like imperative commands;
- exposing internal filters/scope structs the frontend does not need;
- separate fields that differ only by pagination shape;
- mutation return values that do not help the client update cache/UI;
- strings like `"ok"` when returning the changed object is more useful.

Prefer object relationships, typed return objects, and one coherent API surface.

## Errors

Flag:

- `{:error, :unauthorized}` or ad-hoc strings when project error helpers exist;
- resolvers converting all context failures into `"failed to do thing"`;
- returning non-user-facing internal errors directly when the boundary should shape them.

Prefer structured project errors where available, such as argument errors and unauthorized errors. Let middleware/framework code translate domain errors instead of inventing local response shapes.

## Dataloader, Batch, And Connections

Flag:

- field resolvers that run one query per parent;
- status/count resolvers that recalculate per node;
- `Connection.from_list` for DB-backed data because it paginates an already-materialized list instead of pushing pagination to the database;
- loading all records into memory to paginate;
- resolvers with per-parent preload/query behavior.

Prefer:

- dataloader for associations;
- dataloader by schema/id for synthetic map nodes that carry foreign keys;
- Absinthe `batch/3` for computed fields that do not map cleanly to dataloader;
- context batch functions that accept list IDs and return maps only when the field is genuinely computed or cannot be expressed through dataloader cleanly;
- DB-backed pagination;
- resolvers that minimally adapt batch results.

Check the project's dataloader convention before accepting custom batch resolver code. If a resolver only batches `id in ids` for normal schemas, prefer existing dataloader helpers or manual schema/id loading:

```elixir
loader
|> Dataloader.load(:db, Event, event_id)
|> on_load(fn loader ->
  {:ok, Dataloader.get(loader, :db, Event, event_id)}
end)
```

Batch loaders and dataloader sources still need the same auth/org/account scoping as normal queries. Do not accept a batch helper that bypasses the boundary's scoping just because it avoids N+1s.

For Absinthe enums backed by existing domain/query string values, prefer `value(:thing, as: "thing")` over resolver functions that convert strings to atoms. Remember enum input args also arrive as the configured internal values.

Example batch shape:

```elixir
def batch_event_statuses(args, event_ids) do
  args
  |> Events.status_query(event_id: event_ids)
  |> Repo.all()
  |> Map.new(fn row -> {row.event_id, row.status} end)
end
```

## Testing Absinthe Code

Heavy behavior tests usually belong at the context/domain layer when resolvers are thin.

Resolver tests should cover:

- argument coercion;
- auth/org boundary behavior;
- GraphQL return shape;
- error translation;
- integration between field shape and context API.

Do not duplicate every context behavior through GraphQL unless the resolver materially changes behavior.

For GraphQL fields over domain/query behavior, expect context tests for semantics and resolver tests for boundary wiring. Resolver tests should prove enum coercion, argument flattening, auth/org boundary behavior, connection shape, and field loading; they should not be the only proof of complex query semantics.
