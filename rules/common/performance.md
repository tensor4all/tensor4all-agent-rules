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
- Tensor-sized loops carry no runtime mode flags. This covers
  conjugation/op flags, zero-scale shortcuts, input and output source
  choices, and any other boolean or enum. Dispatch once at the outer
  boundary and specialize the loop body. Do not make a dynamic call per
  element.
- Elementwise, map and zip work over strided tensors uses the ecosystem's
  shared strided-array library; there are no new hand-written traversal
  loops in higher-level packages. Extend that library when it lacks an
  operation.
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
  object. For Rust crates, [`../rust/performance.md`](../rust/performance.md),
  "CPU Parallelism", says which design the audit checks.

## Parallel Scaling

A parallel split must keep working when the thread count `p` grows to the
largest host the code runs on (64 cores and more), not only at the 4 to 8
threads of the development machine. Check every new or changed parallel region
against these failure modes before measuring it:

- **Shared operand read once per task.** If each task reads the same large
  operand (a basis, a panel, a factor), total traffic is `p × size` and the
  region becomes bandwidth-bound once the operand exceeds a thread's share of
  cache. Pack or stage the shared operand once and let tasks read the shared
  copy, or split in two dimensions so each task reads `size / r`.
- **`O(p)` partial results.** Splitting a reduction into one partial per thread
  costs `p × partial` memory and an `O(p)` combine. Combine in a tree or bound
  the number of partials independently of `p`.
- **Fixed task caps and fixed split counts.** A constant such as "at most four
  tasks" or a split count unrelated to `p` leaves threads idle as `p` grows.
  Derive the split from the shapes and the thread budget.
- **Granularity and thresholds that ignore `p`.** A threshold tuned on one
  thread count can make tasks so small at high `p` that dispatch dominates.
  Thresholds come from the work per task and the dispatch cost model.
- **Contention that grows with `p`.** Barriers, atomics and locks on a shared
  counter or buffer inside the region.
- **Cache domains.** On hosts with several L3 domains (CCX/CCD, NUMA), data
  shared across domains costs more than within one; do not assume one
  shared L3.

Compare the split with what the reference implementation does (for dense
linear algebra, reference LAPACK and an open BLAS such as OpenBLAS, which
parallelize inside BLAS). A split that only wins at the development host's
thread count, or only against a regressed baseline, is not adopted. Scaling
evidence covers at least 1, 8 and the largest available thread count, and
reports per-call traffic (counters or a computed schedule) when the claim is
about memory traffic. Judge by mechanism: fix a split whose cost grows with
`p`; do not tune constants to one host.

## Evidence

- Measure scaling across representative sizes, shapes, layouts, and thread
  counts for performance-sensitive changes. One fixed-size benchmark is
  insufficient when the algorithmic shape changed.
- When leaving a known tradeoff in place, document why the input size is
  bounded or the cost acceptable.

## Optimising Against Proprietary Software

A vendor library (MKL, cuBLAS, cuTENSOR, a closed BLAS) is a yardstick, not a
source. When a change is measured against one:

- **Allowed:** timing the vendor through its public API under the same
  conditions as our code (the campaign's vendor arms); reading its public
  documentation for what an API does and what its parameters mean; stating a
  hypothesis about how it reaches a number ("it probably pads the leading
  dimension") and testing that hypothesis on *our* code; reading published
  papers and reference implementations whose licence is compatible (reference
  LAPACK is BSD-3) and citing them in the module header.
- **Not allowed:** disassembling, decompiling or tracing the vendor binary;
  sampling a profiler into its internals to learn its structure; reading its
  source where one exists (leaked, under NDA, or as a licensed component);
  carrying over constants, thresholds or orderings that were learned from it
  rather than measured on our code; translating a reference implementation's
  code without its licence notice, even when the licence would permit it.
- **Record the provenance** in the change's worklog and ledger row: which
  papers the algorithm comes from, that the vendor was timed through its public
  API only, and that no vendor or (if so) no LAPACK code was consulted. A design
  choice should trace to a measured breakdown of *our* code, not to what the
  vendor is believed to do.

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
