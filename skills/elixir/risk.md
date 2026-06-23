# Production Risk Review

Use this file for changes where correctness depends on existing production data, long-running data movement, background execution, or subtle domain state transitions.

Do not hardcode one company's risky domains. Discover risk from the codebase under review.

## Risk Discovery

Search for evidence before deciding what is risky:

- schemas, migrations, constraints, indexes, and data-shape comments;
- context APIs and tests around state transitions;
- existing jobs, imports, exports, syncs, and reconciliation code;
- git history and prior fixes around the touched modules;
- GraphQL/frontend usage when derived state drives UI behavior;
- reporting queries, materialized tables, counters, and denormalized fields.

Common high-risk shapes include money, ledger-like data, inventory, fulfillment, state machines, assignment/location semantics, imports/integrations, derived counters/statuses, historical/audit rows, and materialized workflow state. Treat these as examples, not a fixed list.

Flag:

- partial updates that can leave related state inconsistent;
- ignored update counts or ignored `{:error, _}` results;
- tests that only assert the call returned `{:ok, _}`;
- race-prone swaps that could be one atomic upsert/update;
- transactions that do not roll back on failure;
- stale denormalized or derived state that can make users or reports see a lie;
- behavior that depends on current data shape but lacks a migration/backfill plan.

Preferred tests assert the final domain effect, not just the function return.

## Reporting And Scale-Sensitive Queries

When a change touches reporting, analytics, audit/history views, exports, dashboards, reconciliation, or any domain query that operates over many records, do not review only the local callsite. Explore the codebase for similar queries in the same domain before accepting the shape.

Check whether the new query:

- duplicates an existing report/query with slight semantic drift;
- changes semantics that other callsites, reports, counters, or UI states assume;
- leaves out filters, joins, preload strategy, pagination, batching, or index-aware predicates that similar queries already use;
- moves reporting/query semantics into a resolver/controller/job instead of the query/domain layer;
- uses a one-off query shape where the project already has Queryable/schema-owned filters;
- builds a query that looks fine on fixtures but is likely expensive at production scale.
- adds filters, joins, or orderings without considering existing indexes and query plans.

Use existing domain queries as evidence. If nearby code is heavily optimized and the new code is not, say that directly and explain which optimization appears to matter: selective predicates, join order, `exists`, distinct/window usage, aggregate shape, avoiding preloads, batching, or pagination.

When scale is uncertain, suggest safe read-only Ecto queries the author can run against production or a production-like replica to validate counts and distribution. Keep these specific to the touched schema and avoid asking for secrets or raw data dumps.

Useful validation snippets include:

```elixir
# How many rows can this report touch?
query
|> exclude(:select)
|> select([r], count(r.id))
|> Repo.one()

# How many rows match the riskiest filter branch?
Schema.query(status: :active, inserted_after: cutoff)
|> Repo.aggregate(:count, :id)

# Are we accidentally multiplying rows through joins?
query
|> exclude(:select)
|> select([r], {count(r.id), count(r.id, :distinct)})
|> Repo.one()
```

If the project has tooling for `EXPLAIN`, query stats, replicas, or production console access, recommend using that existing path. Do not invent unverifiable performance claims; frame them as validation requests unless the query shape is clearly wrong from code.

For high-risk reports or backfills, ask for enough observability to know whether the change is safe: expected row counts, batch progress, failure counts, retry behavior, and a rollback or stop plan.

## Long-Running Data Changes

For large data movement, prefer the project's existing backfill/job framework over deployment-blocking migrations. If the project has no framework, suggest a small Oban job, script, or Mix task with the semantics below instead of stuffing large updates into migrations.

Good backfill APIs expose:

- a target query that can be inspected and bounded;
- deterministic batch ordering;
- configurable batch size;
- idempotent chunk execution;
- progress and remaining-count reporting;
- retry-safe behavior if the process crashes mid-run;
- clear completion criteria;
- separation between deploy-safe schema migrations and large data movement.

Flag:

- huge updates inside migrations;
- migrations that call current schema/context modules instead of using migration-local schemas or raw SQL;
- backfills without resume/retry behavior;
- data movement that cannot be safely re-run;
- unclear ownership between migration, backfill, and runtime job;
- jobs that query one row at a time when batch processing is possible.

## Queryable Coupling

When a project has a Queryable-style pattern, backfills and risky state changes should usually build on the same query/filter infrastructure as runtime code.

Prefer:

- schema-owned exact filters for selecting the target records;
- flat keyword filters over nested job-specific wrapper shapes;
- named/idempotent joins for association filters;
- one target query shared by dry-run/count/run paths;
- source-specific query clauses kept in the query layer, not in job orchestration.

Flag:

- backfill-specific mini DSLs that duplicate existing Queryable filters;
- dry-run/count/run paths that build slightly different queries;
- job modules owning schema association details that already belong to query modules;
- GraphQL/import/frontend-shaped filters leaking into backfill APIs;
- reporting or export code bypassing established Queryable filters and changing domain semantics without making that choice explicit.
