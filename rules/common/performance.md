# Common Performance Rules

Tensor4all projects often contain active optimization work. Do not assume nearby
existing code is performance-correct. Before copying an implementation pattern,
check whether it is a legacy tradeoff, a reference path, or a known migration
target.

## General Checklist

- Avoid accidental `O(n^2)` or worse behavior in graph construction, metadata
  propagation, planning, key hashing/equality, scheduling, and execution.
- Do not repeatedly clone, hash, format, or scan whole histories, metadata
  scope lists, input maps, structural keys, or live operand sets inside
  per-node or per-op loops.
- Avoid dense materialization whose cost scales with an unconstrained product
  of tensor, site, batch, or index dimensions unless the API is explicitly
  dense, reference, or debug behavior.
- Do not allocate heap buffers inside hot loops. Pre-allocate, reuse scratch,
  use views, or move the allocation to a documented boundary.
- Do not zero-initialize buffers that will be fully overwritten.
- Prefer incremental offsets, precomputed strides, and strided/backend-native
  views over repeated per-element coordinate decoding.
- Cache keys should be compact structural fingerprints or incremental hashes,
  not debug strings for whole programs or large objects.
- Long-lived caches need explicit owners, bounded defaults, clear/configure
  APIs, and useful stats.

## Backend And Device Boundaries

- Do not introduce hidden CPU/GPU transfers or hidden host round-trips.
- GPU kernels must map the output or update domain to the launch domain. A
  single thread must not loop over an unbounded tensor domain.
- Reductions on GPU must not hide unbounded serial work inside one thread. Use
  a documented strategy and threshold when a unit-thread fallback is allowed.
- CPU threading policy should have one source of truth per repository or
  runtime object.

## Evidence

- Measure scaling across representative sizes, shapes, layouts, and thread
  counts for performance-sensitive changes.
- A single fixed-size benchmark is not enough evidence when the algorithmic
  shape changed.
- When leaving a known performance tradeoff in place, document why the input
  size is bounded or why the cost is acceptable.
