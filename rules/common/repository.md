# Common Repository Rules

## Source Of Truth

- Source code, public docs, generated API references, and repository-local
  `REPOSITORY_RULES.md` files have different roles. Do not duplicate detailed
  implementation facts across all of them.
- Keep project `AGENTS.md` files short. They should orient the agent, point to
  shared rules, and list repository-specific overrides.
- Repository-local rules override shared rules when they are more specific.
  Shared rules should capture durable cross-project policy.

## Public Surface

- Keep public APIs deliberate and small. Do not expose implementation details
  just because another module or package needs them.
- Public APIs are contracts. Before adding or keeping a public item, ask
  whether downstream users should rely on it.
- Existing public APIs that violate current policy are migration targets, not
  patterns to copy.

## Layering

- Fix behavior at the owning abstraction. Avoid compatibility shims,
  duplicated logic, and downstream reach-through into lower layers.
- Use repository-local helper APIs and established abstractions before adding
  new ones.
- If a needed behavior does not fit the current abstraction, refine the
  abstraction instead of patching around it locally.

## Dependency And Boundary Discipline

- Shared dependencies should be declared once at the workspace or package
  policy layer when the ecosystem supports it.
- Keep FFI, language bindings, generated docs, and tutorial code synchronized
  with the implementation they expose.
- Do not add hidden device, backend, or dense-layout conversions across API
  boundaries. Conversion boundaries should be explicit in names, docs, or code.
