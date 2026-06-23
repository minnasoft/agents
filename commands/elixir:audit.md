---
description: Audit Elixir/Phoenix code for refactoring and pattern opportunities
agent: staff
subtask: false
---

Audit this Elixir/Phoenix codebase area for refactoring opportunities:

$ARGUMENTS

Do not edit files.

Resolve the target as a directory, context/module namespace, feature area, test area, or whole codebase slice. If it is ambiguous, choose the smallest useful scope and state it.

Read `SKILL.md`, `core.md`, `architecture.md`, `aesthetics.md`, `audit.md`, and relevant domain files. Include `queryable.md` only when the project has a Queryable-style pattern.

Map enough shape before judging:

- modules, public APIs, submodules, and call sites;
- tests and describe blocks;
- query/schema conventions;
- resolver/controller/job boundaries if relevant;
- similar patterns elsewhere in the codebase.

Fan out to hidden `research` and `review` workers when useful, using focused audit lenses like module/API shape, Queryable/database duplication, test organization, boundary responsibilities, `defdelegate`/public API surface, and helper/docs/comment aesthetics. For non-trivial audits, include one dedicated style/prose or aesthetics reviewer plus separate domain reviewers for the highest-risk areas.

Return inspected scope, intentionally skipped scope, and opportunities ordered by leverage. Use code anchors plus ideal shape, divergence evidence, maintenance/caller cost, smallest safe migration path, confidence, and next-step judgment: `plan`, `build`, or `defer`. Reject aesthetic churn and speculative abstractions.

Audit is a discovery workflow, not implementation. If the user selects an opportunity afterward, `/elixir:plan` should refine the design when tradeoffs remain; `/elixir:build` can execute directly when the smallest safe path is obvious.
