# Julia Performance Rules

## Allocation And Type Stability

- Avoid avoidable allocations in inner loops. Use views, preallocated output
  buffers, and in-place APIs when the public API allows it.
- Keep hot functions type-stable. Check inference when adding polymorphic or
  closure-heavy code in performance-sensitive paths.
- Avoid global mutable state in hot paths unless it is explicitly cached,
  bounded, and thread-safe.
- Do not hide full dense materialization behind high-level tensor-network APIs.

## Tensor And Tensor-Network Work

- Preserve structured tensor representations such as identity, diagonal, copy,
  one-hot, selector, and topology-routing tensors when the structure is
  semantically available.
- Do not replace structured tensor-network operations with full dense arrays in
  production paths unless the API is explicitly dense/reference/debug.
- Long tensor-network tests and examples should use scalable residuals,
  sampled evaluations, or structural checks rather than full dense conversion.

## Bindings

- Keep Julia wrappers aligned with the C API and Rust commit they build
  against.
- Do not paper over missing C API behavior with expensive Julia-side dense
  fallbacks unless the wrapper is explicitly named and documented as dense or
  reference behavior.
