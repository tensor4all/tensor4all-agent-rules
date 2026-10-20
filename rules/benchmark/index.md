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

Read `measurement.md` before you run anything beside a measurement, and `tables.md` before you
publish one.

Campaign-specific facts - which cores this host measures, which suite declares which grid - live in
the campaign's own `AGENTS.md` and descriptor, never here: a rule that hardcoded a core set would be
wrong on the next machine.
