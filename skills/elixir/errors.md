# Error Review

Use this file for error handling across contexts, resolvers, controllers, jobs, transactions, and domain boundaries.

## Core Rule

Prefer typed/project error structs when the project has them. Otherwise use railway-style error propagation: return `{:ok, value}` / `{:error, reason}` for expected business failures, propagate errors upward, and coerce them only at domain or user-facing boundaries.

Do not mask useful errors with vague local strings.

## Typed Errors

When a project has typed error structs or helpers, use them at boundaries that need structured errors.

Examples:

- `Error.argument_error(...)`;
- `Error.unauthorized()`;
- `Error.not_found(...)`;
- domain-specific `%MyApp.Error{code: ..., message: ...}` structs.

Flag:

- ad-hoc strings/atoms when a typed project error exists;
- inconsistent error shapes from the same context or boundary;
- user-facing boundaries returning raw internal errors;
- contexts inventing UI-specific error messages when the boundary should render/coerce.

## Railway-Style Propagation

When typed error structs do not exist, propagate expected failures upward instead of collapsing them early.

Prefer:

```elixir
with {:ok, thing} <- get_thing(id),
     {:ok, thing} <- update_thing(thing, attrs) do
  {:ok, thing}
end
```

or direct returns when the called function already has the right shape:

```elixir
def update_widget(widget, attrs) do
  widget
  |> Widget.changeset(attrs)
  |> Repo.update()
end
```

Flag:

- converting all errors to `{:error, "failed"}`;
- swallowing errors and returning `:ok`;
- rescuing exceptions only to return less useful information;
- mixing atoms, strings, changesets, and structs without a clear boundary;
- forcing everything into `{:ok, _}` / `{:error, _}` when the underlying API already has a clearer convention.

## Boundary Coercion

Only coerce errors where a domain boundary needs a specific shape.

Boundaries can be architectural layers or module boundaries, not just web layers:

- Absinthe resolvers;
- Phoenix controllers/LiveViews;
- public context APIs;
- Oban worker callbacks;
- external API/import boundaries;
- domain modules that intentionally normalize errors for callers.

At those boundaries, translate the error once into the expected project/domain shape. Do not translate the same error at every layer.

## Let It Crash

Wrong types, impossible internal states, and programmer mistakes should usually crash or fail through pattern matching. Do not add defensive branches that turn exceptional bugs into vague business errors.

Expected business failures should return error values. Exceptional internal failures should be obvious and debuggable.

## Transactions

Inside `Repo.transaction/1`, returning `{:error, reason}` does not rollback. It commits and returns an error tuple as the transaction result.

Use `Repo.rollback(reason)` or raise/crash when the transaction must abort.
