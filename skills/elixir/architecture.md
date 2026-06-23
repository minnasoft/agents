# Architecture Lens

Use this file when reviewing, auditing, or building code where the shape matters, not just the local diff.

The stance is senior consulting architect with broad authority. You are not here to politely preserve accidental structure. You are here to decide what the code should have looked like if someone with taste had built it first, then move the current code toward that shape without blowing up the project.

This is not license to rewrite everything. It is a way to make opinionated, iterative recommendations.

## Core Question

For the task or code at hand, answer three questions:

1. What is the ideal shape if we were building this fresh with minnasoft conventions?
2. Where does the current code diverge from that shape, and what does that cost?
3. What is the smallest safe sequence of moves from here to there?

Do not stop at "fix this line." Name the design you wanted before this code existed.

## Ideal-First Review

Before giving findings, reconstruct the design path you would have taken:

- the domain/module boundary you would choose;
- the deepest owning submodule that can carry the concept without leaking across layers;
- the public API you would want callers to use;
- where query composition belongs;
- where boundary parsing/coercion belongs;
- what tests should anchor the behavior;
- what production-data or performance facts would change the design;
- which existing project patterns should be reused or replaced.

Then compare the current implementation against that ideal.

Flag divergence when it creates:

- awkward public API shape;
- duplicated query/filter logic;
- boundary/domain leakage;
- hard-to-test behavior;
- repeated private-helper confetti;
- broad contexts absorbing a coherent subdomain;
- brittle tests that mirror implementation instead of behavior;
- performance or production-data risk.

Do not flag divergence just because another shape is also possible. The better shape must reduce real cost.

When suggesting a submodule, default to the deepest layer that still owns the concept. Prefer `Product.Sigs` over a broad `Sigs` module when the behavior is product default-SIG policy. Prefer a higher-level module only when multiple layers genuinely need the same public concept and the module would not blur different reasons to change.

Before extracting a subcontext during build, gather evidence when the answer is not obvious:

- behavior clusters;
- call sites and tests;
- deepest owner candidates;
- do-now risk versus follow-up value;
- name options with a recommendation.

Split without asking only when the extraction is small, same-layer, private-ish, needed for the current fix, and clearly named. Ask before splitting when the diff gets large, ownership/naming is uncertain, or tests need meaningful moves.

## Iterative Migration

Prefer migration paths that can be done in slices:

- introduce the target module/API shape;
- move one behavior/query/source at a time;
- add or move tests before widening the change;
- update internal callers before public callers when possible;
- use narrow `defdelegate` only when the top-level context should remain the public entrypoint;
- delete old wrappers and compatibility paths after call sites move;
- stop when the next step no longer belongs to the current task.

Every architecture recommendation should include the first useful step. If the ideal shape is too large for the current work, say what to do now and what to leave as follow-up.

When a follow-up matters but is too large for the current slice, consider leaving a concrete `TODO` comment at the relevant seam so the migration path stays visible in the code. The TODO should name the next architectural move, not vaguely ask someone to clean things up later.

If a subcontext split is deferred, prefer final-summary follow-up unless there is a real code seam where a future engineer would otherwise miss the next move.

## Output Shape

For reviews and audits, use this shape when architectural framing matters:

```text
Ideal shape: what we would have built first.
Current divergence: where this code differs and why that matters.
Move toward it: smallest safe step now, plus follow-up sequence if useful.
```

For builds, use the same reasoning before implementation:

```text
Chosen shape: module/API/query/test shape to build.
Why this shape: relevant minnasoft conventions and local project evidence.
Implementation slices: ordered safe steps.
Verification: tests/checks that prove the shape and behavior.
```

## Aesthetic Lens

Use `aesthetics.md` with this file. Architecture decides the shape we should have built first; aesthetics decides whether that shape has a beautiful surface, named sharp edges, honest debt, and boundaries that earned their names.

When the ideal architecture is bigger than the current task, use the aesthetic lens to choose the smallest slice that improves the public surface without hiding the remaining edge.
