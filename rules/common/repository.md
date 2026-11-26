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

## Build Profiles

- **Every repository that runs CI defines `[profile.ci]` and its workflows use it** (`cargo test --profile ci`,
  `cargo nextest --cargo-profile ci`, or the equivalent). The stack is release-mode first: `tensor4all-rs` inherits
  `release` with `incremental = false` and `strip = "symbols"`; `tenferro-rs` inherits `test` with the same two
  settings. A repository without the profile builds its CI in the dev profile, where Rust keeps **debug info on by
  default**.
- **Why this is not style.** In the dev profile `faer_ffi` takes over an hour to compile with debug info, and the build's
  memory is real: a worker building inside a herdr pane on a shared host grew that unit to 25.9 GiB, which is what made
  `systemd-oomd` pick it as a victim and kill the session. Two workers died that way in one night.
- **When a profile inherits `release`, decide about `debug-assertions` explicitly** rather than letting it inherit. Those
  assertions are behaviour, not debug info: turning them off changes what the tests catch, and if a suite depends on them
  that has to be a decision with a test behind it.
- **Local workarounds are not the fix.** `CARGO_PROFILE_DEV_DEBUG=0` on a command line is the escape hatch when debug info
  is wanted in the other direction; it is not a substitute for the repository's profile, because every script that forgets
  it pays the hour again.
- Run `scripts/check_build_profiles.py` in the advisor checkout to see the current state across the organisation; it
  reports both "no profile" and "profile defined but no workflow uses it".

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
- An agent may publish. It prepares and validates first, and the maintainer's
  approval is required **per release**, given in the conversation for that
  release: "finish this PR", "wrap up the work" or a general release request is
  not approval to publish. When approval is not explicit, stop and report rather
  than infer it.
- **New packages are the case that needs care.** Never publish a package that is
  not already on the registry without the maintainer's approval naming that
  package; when one release contains several new packages, ask for each and
  record the approvals. An approval is stale once the package exists but the
  target version does not.
- Publishing with the maintainer at the keyboard is still allowed, and often
  preferable for a first release of a new package. A guarded handoff script is
  the way to hand publication over without retyping commands: mode 0700,
  `bash -n` clean, non-append, restart-safe, TTY required, re-runs the fail-closed
  preflight, then asks for one exact lowercase confirmation immediately before
  invoking the publish helper with the execute flag. Pin the helper and canonical
  workflow with checksums computed at generation and re-verified at run time.
  Write the script outside the release worktree when untracked files abort the
  release. Whether the agent executes or hands over is the maintainer's call;
  either way no check may be bypassed.
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

## AI-Ready Issue Burn

Repositories that permit autonomous AI issue implementation must use a
maintainer-controlled `ai-ready` label with the description `Human-reviewed
issue authorized for autonomous AI implementation` and color `7057FF`. The
label records a human readiness decision; agents must not infer readiness from
an issue's apparent detail or apply the label during ordinary issue intake.

Maintainers may apply `ai-ready` only after human review confirms all of the
following:

- Intended and current behavior, implementation scope, and non-goals are
  explicit.
- Acceptance criteria and focused regression or verification coverage are
  precise enough for implementation without a new design decision.
- Affected packages, public APIs, dependencies, backends, devices, dtypes,
  feature flags, numerical and AD behavior, documentation, language bindings,
  cross-repository impact, and design records are identified or explicitly not
  applicable.
- Every required public-API, dependency, architecture, backend, numerical,
  provenance, compatibility, product, roadmap, and landing-order decision has
  already been accepted.
- Relevant reproduction details, source locations, references, and provenance
  constraints are present.

An AI agent may autonomously implement an open issue only while it carries the
`ai-ready` label. The issue and its linked accepted specifications are the scope
boundary. The label authorizes implementation, focused validation, and
preparation of a reviewable pull request; it does not authorize merge,
publication, release, external communication, destructive operations, scope
expansion, or bypassing human review and required CI.

If implementation exposes missing or contradictory requirements, requires a
new decision, or would expand the accepted boundary, the agent must stop the
issue burn and request maintainer clarification. A maintainer must remove
`ai-ready` until the issue is ready again.

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

Review the inner loops of every changed numerical path. Look for
per-element branches, dynamic calls, allocation and coordinate decoding.
Deferring or skipping benchmarks never waives this structural check.

Multi-agent or cross-model audits are optional and only on explicit request.
Human approval and required CI remain separate requirements. Correctness
defects and required measurements without valid evidence still block
completion.

After a correction, review the delta and rerun affected checks. Unaffected
evidence need not be regenerated because the commit changed; record why it
still applies.
