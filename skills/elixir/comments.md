# Code Comments

Be conservative with comments. Comments should explain why something exists, what non-obvious business/technical context matters, or what future work is needed. Comments that restate the code are noise.

Allowed comment types:

- `NOTE`: context the reader probably cannot infer from the code alone.
- `HACK`: intentional shortcut, semantic abuse, compatibility compromise, or simple-but-not-quite-right solution.
- `TODO`: future work that can be cleaned up or ticketed.
- `WARN`: explicit warning about a sharp edge, surprising invariant, or change that can break behavior.

Do not add other tags by default. `FIXME` should usually be `TODO` or `HACK`. `PERF` and `SECURITY` should usually be `NOTE` or `WARN`. Avoid `XXX`.

## Format

Short comments:

```elixir
# NOTE: The payment provider sends this field only for legacy subscriptions.
```

Long comments align continuation text under the message:

```elixir
# NOTE: This looks redundant, but imports can reprocess the same source data
#       multiple times while cleaning mappings. Keeping the source id lets us
#       distinguish imported rows from rows created organically.
```

Use this format for all four tags:

```elixir
# TYPE: first line
#       continuation line
#       continuation line
```

## NOTE

Use `NOTE` for useful context, not narration.

Good:

```elixir
# NOTE: This must use the primary repo because the next query depends on the
#       row inserted above. A replica can lag and miss the new invoice.
invoice = Repo.one(query)
```

Good:

```elixir
# NOTE: The order of these clauses matters. Package items must be reassigned
#       before standalone items so the later position swap sees the final set.
case order_item do
  ...
end
```

Bad:

```elixir
# NOTE: Gets the invoice by id.
invoice = Billing.get_invoice(id)
```

That just repeats the code. Delete it.

## HACK

Use `HACK` when the code intentionally takes a shortcut or abuses semantics because that is simpler for now.

Add a `HACK` automatically when a change intentionally preserves weird compatibility behavior, accepts a hard product/layout limit, or avoids generalizing a mechanism that looks like it wants pagination, looping, abstraction, or a broader model. Put the comment at the policy seam, not at a random caller or template. Name the limit, why it is intentional, and what should happen if the need grows.

Good:

```elixir
# HACK: We key this by location id instead of org id because historical data
#       has duplicate external ids across locations. Remove this once the
#       migration backfills globally unique external ids.
dedupe_key = {location_id, external_id}
```

Good:

```elixir
# HACK: This relies on preload idempotency so callers can batch-preload the
#       collection while this function still works correctly for one record.
order = Repo.preload(order, :items)
```

Good:

```elixir
# HACK: This template only has two top medication labels. If label 2 ever needs
#       more overflow space, defer to product before adding pagination/new labels.
#       When there is no overflow, label 2 intentionally duplicates label 1 to
#       preserve the pre-existing label output.
```

Bad:

```elixir
# HACK: Fix this later.
```

That is not enough. Say what is hacky and what would remove it.

## TODO

Use `TODO` for concrete future work. Keep it short and ticket-shaped.

Use TODOs as breadcrumbs for architectural migrations that are intentionally split across slices. If a review/build identifies the ideal shape but the whole move is too large for the current task, leave a TODO near the seam that explains the next move.

Good:

```elixir
# TODO: Replace this legacy status mapping after all partner imports use
#       normalized invoice states.
```

Good:

```elixir
# TODO: Move this into Inventory once orders no longer call stock internals directly.
```

Bad:

```elixir
# TODO: Clean this up.
```

Too vague. Clean what up? Why? What should the next engineer do?

## WARN

Use `WARN` for sharp edges where changing something casually can break behavior.

Good:

```elixir
# WARN: Do not change this to Repo.replica(). The next step reads immediately
#       after writing and must see hot data.
assignments = Repo.all(query)
```

Good:

```elixir
# WARN: This timeout must stay below the Oban job deadline or the worker can
#       keep retrying after the upstream request has already been cancelled.
```

Bad:

```elixir
# WARN: This is important.
```

Say what breaks and why.

## Review Smells

Flag comments that:

- restate exactly what the code does;
- explain obvious syntax;
- are stale or contradicted by the implementation;
- use unapproved tags like `FIXME`, `XXX`, `PERF`, or `SECURITY`;
- say `temporary` without explaining the removal condition;
- hide confusing code instead of making the code clearer;
- are long because the code shape is bad, not because the domain context is hard.

Prefer deleting comments when the code can be made self-explanatory. Prefer adding comments when the code is already clear but the reason behind it is not.
