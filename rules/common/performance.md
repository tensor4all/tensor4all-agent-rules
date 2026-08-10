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

## Performance-Gated Experiment Protocol

Human/process protocol. This section is intentionally not routed to the
diff-scoped review bot.

- Performance candidates found by static/source audit alone must pass a
  need-before-implementation gate before code changes start. First measure the
  candidate path's share of an end-to-end workload, or another
  issue-predeclared representative workload. If the path does not occupy a
  meaningful share under the predeclared threshold, record the measurement or
  argument in the issue and close or defer it without implementation. A
  microbenchmark of the targeted helper is useful after the need is established,
  but by itself it proves only effect, not that the optimization is worth its
  semantic, aliasing, cache, or maintenance risk.
- Before running a candidate, record the baseline commit, candidate commit,
  benchmark source, build profile, hardware and affinity configuration,
  provider/thread settings, complete case list, comparison statistic,
  acceptance threshold, repetition policy, and host-noise observables and
  thresholds. Candidate results must not influence these choices.
- Run the complete baseline/candidate suite as one paired experiment. Do not
  selectively retry, omit, replace, or promote individual favorable cases.
- If a predeclared host-noise or validity gate fails, classify the entire paired
  experiment as `INCONCLUSIVE`. Reconsideration requires a complete paired
  rerun under the same predeclared protocol, not a selective retry.
- Record every measured case, confidence interval, validity observation, and
  regression in the worklog. A negative or inconclusive primary result is
  evidence and must not be rewritten as success because secondary cases
  improved.
- Promote a performance-gated change only when its predeclared primary gate and
  all required non-regression/correctness gates pass. Do not relax thresholds,
  redefine the primary metric, or add post-hoc exclusions after seeing the
  candidate.

## Cache Ownership

- Long-lived runtime/compiler caches must be owned by an explicit top-level
  runtime object, not hidden in thread-local/global state or buried in backend
  internals.
- Every cache must have a bounded default, a user-facing way to configure that
  bound, and a user-facing way to clear it.
- Every cache must expose user-facing introspection for the number of retained
  entries and retained bytes. Report retained bytes as the cache's owned/logical
  payload estimate, not operating-system RSS or allocator arena usage.
- Top-level runtime objects that own multiple caches must provide aggregate
  clear and aggregate stats APIs in addition to cache-specific controls.
- Backend resource pools such as buffer pools may live on the backend, but they
  still need explicit limit/clear controls, stats APIs, and documentation.
- Backends that own resource pools or runtime contexts are construct-once-and-
  reuse values. Examples should bind the backend once and reuse it across
  related operations; they must not present per-call construction chained
  directly into an operation as the normal idiom.
- Do not add a new cache without documenting its owner, lifetime, default
  capacity, memory behavior, entry/byte accounting, and
  clear/configuration/stats path.

## Complexity Budget

- Do not introduce accidental `O(n^2)` behavior in graph construction,
  metadata propagation, key hashing/equality, compilation, or execution
  scheduling. If a quadratic algorithm is intentional, document why the input
  size is bounded or why the tradeoff is acceptable with an `// INVARIANT:`
  marker (see Invariant Markers in `common/repository.md`).
- Avoid repeatedly cloning, hashing, formatting, or scanning whole graph
  histories, metadata scope lists, input maps, or structural keys inside
  per-node/per-op loops. Prefer stable IDs, interning, cached fingerprints with
  exact equality checks, or persistent/shared data structures.
- When optimizing compiler or graph-build overhead, measure scaling across
  increasing input sizes, not only one fixed benchmark case.
