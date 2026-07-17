# Common Docs And Tests Rules

## Documentation Audits

- For user-facing documentation, architecture, public API, or onboarding changes,
  prefer a source-blind documentation audit when practical: provide a reviewer
  or cheap/non-frontier agent only the rendered or compiled documentation
  artifact, not source files or repository links, and ask it to explain the
  architecture, usage path, unknowns, and documentation gaps.
- Use the findings to improve the docs or record follow-up issues before
  treating the documentation as complete. After updating user-facing docs, note
  briefly to the user or maintainer when this protocol is worth running.
- Stronger variant: when the question is whether the docs are enough to *use*
  the project (not just understand it), run a source-blind build test — have the
  doc-only agent write a minimal integration from the compiled docs alone, then
  compile-check that code against the real project. An explain-only audit can
  pass while a doer is still blocked (for example, missing trait or function
  signatures, or how to construct core objects); compiling the doc-faithful code
  surfaces those gaps concretely.

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

## Local And Hosted Validation

- Ordinary local development and focused edit-test loops should use
  non-release builds with incremental compilation enabled. Keep local checks
  proportional to the changed surface instead of requiring every contributor
  to rebuild and run the complete workspace before each pull request.
- Local pull-request preparation should run formatting and focused tests for
  changed code, relevant documentation checks for documentation-only changes,
  and focused CI-helper checks for CI-only changes. Unknown paths should fall
  back to the conservative code-change policy.
- Hosted CI owns comprehensive workspace tests, coverage enforcement, feature
  and backend matrices, documentation builds, hardware-dependent tests, and
  clean builds with incremental compilation disabled.
- Use release mode locally when optimization semantics matter: benchmarks,
  performance claims, release-only failures, unsafe or optimization-sensitive
  behavior, or an explicit maintainer request. Release mode is not the default
  for ordinary correctness-oriented edit-test loops.
- Repository-local policy may require stricter local validation for a specific
  project or change class. Such overrides should explain why hosted CI alone is
  insufficient for that risk.

## Benchmarks

- Use release-mode benchmarks for performance claims.
- Pin relevant thread counts and backend configuration when comparing CPU or
  backend behavior.
- Use value-dependent `black_box` inputs and outputs. Black-boxing only a
  shape, length, or rank can hide wrong-but-shaped results.
- Keep setup cost separate from the operation being measured, or state that the
  benchmark intentionally includes setup cost.
