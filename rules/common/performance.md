# Common Performance Rules

Tensor4all projects contain active optimization work; nearby code is not proof
of performance-correctness. Before copying a pattern, check whether it is a
legacy tradeoff, a reference path, or a known migration target.

## General Checklist

- No accidental `O(n^2)` or worse in graph construction, metadata propagation,
  planning, key hashing/equality, scheduling, or execution.
- Do not repeatedly clone, hash, format, or scan whole histories, metadata
  scope lists, input maps, structural keys, or live operand sets inside
  per-node or per-op loops.
- No dense materialization scaling with an unconstrained product of tensor,
  site, batch, or index dimensions unless the API is explicitly dense,
  reference, or debug.
- No heap allocation inside hot loops: pre-allocate, reuse scratch, use views,
  or move the allocation to a documented boundary.
- Do not zero-initialize buffers that will be fully overwritten.
- Prefer incremental offsets, precomputed strides, and strided/backend-native
  views over per-element coordinate decoding.
- Cache keys are compact structural fingerprints or incremental hashes, not
  debug strings of whole programs or large objects.
- Do not key persistent or hot-path caches by raw index vectors (`Vec<usize>`,
  `Vector{Int}`, or equivalent): each lookup pays an O(length) hash and
  equality walk, each insert clones the vector, and retained keys can rival the
  payload in memory. Encode the tuple as a compact integer (mixed-radix flat
  index with width chosen from the index-space size) or intern to stable IDs
  when the key space is unbounded.
- Long-lived caches need explicit owners, bounded defaults, clear/configure
  APIs, and useful stats.

## Backend And Device Boundaries

- No hidden CPU/GPU transfers or host round-trips.
- GPU kernels map the output or update domain to the launch domain; a single
  thread must not loop over an unbounded tensor domain.
- GPU reductions must not hide unbounded serial work in one thread. Use a
  documented strategy and threshold where a unit-thread fallback is allowed.
- CPU threading policy has one source of truth per repository or runtime
  object.

## Evidence

- Measure scaling across representative sizes, shapes, layouts, and thread
  counts for performance-sensitive changes. One fixed-size benchmark is
  insufficient when the algorithmic shape changed.
- When leaving a known tradeoff in place, document why the input size is
  bounded or the cost acceptable.

## Performance-Gated Experiment Protocol

Human/process protocol; intentionally not routed to the diff-scoped review bot.

- Candidates found by static/source audit alone pass a
  need-before-implementation gate before code changes: first measure the path's
  share of an end-to-end or issue-predeclared representative workload. Below
  the predeclared threshold, record the measurement or argument in the issue
  and close or defer without implementation. A helper microbenchmark is useful
  once need is established but proves only effect, not that the optimization is
  worth its semantic, aliasing, cache, or maintenance risk.
- Before running a candidate, record baseline commit, candidate commit,
  benchmark source, build profile, hardware and affinity, provider/thread
  settings, complete case list, comparison statistic, acceptance threshold,
  repetition policy, and host-noise observables and thresholds. Candidate
  results must not influence these choices.
- Run the complete baseline/candidate suite as one paired experiment. Never
  selectively retry, omit, replace, or promote favorable cases.
- If a predeclared host-noise or validity gate fails, the entire paired
  experiment is `INCONCLUSIVE`; reconsideration requires a complete paired
  rerun under the same protocol.
- Summarize the decision, primary result, validity, and remaining limitations
  in the work log. Retain every measured case, confidence interval, validity
  observation, regression, and reproduction-critical setting in the experiment
  results, linked from the work log; do not copy the full results or command
  history into it. A negative or inconclusive primary result is evidence; do
  not rewrite it as success because secondary cases improved.
- Promote only when the predeclared primary gate and all required
  non-regression/correctness gates pass. No relaxed thresholds, redefined
  primary metric, or post-hoc exclusions after seeing the candidate.

## Cache Ownership

- Long-lived runtime/compiler caches are owned by an explicit top-level runtime
  object, not thread-local/global state or backend internals.
- Every cache has a bounded default, a user-facing way to configure the bound,
  and a user-facing way to clear it.
- Every cache exposes user-facing counts of retained entries and retained
  bytes; bytes are the cache's owned/logical payload estimate, not RSS or
  allocator arena usage.
- Top-level runtime objects owning multiple caches provide aggregate clear and
  aggregate stats APIs as well as cache-specific controls.
- Backend resource pools (e.g. buffer pools) may live on the backend but still
  need limit/clear controls, stats APIs, and documentation.
- Backends owning pools or runtime contexts are construct-once-and-reuse
  values. Examples bind the backend once and reuse it; never present per-call
  construction chained into an operation as the normal idiom.
- No new cache without documenting owner, lifetime, default capacity, memory
  behavior, entry/byte accounting, and clear/configuration/stats path.

## Complexity Budget

- Do not introduce accidental `O(n^2)` behavior in graph construction,
  metadata propagation, key hashing/equality, compilation, or execution
  scheduling. An intentional quadratic algorithm documents why the input is
  bounded or the tradeoff acceptable with an `// INVARIANT:` marker (see
  Invariant Markers in `common/repository.md`).
- Prefer stable IDs, interning, cached fingerprints with exact equality checks,
  or persistent/shared data structures over rescanning per node/op.
- When optimizing compiler or graph-build overhead, measure scaling across
  increasing input sizes, not one fixed case.
