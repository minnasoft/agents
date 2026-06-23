# Oban Review

Use this file for Oban workers and background job orchestration.

## Jobs Are Orchestrators

Workers should be thin wrappers over business logic.

They should:

- parse job args;
- call context/domain functions;
- return success/error in a way Oban can retry;
- allow crashes when stacktraces are useful;
- be idempotent or explicitly safe to retry.

They should not:

- contain complex business workflows directly;
- swallow errors that should retry;
- over-handle impossible internal states;
- obscure failures behind generic strings;
- load huge datasets into a single job.

Use `Oban.Pro.Worker` features only when they solve a real problem. Structured args, encrypted args, recorded results, chains, deadlines, hooks, and signals are useful tools, not default ceremony.

## Inserting Jobs With Database Writes

If enqueued jobs are coupled to database writes, insert the jobs in the same `Repo.transaction` as the records they depend on.

Use this when:

- creating records and enqueueing follow-up jobs for those records;
- updating state and enqueueing a job that assumes the new state exists;
- inserting a parent row and child processing jobs;
- scheduling work that should not exist if the surrounding DB changes rollback.

Flag:

- records inserted first, then jobs enqueued outside the transaction;
- jobs enqueued before the data they depend on is committed;
- partial success where DB rows exist but jobs do not, or jobs exist for rolled-back rows;
- enqueue logic hidden in callbacks where transaction boundaries are unclear.

Prefer the caller/context function owning both the database changes and the job insertion so the transaction boundary is obvious.

## Error And Retry Semantics

Because jobs have no human user, stacktraces are often helpful. User-facing niceness belongs at user-facing boundaries.

Flag:

- rescue blocks that convert useful crashes into vague errors;
- `:ok` returns after partial failure;
- ignored return values from context calls;
- non-idempotent operations in retryable jobs;
- jobs that cannot safely resume.

Prefer:

- clear crash/error tuple behavior;
- idempotent context operations;
- small orchestration around tested domain functions;
- structured args that are validated at job start.

Idempotency should usually live in the domain operation, not only in worker guard clauses. Use job uniqueness/deduplication when duplicate enqueue is possible and harmful.

Retry, discard, cancel, and snooze behavior should match the error type. Do not retry permanent validation failures forever, and do not discard transient upstream or database failures that should retry.

Scheduled jobs should remain safe if they run late, run twice, or retry after related data changes.

## Chunking And Backfills

For large work:

- chunk by IDs or query windows;
- keep jobs idempotent;
- persist progress through the job/backfill framework when appropriate;
- avoid one giant transaction/job;
- avoid loading all rows into memory.

If a job exists only to perform a data backfill, review it with `database.md` and `performance.md` too.

## Many Records And Fanout

For work across many records, prefer Oban Pro composition tools when they are available and match the shape of the work.

Use domain shards by default rather than one job per row:

- `org_id`;
- `location_id`;
- `client_id`;
- account/cohort/source IDs;
- date windows or ID ranges.

Prefer one job per row only when isolation/retry semantics clearly matter and volume is bounded.

### Batch

Prefer `Oban.Pro.Batch` for broad parallel work where many independent jobs belong to the same operation and you need group-level callbacks or progress tracking.

Batch fits:

- many independent child jobs;
- completion/discard/retry callbacks;
- group progress/status;
- callback work after all jobs complete or exhaust.

When a batch callback needs to inspect lots of jobs, consider `Batch.stream_jobs/2` inside `Repo.transaction` rather than loading every job into memory.

### Chunk

Prefer `Oban.Pro.Chunk` for throughput batching: many similar jobs should be processed together by size, timeout, and optional partitioning.

Chunk fits:

- bulk API calls;
- batched DB operations;
- high-volume ingestion;
- jobs partitioned by args/meta like `account_id`, `location_id`, or source;
- work where one process can handle a list of jobs more efficiently than one job at a time.

Use chunk result handling to mark individual jobs as errored, cancelled, discarded, or snoozed when only part of the chunk fails.

### Workflow

Use `Oban.Pro.Workflow` for multi-stage work with explicit dependencies. Workflows are DAGs, not just a nicer batch API.

Workflow fits:

- sequential stages;
- fan-out/fan-in dependencies;
- a final step that must wait for multiple upstream steps;
- dynamic sub-workflows or grafting when later work depends on data discovered mid-flow;
- durable state-machine style work with signals.

Be cautious with recorded workflow results and cascades. They can simplify dependency wiring, but avoid storing large/intermediate data in job output when IDs plus re-querying would be clearer and smaller.

### Recursive Parent Fanout

Parent jobs that recursively enqueue child jobs are acceptable for simple work, especially when Oban Pro is not available. Prefer Batch/Chunk/Workflow when available for non-trivial orchestration.

If using parent fanout, require:

- bounded fanout;
- idempotent parent and child jobs;
- child work that can retry independently;
- uniqueness or dedupe where duplicate fanout is possible;
- domain-shard granularity rather than one job per row by default;
- clear completion semantics if anything needs to happen after children finish.

If completion tracking matters, a bare parent fanout is usually the wrong abstraction. Use `Oban.Pro.Batch` if available.

## Testing Jobs

Job tests should focus on orchestration and retry/error behavior.

Prefer domain tests for the underlying business logic. Worker tests should prove the worker passes clean inputs, handles expected returns, and behaves correctly on failure/retry paths.
