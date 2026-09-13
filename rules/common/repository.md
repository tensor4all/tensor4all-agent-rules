# Common Repository Rules

## Source Of Truth

- Source code, public docs, generated API references, and repository-local
  `REPOSITORY_RULES.md` have different roles. Do not duplicate detailed
  implementation facts across them.
- Keep project `AGENTS.md` short: orient the agent, point to shared rules, list
  repository-specific overrides.
- Repository-local rules override shared rules when more specific. Shared rules
  capture durable cross-project policy.

## Proportionate Workflow

- Work directly by default. Subagents, panels, and independent AI reviews
  require an explicit user request; different model families are not a
  mandatory gate.
- Read applicable rules once and revisit affected sections when scope changes.
  Do not reload unchanged policy files before every small edit.
- Routine fixes need a focused test and a self-review of the final diff, not a
  separate design approval. Record a short design for changes to public
  contracts, architecture, or safety-critical boundaries.
- A correction within the agreed design does not restart design approval or a
  full review. Recheck the affected behavior; reopen decisions only when scope
  or semantics change.
- If work stalls, reassess and report blockers instead of adding infrastructure
  or repeating failed delegation/review cycles.
- None of this waives required CI, human approvals, numerical correctness,
  memory safety, or data preservation.

## Public Surface

- Keep public APIs deliberate and small. Do not expose implementation details
  because another module or package needs them.
- Public APIs are contracts: before adding or keeping a public item, ask whether
  downstream users should rely on it.
- Existing public APIs that violate current policy are migration targets, not
  patterns to copy.

## Layering

- Fix behavior at the owning abstraction. No compatibility shims, duplicated
  logic, or downstream reach-through into lower layers.
- Use repository-local helpers and established abstractions before adding new
  ones.
- If a behavior does not fit the current abstraction, refine the abstraction
  instead of patching around it.

## Publication And Release Safety

Registry publication is irreversible. For every publishing repository:

- Maintain a canonical, versioned publication DAG for all publishable
  packages, derived from the package dependency graph (topological order over
  cross-package normal/build dependencies). A package publishes only after
  every dependency exists on the registry at the target version.
- A publishable package must not carry a **versioned** dev-dependency on a
  package that publishes later: versioned requirements registry-resolve during
  packaging even without verification. Path-only (unversioned) dev-dependencies
  are safe for cross-layer tests because dev-dependencies are stripped from
  published manifests; a declared dev version must still match the workspace
  version.
- The publishable-package graph must not contain a publication cycle over
  versioned edges (unversioned path dev-edges cannot create one).
- Publication is human-only. Agents prepare and validate, then stop: never
  execute publication or type the final confirmation.
- Generate a guarded handoff script for the human instead of asking them to
  retype commands: mode 0700, `bash -n` clean, non-append, restart-safe, TTY
  required, re-runs the fail-closed preflight, then asks for one exact
  lowercase confirmation immediately before invoking the publish helper with
  the execute flag. Pin the helper and canonical workflow with checksums
  computed at generation and re-verified at run time. Write the script outside
  the release worktree when untracked files abort the release.
- Release-time revalidation is change-aware: classify the diff between the
  previous release and the tag (helper/workflow-only, publication-metadata
  only, semantic manifest, or source/ambiguous) and run only the lane the
  strongest change demands. Source or ambiguous changes run full validation.
- Before skipping a rerun on the strength of prior CI, verify every required
  check run for the exact release commit (per-repository canonical query, e.g.
  GitHub check-runs with `--paginate`): commit matches,
  `status == "completed"`, `conclusion == "success"`. Anything else fails
  closed and reruns the applicable tier.

## Dependency And Boundary Discipline

- Declare shared dependencies once at the workspace or package policy layer
  when the ecosystem supports it.
- Keep FFI, language bindings, generated docs, and tutorial code synchronized
  with the implementation they expose.
- No hidden device, backend, or dense-layout conversions across API boundaries.
  Make conversion boundaries explicit in names, docs, or code.

## Invariant Markers

- One canonical marker, `// INVARIANT: <why this is valid, bounded, or
  intentional>`, records non-obvious invariants kept for performance or by
  design: rank-bounded quadratic loops, checked-by-construction arithmetic,
  ownership-required copies, semantic zero-fill, constant-bounded scans, and
  similar audit false-positive hotspots. Keep `// SAFETY:` for unsafe blocks.
- State the invariant concretely enough to re-verify: name the validation site,
  the bound, or the owning contract, not just intent.
- `#[allow(...)]` for repository-mandated lints needs an adjacent
  `// INVARIANT:` line. Allowlist and ratchet-baseline entries in audit tooling
  need a rationale string; an entry without one is a defect.
- Audit tooling and prompts, human or AI, must not flag a site governed by an
  `// INVARIANT:` marker as a violation. Check whether the stated invariant
  still holds and report only when it does not.
- Reject unsupported findings with source evidence. Add a marker or test only
  when the invariant needs clarification or regression protection; an incorrect
  review report alone requires no code change.

## API Evolution

- When a bug or requirement exposes an API design mismatch, fix the canonical
  contract, not a parallel escape hatch.
- No `try_*` compatibility escapes beside a panicking canonical API, and no
  deprecated panicking shims unless a maintainer explicitly requires a
  compatibility window (document the window and its end date).
- Prefer removing or renaming a public item over a shim under the old name;
  early-development repositories remove obsolete APIs directly.

## Output-Update Naming Principle

- Mutation semantics must be visible in the name: unsuffixed returns an owned
  result, `_mut`-style mutates in place, `_into`-style writes into a
  caller-provided output.
- An `_into`-style overwrite must not read the previous output value.
- Accumulation and destructive in-place update get their own explicit
  vocabulary. Concrete suffixes are per-repository and documented in
  repository-local rules.

## File Organization

- Keep source files small and focused, but never split solely to reduce line
  count. ~1000 lines is a soft review trigger, not a limit; one coherent
  concern may stay larger until a clear behavior, abstraction, feature, or
  ownership boundary exists.
- A split must make the code easier to reason about: parsing, plan
  computation, execution, dispatch, public API, operation families, backend
  glue, validation, cache ownership. No `part1`/`part2` splits and no tiny
  files that scatter one concept across many modules.
- Line count decides where to inspect first; responsibility, change frequency,
  public/private boundaries, and human navigation decide whether and how to
  split.

## Work Logs And Design Records

- Keep a lightweight work log for nontrivial multi-phase changes, non-obvious
  design choices, or performance experiments. Small fixes and AI assistance
  alone do not require one; a commit or PR explanation suffices.
- Record the chosen approach and why, important alternatives, verification
  conclusions, and remaining constraints or unverified areas. Do not list
  commands run (Cargo or otherwise), files read, agent activity, or the
  chronology of edits and reviews. Mention failed attempts only when they
  explain a decision or remaining limitation.
- Keep one record per change theme. Update it when decisions, conclusions, or
  constraints change, not after every correction. Historical work logs need
  not be rewritten to match this format.
- Link detailed evidence or existing reproduction instructions only when
  needed; retain special settings essential to reproduce a result. Do not
  duplicate execution histories elsewhere to satisfy the work-log policy.
  Required validation and performance evidence remain unchanged.
- Link the work log from the PR rather than repeating it. Preserve the record
  independently of squash or non-squash merge; this policy does not choose a
  merge method. Read a linked record when reviewing the choices it explains.
- When a PR establishes or changes durable design intent, update the relevant
  `docs/design/` document in the same PR. Work logs hold session-level
  rationale; design docs hold decisions future work must follow.
- Explain a rejected finding briefly with source evidence. Add a comment or
  regression test only when a real non-obvious invariant warrants it; do not
  modify correct code to close an inaccurate report.
- Before adding an audit or repository rule, inventory nearby rules and merge,
  tighten, or relocate overlaps. Prefer one sharper general rule over many
  narrow bullets.

## Final Integration Review

For multi-phase work, the main agent checks the integrated result against the
agreed requirements before declaring completion: identify the candidate state,
relevant tests, unresolved findings, and unavailable hardware; check affected
architecture, lifecycle, numerical, performance, and documentation boundaries.
Do not create fixed reviewer roles or duplicate task-level evidence.

Multi-agent or cross-model audits are optional and only on explicit request.
Human approval and required CI remain separate requirements. Correctness
defects and required measurements without valid evidence still block
completion.

After a correction, review the delta and rerun affected checks. Unaffected
evidence need not be regenerated because the commit changed; record why it
still applies.
