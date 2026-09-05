# Common Repository Rules

## Source Of Truth

- Source code, public docs, generated API references, and repository-local
  `REPOSITORY_RULES.md` files have different roles. Do not duplicate detailed
  implementation facts across all of them.
- Keep project `AGENTS.md` files short. They should orient the agent, point to
  shared rules, and list repository-specific overrides.
- Repository-local rules override shared rules when they are more specific.
  Shared rules should capture durable cross-project policy.

## Proportionate Workflow

- Work directly by default. Subagents, panels, and independent AI reviews are
  optional and require an explicit user request; using different model families
  is not a mandatory gate.
- Read the applicable rules once and revisit affected sections when scope changes.
  Do not repeatedly load unchanged policy files before every small edit.
- Routine fixes need a focused test and a self-review of the coherent final diff,
  not a separate design approval. Record a short design for changes to public
  contracts, architecture, or safety-critical boundaries.
- A correction within the agreed design does not restart design approval or a
  full review. Recheck the affected behavior; reopen decisions only when their
  scope or semantics change.
- If work stalls, reassess the approach and report blockers rather than adding
  more infrastructure or repeating unsuccessful delegation/review cycles.
- These workflow defaults do not waive required CI, human approvals, numerical
  correctness, memory safety, or data preservation.

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

## Invariant Markers

- Use one canonical marker, `// INVARIANT: <why this is valid, bounded, or
  intentional>`, for source comments that record a non-obvious invariant kept
  for performance or by design: rank-bounded quadratic loops,
  checked-by-construction arithmetic, ownership-required copies, semantic
  zero-fill, constant-bounded scans, and similar audit false-positive
  hotspots. Keep `// SAFETY:` for unsafe-block justifications.
- The marker must state the invariant concretely enough that a later reader
  can re-verify it — name the validation site, the bound, or the owning
  contract — not just assert intent.
- `#[allow(...)]` suppressions for repository-mandated lints must carry an
  adjacent `// INVARIANT:` line. Allowlist and ratchet-baseline entries in
  audit tooling must carry a rationale string; an entry without a reason is a
  defect.
- Audit tooling and audit prompts, human or AI, must not flag a site governed
  by an `// INVARIANT:` marker as a violation. They must instead check whether
  the stated invariant still holds and report only when it does not.
- Reject unsupported findings with source evidence. Add a marker or test only
  when the invariant needs clarification or regression protection; an incorrect
  review report alone does not require a code change.

## API Evolution

- When a bug or requirement exposes an API design mismatch, fix the canonical
  contract instead of adding a parallel escape hatch.
- Do not add `try_*` compatibility escapes beside a panicking canonical API,
  and do not keep deprecated panicking shims unless a maintainer explicitly
  requires a compatibility window (document the window and its end date).
- Removing or renaming a public item is preferred over retaining a shim under
  the old name; early-development repositories remove obsolete APIs directly.

## Output-Update Naming Principle

- The mutation semantics of an operation must be visible in its name: an
  unsuffixed operation returns an owned result, an `_mut`-style name mutates
  in place, and an `_into`-style name writes into a caller-provided output.
- An `_into`-style overwrite name must not read the previous output value.
- Accumulation and destructive in-place update get their own explicit
  vocabulary. Concrete suffix choices stay per-repository and must be
  documented in the repository-local rules.

## File Organization

- Keep source files small and focused, but do not split files solely to reduce
  line count. Treat ~1000 lines as a soft review trigger, not a mechanical
  limit. A file that remains one coherent concern may stay above that size
  until there is a clear behavior, abstraction, feature, or ownership boundary
  to extract.
- When splitting, the split must make the code easier to reason about: prefer
  boundaries such as parsing, plan computation, execution, dispatch, public
  API, operation families, backend glue, validation, or cache ownership.
  Avoid arbitrary `part1` / `part2` splits and avoid tiny files that force
  readers to chase one concept across many modules.
- Use line count to decide where to inspect first; use responsibility, change
  frequency, public/private API boundaries, and human navigation to decide
  whether and how to split.

## Work Logs And Design Records

- Use a concise work log for multi-phase work or non-obvious design tradeoffs.
  A small fix can record its rationale and checks in the PR body; AI assistance
  alone does not require another document.
- Record decisions, verification and unresolved risks once. Link that record
  instead of duplicating it across a design, work log, issue and PR.
- Work logs are curated decision records, not transcripts or per-edit approval
  ledgers. Read a linked record when reviewing the design choices it explains.
- When a PR establishes or changes durable design intent, update the
  appropriate document under `docs/design/` in the same PR. Use work logs for
  session-level rationale and design docs for decisions future implementation
  and review should continue to follow.
- Explain a rejected finding briefly with source evidence. Add a nearby comment
  or regression test when an actual non-obvious invariant warrants it; do not
  modify correct code merely to close an inaccurate review report.
- Before adding a new audit or repository rule, inventory nearby existing rules
  and merge, tighten, or relocate overlapping guidance when possible. Prefer
  one sharper general rule over many narrow bullets that future agents must
  reconcile.

## Final Integration Review

For multi-phase work, the main agent checks the integrated result against the
agreed requirements before declaring completion. Identify the candidate state,
relevant tests, unresolved findings and unavailable hardware. Check affected
architecture, lifecycle, numerical, performance and documentation boundaries;
do not create a fixed set of reviewer roles or duplicate task-level evidence.

A multi-agent or cross-model audit is optional, only when explicitly requested.
Human approval and required CI remain separate requirements. Correctness defects
and required measurements without valid evidence still block completion.

After a correction, review the relevant delta and rerun affected checks. Unaffected
evidence need not be regenerated solely because the commit changed. Record why
it still applies; do not restart every review lane for a small fix.
