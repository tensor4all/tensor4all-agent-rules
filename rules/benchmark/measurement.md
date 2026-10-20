# Measuring on a shared host

Rules for anyone - human or agent - who runs or sits near a benchmark measurement. They are about
the machine, not about the code: a measurement is evidence only if nothing else was using the cores
it ran on.

## Bind yourself off the measured cores

**Any command you run while a measurement is in flight must be pinned to cores the measurement is
not using.** That includes a diagnostic `verify`, a build, a `git status` in a loop, an editor's
indexer, and anything you launch "just to check".

The campaign's suite declares what it measures (`cpu_sets` in the suite, or `threads.cpu_sets` in the
run manifest); everything else on the host is the observer's. Discover the split rather than
assuming it:

```sh
lscpu -e | column -t                       # L3 map: cores sharing an L3 cache are one domain
cat /sys/devices/system/cpu/cpu*/cache/index3/shared_cpu_list
ps -eo pid,psr,%cpu,comm --sort=-%cpu | head   # who is already busy, and where
```

Then pin:

```sh
taskset -c <observer cores> <command>
```

**Worked example.** On a 24-CPU host the campaign measured `4-11` (the 8 MiB L3 domain) and left
`0-3,12-15` (the 16 MiB domain) to the observer. A diagnostic verify was started without pinning;
it landed on `4-11`, the guard saw the cores busy after the run (`pinned.sh: cpus 4-11 busy after
run`), and an hour of measurement was discarded. The work was not wrong — it was unverifiable.

## One measurement at a time

A campaign holds one lock across build, verify and timing. Do not start a second campaign, a build
or a commit while it is held: the recorder checks that the measured sources did not move and stops
the run rather than publishing something it cannot vouch for.

## A discarded run stays discarded

The guard discards a run whose cores were busy before or after it. Do not re-run until it passes:
the discard is the evidence that something else was using the machine, and a run that only passes
on the fourth attempt is a run whose conditions nobody recorded.

## Prime, then time

Priming is time-based and identical on every cell (an empty loop for a fixed wall time). A fixed
call count is not a substitute: after an idle gate the machine reads low for the first second or
two of sustained work, and that is large enough to look like a kernel defect.

## A budget is declared, not inherited

The thread count is part of the measurement, and the environment is not: the recorder removes
`RAYON_NUM_THREADS`, `OMP_NUM_THREADS` and their siblings from the measured process, and the harness
reads its own budget back and refuses to report a request that was clamped. When a suite declares
`threads: [8]`, the CPU set paired with it must hold at least eight CPUs - a suite that keeps a
two-entry `cpu_sets` list beside a one-entry `threads` list once measured eight threads on one core,
and the manifest recorded `cpu_sets: {'8': '4'}`. Check the pairing before you record.
