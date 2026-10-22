---
name: benchmark-campaign
description: Run, re-record, extend or compare a tensor4all benchmark campaign (tlinalg-benchmark, tcontract-benchmark, faer-benchmark, tenferro-benchmark, strided-rs-benchmark-suite). Use when asked to benchmark something, record a campaign page, add a vendor arm, compare two revisions, or read a campaign's numbers. Covers one-measurement-at-a-time discipline, the idle gate, what a run records, vendor arms, coverage and evidence.
---

# Benchmark Campaign

A campaign publishes **one page per measured run** under `result/`, generated from the raw run
under `data/results/`, with `result/INDEX.md` generated over all of them. The campaign's own
`AGENTS.md`, `docs/timing-policy.md` and `docs/results.md` are the specifics; this is the part
that is the same everywhere and the part that has cost real time.

## One measurement at a time, on a quiet machine

- The recorder holds a lock **per checkout**. Two campaigns, or two worktrees of one campaign,
  do *not* serialize with each other: run one, then the other. Never two at once.
- Every timing goes through `benchmarks/scripts/pinned.sh`, which pins the measured CPUs **and
  their SMT siblings** and **discards** any run whose window was not idle (`busy after run`).
  A discarded run is the protocol working, not a flake: repeating it unchanged fails the same way.
- **Your own session is load.** The agent process itself, a `cargo build` you started, a script
  in another worktree — anything landing on the measured set can exceed a 5% gate. Before a
  recording:
  - build the arm *first*, pinned off the measured set, so the recorder's own build is a cache hit;
  - pin the agent's own process off the measured set for the duration, e.g.
    `taskset -pc 0-3 $(pgrep -x pi)`, and restore its affinity when the run is done;
  - then leave it alone: poll with a single long `sleep`, not with a stream of commands.
- The longest cases (a tall verify, an 8T timing row) have the widest window and are therefore
  the ones most often discarded. That is not evidence that anything is wrong with them.

## What a run actually measures

- A run records the **pin**, not whatever the checkout happens to contain: the recorder
  materialises `pins/<library>.rev` in the checkout it is handed and records that revision as
  the page's identity. Move the pin **before** recording, never after.
- It verifies the whole declaration **before** it times anything, and publishes nothing unless the
  grid is clean. Never weaken a check to make a run publishable: fix the library or the
  reference. A reference that is correct but quadratic in the wrong place — forming a full `Q`
  where only the thin one is used, on a 4096×256 shape — is a harness defect with the same
  standing as a wrong number, and it makes whole populations unmeasurable.
- One vendor library owns the BLAS/LAPACK symbols, so **an arm is a build and a page**; two arms
  never share a manifest, and the vendor's own thread count is set and read back. The vendor's
  identity is part of the record, including the compiler that built it (a from-source vendor
  needs a Fortran compiler for LAPACK, and which one is an identity, not a detail). If the
  vendor runtime needs a loader path, bake it in as an inherited `RPATH`, not a `RUNPATH` — the
  recorder executes the harness directly, and an arm whose binary cannot start is not recordable.

## Coverage

- `full` means every axis the declaration names matches: populations, thread budgets, dtypes,
  order, and the engines of that arm. A case a provider cannot take is emitted as `skipped` with
  a truthful note, never dropped silently.
- A required hardware profile needs a page for **every** arm, so `required_profiles` moves only
  when the last arm is in — adding it earlier fails the campaign's own index check.

## Evidence

- Page, its raw `data/results/**` run and the regenerated `result/INDEX.md` travel in **one
  commit**; the index is generated, never hand-edited.
- Quote a number with its revision, pin and hardware profile. Say "scan" when the run is a single
  complete set; a claim needs an A/A pair, not one run read twice.
- A negative result is a result: record the candidate that lost *with the measurement that killed
  it*, so that the next session does not spend the day on it again.

## Diagnostics under `probes/`

A standalone driver that links the measured library by path and is run by hand - `faer-benchmark`
keeps three - answers "where does the time go", not "is this change faster". The driver stays
untracked in `probes/`; the numbers it produced go into the `docs/worklogs/` entry of the decision
they fed, and are labelled a diagnostic where they are quoted.

It is outside `pinned.sh` and outside the recorder, so the pin is yours: bind yourself off the
measured set, declare the CPU set per thread count, the statistic and the repetition count *before*
the first run, and report the values you got rather than a rounded factor.

A diagnostic never adopts a change. Re-confirm it with a paired campaign run
(`record_run.py ... --no-publish` plus `candidate_compare.py`); the predeclared experiment protocol
in [`../../rules/common/performance.md`](../../rules/common/performance.md) is what the pair has to
satisfy.
