# Common Docs And Tests Rules

## Documentation Audits

- For user-facing documentation, architecture, public API, or onboarding
  changes, prefer a source-blind documentation audit when practical: give a
  reviewer or cheap/non-frontier agent only the rendered or compiled
  documentation artifact (no source files or repository links) and ask it to
  explain the architecture, usage path, unknowns, and documentation gaps.
- Improve the docs or record follow-up issues from the findings before treating
  the documentation as complete. After updating user-facing docs, note briefly
  to the user or maintainer when this protocol is worth running.
- Stronger variant, when the question is whether the docs suffice to *use* the
  project: a source-blind build test. The doc-only agent writes a minimal
  integration from the compiled docs alone, and that code is compile-checked
  against the real project. An explain-only audit can pass while a doer is
  still blocked (missing trait or function signatures, how to construct core
  objects).

## Examples

- User-facing examples verify behavior, not just compile.
- Shape-only, finite-only, non-empty, non-zero, or positive-rank assertions are
  weak smoke checks unless that property is the behavior being taught. Prefer
  known values, algebraic identities, reconstruction residuals, or meaningful
  structural invariants.
- Examples needing special hardware may be compile-checked separately; the docs
  must say what cannot run in ordinary CI.

## Tests

- Tests follow implementation ownership: private details are tested in the
  crate/package that owns them.
- Small reference tests may materialize dense data. Long or scalable regression
  tests use comparisons that fail quickly if an accidental dense path appears.
- Approximate numerical tests report useful residuals (max absolute error,
  relative norm error).
- Do not relax tolerances, skip failing coverage, or remove checks without a
  clear reason.

## Local And Hosted Validation

- Ordinary local development and focused edit-test loops use non-release builds
  with incremental compilation enabled. Keep local checks proportional to the
  changed surface; do not require rebuilding and running the whole workspace
  before each PR.
- Local PR preparation runs formatting and focused tests for changed code,
  relevant documentation checks for documentation-only changes, and focused
  CI-helper checks for CI-only changes. Unknown paths fall back to the
  conservative code-change policy.
- Hosted CI owns comprehensive workspace tests, coverage enforcement, feature
  and backend matrices, documentation builds, hardware-dependent tests, and
  clean builds with incremental compilation disabled.
- Use release mode locally only when optimization semantics matter: benchmarks,
  performance claims, release-only failures, unsafe or optimization-sensitive
  behavior, or explicit maintainer request.
- Repository-local policy may require stricter local validation for a specific
  project or change class; the override must explain why hosted CI alone is
  insufficient.
- Default build profiles do not carry full debug information. Keep default
  local and release profiles lean; provide an opt-in profile or documented
  one-command override for debugger sessions. Line-table-only debug info
  usually suffices for backtraces. Comprehensive CI profiles strip symbols.
- When hosted CI owns a gate's measurement (coverage is canonical), the local
  pre-PR gate may be attestation-based: an explicit flag or statement that the
  changed code was reviewed for that property. The check must fail when the
  attestation is absent; silence is not attestation.
- Build directories accumulate stale artifacts from dependency and feature
  churn that no setting prevents. Repositories document a pruning mechanism
  (age-based sweep or periodic full clean); agents propose a cleanup when a
  build directory is clearly dominated by artifacts no current build uses.

## Benchmarks

- Release-mode benchmarks for performance claims.
- Pin thread counts and backend configuration when comparing CPU or backend
  behavior.
- Use value-dependent `black_box` inputs and outputs; black-boxing only a
  shape, length, or rank hides wrong-but-shaped results.
- Keep setup cost separate from the measured operation, or state that setup is
  intentionally included.

## Doc Examples

- Every public type, trait, and function has minimal but sufficient usage
  examples in its doc comments (`/// # Examples`); `#[doc(hidden)]` items are
  exempt.
- Doc examples must NOT use `ignore` or `no_run`. Every example compiles AND
  runs as a doctest. Use `compile_fail` only for intentional compile errors. If
  an example cannot run as a doctest, refactor it until it can.
- An example demonstrates real usage. Path or assignment statements alone
  (`let _method = Type::method;`) pass the doctest gate without documenting
  anything and are not acceptable.
- When a canonical public API changes from infallible or panicking to
  `Result`-returning, update its examples, tutorials, and guides to propagate
  errors with `?` or handle the documented error explicitly. Do not preserve
  the old call shape with `unwrap()`/`expect()`. Tests may keep an intentional
  assertion boundary; an example may unwrap only a locally proven invariant
  whose proof is explicit.
- Non-trivial user-facing snippets have an executable source of truth: a
  checked example, test, or doctest rather than hand-copied Markdown code. A
  Markdown page that must contain a copied snippet gets an automated sync or
  extraction check that fails on drift. Crate READMEs with code fences need an
  executable sync mechanism (e.g. `#![doc = include_str!("../README.md")]`).
- Guide code demonstrating a workflow compiles in CI. When execution needs
  special hardware or external libraries, CI still compile-checks the example
  with the required features, and the guide documents the command that runs it
  on a configured machine.
- Examples calling backend operations bind the backend to a local variable and
  reuse it, not chain `Backend::new().op(...)` beyond a single trivial
  construction example.

## Public Result Error Documentation Gate

- Every public function, inherent method, and public-trait method returning
  `Result` documents a `# Errors` section naming the concrete variants or
  failure conditions the caller can observe; a generic "returns an error on
  failure" is insufficient.
- Documentation for traced or symbolic APIs describes validation deferred to
  compile or execution and identifies the applicable phase when that is part of
  the contract. Intentional panics use `# Panics`; deferred symbolic checks use
  `# Deferred errors`.
- `scripts/check-public-error-docs.py` audits the full Rust workspace and the
  Rust files changed by a PR and is a required CI gate. Keep it enabled without
  clippy or source-level allowlists; add the documentation at the API source.
