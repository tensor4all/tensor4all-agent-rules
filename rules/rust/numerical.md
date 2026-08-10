# Rust Numerical Rules

## Correctness

- Numerical algorithms need tests for representative values, edge cases,
  layout variants, dtype variants, and error branches.
- AD rules need oracle or finite-difference coverage before being treated as
  supported mainline behavior.
- Decompositions should be checked with reconstruction or residual tests when
  possible, not only output shapes or sorted spectra.

## Error Handling

- Public Rust library APIs should prefer crate-local typed errors over
  unstructured `anyhow::Result` unless the repository explicitly allows it.
- Error messages should preserve enough context for C API layers and language
  bindings to report useful diagnostics.

## FFI And Bindings

- Keep Rust, C API, and language-binding contracts synchronized.
- Do not expose low-level Rust internals through FFI to avoid adding the right
  high-level API.
- Flat-buffer order, ownership, device residency, and error semantics must be
  explicit at FFI boundaries.

## Typed Errors

- Public validation APIs return crate error types, never `String` or `&str`.
- Conversions preserve the `source()` chain across crate boundaries; converting
  a typed error to `String` is permitted only for display/logging, vendor/FFI
  arguments, final serialization, or an explicit message-only external
  protocol boundary.
- Equivalent operation paths (for example eager and traced/symbolic
  variants) must report the same validation kind and payload for the same
  known input; the phase at which the failure is detected (graph build,
  compile, or execution) is a separate orthogonal axis, not a different error
  kind.
- Classify failures by kind rather than by opaque text: validation facts use a
  shared typed vocabulary for shape, rank, axis, dtype, configuration, and
  invalid arguments; unsupported operations and unsupported dtypes use an
  `Unsupported` category with a typed local source when one exists; numeric
  domain failures (singularity, non-convergence, division by zero) use a
  numerical-failure category retaining the owning crate's typed source;
  backend/kernel errors use a source-preserving wrapper; file/stream/
  serialization failures use an I/O category retaining their source; invalid
  runtime state uses a runtime-state category; impossible internal invariants
  remain `Internal`. These categories must not be used as catch-alls for known
  input validation.
- When a documented alternative API, explicit conversion, feature, or
  supported-value set provides one reliable remediation, append it to the
  diagnosis as `<what failed>; <what to do>`. Do not invent a remedy for an
  arbitrary failure when no universal next action exists.
- Error enums use `thiserror` (or the repository's established derive) with
  `#[source]` preservation and structured fields. Prefer typed local sources
  over string payloads for recurring cases.
