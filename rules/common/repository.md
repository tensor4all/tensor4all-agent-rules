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

## Publication And Release Safety

Releasing crates or packages to a registry is irreversible. These rules apply
to every repository that publishes:

- Maintain a canonical, versioned publication DAG for all publishable
  packages, derived from the package dependency graph (topological order over
  cross-package normal/build dependencies). A package must publish only after
  every package it depends on exists on the registry at the target version.
- A publishable package must not carry a **versioned** dev-dependency on a
  package that publishes later in the canonical order: versioned requirements
  resolve against the registry during packaging even without verification.
  Path-only (unversioned) dev-dependencies are safe for cross-layer tests
  because dev-dependencies are stripped from published manifests and never
  registry-resolve for consumers; a declared dev version must still match the
  workspace version.
- The complete dependency graph among publishable packages must not contain a
  publication cycle over versioned edges (versioned edges are the only ones
  that registry-resolve at package time; unversioned path dev-edges cannot
  create a publish-time cycle).
- Publication is a human-only action. Agents prepare and validate, then stop:
  they never execute publication and never type the final confirmation.
- For human execution, generate a guarded handoff script instead of asking
  the human to retype commands. The generated script must be mode 0700,
  `bash -n` clean, non-append, restart-safe, require a TTY, re-run the
  fail-closed preflight, and then ask for one exact lowercase confirmation
  immediately before invoking the publish helper with the execute flag. Pin
  the helper and canonical workflow with checksums computed at generation and
  re-verified at run time, so a later helper change cannot silently alter
  publication behavior. Write the script outside the release worktree when
  untracked files abort the release.
- Revalidation at release time is change-aware: classify the diff between the
  previous release and the tag (helper/workflow-only, publication-metadata
  only, semantic manifest, or source/ambiguous) and run only the validation
  lane the strongest change demands. Conservative by default: source or
  ambiguous changes run the full validation.
- Before skipping a rerun on the strength of previously passed CI, verify
  every required check run for the exact release commit (per-repository
  canonical query, e.g. GitHub check-runs with `--paginate`), requiring the
  run's commit to match, `status == "completed"`, and
  `conclusion == "success"`; anything else fails closed and reruns the
  applicable tier.

## Dependency And Boundary Discipline

- Shared dependencies should be declared once at the workspace or package
  policy layer when the ecosystem supports it.
- Keep FFI, language bindings, generated docs, and tutorial code synchronized
  with the implementation they expose.
- Do not add hidden device, backend, or dense-layout conversions across API
  boundaries. Conversion boundaries should be explicit in names, docs, or code.
