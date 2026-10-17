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
- Bounded-rank inline metadata (shape, strides, offsets, axis lists: `Copy`
  integers carried by tensors, views and plans) uses `tinyvec`, the single
  inline-vector crate across tensor4all repositories. Use
  `tinyvec::ArrayVec<[T; N]>` for a fixed capacity: it is `Copy`, so a copy is
  a bitwise move. Use `tinyvec::TinyVec<[T; N]>` where the capacity may spill
  to the heap: an inline clone is still a bitwise copy. Do not use
  `smallvec::SmallVec` or `arrayvec::ArrayVec` for this metadata, because
  their `clone` is element-wise. smallvec 1.x routes it through `From<&[T]>`
  (an iterator collect), with `memcpy` only under the nightly
  `specialization` feature, and the 2.0 betas use `extend`. Measured on an
  EPYC 7713P (smallvec 1.16.3, arrayvec 0.7.8, tinyvec 1.13.3), a clone at
  rank 2-8 takes 3.6-5.5 ns (`SmallVec`), 2.0-4.4 ns (`arrayvec::ArrayVec`),
  0.65 ns (`tinyvec::ArrayVec`) and 0.97 ns (`tinyvec::TinyVec`)
  (tensor4all/tenferro-rs#2180, tensor4all/strided-rs#306). `tinyvec` needs
  `Default` items, which integers satisfy. Existing `SmallVec` uses migrate
  when touched, and a hot path never calls `clone()` on one.
  `TinyVec` matches on its inline/heap variant on every access, so a hot
  inner loop takes `as_slice()` once, or uses a fixed `[T; N]` cursor with a
  separate spill, instead of indexing the `TinyVec` per element. A `TinyVec`
  cursor inside a strided-rs `reduce_axis` loop cost about 26 extra
  instructions per output (strided-rs#306 B2).

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
- Hoist every runtime mode out of tensor-sized loops. This includes conj
  flags, `beta == 0`, C/D source choice and `Option`/enum operands. Use one
  `match` at the boundary, monomorphized bodies (generics, const generics,
  type-level ops such as strided-view `ElementOp`), and no per-element
  `dyn Fn` call.
- Elementwise/map/zip over strided views goes through `strided-rs`
  (`strided-basic` `map_into`, `zip_map{2,3,4}_into`, `mul_into`, `fma`,
  ...). Add a missing operation to strided-rs instead of hand-writing an
  odometer loop downstream.

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
  compilation on for edit-test loops. `debug = 1` and `debug = "line-tables-only"`
  are debug info too — both emit DWARF — so the override is
  `CARGO_PROFILE_DEV_DEBUG` or a named opt-in profile, never a default one.
- No default profile carries debug info, `release` and `bench` included; it
  multiplies the build directory and the shared compiler cache by gigabytes for
  something no test reads. When a workspace needs readable backtraces or a
  debugger, add one narrow profile that inherits the standard one
  (`release-debug`, `profiling`) instead of enabling it for everyone. See
  [`build-artifacts.md`](build-artifacts.md#debug-info-is-not-a-default).
- The profile used by comprehensive CI runs disables incremental compilation
  and sets `strip = "symbols"`.
- Dependency `rev =` bumps and feature churn leave orphaned rlibs and test
  binaries in `target/` that no profile setting removes. Document the pruning
  mechanism (age-based sweep tool or periodic `cargo clean`) and propose a
  cleanup when `target/` growth is dominated by stale artifacts. See
  [`build-artifacts.md`](build-artifacts.md) for what must be preserved and the
  cache-convergence step.

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
