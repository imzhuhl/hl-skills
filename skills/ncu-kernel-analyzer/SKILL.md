---
name: ncu-kernel-analyzer
description: Analyze CUDA kernel performance from NVIDIA Nsight Compute profiling reports (.ncu-rep files). Use this skill whenever the user mentions ncu-rep files, CUDA kernel profiling, Nsight Compute reports, kernel performance analysis, GPU performance bottlenecks, or wants to understand why a CUDA kernel is slow. Also trigger when the user asks about occupancy, memory throughput, warp stalls, compute utilization, or other GPU performance metrics in the context of profiling data.
---

# NCU Kernel Analyzer

Analyze CUDA kernel performance using NVIDIA Nsight Compute (ncu) profiling reports. The goal is to help users understand where their kernels are bottlenecked and what concrete steps they can take to improve performance.

## Workflow

### 1. Get the .ncu-rep file

Ask the user for the path to their `.ncu-rep` file if they haven't provided one. Confirm the file exists before proceeding.

### 2. Export and list kernels

The `.ncu-rep` format is binary — use `ncu` CLI to export data as CSV for analysis.

Start with a summary export to see what kernels were profiled:

```bash
ncu --import <file>.ncu-rep --csv > /tmp/ncu_summary.csv
```

Read the CSV and present the user with a list of unique kernel names found in the report. Include invocation counts if a kernel appears multiple times. Ask which kernel they want to analyze.

### 3. Export detailed metrics for the selected kernel

Once the user picks a kernel, export the detailed profiling data. Run these exports — you'll need different views for a thorough analysis:

```bash
# Detailed metrics grouped by category (memory, compute, occupancy, etc.)
ncu --import <file>.ncu-rep --csv --page details > /tmp/ncu_details.csv

# Source-level hotspot data (maps metrics back to source/SASS lines)
ncu --import <file>.ncu-rep --csv --page source > /tmp/ncu_source.csv
```

If the details export is very large, you can filter to the specific kernel:

```bash
ncu --import <file>.ncu-rep --csv --page details --kernel-name "<kernel_name>" > /tmp/ncu_details.csv
```

The raw page (`--page raw`) contains every collected metric — it's large but useful when you need a specific counter that doesn't appear in the details view. Only export it if needed:

```bash
ncu --import <file>.ncu-rep --csv --page raw --kernel-name "<kernel_name>" > /tmp/ncu_raw.csv
```

### 4. Find the CUDA source code

Search the project directory for the kernel's source code. CUDA kernels are typically defined with `__global__` in `.cu` or `.cuh` files. The kernel name from the profiler often includes template parameters and namespace qualifiers — strip those to find the base function name, then search:

```bash
# Search for the kernel function definition
grep -rn "__global__.*<kernel_base_name>" --include="*.cu" --include="*.cuh" .
```

If the source isn't found in the current directory, ask the user where the source lives. Having the source is important for correlating hotspot data with actual code, but the analysis can still proceed without it using just the metrics.

### 5. Analyze performance

This is the core of the skill. Combine the profiling metrics with source code understanding to identify bottlenecks and suggest improvements.

#### Key metrics to examine

Work through these categories systematically. Not every category will be relevant — focus on where the data shows problems.

**Compute utilization**
- SM throughput / compute throughput — is the kernel compute-bound?
- Achieved occupancy vs theoretical occupancy — low occupancy often means register pressure or shared memory limits
- Warp execution efficiency — divergent branches waste lanes

**Memory performance**
- Global memory throughput vs device peak — how close to the memory bandwidth ceiling?
- L1/L2 cache hit rates — poor locality means wasted bandwidth
- Global load/store efficiency — coalescing issues show up here (efficiency < 100% means uncoalesced accesses)
- Shared memory bank conflicts — check for high replay counts

**Latency and stalls**
- Warp stall reasons — the profiler breaks down why warps are stalled (memory dependency, execution dependency, synchronization, etc.). This is often the most actionable data.
- Memory latency — high latency with low throughput suggests the kernel isn't hiding latency well (not enough warps in flight)

**Occupancy limiters**
- Registers per thread — high register usage limits how many warps can run concurrently
- Shared memory per block — same effect as registers
- Block size — too small means underutilized SMs

**Instruction mix**
- Integer vs floating-point vs memory instruction ratio
- Special function unit (SFU) usage — transcendentals are expensive

#### Source-level analysis

If source data is available from the `--page source` export, correlate the hottest lines with the metrics above. Identify:
- Which loops or operations consume the most cycles
- Where memory access patterns cause stalls
- Whether there are obvious optimization opportunities (e.g., a loop that could use shared memory tiling)

### 6. Present findings

Structure the analysis as:

**Kernel Overview**
- Kernel name, grid/block dimensions, register usage, shared memory usage
- Achieved occupancy, execution time, throughput numbers

**Bottleneck Analysis**
- Rank the bottlenecks by impact (most significant first)
- For each bottleneck: what the metrics show, why it's a problem, and what the root cause likely is
- Reference specific source lines when possible

**Optimization Recommendations**
- Concrete, actionable suggestions tied to each bottleneck
- Prioritize by expected impact
- Note any tradeoffs (e.g., "using shared memory tiling will improve memory throughput but increases shared memory usage, which may reduce occupancy")

**Summary**
- One-paragraph summary: what's the dominant bottleneck and the single highest-impact change to make first

## Common bottleneck patterns and fixes

These are reference patterns — use them to inform your analysis, not as a checklist to recite.

| Pattern | Symptom | Typical fix |
|---------|---------|-------------|
| Uncoalesced global memory access | Low global load/store efficiency | Restructure data layout (AoS → SoA), align accesses |
| Register spilling | High local memory traffic, low occupancy | Reduce register pressure: simplify expressions, use `__launch_bounds__`, consider trading registers for shared memory |
| Low occupancy | Achieved occupancy << theoretical | Reduce registers/shared memory per block, increase block size, use occupancy calculator |
| Shared memory bank conflicts | High shared memory replay rate | Pad shared memory arrays, restructure access patterns |
| Warp divergence | Low warp execution efficiency | Reorganize data/control flow so threads in a warp take the same path |
| Memory-bound with low throughput | High memory stalls but throughput well below peak | Increase arithmetic intensity (compute more per byte loaded), prefetch, increase occupancy to hide latency |
| Compute-bound | High compute throughput near peak, execution stalls | Algorithmic optimization, reduce redundant computation, use faster math (`__fmaf`, fast intrinsics) |
| Excessive synchronization | High sync stall percentage | Reduce `__syncthreads()` calls, restructure algorithm to need fewer barriers |
| Small grid (tail effect) | Low SM utilization, few blocks | Increase parallelism: more blocks, persistent kernel pattern, or fuse with other work |

## Notes

- If `ncu` is not found on PATH, ask the user to ensure NVIDIA Nsight Compute is installed and `ncu` is accessible. It's typically at `/usr/local/cuda/bin/ncu` or similar.
- Some metrics may not be available depending on the GPU architecture and profiling configuration. Work with what's there.
- For kernels with multiple invocations, the profiler may show per-invocation data. Consider whether to analyze a single invocation or aggregate — ask the user if it matters.
- When the CSV files are very large, focus on reading the most relevant sections rather than loading everything at once.
