---
name: elixir
description: Use when planning, reviewing, auditing, or building Elixir, Phoenix, Ecto, Absinthe, or Oban code with minnasoft's conventions for layer correctness, query/API shape, refactor opportunities, errors, tests, transactions, migrations, performance, and defensive slop.
compatibility: opencode
---

# Elixir

Use this skill when asked to plan, review, audit, or build Elixir/Phoenix/Ecto/Absinthe/Oban code with minnasoft's opinionated engineering conventions.

The goal is to catch code that is awkward, over-defensive, layer-confused, semantically weak, inefficient, or insufficiently tested.

These are minnasoft conventions, not universal Elixir law. Check the local project's conventions before flagging something; prefer local consistency unless it creates correctness, performance, API, or maintenance cost. Explicit hard rules in this skill still apply.

Be blunt, specific, and short. Do not insult the author. For shape work, think from the ideal design first, then choose small safe moves toward it. PR comments must follow `prose.md`.

## Router

Always read `core.md` first. It defines the shared review priorities, evidence rules, error-handling boundaries, output shape, and severity. Do not review without it.

Then add only the files that match the task:

- Ecto/query/schema/migration/backfill: `database.md`, usually `tests.md`, plus `queryable.md` only if the project uses a Queryable pattern.
- planning, architecture, audits, build planning, API shape, helper extraction, or taste: `architecture.md`, `aesthetics.md`, plus `audit.md` for audits.
- GraphQL, HTTP/LiveView, jobs, errors, performance, security, style, comments, prose, or production risk: read the matching domain file when that concern is in scope.
- Tests: read `tests.md` whenever behavior, coverage, fixtures, or regression protection matter.

## Output

Use `architecture.md` for plan shape, commit slices, and ideal/current/first-move framing. Use `core.md` for review finding shape, code anchors, and severity. Use `audit.md` for audit opportunity shape. Use `prose.md` for inline PR comments. Add architecture framing only when it clarifies the issue.

## Automated Review Prompt

```text
Review this Elixir/Phoenix/Ecto/Absinthe diff with strictness. Prioritize semantic correctness, layer correctness, and minimal direct code. Look for defensive slop, unnecessary wrappers/helpers, awkward APIs, misplaced GraphQL/HTTP parsing, duplicated query/filter infrastructure, stale compatibility branches, transaction rollback bugs, ignored failure returns, N+1 behavior, unsafe migrations/backfills, weak tests, and path/security boundary issues.

Search call sites, tests, schema/query definitions, migrations, and git history before preserving legacy logic or recommending deletion. Push request/string parsing to resolvers/controllers; keep contexts typed and domain-native; keep exact filters near schema/query APIs; use batch/dataloader/preload pragmatically; prefer structured project errors at user-facing boundaries; let exceptional internal invariant/type failures crash clearly.

Report findings first, ordered by severity, with code anchors and the smallest direct fix. Anchor by module/function/field/query/loop/test behavior instead of raw line numbers. Be strict about API/query shape, GraphQL boundary flattening, dataloader conventions, Queryable usage, enum internal values, and layer-appropriate tests. Calibrate severity to proven impact: do not label maintainability concerns as blocking, do not invent security findings, and ask direct product-semantics questions when behavior is ambiguous. Label style-only feedback as nitpick, but call real module/API reshaping a structure suggestion. Do not invent abstractions unless they clearly reduce complexity.
```
