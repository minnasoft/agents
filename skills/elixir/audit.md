# Codebase Refactor Audit

Use this file when asked to audit an Elixir/Phoenix codebase area for refactoring opportunities, new patterns, module splits, `defdelegate` boundaries, or test organization cleanup.

This is not PR review mode. Do not report every possible cleanup as a finding. An audit should identify high-leverage opportunities with evidence, confidence, and a safe migration path.

Use `architecture.md` for the high-level stance: reconstruct the shape you would have built first, then explain how to move the current codebase toward it incrementally. Use `aesthetics.md` to judge whether the public surface is beautiful, the sharp edges are named, and helper/module boundaries earned their names.

## Audit Workflow

Start by mapping the current shape:

- top-level context modules and submodules;
- public functions and their call sites;
- schema/query modules and query conventions;
- resolver/controller/job boundary modules;
- tests, describe blocks, and fixture/helper usage;
- similar domains that already solved the same shape better.

Use search and LSP-style navigation before judging. A refactor opportunity needs evidence from repeated patterns, caller pain, confusing boundaries, test mismatch, duplicated query semantics, or local code fighting an existing convention.

## Output Shape

Use this shape for each opportunity:

```text
[impact/confidence] code anchor - Opportunity statement.

Evidence: repeated examples, call sites, tests, or nearby better pattern.
Why it matters: concrete maintenance, testing, API, or semantic cost.
Smallest safe path: incremental move with likely module/API names.
Next step: `plan`, `build`, or `defer`, with why.
```

Anchor opportunities by module, public API, query shape, test behavior, or workflow. Use file paths when useful; do not lead with raw line numbers.

Use `plan` when the opportunity needs design discussion, ownership/naming decisions, or product semantics. Use `build` when the first slice is obvious and low-risk. Use `defer` when the opportunity is valid but not worth interrupting current work.

Impact:

- `high`: repeated shape causes broad maintenance cost, unsafe duplication, hard-to-test behavior, or semantic drift.
- `medium`: localized refactor simplifies a context/API/test area with clear evidence.
- `low`: valid cleanup with modest payoff.

Confidence:

- `high`: verified through call sites, tests, and matching project patterns.
- `medium`: evidence is strong but some product/domain semantics are unknown.
- `low`: plausible, but needs owner confirmation or more runtime/data evidence.

Reject aesthetic churn. Do not recommend a new pattern just because the current code is not your favorite shape.

## Subdomain Modules

Large contexts often start as convenient public APIs and then become junk drawers. Look for coherent clusters that want a submodule.

Flag when one broad module contains a cluster of:

- public functions around one noun/concept;
- private helpers all serving one workflow or query family;
- repeated prefixes in function names;
- resolver/controller/job code that has to know source-selection details;
- tests where several describe blocks clearly belong to the same smaller domain;
- query functions that share bindings, filters, unions, or source-specific branches.

Prefer a small subdomain module when it makes the public API easier. Default to the deepest owning layer that still works; move higher only when multiple layers genuinely share the same public concept.

```elixir
defmodule Billing.History do
  def query(scope, filters) do
    ...
  end
end
```

Then the broad context can expose only the API it actually wants callers to use.

Do not suggest a submodule for one function, one helper, or code that is easier to read inline. The split must reduce caller confusion or module sprawl, not just move lines around. Do not group code under one noun when the behaviors have different owners or reasons to change.

During build work, treat submodule extraction as a decision point:

- do it without asking when it is small, same-layer, private-ish, needed for the current fix, and clearly named;
- ask when naming/ownership is uncertain, the diff grows large, or tests need meaningful moves;
- defer when it is not needed for the current slice, but note the follow-up in the final summary or a concrete `TODO` at a real seam.

Commit boundaries should match the extraction's weight: separate commit for larger or reusable extractions, same behavior commit for tiny local extractions that make the fix clean.

## `defdelegate` And Public API Shape

Use `defdelegate` when a top-level context should present a stable public API while a submodule owns the implementation.

Good fits:

- a submodule owns a coherent domain but callers should still enter through the top-level context;
- many callers reach into `Context.Submodule.function/arity` because the top-level context lacks a forwarding API;
- the top-level function would only delegate without adapting args, errors, telemetry, or transactions;
- the delegate list is short enough to make the public API obvious.

Flag:

- sibling contexts reaching into internal submodules instead of the owning context;
- hand-written wrappers that only call a submodule with the same args;
- public APIs split across top-level and submodule call sites without a clear rule;
- `defdelegate` lists that expose an entire internal module instead of a curated API.

Prefer hand-written wrapper functions over `defdelegate` when the boundary must normalize args, enforce auth/org scope, translate errors, wrap transactions, or preserve a clearer domain name.

## Pattern Introduction

Suggest a new pattern only when local duplication proves the need.

Good candidates:

- repeated exact filters that should use an existing Queryable/schema-owned pattern;
- several resolvers/controllers doing the same boundary coercion;
- repeated batch-loading shapes that should use Dataloader/batch conventions;
- repeated job chunking/backfill shape that should use existing job/backfill infrastructure;
- repeated test setup that can become clearer fixtures/tags without hiding scenario details.

Bad candidates:

- one-off helper extraction;
- abstracting before there are real call sites;
- creating a framework because two functions look vaguely similar;
- hiding domain differences behind a generic helper.

If the codebase already has a pattern, prefer extending it over inventing a parallel one. If the existing pattern is bad, say why and propose a migration path rather than adding more exceptions.

## Query And Reporting Audits

For query-heavy modules, compare across the domain before suggesting a refactor.

Look for:

- duplicated query semantics with slightly different filters;
- reporting/export queries bypassing schema-owned filters;
- local call sites missing optimizations used elsewhere;
- nested wrapper filters that should be flat domain filters;
- resolver/job code owning joins, source selection, or aggregation shape;
- dry-run/count/run paths building different target queries.

When the project has a Queryable pattern, prefer consolidating exact filters and named joins there. Keep semantic predicates named and close to schemas/query modules unless they truly belong to a context workflow.

## Test Organization Audit

Test files should generally mirror the modules they test, unless the project has a different clear convention.

Check:

- whether each public module has a corresponding test module/file;
- whether broad context tests have become dumping grounds for several subdomains;
- whether tests for a submodule still live under the broad parent test file after implementation moved;
- whether resolver/controller tests are proving boundary behavior while context tests prove domain behavior;
- whether fixtures/helpers are hiding scenario details or making tests easier to read.

`describe` blocks should generally map 1:1 to public functions or public APIs.

Flag:

- describe blocks grouping unrelated functions;
- vague describes like `"validations"`, `"errors"`, or `"edge cases"` when a function/API name would be clearer;
- one describe covering many public functions because the module is too broad;
- tests named after implementation branches instead of behavior/invariants;
- private helper tests that make refactors brittle;
- missing tests for public functions that carry real behavior.

Accept exceptions:

- integration tests that intentionally cover a workflow across modules;
- generated/table-driven parser or converter tests;
- boundary tests grouped by route/field/event when that is the public API;
- tiny modules where a separate test file would add noise.

Do not demand mechanical 1:1 structure when it fights readability. Use it as a default audit heuristic, not a law.

## Migration Planning

Every audit recommendation should include a safe path.

Prefer incremental moves:

- introduce the submodule with one public entry point;
- move tests with the behavior;
- add top-level `defdelegate` only for the curated public API;
- update internal callers before external callers when possible;
- delete old wrappers once call sites are migrated;
- keep behavior unchanged unless the audit explicitly calls out a semantic fix.

Avoid giant rewrite recommendations. If the opportunity is large, split it into phases and name the first useful slice.
