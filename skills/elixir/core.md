# Core Review Rubric

Use this file for every review. It defines the shared instincts that all domain files assume.

## Philosophy

Review for code that is direct, domain-shaped, layer-correct, and production-honest.

Good code makes the correct behavior obvious. Bad code often hides simple behavior behind helpers, preserves old API shapes without proof, parses framework inputs in the wrong layer, or passes tests that do not assert the real business effect.

The default bias:

- delete before adding; backwards compatibility is a discussion, not an assumption.
- inline before extracting; private helper extraction has a high bar because it can make code harder to skim.
- use existing infrastructure/abstractions/patterns before inventing local abstractions;
- model the domain honestly instead of preserving accidental shapes;
- validate assumptions through code search, tests, git history, or production facts.

These are minnasoft defaults. Prefer explicit local project conventions when they exist, unless the local convention creates correctness, performance, API, or maintenance cost.

## Priorities

Balance three priorities:

- Semantic correctness: the code must model the business/domain behavior correctly.
- Layer correctness: parsing, querying, validation, orchestration, and persistence belong in the right place.
- Minimal directness: the code should be the smallest clear change with the least unnecessary API surface.

Prefer the solution that preserves all three. If one must lose, say which one and why.

## Evidence Before Legacy Preservation

Before preserving or deleting legacy behavior, search:

- call sites;
- tests and factories;
- schemas, migrations, and backfills;
- GraphQL schema fields and frontend queries;
- git history/blame;
- production-data clues if available from code, comments, or migration history.

If evidence is unavailable, ask a specific question or leave a non-blocking note describing what evidence would settle it.

Even if legacy callsites, tests, etc exist; question if they could be removed or migrated easily. Prefer updating code over keeping legacy implementations around.

## Defensive Slop Boundaries

"Avoid defensive slop" does not mean "never handle errors." It means handle errors at the layer that owns them.

- Internal invariant violation or wrong type in business logic: let it crash or pattern match assertively.
- Expected business failure: return the domain's normal error shape, usually `{:error, reason}` or project error structs.
- User-facing resolver/controller boundary: return useful domain errors and let middleware/frameworks translate them.
- Oban/background orchestration: keep jobs light; crashes and error tuples are useful because retry and stacktraces are useful.
- External input, files, path construction, auth, permissions, imports: validate deliberately. That is not slop.

## Context Boundaries

Prefer top-level context APIs over reaching into sibling contexts' internal modules.

For example, code in `Fulfillment` should generally call a public `Inventory` API rather than reaching into `Inventory.Stock.SomeInternalModule`. The owning context should hide its storage/query/internal module layout behind its public API.

Flag:

- sibling top-level contexts reaching into each other's internal/sub-context modules;
- bypassing a top-level context API when one exists;
- coupling one domain to another domain's storage/query internals;
- adding cross-context calls that make future boundary cleanup harder.

Accept:

- aliasing structs from sibling contexts for pattern matching;
- temporary direct calls when no top-level API exists and fixing it would require rewriting the world;
- local exceptions when the review explicitly calls out the boundary debt or suggests a follow-up.

Prefer adding or using a small top-level API that preserves the owning context's boundary.

## Subdomain Shape

When a change introduces several related query functions, public APIs, schemas, resolver fields, or source-specific helpers around one coherent concept, consider whether that concept wants a small subdomain module instead of more functions in an already broad context.

Flag:

- broad contexts gaining a cluster of private helpers that all describe one new subdomain;
- public function proliferation like `created_history_query`, `updated_history_query`, and `deleted_history_query` when one scoped `history_query/2` or `History.query/2` API would be clearer;
- resolver/context code owning source selection and union/query orchestration that belongs in the domain/query layer.

Prefer one tiny public entry point with private source-specific implementations when that makes the API easier to read. Do not demand a new module for a single small function or when extraction would hide more than it clarifies.

## Review Workflow

1. Understand the domain behavior the diff claims to implement.
2. Search before preserving legacy paths or recommending deletion.
3. Look for layer leaks: request parsing in contexts, business logic in boundaries, query details far from schemas, migration logic coupled to current app code.
4. Look for slop: unnecessary helpers, wrappers, compatibility branches, shape-preserving code, caller-built helper internals, or APIs that keep old awkwardness alive.
5. Look for correctness risks: transaction rollback mistakes, ignored return values, masked errors, missing constraints, bad test assertions, N+1s, race conditions, and deploy-unsafe migrations.
6. Write findings first, ordered by severity, using code anchors instead of raw line numbers.
7. Suggest the smallest direct fix that improves the API or behavior. Prefer existing abstractions when they fit. Do not invent new abstractions unless they clearly reduce complexity.

## Output Style

For review findings (not GH PR reviews):

- Lead with bugs, regressions, production risks, and missing tests.
- Be direct and specific. Vague comments like "consider improving this" are useless.
- Ask questions only when a product/data fact is genuinely unknowable from the code.
- If something is stylistic, label it as nitpick and do not pretend it blocks the PR.
- Prefer "this leaks GraphQL parsing into the context" over "this is not clean".

Finding shape is loose, not a form to fill mechanically:

```text
[severity] code anchor - problem statement.

why it matters, if non-obvious.
smallest direct fix, preferably naming the layer/API it should move to.
```

Use module/function/field/query/loop/test behavior anchors. File paths are fine for orientation; raw line numbers are only for users who ask for them.

Severity guidance:

- `blocking`: bug, data risk, security risk, broken deploy, transaction correctness, missing essential test.
- `major`: wrong layer/API shape likely to create maintenance or caller problems.
- `minor`: readability/style inconsistency with real cost.
- `nit`: preference; should not block.

Calibrate severity by proven impact, not by reviewer taste. Do not upgrade maintainability or API-shape comments to `blocking` unless the shape causes actual broken behavior, data risk, auth exposure, deploy failure, or blocks safe future behavior immediately. If the issue is structural but non-trivial, call it a structure suggestion or `major`, not a nitpick.

## Implementation Guidance

- Default to small, direct changes.
- Delete dead code instead of preserving legacy branches without proof.
- Keep the public API pleasant even if the internal implementation needs a little work.
- Do not invent generic helpers unless there are enough real call sites and the helper improves readability.
- Add behavior-focused tests that assert final domain state.
