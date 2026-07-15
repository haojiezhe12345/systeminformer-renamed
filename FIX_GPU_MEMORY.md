# Fix: GPU/NPU dedicated and shared memory values inflated vs Task Manager

## Problem

Per-process GPU memory values (Dedicated GPU memory, Shared GPU memory) are much
higher than what Task Manager reports. For long-running processes like dwm.exe
(50 days uptime), the discrepancy is extreme — e.g. 103.85 GB vs 1 GB.

## Root Cause

Two separate data sources both reported **committed** bytes instead of **resident**
bytes (currently in VRAM):

1. **D3DKMT path** (`D3DKMT_QUERYSTATISTICS_PROCESS_SEGMENT`):
   `BytesCommitted` counts all allocations including evicted/paged-out GPU memory.
   Over time, many allocations get evicted from VRAM but remain "committed" in
   tracking, inflating the value.

2. **Perf counter path** (`GPU Process Memory` GUID, counters 4/5):
   "Dedicated Usage" and "Shared Usage" counters also report committed bytes,
   not resident bytes.

Task Manager shows resident GPU memory — memory currently in VRAM.

## Fix

### Memory data source (`EtpGpuUpdateProcessSegmentInformation` / `EtpNpuUpdateProcessSegmentInformation`)

Changed from `BytesCommitted` to summing resident bytes:

```c
// Before (inflated):
bytesCommitted = queryStatistics.QueryResult.ProcessSegmentInformation.BytesCommitted;

// After (correct):
for (ULONG p = 0; p < D3DKMT_QUERYSTATISTICS_SEGMENT_PREFERENCE_MAX; p++)
    residentBytes += processSegmentInfo.VideoMemory.AllocsResidentInP[p].Bytes;
residentBytes += processSegmentInfo.VideoMemory.AllocsResidentInNonPreferred.Bytes;
```

Fallback for pre-Windows 8 still uses `BytesCommitted`.

### Update loop (`EtGpuProcessesUpdatedCallback` / `EtNpuProcessesUpdatedCallback`)

Moved `EtpGpuUpdateProcessSegmentInformation()` / `EtpNpuUpdateProcessSegmentInformation()`
to always execute (before the `EtGpuD3DEnabled` check), so memory always comes from
D3DKMT resident bytes. The D3D perf counters are still used for GPU utilization
(engine/node usage) only.

Before:
```c
if (EtGpuD3DEnabled)
{
    // Memory from perf counters (committed bytes) ← bug
    EtLookupProcessGpuMemoryCounters(...);
    // Utilization from perf counters
    EtLookupProcessGpuUtilization(...);
}
else
{
    // Memory from D3DKMT (committed bytes) ← bug
    EtpGpuUpdateProcessSegmentInformation(block);
    // Utilization from D3DKMT
    EtpGpuUpdateProcessNodeInformation(block);
}
```

After:
```c
// Memory always from D3DKMT (resident bytes) ← fixed
EtpGpuUpdateProcessSegmentInformation(block);

if (EtGpuD3DEnabled)
{
    // Utilization from perf counters
    EtLookupProcessGpuUtilization(...);
}
else
{
    // Utilization from D3DKMT
    EtpGpuUpdateProcessNodeInformation(block);
}
```

## Files Changed

| File | Change |
|---|---|
| `plugins/ExtendedTools/gpumon.c` | GPU: resident bytes in segment query + always-run memory path |
| `plugins/ExtendedTools/npumon.c` | NPU: same fix (identical bug pattern) |

## Key Data Structures

- `D3DKMT_QUERYSTATISTICS_PROCESS_SEGMENT_INFORMATION` — per-process per-segment GPU stats
  - `BytesCommitted` — all committed allocations (inflated)
  - `VideoMemory.AllocsResidentInP[].Bytes` — resident in preferred segments (correct)
  - `VideoMemory.AllocsResidentInNonPreferred.Bytes` — resident in non-preferred (correct)
- `D3DKMT_QUERYSTATISTICS_SEGMENT` (system-wide) — already used `BytesResident` correctly

## Files Involved

- `plugins/ExtendedTools/gpumon.c` — GPU monitoring, per-process and system-wide
- `plugins/ExtendedTools/npumon.c` — NPU monitoring, same structure as GPU
- `plugins/ExtendedTools/counters.c` — Windows perf counter data source (still used for utilization)
- `plugins/ExtendedTools/d3dkmt/d3dkmthk.h` — D3DKMT API type definitions
