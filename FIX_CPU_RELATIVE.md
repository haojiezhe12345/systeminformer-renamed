# Fix: CPU (relative) column incorrect when process affinity is limited

## Problem

The "CPU (relative)" column (one core = 100%) shows lower values when a process's
affinity mask excludes some CPUs. For example, a process using 500% on a 32-core
system drops to 250% if its affinity is limited to 16 cores.

## Root Cause

`CpuUsage` is computed as:

    CpuUsage = processCyclesDelta / PhCpuTotalCycleDelta

`PhCpuTotalCycleDelta` is the total cycle time across **all** system processors.
So `CpuUsage` already represents the fraction of total system CPU capacity.

To convert to "cores used" (where 1 core = 100%), you must multiply by the
**total system CPU count**, not the process's affinity count.

The buggy code multiplied by `AffinityPopulationCount` — the number of CPUs the
process is allowed to run on — instead of `PhSystemProcessorInformation.NumberOfProcessors`.

## Fix

Changed three locations to use `PhSystemProcessorInformation.NumberOfProcessors`:

| File | Line | Context |
|---|---|---|
| `SystemInformer/proctree.c` | ~4207 | Process tree "CPU (relative)" column display |
| `SystemInformer/prpgstat.c` | ~393 | Process properties statistics page |
| `SystemInformer/thrdlist.c` | ~1784 | Thread list "CPU (relative)" column display |

This is consistent with the column header total calculation at `proctree.c:5504`
and the thread sort comparison at `thrdlist.c:1119`, which already used
`PhSystemProcessorInformation.NumberOfProcessors`.

## Key Data Flow

```
procprv.c: PhCpuTotalCycleDelta = sysTotalCycleTime  (sum of all CPUs)
thrdprv.c: CpuUsage = CyclesDelta / PhCpuTotalCycleDelta
proctree.c: cpuUsage = CpuUsage * 100 * NumberOfProcessors  (was: AffinityPopulationCount)
```

## Files Involved

- `SystemInformer/procprv.c` — process provider; computes `CpuUsage` and `PhCpuTotalCycleDelta`
- `SystemInformer/thrdprv.c` — thread provider; computes per-thread `CpuUsage`
- `SystemInformer/proctree.c` — process tree list; displays "CPU (relative)" column
- `SystemInformer/thrdlist.c` — thread list; displays and sorts "CPU (relative)" column
- `SystemInformer/prpgstat.c` — process properties statistics page
