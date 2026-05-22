# Common Docs And Tests Rules

## Examples

- User-facing examples should verify behavior, not just compile.
- Shape-only, finite-only, non-empty, non-zero, or positive-rank assertions are
  weak smoke checks unless that exact property is the behavior being taught.
- Prefer known values, algebraic identities, reconstruction residuals, or
  meaningful structural invariants.
- Examples that require special hardware may be compile-checked separately, but
  the docs must say what cannot run in ordinary CI.

## Tests

- Tests should follow implementation ownership. Private implementation details
  belong in the crate/package that owns them.
- Small reference tests may materialize dense data. Long or scalable regression
  tests should use comparisons that would fail quickly if an accidental dense
  path appeared.
- Approximate numerical tests should report useful residuals such as max
  absolute error or relative norm error.
- Do not relax tolerances, skip failing coverage, or remove checks without a
  clear reason.

## Benchmarks

- Use release-mode benchmarks for performance claims.
- Pin relevant thread counts and backend configuration when comparing CPU or
  backend behavior.
- Use value-dependent `black_box` inputs and outputs. Black-boxing only a
  shape, length, or rank can hide wrong-but-shaped results.
- Keep setup cost separate from the operation being measured, or state that the
  benchmark intentionally includes setup cost.
