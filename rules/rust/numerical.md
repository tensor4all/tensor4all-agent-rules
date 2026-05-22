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
