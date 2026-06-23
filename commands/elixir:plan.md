---
description: Plan Elixir/Phoenix changes before implementation
agent: staff
subtask: false
---

Plan this Elixir/Phoenix change without editing files:

$ARGUMENTS

Resolve whether the target is a feature request, bug, refactor, audit opportunity, review finding, file/module area, or pasted context. If it is ambiguous, make the smallest reasonable assumption and state it.

Use the `elixir` skill as minnasoft's planning rubric. Read `SKILL.md`, `core.md`, `architecture.md`, `aesthetics.md`, and relevant domain files. Include `audit.md` only when planning from an audit opportunity or codebase-area cleanup.

Inspect enough code to choose a shape. Use hidden `research` when call sites, tests, local patterns, git history, data shape, or owner boundaries matter. Use hidden `review` when focused challenge would materially improve the plan; for non-trivial plans, prefer parallel style/API-shape and domain-risk lenses over one broad reviewer.

Use the question tool when available and needed. Ask the user questions only when product behavior is unknowable from code, multiple viable designs have meaningful tradeoffs, the ideal fix is bigger than the requested task, or commit/scope boundaries are ambiguous. Otherwise state assumptions and produce the plan.

Return a short decision-shaped plan:

- chosen shape and public API;
- evidence and conventions that justify it;
- implementation slices and commit boundaries;
- verification per slice;
- explicit deferrals or follow-up opportunities;
- open questions, only if they block confidence.

Stop after the plan. Do not edit files, stage, commit, post comments, or run mutating commands.
