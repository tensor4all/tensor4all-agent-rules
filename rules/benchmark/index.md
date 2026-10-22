# Benchmark rules

Measurement and publication discipline for the campaigns under `tensor4all`. Load these when you
run, record, publish or read a benchmark - not when you write library code (that is the rest of this
repository).

- [`measurement.md`](measurement.md): **bind yourself off the measured cores**, one measurement at
  a time, a discarded run stays discarded, priming, a declared thread budget.
- [`evidence.md`](evidence.md): pins resolved from git, what a run records, checked versus timed
  cells, and the sentences a page may not write.
- [`tables.md`](tables.md): one table one question, rows are implementations, internal comparisons
  are a different table, name the subject.

The same three questions, at the level of a whole campaign: [`../../skills/benchmark-campaign/SKILL.md`](../../skills/benchmark-campaign/SKILL.md)
- the lock is per checkout, your own agent session is load and must be pinned off the measured set
(`taskset -pc 0-3 $(pgrep -x pi)`) and restored after, and a negative result is recorded with the
measurement that killed it.

Before you run a candidate at all, [`../common/performance.md`](../common/performance.md)'s
*Performance-Gated Experiment Protocol* applies: baseline and candidate commit, build profile,
hardware and affinity, comparison statistic, threshold and repetition policy are recorded **before**
the run, and the suite is one paired experiment with no selective retry. A standalone driver under a
campaign's `probes/` is a diagnostic, not that experiment.

Read `measurement.md` before you run anything beside a measurement, and `tables.md` before you
publish one.

Campaign-specific facts - which cores this host measures, which suite declares which grid - live in
the campaign's own `AGENTS.md` and descriptor, never here: a rule that hardcoded a core set would be
wrong on the next machine.
