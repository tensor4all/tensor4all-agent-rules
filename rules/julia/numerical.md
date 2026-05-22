# Julia Numerical Rules

## Validation

- Numerical examples and tests should assert meaningful values, residuals, or
  structural invariants.
- Avoid tests that only assert shape, non-emptiness, finiteness, or successful
  execution unless that property is the behavior under test.
- For decompositions, fitting, interpolation, and tensor-network contraction,
  report residuals in a form that helps diagnose failures.

## Cross-Language Consistency

- Julia bindings should preserve Rust/C API semantics for layout, ownership,
  errors, and index identity.
- When a Julia feature depends on a new Rust or C API symbol, update the Rust
  dependency pin only after the producing Rust change is available at that pin.
- Do not expose ID-only identity helpers as user-facing selection APIs when the
  full object identity carries tags, prime levels, directions, or other
  metadata.
