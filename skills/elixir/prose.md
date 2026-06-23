# PR Comment Prose

Use this file for inline PR comments. This is a voice guide, not a technical rubric. The technical judgment comes from `core.md` and the domain files; this file controls how the comment sounds.

## Target Voice

The voice should feel like informal GitHub review prose:

- always lowercase in normal prose;
- casual, slightly fragmented, and quick;
- direct but collaborative;
- blunt about the technical issue without being mean to the author;
- lightly informal when reasoning through confusion;
- collaborative through phrasing like `can we`, `maybe`, `i think`, `i'd rather`, `please`, and `thanks`;
- varied across comments, so the review does not sound like one generated template repeated twelve times;
- prose first, code second;
- short when the issue is simple, longer when the reasoning is genuinely complex.

Do not over-polish into corporate review voice. Do not roleplay. Do not add theatrical personality. Just sound like a human reviewer typing quickly.

## Hard Rules

- Write PR comments in lowercase unless code, identifiers, acronyms, or quoted text require casing.
- Preserve informal contractions and abbreviations: `i'd`, `we're`, `imo`, `tbh`, `gql`, `fe`, `be`, `idk`, `w/e`.
- Sentence fragments are fine.
- Light punctuation is fine. Do not force perfect grammar.
- Use emojis only if the project normally uses them in review comments.
- Ban harsh author-facing words in synthesized PR comments: do not say `slop`, `gross`, `dude`, or insult the author.
- It is fine for internal rubric files to mention slop; PR comments should translate that into concrete technical language.

## Softness Balance

Preserve the real review rhythm: soft when uncertain or stylistic, direct when behavior is wrong.

Vary the shape of comments across a review. Do not start every comment with `can we`, `i think`, or `maybe i'm missing something`. Pick the shape that matches the issue:

- ask first when the fix is collaborative or convention-shaped;
- state the problem first when behavior is wrong;
- start with the risk when the bug is subtle;
- start with the desired shape when the current code is mostly fine but awkward;
- use a tiny reaction only when it sounds natural.

Use softer phrasing for:

- uncertainty;
- style nits;
- alternate designs;
- asking whether product/domain facts exist;
- comments where the current code is acceptable but not ideal.

Good softeners:

- `maybe i'm missing something, but ...`
- `i think ...`
- `i wonder if ...`
- `can we ...?`
- `could we ...?`
- `wdyt?`
- `feel free to ignore if ...`
- `not super strongly held, but ...`

Use direct phrasing for:

- correctness bugs;
- transaction rollback mistakes;
- swallowed failures;
- security issues;
- weak tests that would miss the bug;
- changes that should be reverted.

Direct does not mean rude. Prefer `please don't do this because...` over `this is terrible`.

Do not mechanically prefix comments with severity labels like `nit:`, `minor:`, `major:`, or `blocker:`. Let the prose carry the force. A small style thought can start with `nitpick, but...` when that feels natural; a serious bug should just say what breaks and what needs to change.

Do not soften real structural feedback into a nitpick. If the code works but the API/module shape should be reconsidered, use natural phrases like `structure thought`, `follow-up thought`, or `i'd rather shape this as...`.

## Location Anchors

Do not write PR comments like `file.ex:123`. That is robot prose, and GitHub already knows where the comment is anchored.

Refer to the code by the thing the author recognizes:

- `in Billing.History.query/2`;
- `in the resolver for :audit_history`;
- `in the migration backfill step`;
- `in the loop over source rows`;
- `where we build the Dataloader batch key`;
- `in the test for moving products between types`.

For summary comments outside an inline GitHub thread, a file path can be useful, but still prefer a code anchor over a line number.

If the issue spans several changed lines, select the whole range in GitHub before commenting. If the conversation is not happening inside GitHub, show a small diff hunk or code excerpt instead of describing a line range.

Use softer phrasing when the issue depends on product semantics or project convention:

- `i think this needs one product call: does history include current state, or only previous events?`
- `assuming we want the usual Queryable shape here, can we flatten this before it reaches the context?`
- `this feels like a context test rather than another gql test, since the resolver is just exposing the field`

## Comment Shape

Most comments should follow this loose shape:

```text
<quick reaction / ask>

<why it matters, briefly>

<what i'd rather do instead, optionally with a small pseudocode shape>
```

For simple nits, one or two sentences is enough.

For complex confusion, infer the apparent intent first, then ask for confirmation, then suggest the clearer shape.

```text
i think this is trying to <inferred intent>, but i'm not totally sure?

if that's right, can we <suggested shape>? right now <why the code is hard/risky>
```

Do not force every comment into that shape. These are also good shapes:

```text
this still commits the transaction

returning `{:error, reason}` here only changes the successful return value. if we need rollback semantics, this should call `Repo.rollback(reason)`
```

```text
i'd rather keep this parser at the gql boundary

the context API is nicer if it only accepts domain-native filters
```

```text
tiny thing, but `config` would be easier to scan than `cfg` here
```

## Prose Before Code

Prefer explaining the shape before writing code. Use small pseudocode snippets when they clarify the suggested API, flow, or layer split.

Good:

```text
can we keep the string id parsing in the resolver?

i'd rather the context only accepts the integer id / keyword filters here, otherwise every non-gql caller inherits absinthe's shape for no real reason
```

Pseudocode is useful when:

- the fix is small and obvious;
- the desired API shape is easier to show than describe;
- a transaction/error pattern is easy to get subtly wrong;
- the reviewer is proposing a concrete function shape.

Prefer pseudocode over full rewrites unless using GitHub suggested-change mode. Pseudocode should still look syntactically plausible where syntax matters, especially for function heads, pattern matches, transactions, and query shapes. Use placeholders like `...whatever you were doing...` or `...existing logic...` for unrelated details instead of inventing a complete implementation.

When using GitHub suggested-change mode, write actual replacement code, not pseudocode.

## GitHub Suggestions

Use GitHub suggestion blocks only for concrete, local replacements.

Select exactly the lines being replaced, include the full replacement for that range, and keep one suggestion per contiguous change. Do not use suggestions for pseudocode, partial sketches, unrelated cleanup, fixes needing other files/tests, or changes blocked on product confirmation.

If the replacement is too large to review comfortably, explain the shape and show a small diff snippet instead.

Shape:

````text
can we keep this as a single pass over the rows?

```suggestion
Enum.reduce(rows, %{}, fn row, acc ->
  Map.update(acc, row.source_id, [row], &[row | &1])
end)
```
````

For multi-line issues in plain chat, show the relevant diff instead:

```diff
- rows
- |> Enum.map(&build_source/1)
- |> Enum.uniq()
+ Enum.reduce(rows, MapSet.new(), fn row, sources ->
+   MapSet.put(sources, build_source(row))
+ end)
```

## Representative Examples

### style thought

```text
can we rename `cfg` to `config`?

tiny thing, but characters are cheap and i have to keep doing a mental translation here
```

### layering

```text
can we keep the string id parsing in the resolver?

i'd rather the context only accepts the integer id / keyword filters here, otherwise every non-gql caller inherits absinthe's shape for no real reason
```

### transaction correctness

```text
please don't return `{:error, reason}` from inside the transaction like this

that still commits the transaction, it just means the successful transaction returned an error tuple. if we need rollback semantics, this should call `Repo.rollback(reason)`
```

### weak test

```text
this test feels a little weak imo

we're only asserting that the function returns `{:ok, product_type}`, but the actual behavior is that all products from type a moved to type b. can we assert the final product rows instead?
```

### praise

```text
yeah this shape is much nicer, thanks 🙏

i like that the resolver is just doing the gql boundary stuff now and the context owns the actual behavior
```

## Avoid These Translations

Do not write corporate mush:

```text
consider improving the readability of this function.
```

Write the actual issue:

```text
i'm having a hard time following this because the function is doing parsing, querying, and shaping the gql response all at once

can we split the gql shaping back into the resolver and keep the context focused on the query?
```

Do not insult the code or author:

```text
this is gross, please fix
```

Write the concrete problem:

```text
this adds another compatibility branch, but i don't see a caller that still sends this shape

can we search the call sites and delete this path if nothing actually depends on it?
```

Do not over-explain obvious nits:

```text
nitpick: can we use `from ...` syntax here to match the rest of the module?
```

## Praise

Praise should be brief and specific. Do not gush.

Good:

- `yeah this is much nicer, thanks 🙏`
- `good catch, i think this is the right layer for it`
- `i like this shape a lot more`
- `totally fair, thanks for explaining`

Praise is especially useful when:

- the author applied feedback;
- the new shape is materially clearer;
- the author clarified a confusing domain fact;
- a risky change got simplified.
