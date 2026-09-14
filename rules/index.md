# Tensor4all Agent Rules Index

Read only the files relevant to the task: common rules first, then
language-specific rules when the task touches that language.

## Common

- [`common/repository.md`](common/repository.md): source of truth, API surface,
  layering, dependency boundaries, publication/release safety, invariant
  markers, API evolution, output-update naming, file organization, work logs,
  final integration review, repository-local overrides.
- [`common/performance.md`](common/performance.md): performance checklist,
  performance-gated experiment protocol, cache ownership, complexity budget.
- [`common/docs-and-tests.md`](common/docs-and-tests.md): documentation audits,
  doc-example policy (no `ignore`/`no_run`), public `Result` error-doc gate,
  tests, benchmarks, local vs hosted validation.
- [`common/provenance.md`](common/provenance.md): recording third-party
  references in source, copyright for ports and translations, scientific
  credit and citation policy, permission-gated upstream bug feedback.
- [`common/agent-consumers.md`](common/agent-consumers.md): serving downstream
  users and coding agents: bundled usage skills, remedy clauses in errors,
  package-index metadata gates, verified llms.txt indexes.

## Rust

- [`rust/index.md`](rust/index.md): entry point; unsafe boundary, unit-test
  organization, debug/enum hygiene.
- [`rust/performance.md`](rust/performance.md): allocation, slicing, linalg,
  GPU kernels, uninitialized/scratch acquisition, build profiles, threading.
- [`rust/numerical.md`](rust/numerical.md): numerical correctness, AD
  validation, typed-error classification.
- [`rust/build-artifacts.md`](rust/build-artifacts.md): where Cargo and compiler
  caches put build outputs; sweeping stale `target/` trees safely.

## Julia

- [`julia/index.md`](julia/index.md): entry point.
- [`julia/performance.md`](julia/performance.md): allocation, type stability,
  tensor-network performance.
- [`julia/numerical.md`](julia/numerical.md): numerical validation and bindings.

## Loading Policy

- Do not bulk-load the repository by default.
- Load common rules for any cross-repository implementation work.
- Load agent-consumers rules for docs sites, packaging or release metadata,
  error messages, usage skills, or README quickstarts of a user-facing library.
- Load Rust rules for Rust crates, C API layers, backend code, or Rust docs.
- Load Julia rules for Julia packages, wrappers, examples, or docs.
- When project-local rules conflict with shared rules, follow the more specific
  project-local rule and document the reason when it affects a PR.
