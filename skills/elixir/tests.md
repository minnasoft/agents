# Tests Review

Use this file for test changes and for evaluating whether implementation changes are properly covered.

## Core Principle

Tests should prove behavior and production invariants.

Flag tests that pass only because they assert a weak return shape, omit real data constraints, or accidentally avoid the bug.

## Describe Blocks

`describe` blocks should usually map 1:1 to the public function or API being tested.

Prefer:

```elixir
describe "update_order_item_status/2" do
  test "updates the given order item" do
    ...
  end

  test "also updates package items when the item is in a package" do
    ...
  end

  test "returns an error when the order item does not exist" do
    ...
  end
end
```

Inside a describe block, individual tests should assert invariants and meaningful behavior for that function/API, not mirror private implementation branches.

Each test should usually prove one scenario or invariant. Multiple assertions are fine when they all support the same invariant, but multiple scenarios should usually be separate tests.

Table-driven/generated tests are fine when the cases are a known set of examples for the same invariant. This is especially good for parsers, converters, validators, coercion helpers, formatters, and compatibility inputs.

Flag:

- describe blocks named after vague behavior instead of the function/API under test;
- describe blocks that group unrelated functions together;
- tests named after implementation details rather than invariants;
- tests that cover several scenarios at once when those scenarios could be named separately;
- one describe block covering a broad workflow when several public functions are actually under test;
- private helper behavior tested directly instead of through the public function that owns it.

Good test names describe the invariant:

- `updates all products from the old type to the new type`;
- `does not create duplicate rows when re-run`;
- `returns an argument error when both item_id and item_ids are provided`;
- `uses the caller's org scope when listing records`.

## Table-Driven Tests

Use generated tests when many inputs assert the same invariant and the list of cases is itself useful documentation.

Good fits:

- date/string parsers;
- coercion helpers;
- validation edge cases;
- normalizers/formatters;
- known compatibility inputs;
- mapping tables with many equivalent examples.

Prefer:

- a module attribute or local data structure containing the cases;
- comments grouping families of cases when useful;
- generated test names that include the input and expected output;
- `Macro.escape/1` for expected values that need safe compile-time injection;
- one generated test per case when failures should identify the exact input.

Example shape:

```elixir
@test_cases %{
  # slash separators
  "09/11/1980" => ~D"1980-09-11",
  "05/02/72" => ~D"1972-05-02",

  # invalid values
  "NONE" => nil,
  "Anything else" => nil
}

for {test_string, date} <- @test_cases do
  test_name = "given string `#{test_string}`, returns `#{inspect(date)}`"

  test test_name do
    assert unquote(Macro.escape(date)) ==
             ConverterHelper.coerce_date_of_birth(unquote(test_string))
  end
end
```

Avoid table-driven tests when each case is actually a different behavior that deserves its own setup, name, or explanation. A giant table should not hide unrelated scenarios just to look tidy.

## Test Context, Setup, And Tags

Using ExUnit context is a good pattern when tests share setup or when setup varies by tags.

Prefer:

- `test "name", ctx do` when the test uses context values;
- `setup` for per-test fixtures and state;
- `setup_all` only for safe shared values that do not leak mutable state between tests;
- tags to parameterize fixture setup and make scenario differences obvious;
- context-driven setup over repeated inline fixture boilerplate when many tests share the same shape.

Example shape:

```elixir
setup ctx do
  account = account_fixture(ctx[:account] || [])
  rule = rule_fixture(ctx[:rule] || [])

  {:ok, account: account, rule: rule}
end

@tag account: [tier: :basic]
@tag rule: [minimum_tier: :pro]
test "does nothing when the account does not match the rule", ctx do
  refute_applies(ctx)
end
```

Tags are especially useful when the invariant is the same but inputs vary by scenario. They keep the test name focused on behavior while setup data stays close to the test.

Flag:

- context maps passed around when direct local variables would be clearer;
- giant `setup` blocks that obscure what each test actually needs;
- tags that hide important scenario details instead of clarifying them;
- `setup_all` creating mutable/shared database state that can leak between tests.

## Concurrent Tests

Prefer concurrent tests when possible. In ExUnit, use `async: true` for tests/modules that do not depend on shared mutable state.

If a test cannot run concurrently, the reason should be clear:

- shared global process state;
- application env mutation;
- non-sandboxed external services;
- feature flags or mocks that are global;
- shared filesystem paths;
- named processes/registries with global names;
- database behavior outside the SQL sandbox.

When concurrency causes failures, investigate the shared state. Do not immediately remove `async: true`, over-scope queries, add sleeps, or weaken assertions just to make the failure disappear.

Flag:

- disabling async without explaining the shared resource;
- test changes that hide data pollution instead of proving/fixing its source;
- sleeps/retries used to mask races;
- broad query scoping added only because another async test might leak data;
- global mocks/flags/env changes that are not isolated.

Prefer fixing the underlying isolation issue: sandbox ownership, per-test unique names/paths, Mimic/stub isolation, explicit process cleanup, or moving truly global tests into a small non-async module with a comment explaining why.

## Behavior-Focused Assertions

Flag:

- tests that only assert `{:ok, _}`;
- tests that do not assert the final domain/data state;
- tests that call the same implementation path they are meant to validate;
- assertions that pass without checking anything when lists are empty;
- tests that scope away possible duplicate/extra records instead of catching them.

Prefer:

- exact state assertions when cardinality matters;
- asserting changed rows, not just returned tuples;
- testing mixed valid/invalid input when validation is the point;
- regression tests that fail on the original bug;
- direct assertions that give useful ExUnit failures.

Example:

```elixir
# Weak
assert {:ok, %ProductType{}} = Inventory.reassign_product_type(old_type.id, new_type.id)
```

Better:

```elixir
assert {:ok, %ProductType{id: ^new_type_id}} =
         Inventory.reassign_product_type(old_type.id, new_type_id)

assert [] = Product.query(product_type_id: old_type.id) |> Repo.all()
assert [%Product{}, %Product{}] = Product.query(product_type_id: new_type_id) |> Repo.all()
```

## Layer-Appropriate Tests

Prefer:

- context/domain tests for business behavior;
- resolver/controller/LiveView tests for boundary behavior, auth, coercion, returned shape, and user-visible state;
- worker tests for job orchestration and retry/error behavior;
- schema/changeset tests for persistence validation and constraints.

Flag:

- heavy resolver tests for thin pass-through resolvers while contexts are under-tested;
- UI tests used to prove deep domain logic;
- duplicated boundary tests that do not add coverage;
- implementation-detail assertions that make refactors brittle.

For GraphQL additions, distinguish domain behavior from boundary shape:

- context/domain tests should cover business semantics, query filtering, ordering, current-vs-history rules, and returned records;
- resolver/controller tests should cover field exposure, auth/org scoping, ID/arg coercion, connection shape, and resolver integration;
- do not demand duplicate GraphQL tests for every context scenario when the resolver is thin;
- do not accept only GraphQL tests when the core behavior belongs in the context/query layer.

For union/source-specific query behavior, test source-specific filters at the context layer. If a filter cannot apply to one source and intentionally returns no rows for that source, make that behavior obvious with a regression test.

## Fixtures And Production Invariants

Fixtures should reflect production realities.

Flag:

- fixtures missing required production fields like `org_id` when the domain requires them;
- tests preserving stale legacy loopholes;
- factories that create impossible states unless the test is explicitly about impossible/corrupt data;
- test changes that hide data pollution without proving the pollution exists.

## Custom Test Helpers

Custom assertion helpers are suspicious when they hide useful failures.

Prefer direct assertions unless the helper improves repeated readability without reducing failure quality. Use `flunk` for semantic test failure when needed rather than crashing with vague errors.

Multiple assertions are fine when they document parts of the behavior. Do not cargo-cult one-assertion tests.
