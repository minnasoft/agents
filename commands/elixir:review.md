---
description: Fanout Elixir/Phoenix review with critical consolidation
agent: staff
subtask: false
---

Review this Elixir/Phoenix target:

$ARGUMENTS

Do not edit files.

Resolve whether the target is a PR, branch/ref/range, commit, file, line, pasted hunk, or topic. If it is ambiguous, make the smallest reasonable assumption and state it.

Use the `elixir` skill as minnasoft's review rubric. Include `architecture.md` and `aesthetics.md` only when shape, boundaries, helper extraction, docs/comments, or public API taste matter.

Target resolution:

- if a PR number is given, inspect its base/head diff;
- if a ref/range is given, use that exact range;
- if the user asks for the current diff, inspect staged and unstaged changes;
- if a file, line, hunk, or topic is given, keep the review scoped unless verification requires nearby code.

Fan out to hidden `review` workers when useful, and by default for non-trivial diffs. Include one dedicated style/prose reviewer. Add separate domain reviewers for the touched risk areas: data/query, boundary/API shape, tests, runtime/performance/security, migrations/backfills, jobs, GraphQL/HTTP, or architecture. Critically consolidate worker feedback and reject stale, speculative, over-severe, duplicated, or convention-confused claims.

Return findings first, ordered by severity, with code anchors and smallest direct fixes. For inline GitHub comments, follow `prose.md`. If there are no findings, say what was checked.

Review is primarily for PRs, diffs, ranges, and other people's work. Do not turn it into a refactor wishlist. When fixes are useful, include a short handoff describing what `/elixir:build` should apply, and use `/elixir:plan` first only when the fix needs a design decision.
