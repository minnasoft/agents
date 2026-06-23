# Style And Naming Review

Use this file for naming, readability, helper/wrapper smell, one-line defs, pipelines, and consistency.

Style feedback should improve scanning or consistency. Label pure style as nitpick. Do not block on it unless inconsistency creates real maintenance cost.

## Naming

Names should be honest, domain-specific, and skim-friendly.

Flag:

- unclear short names like `r`, `rv`, `o1`, or `res` outside tiny obvious scopes;
- redundant domain prefixes like `order_order_items` when `list_order_items` or `order_item_query` is clearer;
- helper names that claim validation/safety they do not provide;
- GraphQL field names that read like imperative RPC endpoints;
- inconsistent terminology or casing.

Characters are cheap. Use the longer name if it removes mental translation.

## Function Name Grammar

Most context/domain functions should use a boring verb prefix unless there is an exceptionally good reason not to.

Prefer these shapes:

- `get_` for fetching one thing;
- `list_` for fetching many things;
- `paginate_` for cursor-based pagination return values;
- `batch_` for batch loaders, especially Absinthe/GQL batch resolver support;
- `update_` for changing an existing thing;
- `delete_` for deleting/removing a thing.

Names outside this set need to earn their existence by describing a real domain operation. `register_user`, `complete_order`, or `submit_invoice` can be fine as public workflow/orchestration names, but they should usually delegate internally to boring primitives like `update_user`, `update_order`, `create_invoice`, etc.

Strongly question:

- bespoke verbs that only mean `get`, `list`, `update`, or `delete`;
- names that preserve awkward UI/GQL wording in context APIs;
- helper names that describe implementation mechanics instead of domain behavior;
- redundant prefixes like `order_order_items` when `list_order_items` says the same thing better;
- API pairs where one name is a thin wrapper over another but both remain public.

The boring name is usually the better name. If the function is not a real domain workflow, do not give it a dramatic workflow name.

## Helpers And Wrappers

Prefer fewer names and fewer indirections unless they buy real clarity.

Flag:

- private helpers with one or two trivial call sites;
- wrappers that only delegate to another resolver/context function;
- functions that adapt awkward old shapes instead of removing the awkwardness;
- new modules/abstractions before the domain needs them;
- helper names hiding simple operations that are clearer inline;
- private helpers that call smaller private helpers without adding a meaningful domain boundary.

Accept duplication when it keeps behavior easier to read. Extract only when the name captures a real domain concept or removes meaningful complexity.

Private helpers should have a high bar:

- one call site is usually not enough;
- two trivial call sites are usually not enough;
- a private helper that only sequences smaller private helpers should strongly consider being inlined;
- a private helper that merely names a few lines of obvious code is usually worse than the obvious code;
- a private helper is justified when it isolates real complexity, names a domain concept, or prevents meaningful repeated bugs.

When in doubt, inline first. Extract later when the repetition or complexity proves it wants a name.

For cohesive source-selection flows, prefer one readable function over helper confetti. A small `for` over source query functions plus a local `case` for the empty case can be easier to review than several one-use helpers.

Good shape:

```elixir
query_funs = [
  {"created", &created_query/1},
  {"updated", &updated_query/1}
]

queries =
  for {event_type, query_fun} <- query_funs,
      is_nil(event_types) or event_type in event_types do
    query_fun.(filters)
  end
```

Avoid public helper proliferation when the argument already carries the scope. Prefer `history_query(%Order{}, filters)` or `History.query(%Order{}, filters)` over names like `order_history_query(%Order{}, filters)` unless the extra prefix removes real ambiguity.

## Function Style

Recurring preferences:

- no one-line function definitions; use `do ... end`;
- avoid pointless pipelines for a single call;
- use consistent Ecto query style within a module;
- avoid binding variables just to satisfy shape checks;
- avoid `%{field: _} = attrs` when the shape is not the point; (prefer guards, or just nothing)
- prefer direct pattern matching when it documents an invariant.

This applies even when the body is tiny. Consistent multi-line definitions are easier to scan, diff, annotate, and expand during review.

Do not introduce new one-line function definitions in build work. In review work, treat touched one-line defs as a standing nitpick. They are ugly because they compress the function boundary, make diffs worse, and leave less room for future changes.

## Function Ordering

Colocate public functions with the private functions that explain them. Do not separate all public functions at the top and all private helpers at the bottom by default; that makes the reader jump around to understand one API.

Default ordering:

- put a public function first, then the private helpers used mainly by that public function;
- when multiple public functions share one private helper cluster, put those public functions together, then put the shared private helpers immediately after them;
- keep unrelated public/helper clusters separate so each public API can be read in one pass;
- leave broadly shared low-level helpers near the end only when no smaller public cluster owns them.

This is a reading-order rule, not permission to create extra helpers. Inline first when the helper has not earned a name.

## Control Flow

Prefer flat, readable control flow over nested pyramids.

Flag:

- deeply nested `case` / `if` / `with` blocks;
- `with` blocks that only rewrap or mask errors;
- nested error handling where direct propagation would work;
- extracting every branch into tiny private helpers to hide nesting;
- private helper chains that make the reader jump around to understand one flow.

Prefer:

- `cond` for mutually exclusive predicates;
- `with` for sequential fallible operations;
- pattern matching when it documents invariants;
- direct error propagation when the called function already returns the right shape;
- keeping logic in one function unless there is a natural domain or complexity boundary.

Do not turn nesting into private-helper confetti. A slightly longer but flat function is usually better than forcing the reader through five private helpers.

## Truthy Fallbacks

Prefer simple truthy/fallback expressions when the nil/false semantics are exactly what the code wants.

Good shapes:

```elixir
value = Map.get(attrs, :thing) || :something_else
```

```elixir
thing = condition && truthy_value || falsey_value
```

These are often clearer than nested `if` / `case` blocks when the fallback is natural and `nil`/`false` should be treated the same.

Flag:

- verbose `case` / `if` forms that only implement a simple fallback;
- fallback branches that duplicate defaulting already handled by `||`;
- clever truthy expressions where `false` is a valid meaningful value and must not fall through;
- truthy/fallback chains that become hard to read.

Use explicit `case` / `if` when `false` and `nil` need different behavior, when the predicate has side effects, or when the branches need names/explanation.

## Examples

Avoid mental translation:

```elixir
# Bad
Enum.map(cfgs, fn cfg -> cfg.name end)
```

```elixir
# Better
Enum.map(configs, fn config -> config.name end)
```

Avoid redundant prefixes:

```elixir
# Bad
def order_order_items(order, args) do
  ...
end
```

```elixir
# Better, depending on behavior
def list_order_items(order, args) do
  ...
end

def order_item_query(filters) do
  ...
end
```

Prefer boring context verbs:

```elixir
# Bad unless this is truly a domain workflow
def activate_membership(user, attrs) do
  ...
end
```

```elixir
# Better if this is just a state change
def update_membership(membership, attrs) do
  ...
end
```

Avoid private helper confetti:

```elixir
# Bad
def create_thing(attrs) do
  attrs
  |> normalize_attrs()
  |> put_defaults()
  |> insert_thing()
end

defp normalize_attrs(attrs) do
  Map.update(attrs, :name, nil, &String.trim/1)
end

defp put_defaults(attrs) do
  Map.put_new(attrs, :enabled, true)
end
```

```elixir
# Better, unless those helpers have real reuse/domain meaning
def create_thing(attrs) do
  attrs
  |> Map.update(:name, nil, &String.trim/1)
  |> Map.put_new(:enabled, true)
  |> Thing.changeset()
  |> Repo.insert()
end
```
