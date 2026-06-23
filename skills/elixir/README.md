# Elixir Expert

> Because your Elixir reviewer deserves taste too.

The Elixir expert is minnasoft's OpenCode setup for Elixir/Phoenix plan, review, audit, and build loops.

It gives a project one visible coordinator, the hidden `research`/`eng`/`review` workers, slash commands, and a progressively loaded pile of Elixir opinions.

The point is not to be a generic answer to everything. This is just the minnasoft-shaped way we like agents to read Elixir code:

- direct about bugs
- fussy about API shape
- allergic to defensive slop
- suspicious of query duplication
- architecturally bossy about the shape we should have built first
- weirdly serious about beautiful public APIs
- very normal about hating `Ecto.Multi`
- cute enough to keep around

## Parts

### Agents

- `agents/staff.md`: visible plan/review/audit/build coordinator. Use with `@staff`, `/elixir:plan`, `/elixir:review`, `/elixir:audit`, or `/elixir:build`.
- `agents/research.md`: hidden read-only worker for git history, local patterns, call sites, and evidence gathering.
- `agents/eng.md`: hidden edit-capable worker for one approved implementation slice at a time.
- `agents/review.md`: hidden read-only worker for focused review lenses.

### Commands

- `/elixir:plan <task>` investigates enough to choose a shape, asks only blocking questions, and stops before editing.
- `/elixir:review <target>` reviews a PR, range, file, hunk, topic, or current diff.
- `/elixir:audit <target>` audits a codebase area for refactor opportunities, pattern adoption, module/API shape, `defdelegate` boundaries, and test organization.
- `/elixir:build <task>` routes through `staff`, delegates implementation to `eng`, and reviews each slice before commit.

### Skill

- `skills/elixir`: minnasoft's Elixir/Phoenix/Ecto/Absinthe/Oban conventions for plan, review, audit, and build workflows.

```text
skills/elixir/
├── SKILL.md       # router
├── core.md        # always read
├── architecture.md # ideal-shape and migration-path lens
├── aesthetics.md  # beautiful API, named edges, and helper taste
├── audit.md       # codebase refactor audits
├── database.md    # Ecto/query/migration review
└── ...            # domain-specific lenses
```

## How It Thinks

`@staff` coordinates plan, review, audit, and build work. It loads the relevant skill files, delegates bounded research/implementation/review, and owns final judgment. Hidden workers provide evidence, implementation slices, or focused challenge; they are not authorities.

The conventions are minnasoft defaults, not universal Elixir law. The default stance is ideal-shape first, then safe iterative slices toward a better public API and clearer boundaries.

## Plan

Plan a change before editing:

```text
/elixir:plan add a paginated audit history GraphQL field
```

Plan from an audit or review discussion:

```text
/elixir:plan option B from the audit: move reporting filters into Reports.Queryable first
```

Planning is no-edit mode. `staff` should inspect enough code to choose a shape, use `research` when evidence matters, ask questions only for genuinely unknowable product/design choices, and return build-ready slices with verification.

## Review

Review a PR against its base branch:

```text
/elixir:review PR #123 against origin/main
```

Review a local branch/range:

```text
/elixir:review origin/main...HEAD
```

Review only one concern in the current diff:

```text
/elixir:review query shape for the new audit history API
```

Review a specific file or hunk:

```text
/elixir:review api/lib/my_app/audit_history.ex
```

```text
/elixir:review this hunk: <paste diff hunk>
```

Draft GitHub comments without posting them:

```text
/elixir:review PR #123; draft github comments, but do not post
```

Drive the coordinator interactively instead of using a command:

```text
@staff review the current diff, fan out data/query, boundary, and tests, then consolidate critically
```

## Audit

Audit a context for refactoring opportunities:

```text
/elixir:audit api/lib/my_app/billing
```

Audit module and test organization:

```text
/elixir:audit api/lib/my_app/catalog and matching tests; focus on submodules, defdelegates, and describe/function alignment
```

Audit whether a pattern should be introduced:

```text
/elixir:audit reporting queries under api/lib/my_app/reports; look for Queryable/filter duplication and scale risks
```

Audit output should point to the next step: use `/elixir:plan` when the opportunity needs design discussion, `/elixir:build` when the first slice is obvious, or defer it when the payoff is not worth the current interruption.

## Build

Build a feature with internal Elixir review loops:

```text
/elixir:build add a paginated audit history GraphQL field
```

Apply review feedback and verify it:

```text
/elixir:build apply the feedback about flattening Queryable filters and replacing custom batch resolvers with Dataloader
```

Focus the build loop on a single layer:

```text
/elixir:build refactor audit history into Audit.History; only fan out query-shape and tests reviewers
```

Use `/elixir:build` to route through `staff`. When multiple commits are natural, staff should plan behavior/layer slices, run the atomic build-review-fix-review-commit loop for each slice, and report rejected review feedback in the final summary.

If the conversation already has an approved `/elixir:plan`, `/elixir:audit`, or `/elixir:review` handoff, `/elixir:build` should preserve that direction unless new code evidence disproves it.
