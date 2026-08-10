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
- Default build profiles should not carry full debug information. Keep the
  default local and release profiles lean; provide an opt-in profile variant
  (or a documented one-command override) for the sessions that actually attach
  a debugger. Line-table-only debug information is usually enough for readable
  backtraces at a fraction of the size. CI profiles used for comprehensive
  runs should strip symbols.
- When hosted CI owns the measurement for a gate (coverage is the canonical
  case), the local pre-PR gate may be attestation-based: an explicit flag or
  statement that the changed code was reviewed for that property. The
  attestation must be explicit and the check must fail when it is absent;
  silence is not attestation.
- Build directories accumulate stale artifacts that no setting prevents:
  dependency version and feature churn leaves orphaned object files behind.
  Repositories should document a pruning mechanism (an age-based sweep tool or
  a periodic full clean) and agents should propose a cleanup when a build
  directory's size is clearly dominated by artifacts no current build uses.

## Benchmarks

- Use release-mode benchmarks for performance claims.
- Pin relevant thread counts and backend configuration when comparing CPU or
  backend behavior.
- Use value-dependent `black_box` inputs and outputs. Black-boxing only a
  shape, length, or rank can hide wrong-but-shaped results.
- Keep setup cost separate from the operation being measured, or state that the
  benchmark intentionally includes setup cost.

## Doc Examples

- Every public type, trait, and function must include minimal but sufficient
  usage examples in its doc comments (`/// # Examples`). `#[doc(hidden)]`
  items are exempt.
- Doc examples (`/// # Examples`) must NOT use `ignore` or `no_run`
  attributes. Every example must compile AND run as a doctest.
- An example must demonstrate real usage. Examples consisting only of path or
  assignment statements (for example `let _method = Type::method;`) satisfy
  the doctest gate without documenting anything and are not acceptable.
- Use `compile_fail` only for examples that intentionally demonstrate compile
  errors. If an example cannot run as a doctest, refactor it until it can.
- When a canonical public API changes from infallible or panicking to
  `Result`-returning, update its user-facing examples, tutorials, and guides to
  propagate recoverable errors with `?` or handle the documented error
  explicitly. Do not preserve the old call shape by appending `unwrap()` or
  `expect()`. Tests may still use an intentional assertion boundary, and an
  example may unwrap only a locally proven invariant whose proof is explicit.
- Non-trivial user-facing code snippets must have an executable source of
  truth: prefer including a checked example, test, or doctest instead of
  copying code into Markdown by hand. If a Markdown page must contain a copied
  snippet, add an automated sync or extraction check that fails when the
  snippet drifts from the executable source. Crate READMEs with code fences
  need an executable sync mechanism (for example
  `#![doc = include_str!("../README.md")]`).
- Guide code that demonstrates a workflow should compile in CI. When runtime
  execution requires special hardware or external libraries, CI must still
  compile-check the example with the required feature flags, and the guide must
  document the command that runs the example on a correctly configured machine.
- Examples that call backend operations should bind the backend to a local
  variable and reuse it for related operations instead of chaining
  `Backend::new().op(...)` beyond a single trivial construction example.

## Public Result Error Documentation Gate

- Every public function, inherent method, and public-trait method returning a
  `Result` must document a `# Errors` section. The section must name the
  concrete error variants or failure conditions the caller can observe; a
  generic "returns an error on failure" sentence is insufficient.
- Documentation for traced or symbolic APIs must describe validation deferred
  to compile or execution and identify the applicable phase when that behavior
  is part of the contract. Intentional panics use `# Panics`; deferred symbolic
  checks use `# Deferred errors`.
- `scripts/check-public-error-docs.py` audits the full Rust workspace and the
  Rust files changed by a PR and is a required CI gate. Keep the audit enabled
  without clippy or source-level allowlists; add the concrete documentation at
  the API source instead.
