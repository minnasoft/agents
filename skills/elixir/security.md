# Security Review

Use this file for external inputs, files, paths, filenames, auth/org scoping, atom safety, and boundary validation.

## Boundary Validation Is Not Slop

Validating external input is not defensive slop. It is the boundary's job.

Validate deliberately at:

- HTTP/GraphQL boundaries;
- file/import boundaries;
- auth/permission boundaries;
- third-party webhook/API boundaries;
- background job arg boundaries.

## File And Path Safety

Flag:

- filenames trusted as truth;
- paths built through string concatenation;
- user-controlled strings influencing filesystem paths;
- extension checks based on naive string assumptions;
- directory traversal risk through `../` or nested path segments.

Prefer:

- `Path.join/2` or `Path.join/1`;
- `Path.basename/1` when accepting filenames;
- `Path.extname/1` for extension checks;
- explicit allowlists for extensions/types;
- storing/generated safe filenames rather than trusting user filenames.

Example:

```elixir
if Path.extname(filename) in [".png", ".jpg", ".jpeg"] do
  dir
  |> Path.join(Path.basename(filename))
  |> File.read!()
end
```

## Auth And Org Scoping

Do not invent security findings. Only call something a security issue when there is a concrete external input, auth/org boundary, permission check, path/file boundary, atom creation, or data exposure path in the diff.

If the concern is parent consistency, reporting correctness, import weirdness, or index usefulness, classify it as API/query shape, data consistency, performance, or a nit instead of security.

Flag:

- missing org scoping on queries;
- org scoping added in the wrong layer;
- permissions checked after side effects;
- tests that do not cover cross-org access;
- assumptions that IDs imply access.

Prefer clear boundary checks and context query filters that make org ownership explicit.

## Atom Safety

Flag converting uncontrolled strings to atoms.

Atoms are not garbage collected. Use existing atoms only when the input is finite, controlled, and already known by the VM. Otherwise keep strings or use explicit maps/allowlists.

Also flag external JSON or params decoded directly into atoms. Do not use uncontrolled `String.to_atom/1`; use `String.to_existing_atom/1` only for finite, trusted values and prefer explicit maps.

## SQL And Fragments

Raw SQL and `fragment` calls must be parameterized. Do not interpolate external strings into SQL fragments, order clauses, table names, or filter expressions.

Dynamic field/order/table selection should use explicit allowlists before reaching SQL/Ecto query construction.

## Upload Content

Extension checks are not always enough for user-uploaded files. When the file content matters, validate MIME/content using the project's established upload/file inspection path in addition to safe filenames and paths.

## Error Disclosure

User-facing boundaries should return useful domain errors without leaking internals. Internal/background code can crash when stacktraces are more useful.

Do not mask security-relevant failures so thoroughly that operators cannot debug them.
