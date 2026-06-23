---
description: Build Elixir/Phoenix changes using internal Elixir feedback loops
agent: staff
subtask: false
---

Build this Elixir/Phoenix change:

$ARGUMENTS

`staff` coordinates architecture, scope, commit boundaries, delegation, and final judgment.

Before editing, load/read the `elixir` skill and relevant project conventions. If `staff` cannot invoke `research`, `eng`, or `review`, stop and report the agent-pack misconfiguration.

If an approved plan exists in conversation, execute it unless the user overrides it. If no approved plan exists, plan from the ideal minnasoft shape inline, then take the smallest safe slices toward it. Ask before expanding beyond the requested task. If the user asks for a plan/sign-off, investigate enough to produce a short decision-shaped plan, then stop.

Use `research` for evidence, `eng` for approved implementation slices, `review` to challenge each slice, and specialty subagents only when their domain clearly fits or work is parallelizable. Use multiple `eng` workers only for independent non-overlapping sub-slices; otherwise keep implementation sequential. Tell workers which skill files to read when you know the lens; workers should choose additional relevant files too.

Keep the build loop atomic: plan slices, build one slice, review it with parallel focused reviewers when useful, fix and re-review until no confirmed blocking/major issues remain, commit it, then continue. For non-trivial slices, include one style/prose reviewer plus additional domain reviewers chosen from the touched code. New mid-build tasks become separate slices unless required for the current slice. If the task came from `/elixir:plan`, `/elixir:audit`, or `/elixir:review`, preserve that handoff unless new code evidence disproves it.

Verify by touched layer and explain skipped checks with concrete blockers. Report diff summary, commits, verification, accepted risks, rejected feedback, chosen shape, and follow-up slices.
