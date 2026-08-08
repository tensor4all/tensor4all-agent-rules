# Rust Performance Rules

## Allocation And Copies

- Prefer borrowed slices, strided views, backend-native buffers, and
  metadata-only layout changes over dense copy-in/copy-out.
- Use `Vec::with_capacity` or reusable scratch when every element will be
  written. Do not zero-fill buffers that are immediately overwritten.
- Avoid `to_vec()`, `clone()`, `format!`, `HashSet` construction, or `String`
  fingerprints in hot loops unless the size is bounded and documented.
- Keep dtype dispatch at the outer boundary. Share implementation through
  generics, traits, or macros instead of duplicating `f32`/`f64`/complex bodies.

## Indexing And Slicing

- Validate rank, axes, bounds, steps, output shape, and boundary behavior once
  at the API or planning boundary.
- After validation, carry shape/stride/offset metadata into the inner loop
  instead of repeating checks per element.
- Prefer iterator patterns that help LLVM eliminate bounds checks: direct slice
  iteration, pre-sliced ranges, `chunks_exact`, and explicit pre-loop
  assertions.
- Use unchecked indexing or raw pointers only when the validation invariant is
  close to the unsafe block and covered by boundary tests.
- Avoid per-element `flat -> multi-index` decoding in tensor-sized loops when
  incremental index/offset traversal is possible.

## Dense Layout And Linear Algebra

- Preserve repository-local dense layout semantics. Tensor4all Rust projects
  commonly use column-major dense buffers; check the project rules before
  adding flat-buffer APIs or examples.
- Do not add hidden row-major compatibility shims or silent row-major
  round-trips.
- Use the owning backend/linalg abstraction for GEMM, einsum, decompositions,
  and solves. Do not reimplement kernels in downstream layers.
- When integrating faer, BLAS, LAPACK, or CUDA libraries, keep packing and ABI
  copies at explicit boundaries and reuse scratch where possible.

## Graph, Compiler, And Cache Work

- Do not clone full graph input maps, metadata scopes, root lists, or checkpoint
  chains on every node unless the sizes are known to be small.
- Precompute membership maps or live sets for compiler passes instead of
  rescanning later instructions for every slot.
- Prefer structural cache keys and exact equality checks over formatting whole
  programs into strings on every lookup.

## Build Profiles And Target Hygiene

- Default `[profile.dev]` and `[profile.test]` should set `debug = 0` while
  keeping `debug-assertions` and `overflow-checks` enabled, and keep
  incremental compilation on for edit-test loops. Provide a one-command
  override (for example `CARGO_PROFILE_DEV_DEBUG=1`) or an opt-in profile for
  debugger sessions.
- `[profile.release] debug = true` in a workspace whose normal verification
  runs in release mode multiplies the build directory by gigabytes. Prefer
  `debug = "line-tables-only"` for readable backtraces, with a separate
  `release-debug` inheriting profile for full debugger information.
- The profile used by comprehensive CI runs should disable incremental
  compilation and set `strip = "symbols"`.
- Dependency `rev =` bumps and feature churn leave orphaned rlibs and test
  binaries in `target/` that no profile setting removes. Document the pruning
  mechanism (an age-based sweep tool, or a periodic `cargo clean`), and
  propose a cleanup when `target/` growth is dominated by stale artifacts.

## GPU Kernels

- Launch domains should cover the output or update domain. Avoid
  `CubeCount::new_single()` or equivalent single-thread fallbacks for
  tensor-sized work.
- Kernel bodies should not contain unbounded loops over tensor elements inside
  one logical worker.
- Runtime shape/stride metadata should flow through backend bindings rather
  than being duplicated as large compile-time parameter sets unless the
  specialization is intentional.
