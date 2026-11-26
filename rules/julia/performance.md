# Julia Performance Rules

## Allocation And Type Stability

- Avoid avoidable allocations in inner loops: views, preallocated outputs, and
  in-place APIs when the public API allows.
- Keep hot functions type-stable. Check inference when adding polymorphic or
  closure-heavy code in performance-sensitive paths.
- No global mutable state in hot paths unless explicitly cached, bounded, and
  thread-safe.
- Do not hide full dense materialization behind high-level tensor-network APIs.

## Tensor And Tensor-Network Work

- Preserve structured tensor representations (identity, diagonal, copy,
  one-hot, selector, topology-routing tensors) when the structure is
  semantically available.
- Do not replace structured tensor-network operations with full dense arrays in
  production paths unless the API is explicitly dense/reference/debug.
- Long tensor-network tests and examples use scalable residuals, sampled
  evaluations, or structural checks rather than full dense conversion.
- Do not key persistent caches by `Vector{Int}` multi-indices: each lookup
  hashes and compares the whole vector and retained keys cost O(length) each.
  Encode as a mixed-radix flat integer, widening to `Int128` or a fixed-width
  big integer on overflow, or intern to stable IDs.

## Bindings

- Keep Julia wrappers aligned with the C API and Rust commit they build against.
- Do not paper over missing C API behavior with expensive Julia-side dense
  fallbacks unless the wrapper is explicitly named and documented as dense or
  reference behavior.
