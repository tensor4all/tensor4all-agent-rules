# Rust Numerical Rules

## Correctness

- Numerical algorithms need tests for representative values, edge cases, layout
  variants, dtype variants, and error branches.
- AD rules need oracle or finite-difference coverage before being treated as
  supported mainline behavior.
- Check decompositions with reconstruction or residual tests when possible, not
  only output shapes or sorted spectra.

## Error Handling

- Public Rust library APIs prefer crate-local typed errors over unstructured
  `anyhow::Result` unless the repository explicitly allows it.
- Error messages preserve enough context for C API layers and language bindings
  to report useful diagnostics.

## FFI And Bindings

- Keep Rust, C API, and language-binding contracts synchronized.
- Do not expose low-level Rust internals through FFI to avoid adding the right
  high-level API.
- Flat-buffer order, ownership, device residency, and error semantics are
  explicit at FFI boundaries.

## Typed Errors

- Public validation APIs return crate error types, never `String` or `&str`.
- Conversions preserve the `source()` chain across crate boundaries; converting
  a typed error to `String` is permitted only for display/logging, vendor/FFI
  arguments, final serialization, or an explicit message-only external
  protocol boundary.
- Equivalent operation paths (e.g. eager and traced/symbolic) report the same
  validation kind and payload for the same known input; the detection phase
  (graph build, compile, execution) is a separate orthogonal axis, not a
  different error kind.
- Classify failures by kind, not opaque text: validation facts use a shared
  typed vocabulary for shape, rank, axis, dtype, configuration, and invalid
  arguments; unsupported operations and dtypes use an `Unsupported` category
  with a typed local source when one exists; numeric domain failures
  (singularity, non-convergence, division by zero) use a numerical-failure
  category retaining the owning crate's typed source; backend/kernel errors use
  a source-preserving wrapper; file/stream/serialization failures use an I/O
  category retaining their source; invalid runtime state uses a runtime-state
  category; impossible internal invariants remain `Internal`. None of these are
  catch-alls for known input validation.
- When a documented alternative API, explicit conversion, feature, or
  supported-value set provides one reliable remediation, append it as
  `<what failed>; <what to do>`. Do not invent a remedy when no universal next
  action exists.
- Error enums use `thiserror` (or the repository's established derive) with
  `#[source]` preservation and structured fields. Prefer typed local sources
  over string payloads for recurring cases.
