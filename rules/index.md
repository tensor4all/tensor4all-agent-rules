# Tensor4all Agent Rules Index

Read only the files relevant to the current task. Start with common rules, then
load language-specific rules when the task touches that language.

## Common

- [`common/repository.md`](common/repository.md): source of truth, API surface,
  layering, dependency boundaries, publication/release safety, invariant
  markers, API evolution, output-update naming, file organization, work logs,
  final cross-phase audits, and repository-local overrides.
- [`common/performance.md`](common/performance.md): general performance review
  checklist, the performance-gated experiment protocol, cache ownership, and
  complexity budget for tensor, compiler, cache, and backend work.
- [`common/docs-and-tests.md`](common/docs-and-tests.md): documentation audits,
  doc-example policy (no `ignore`/`no_run`), the public `Result` error-doc
  gate, tests, benchmarks, and validation quality.
- [`common/provenance.md`](common/provenance.md): recording references to
  third-party code in the source, copyright compliance for ports and
  translations, scientific credit via provenance and citation policies, and
  permission-gated upstream bug feedback.
- [`common/agent-consumers.md`](common/agent-consumers.md): serving downstream
  users and coding agents of user-facing libraries: bundled usage skills,
  remedy clauses in error messages, package-index metadata gates, and
  verified llms.txt indexes.

## Rust

- [`rust/index.md`](rust/index.md): Rust-specific entry point; unsafe
  boundary, unit-test organization, and debug/enum hygiene.
- [`rust/performance.md`](rust/performance.md): Rust tensor/backend
  performance rules, including allocation, slicing, linalg, GPU kernels,
  uninitialized/scratch acquisition, and threading principles.
- [`rust/numerical.md`](rust/numerical.md): numerical correctness, AD
  validation expectations, and typed-error classification.

## Julia

- [`julia/index.md`](julia/index.md): Julia-specific entry point.
- [`julia/performance.md`](julia/performance.md): Julia allocation,
  type-stability, and tensor-network performance rules.
- [`julia/numerical.md`](julia/numerical.md): Julia numerical validation and
  bindings expectations.

## Loading Policy

- Do not bulk-load the entire repository by default.
- Load common rules for any cross-repository implementation work.
- Load agent-consumers rules when working on docs sites, packaging or release
  metadata, error messages, usage skills, or README quickstarts of a
  user-facing library.
- Load Rust rules for Rust crates, C API layers, backend code, or Rust docs.
- Load Julia rules for Julia packages, wrappers, examples, or docs.
- If project-local rules conflict with these shared rules, follow the more
  specific project-local rule and document the reason when it affects a PR.
