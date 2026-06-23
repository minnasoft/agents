# Aesthetics

Use this file when code shape, public API beauty, helper extraction, docs voice, comments, or review taste matters.

The core aesthetic is simple: beautiful surface, named sharp edges. Make the caller-facing API pleasant first, then make any weird internal machinery obvious, bounded, and reviewable.

This is not prettiness for its own sake. Beauty is a design tool: it makes the intended shape easier to see, easier to call, and harder to misuse by accident.

## Core Aesthetic

Prefer code that has:

- a small, obvious public API;
- direct call sites with cute, readable names;
- sharp internals named where they live;
- responsibilities sitting at the layer that owns their meaning;
- tests that protect the public behavior instead of the current implementation trivia.

Make ugly-but-honest code prettier when you can do it without hiding risk. If a shortcut remains, name the edge with a boundary, test, `TODO`, `HACK`, `NOTE`, `WARN`, or clear docs.

## Honest Beauty And Conscious Debt

Do not treat rough code as automatically virtuous. Honest roughness is useful when it keeps the tradeoff visible, local, and easy to replace. It becomes bad debt when it hides contracts, global mutation, unstable identity, cross-layer coupling, or behavior that will surprise the next change.

Debt is acceptable when it serves the intended shape and is consciously bounded:

- know where it lives;
- know why it exists;
- know what makes it risky;
- know what would remove it.

Do not launder debt by calling it "simple," "tasteful," or "temporary" while burying the cost. If the code is using a sharp shortcut, say so and keep the beautiful surface honest.

## Surface First

Teach and design from the caller surface inward:

1. What does the caller touch?
2. What public API shape makes that usage obvious?
3. What internal machinery is needed to support it?

Prefer callsite beauty over internal neatness. Internal awkwardness is allowed when the public contract becomes clearer and the awkwardness is named, bounded, and not leaking.

Default hard toward callers passing the natural thing they already have: changeset, struct, context, ids, or domain object. Do not make callers assemble a helper's internal shape. If a helper needs a weird scope map, tuple, or precedence structure, hide that inside the helper. The ugly bit belongs behind the boundary that understands it.

Keyword `opts` or scope maps are allowed only when they are the public API and have earned that shape: small, documented or obvious, caller-facing choices like `timeout:`, `limit:`, `preload:`, `force?:`, or a project-standard scope contract. They are not fine when they are just a disguised internal struct the caller has to build so a helper can avoid owning its own shape. When in doubt, keep the construction inside the helper.

When reviewing docs, examples, and generated explanations, start with the smallest useful caller example before explaining internals.

## Extract Only Earned Boundaries

Inline until the real shape is visible. This is not anti-abstraction; it exposes false abstraction.

Use this rhythm:

1. remove fake abstraction or safety-shaped ceremony;
2. pull the moving parts close enough to read in one pass;
3. make the mechanism plain, even if it is temporarily awkward;
4. watch where meaning accumulates;
5. name only the boundary that earned a name.

A private helper is a design claim. A `defp` must prove it deserves to exist. If the helper body is clearer than the helper name, inline it. If the helper only hides one operation, a comment, a tiny branch, or a little shame, inline it. If the helper only exists so a comment has somewhere to sit, inline the code and put the comment at the use site.

Extract only for a real boundary:

- lifecycle;
- data shape;
- behaviour or extension contract;
- registry or extension point;
- public-ish API;
- domain responsibility;
- schema/query clause;
- boundary translator;
- repeated visual or interaction primitive;
- independently useful test/reasoning unit.

Do not create private helpers that secretly own vocabulary, policy, dispatch, ownership, or cross-layer translation. If a private helper mediates meaning, consider a named module, explicit extension point, schema-owned query API, context API, or boundary translator instead.

Length is a reading cost, not proof that a boundary exists. Do not split code just because the file or function is big. A large function or template can be the right shape when it represents one page, one workflow, or one cohesive mechanism.

Extract low-level machinery when it pollutes the high-level story. After the flow is visible, step back and ask whether a real capability, submodule, public API group, query clause, or boundary emerged.

## Meaning Belongs At Its Layer

Put behavior where the meaning belongs, not where it is convenient today.

Submodules should usually live at the deepest owning layer. A pretty broad name is not enough; the module must share ownership, lifecycle, and reason to change. If only product-default SIG policy is moving, prefer `Product.Sigs`/`Product.DefaultSig` over a global `Sigs` module. If label rendering owns the behavior, keep it under the label/rendering owner.

- Generic query mechanics belong in generic query infrastructure.
- Schema-specific filters and joins belong near schema/query APIs.
- Context modules own business use cases.
- Resolvers/controllers translate API and HTTP vocabulary.
- LiveView/web layers own UI input shaping.
- Jobs own orchestration, retry, and idempotency concerns.
- Tests assert the public contract at the layer that owns it.

Do not make callers compensate for lower-layer design mistakes. Do not leak product semantics into generic infrastructure.

## Sharp Tools Are Allowed

Macros, behaviours, reflection, generated functions, and other sharp tools are allowed when they make the caller surface cleaner and the tradeoff is named.

Do not add arbitration layers, opt-in registries, wrappers, safety gates, or claims systems for hypothetical misuse. Safety belongs where concrete breakage, data risk, or a real invariant demands it.

Correctness still matters. The point is to avoid defensive architecture that makes the API uglier without buying proven safety.

## Comments Name Sharp Edges

Comments are orientation markers, not apologies.

Use the standard labels from `comments.md`:

- `NOTE`: invariant, lifecycle fact, or context the code cannot show.
- `HACK`: intentional shortcut or semantic abuse with a removal condition.
- `TODO`: concrete next move, especially at a real migration seam.
- `WARN`: edge where casual changes can break behavior.

Prefer a concrete `TODO` at a real seam over a vague cleanup wish. Prefer deleting comments when a better shape would make them unnecessary.

## Cute Docs

Cute means scannable, concrete, and pleasant to read. It does not mean mascot voice, fantasy flavor, or replacing explanation with whimsy.

Good docs:

- start from the caller surface;
- show tiny real examples;
- use friendly section names when they improve scanning;
- name sharp edges directly;
- preserve intentional microstyle instead of flattening everything into safe assistant prose.

For PR summaries, lead with behavior and contract changes before implementation machinery. Mention helper names, precedence internals, or template details only when reviewers need that context to understand risk.

Fix correctness, ambiguity, and scanability. Do not sand off clear personality just because it is informal.

## Living Contracts

Protect the current desired surface, not ghosts.

Breaking changes are fine when all current call sites can be updated. Do not add backward compatibility unless there is a real external contract, persisted data concern, shipped API, or explicit user requirement.

In-repo callers are migration targets, not a reason to preserve the old shape forever.

## Tests Serve Design

Tests should protect behavior and public contracts. They should not force production code into awkward seams just for coverage.

Avoid test-only dependency injection, config switches, mocks, or alternate modules unless the seam is already part of the design or the behavior is important enough to justify it.

It is better to leave an obvious hard-to-test branch lightly tested than to make production code uglier for a fake testing seam.

## Review And Subagent Feedback

Use review workers as advisors, not authorities. `staff` owns aesthetic and architectural judgment; `eng` implements approved slices.

Reject feedback that adds ceremony, stale compatibility paths, ghost-contract tests, private-helper confetti, enterprise dependency injection, or speculative guardrails without improving the intended current design.

Iterate only on feedback that makes the chosen shape clearer, safer, faster, or easier to call.
