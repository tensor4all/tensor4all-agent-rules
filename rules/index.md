# Tensor4all Agent Rules Index

Read only the files relevant to the current task. Start with common rules, then
load language-specific rules when the task touches that language.

## Common

- [`common/repository.md`](common/repository.md): source of truth, API surface,
  layering, dependency boundaries, and repository-local overrides.
- [`common/performance.md`](common/performance.md): general performance review
  checklist for tensor, compiler, cache, and backend work.
- [`common/docs-and-tests.md`](common/docs-and-tests.md): documentation audits,
  examples, tests, benchmarks, and validation quality.

## Rust

- [`rust/index.md`](rust/index.md): Rust-specific entry point.
- [`rust/performance.md`](rust/performance.md): Rust tensor/backend
  performance rules, including allocation, slicing, linalg, and GPU kernels.
- [`rust/numerical.md`](rust/numerical.md): numerical correctness and AD
  validation expectations.

## Julia

- [`julia/index.md`](julia/index.md): Julia-specific entry point.
- [`julia/performance.md`](julia/performance.md): Julia allocation,
  type-stability, and tensor-network performance rules.
- [`julia/numerical.md`](julia/numerical.md): Julia numerical validation and
  bindings expectations.

## Loading Policy

- Do not bulk-load the entire repository by default.
- Load common rules for any cross-repository implementation work.
- Load Rust rules for Rust crates, C API layers, backend code, or Rust docs.
- Load Julia rules for Julia packages, wrappers, examples, or docs.
- If project-local rules conflict with these shared rules, follow the more
  specific project-local rule and document the reason when it affects a PR.
