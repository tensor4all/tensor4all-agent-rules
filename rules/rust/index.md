# Rust Rules Index

For Rust work, read [`../common/repository.md`](../common/repository.md),
[`../common/performance.md`](../common/performance.md), and
[`performance.md`](performance.md). Also read [`numerical.md`](numerical.md)
when touching numerical algorithms, linear algebra, AD rules, oracle replay,
C API behavior, or language bindings. Read [`build-artifacts.md`](build-artifacts.md)
when cleaning `target/` trees, diagnosing disk usage from build outputs, or
working with a compiler cache around Rust builds.

## Unsafe Code Boundary

`unsafe` is confined to FFI bindings and backend leaf code and near-absent from
algorithmic layers. Reviewers judge *where* it lives, not the raw count.

- Every `unsafe` block carries a `// SAFETY:` comment naming the validation
  site that proves it (checked precondition, caller-provided bound, or owning
  invariant). Keep unsafe next to its proof and test the boundary conditions of
  new unsafe branches.
- Graph, algorithm, and public API layers stay ~zero `unsafe`. Kernel or FFI
  calls belong behind the backend/FFI seam.
- Count production `unsafe` precisely (skip `// SAFETY:` comments, doc
  comments, string literals, tests, generated code), not with
  `grep -c unsafe`.

## Unit Test Organization

- Production source files hold production code. No inline `#[cfg(test)]`
  blocks in normal modules unless the file is a genuinely tiny leaf module and
  the test is trivially small.
- Prefer `src/<module>/tests/*.rs` with only `#[cfg(test)] mod tests;` in the
  source file. Crate-root `tests/` is for integration tests. Do not `include!`
  test files into modules.
- Split larger extracted test suites by concern rather than one monolithic
  module, so `src/**` readers never scroll through large test blocks.
- Tests follow implementation ownership: public facade crates prefer
  integration tests for user-visible behavior; private details are tested in
  the owning crate. If a crate sets `[lib] test = false`, do not add
  crate-local unit-test entrypoints; move a private helper needing direct unit
  tests into the owning internal crate.

## Debug And Enum Hygiene

- Public types implement `Debug`. Prefer `derive(Debug)` when cheap, useful,
  and not exposing unstable internals. Hand-write a summary for graph, runtime,
  cache, tensor, backend, or FFI wrapper types when derived output would
  materialize data, dump large buffers, leak representation details, or hinder
  future changes.
- Public enums that will grow use `#[non_exhaustive]` deliberately; document
  the policy per repository.
