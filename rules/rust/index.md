# Rust Rules Index

For Rust work, read [`../common/repository.md`](../common/repository.md),
[`../common/performance.md`](../common/performance.md), and
[`performance.md`](performance.md). Also read [`numerical.md`](numerical.md)
when touching numerical algorithms, linear algebra, AD rules, oracle replay,
C API behavior, or language bindings. Read [`build-artifacts.md`](build-artifacts.md)
when cleaning `target/` trees, diagnosing disk usage from build outputs, setting
Cargo profile options that change artifact size, or working with a compiler
cache around Rust builds.

## Minimum Supported Rust Version

A tensor4all crate is a general-purpose library; its `rust-version` is a
promise to downstream users, not a record of the toolchain its authors use.

- Develop and run the ordinary CI jobs on the latest stable Rust, and declare
  as `rust-version` the lowest version the crate actually builds and tests on.
  Do not tie the MSRV to the latest stable or align it across crates for
  uniformity alone.
- Raise the MSRV only for a technical reason: a language or standard-library
  feature the code now uses, or a dependency whose own `rust-version` requires
  it. Name the reason in the PR and in the CHANGELOG entry.
- Every crate declares `rust-version` in `Cargo.toml` and has a CI job that
  builds and tests on exactly that version. A crate with no declared MSRV
  (for example a foundation crate such as `strided-rs`) picks one and adds
  that job.
- A crate that provides a tenferro adapter, or that tenferro depends on, keeps
  an MSRV no higher than tenferro-rs's whenever it can, so that a user of the
  stack is never forced past tenferro's own MSRV. A dependency that must go
  higher is raised with the user before it lands.

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
- A fork of an upstream project keeps the upstream layout for files that came
  from upstream, inline test modules included. Only files the fork adds follow
  this rule; moving upstream tests widens the diff against upstream and makes
  every rebase harder.

## Debug And Enum Hygiene

- Public types implement `Debug`. Prefer `derive(Debug)` when cheap, useful,
  and not exposing unstable internals. Hand-write a summary for graph, runtime,
  cache, tensor, backend, or FFI wrapper types when derived output would
  materialize data, dump large buffers, leak representation details, or hinder
  future changes.
- Public enums that will grow use `#[non_exhaustive]` deliberately; document
  the policy per repository.
