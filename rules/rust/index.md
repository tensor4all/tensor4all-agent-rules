# Rust Rules Index

For Rust implementation work, read:

1. [`../common/repository.md`](../common/repository.md)
2. [`../common/performance.md`](../common/performance.md)
3. [`performance.md`](performance.md)

Also read [`numerical.md`](numerical.md) when touching numerical algorithms,
linear algebra, AD rules, oracle replay, C API behavior, or language bindings.

## Unsafe Code Boundary

`unsafe` is intentionally confined to FFI bindings and backend leaf code, and
is near-absent from the algorithmic layers. Reviewers must not read the raw
count as a red flag without first checking *where* it lives.

- **Document each block.** Every `unsafe` block carries a `// SAFETY:` comment
  naming the validation site that proves it (the checked precondition, the
  caller-provided bound, or the owning invariant). Keep unsafe next to its
  proof and test the boundary conditions of new unsafe branches.
- **Keep the higher layers unsafe-free.** Graph, algorithm, and public API
  layers are ~zero `unsafe` by design. If a kernel or FFI call is needed, it
  belongs behind the backend/FFI seam, not in graph or rule code.
- Count production `unsafe` precisely (for example with a script that skips
  `// SAFETY:` comments, doc comments, string literals, tests, and generated
  code) rather than with a plain `grep -c unsafe`.

## Unit Test Organization

- Keep production source files focused on production code. Do not keep inline
  `#[cfg(test)]` blocks in normal modules unless the file is a genuinely tiny
  leaf module and the test is trivially small.
- Prefer module-local test directories such as `src/<module>/tests/*.rs` and
  leave only `#[cfg(test)] mod tests;` in the source file. Reserve crate-root
  `tests/` for integration tests. Do not use `include!` to inject test files
  into modules.
- When splitting tests, keep a developer reading `src/**` from having to scroll
  through large unit-test blocks: split larger extracted test suites by
  concern rather than keeping one monolithic test module.
- Tests follow implementation ownership: public facade crates prefer
  integration tests for user-visible behavior; private implementation details
  are tested in the crate that owns the implementation. If a crate sets
  `[lib] test = false`, do not add crate-local unit-test entrypoints to it;
  move a private helper needing direct unit tests into the owning internal
  crate instead.

## Debug And Enum Hygiene

- Public types implement `Debug`. Prefer `derive(Debug)` when the output is
  cheap, useful, and does not expose unstable internals. Use a hand-written
  summary for graph, runtime, cache, tensor, backend, or FFI wrapper types when
  derived output would materialize data, dump large buffers, leak internal
  representation details, or make future implementation changes harder.
- Public enums that will grow use `#[non_exhaustive]` deliberately; document
  the policy per repository.
