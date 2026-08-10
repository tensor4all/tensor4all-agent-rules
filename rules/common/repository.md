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
- Rejecting an audit finding as a false positive is complete only when the
  marker (or a source-contract test) has landed at the site, per the
  false-positive ledger rule in Work Logs And Design Records.

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

- Nontrivial refactors, cleanup streams, AI-assisted implementation, and PRs
  that make explicit design tradeoffs must leave a curated work log under
  `docs/worklogs/`. The work log should record the session summary, code and
  documents read, reference implementations considered, decisions made,
  alternatives rejected or deferred, verification performed, and remaining
  risks.
- Work logs are not raw transcripts and are not implementation plans. They are
  reviewer-facing decision records for the completed work. Keep them concise
  enough to review, but specific enough that a later reviewer can understand
  why an abstraction, split, macro, descriptor, public API choice, or deferral
  was selected.
- PR bodies for work that requires a work log must link the relevant
  `docs/worklogs/` file. Reviewers should read linked work logs before
  challenging scope, abstraction choices, or design intent.
- When a PR establishes or changes durable design intent, update the
  appropriate document under `docs/design/` in the same PR. Use work logs for
  session-level rationale and design docs for decisions future implementation
  and review should continue to follow.
- When a bug report or audit finding is a false positive because of an
  intentional invariant, record the evidence in the issue or PR ledger and add
  a nearby `// INVARIANT:` source comment (see Invariant Markers above),
  rustdoc note, or source-contract test when that invariant is not obvious
  from the code. Do not just skip the
  finding; leave enough context that later humans and AI agents do not
  rediscover the same non-bug as suspicious.
- Before adding a new audit or repository rule, inventory nearby existing rules
  and merge, tighten, or relocate overlapping guidance when possible. Prefer
  one sharper general rule over many narrow bullets that future agents must
  reconcile.

## Final Cross-Phase Multi-Agent Audit

Human/process protocol. This section is intentionally not routed to the
diff-scoped review bot.

Repository-scale, multi-phase implementation programs require one final audit
after every phase and its task-local reviews are complete, but before the
umbrella issue or implementation branch is declared ready for integration.

- Audit one exact candidate commit. Every report must name that commit, and an
  auditor must not audit a lane whose implementation or task-local review it
  performed. The lanes may run in batches when agent concurrency is limited.
- Assign a distinct independent auditor to each required lane:
  1. **Specification and architecture:** accepted issues, phase acceptance
     criteria, semantic parity, lowering, and migration compatibility.
  2. **Safety and resource lifecycle:** aliasing, unsafe boundaries,
     lifetimes, locks, buffers, caches, and cleanup on success, error,
     cancellation, and unwind.
  3. **Performance and parallelism:** current-main baseline, fast paths,
     allocations and request/container overhead, worker ownership,
     thread-count and placement control, and backend synchronization.
  4. **Public API and documentation:** facade boundaries, typed errors,
     feature combinations, runnable examples, and source/checker consistency.
  5. **Backend and hardware lanes** relevant to the repository: CPU
     placement and resource arbitration, GPU/multi-device context ownership,
     and cross-device failure handling when such backends exist.
- After all lane reports, a separate integration auditor must check
  cross-phase invariants, duplicated or contradictory findings, and the
  closure evidence.
- Each lane report must record the candidate commit; relevant feature,
  toolchain, and hardware configuration; inspected files, public contracts,
  and issue acceptance criteria; fresh commands and complete result
  classifications; findings classified as `Critical`, `Important`, or
  `Minor`; and explicit limitations or skipped hardware paths. Performance
  results must be classified as `PASS`, `FAIL`, or `INCONCLUSIVE`. Do not infer
  a pass from an implementer's earlier run. Source scanners and mutation tests
  support, but do not replace, call-path review and runtime tests.
- Each lane applies the repository's applicable rule sections (public boundary
  audits, unsafe boundary, materialization/copies, performance-gated
  experiment protocol, cache ownership, documentation policy, work logs) to
  its scope instead of restating their checklists.
- Environment-limited CPU, GPU, or multi-device paths must retain reproducible
  diagnostics and an identified verification owner.
- This gate supplements rather than replaces task-local TDD, specification
  review, code-quality review, CI, and required performance gates.
- The final audit passes only when every `Critical` and `Important` finding is
  fixed and independently re-reviewed; every `Minor` finding is fixed or has a
  written rationale and accepted tracking issue; every required performance
  gate is `PASS`; and the integration auditor reports no unresolved
  cross-phase contradiction. `INCONCLUSIVE` blocks promotion until a valid
  rerun or explicit accepted scope decision is recorded.
- The final worklog must link every lane report, the integration report, the
  exact candidate commit, and the final verification commands.
- Auditing is read-only: audit agents must not modify the candidate while
  reviewing it. A finding fix creates a new exact candidate revision. Before
  the audit can pass, every lane report must be refreshed to name and validate
  that final revision: each auditor reviews the intervening diff, every
  affected lane reruns its relevant evidence, and an unaffected lane may carry
  earlier runtime evidence forward only with a recorded diff-impact rationale.
  The separate integration auditor runs last against the same final revision.
