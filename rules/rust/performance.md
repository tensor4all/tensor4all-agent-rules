# Rust Performance Rules

## Allocation And Copies

- Prefer borrowed slices, strided views, backend-native buffers, and
  metadata-only layout changes over dense copy-in/copy-out.
- Use `Vec::with_capacity` or reusable scratch when every element will be
  written. Do not zero-fill buffers that are immediately overwritten.
- No `to_vec()`, `clone()`, `format!`, `HashSet` construction, or `String`
  fingerprints in hot loops unless the size is bounded and documented.
- Keep dtype dispatch at the outer boundary. Share implementation through
  generics, traits, or macros instead of duplicating `f32`/`f64`/complex
  bodies.

## Indexing And Slicing

- Validate rank, axes, bounds, steps, output shape, and boundary behavior once
  at the API or planning boundary, then carry shape/stride/offset metadata into
  the inner loop instead of re-checking per element.
- Prefer iterator patterns that let LLVM eliminate bounds checks: direct slice
  iteration, pre-sliced ranges, `chunks_exact`, explicit pre-loop assertions.
- Use unchecked indexing or raw pointers only when the validation invariant is
  close to the unsafe block and covered by boundary tests.
- Avoid per-element `flat -> multi-index` decoding in tensor-sized loops when
  incremental index/offset traversal is possible.

## Dense Layout And Linear Algebra

- Preserve repository-local dense layout semantics. Tensor4all Rust projects
  commonly use column-major dense buffers; check project rules before adding
  flat-buffer APIs or examples.
- No hidden row-major compatibility shims or silent row-major round-trips.
- Use the owning backend/linalg abstraction for GEMM, einsum, decompositions,
  and solves. Do not reimplement kernels in downstream layers.
- When integrating faer, BLAS, LAPACK, or CUDA libraries, keep packing and ABI
  copies at explicit boundaries and reuse scratch where possible.

## Graph, Compiler, And Cache Work

- Do not clone full graph input maps, metadata scopes, root lists, or
  checkpoint chains on every node unless the sizes are known to be small.
- Precompute membership maps or live sets for compiler passes instead of
  rescanning later instructions for every slot.
- Prefer structural cache keys and exact equality checks over formatting whole
  programs into strings on every lookup.
- Do not key persistent caches by `Vec<usize>` or other owned index vectors:
  each lookup hashes and compares the whole vector and each insert clones it.
  Encode multi-indices as mixed-radix flat integers with the width selected
  from the index-space size (`u64`, `u128`, then extended integers such as
  bnum's `U256`/`U512`/`U1024`), as in tensor4all-simplett's `FlatIndexer`.

## Build Profiles And Target Hygiene

- Default `[profile.dev]` and `[profile.test]` set `debug = 0`, keep
  `debug-assertions` and `overflow-checks` enabled, and keep incremental
  compilation on for edit-test loops. Provide a one-command override (e.g.
  `CARGO_PROFILE_DEV_DEBUG=1`) or an opt-in profile for debugger sessions.
- `[profile.release] debug = true` in a workspace whose normal verification
  runs in release mode multiplies the build directory by gigabytes. Prefer
  `debug = "line-tables-only"` for readable backtraces, with a separate
  `release-debug` inheriting profile for full debugger information.
- The profile used by comprehensive CI runs disables incremental compilation
  and sets `strip = "symbols"`.
- Dependency `rev =` bumps and feature churn leave orphaned rlibs and test
  binaries in `target/` that no profile setting removes. Document the pruning
  mechanism (age-based sweep tool or periodic `cargo clean`) and propose a
  cleanup when `target/` growth is dominated by stale artifacts.

## GPU Kernels

- Launch domains cover the output or update domain. No
  `CubeCount::new_single()` or equivalent single-thread fallbacks for
  tensor-sized work.
- Kernel bodies must not loop unbounded over tensor elements inside one logical
  worker.
- Runtime shape/stride metadata flows through backend bindings rather than
  being duplicated as large compile-time parameter sets unless the
  specialization is intentional.

## Uninitialized And Scratch Acquisition

- Raw uninitialized acquisition is an `unsafe fn` or returns
  `MaybeUninit`-typed storage. Never hide it inside a safe `fn` returning live
  `T` values. Document the path as full-overwrite-only: every element is
  written before any read.
- Expose a separate zeroed/initialized acquisition for read-before-write
  callers.
- Never fix stale reads by unconditional zero-fill of a shared hot-path
  acquisition, and do not zero-initialize buffers that are provably fully
  overwritten.
- Add regression coverage for both contracts: the uninitialized path stays
  unsafe or `MaybeUninit`-typed, and the zeroed path is safe.

## Threading Principles

- One repository source of truth for parallel thresholds (a declared policy
  mechanism), not per-op copies of the same decision.
- Short-circuit to the serial kernel when the effective thread count is one.
- Library kernels must not reach for an ambient global thread pool outside the
  declared policy mechanism. Non-provider parallel work executes within the
  repository/runtime-owned execution context, so thread policy (including
  one-thread serial execution) is decided at that single source of truth.
- Provider-owned threading (BLAS/OpenMP-style) is controlled by provider
  variables documented per repository; library code must not derive its own
  pool policy from an ambient runtime.
