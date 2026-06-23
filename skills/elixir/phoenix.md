# Phoenix Review

Use this file for controllers, LiveView, plugs, params, assigns, HTTP/user-facing errors, and Phoenix boundary validation.

## Boundary Responsibilities

Controllers and LiveViews are user-facing orchestration boundaries.

They should:

- parse/coerce HTTP params and form/input shapes;
- enforce auth/session/org context where the boundary owns it;
- call contexts with clean domain-native inputs;
- return/render useful user-facing errors;
- keep business behavior in contexts.

They should not:

- contain complex domain workflows;
- duplicate changeset validation;
- mask context errors with generic flash/messages when a meaningful domain error exists;
- pass raw params deep into contexts unless the context owns that external shape.

## Params And Assigns

Flag:

- string param parsing inside contexts;
- assigns used as implicit hidden state across unrelated handlers;
- repeated assign setup that should be a plug/on_mount or explicit local helper;
- LiveView event handlers that do too much business work;
- shape checks that exist only because params are being forwarded too far.

Prefer parsing and normalization near the HTTP/LiveView boundary.

## LiveView Event Shape

LiveView event handlers should be thin boundary functions.

Flag:

- event handlers that implement multi-step business workflows directly;
- repeated param parsing across handlers that should be a small boundary helper;
- assigns used as hidden mutable state for domain behavior;
- handlers that ignore failed context calls and keep rendering stale state.

Prefer handlers that parse params, call a context, update assigns from the result, and render useful errors.

## Forms, Changesets, And Uploads

Changesets should own persistence validation. LiveView/controller code should shape user input and display errors, not duplicate schema validation.

For uploads and files, also apply `security.md`: validate external input deliberately, avoid trusting filenames, and use safe path construction.

## Error Handling

Expected user mistakes should become useful user-facing errors. Exceptional internal failures can crash and produce stacktraces.

Flag:

- swallowing context errors and continuing;
- returning `{:noreply, socket}` after a failed domain operation without communicating failure;
- non-conforming LiveView return shapes;
- generic errors that make debugging harder.

## Phoenix Tests

Controller/LiveView tests should prove boundary behavior:

- params are accepted/rejected correctly;
- auth/session/org behavior is enforced;
- user-visible messages/rendered state are correct;
- context/domain effects are triggered, but deep domain behavior should be tested lower down.

Do not make every domain behavior test go through LiveView just because the UI starts it. That is slow and brittle.
