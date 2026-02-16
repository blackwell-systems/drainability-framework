# Drainability Framework: Supporting Data and Implementation

This directory contains the empirical validation data and implementation documentation supporting the drainability framework paper.

## Overview

The drainability framework establishes that **allocation routing**, not allocator policy, determines whether coarse-grained memory reclamation produces bounded or unbounded retention. This directory contains:

1. **Experimental validation** demonstrating the 238× recycle-rate differential between lifetime-mixed and lifetime-isolated routing
2. **Implementation documentation** from temporal-slab, the allocator used to validate the theorems
3. **Raw data** enabling reproduction and extension of the empirical results

---

## Directory Structure

```
data/
├── README.md                           # This file
├── experimental/                       # Empirical validation data
│   ├── mixed-lifetime/                 # Drainability violation experiment
│   │   ├── mixed_timeseries.csv        # Net slab retention over 200K requests
│   │   ├── mixed_closes.csv            # Empty slabs found per epoch close
│   │   └── mixed_stats.txt             # Summary: 0.28% recycle rate, 493K slabs
│   ├── dual-epoch/                     # Drainability satisfaction experiment
│   │   ├── dual_timeseries.csv         # Net slab retention (plateaus at 2,143)
│   │   ├── dual_closes.csv             # Empty slabs found per epoch close
│   │   └── dual_stats.txt              # Summary: 66.5% recycle rate, 2K slabs
│   ├── plots/                          # Generated visualizations
│   │   ├── figure_drainability_centerpiece.png
│   │   ├── figure_dual_epoch_comparison.png
│   │   └── figure_rss_determinism_200k.png
│   ├── DRAINABILITY_CONDITION.md       # Formal statement and proofs
│   ├── DUAL_EPOCH_SUCCESS.md           # Isolation experiment analysis
│   ├── FINAL_RESULTS.md                # Executive summary
│   └── PHASE22_VALIDATION_SUMMARY.md   # Complete validation methodology
└── implementation/                     # temporal-slab allocator documentation
    ├── semantics.md                    # Epoch/domain contracts and invariants
    └── ARCHITECTURE.md                 # Allocator design and state machine
```

---

## Key Results

### Alignment Theorem (Paper Section 3.2)

**Theorem 1**: A granule is reclaimable at its designated reclaim boundary if and only if every allocation routed to it completes by that boundary.

**Empirical validation:**
- **Mixed-lifetime (violation)**: 0.28% of granules reclaimable at close → linear growth
- **Dual-epoch (satisfaction)**: 66.5% of granules reclaimable at close → bounded plateau

### Pinning Growth Theorem (Paper Section 5.4)

**Theorem 3**: If a positive fraction p of granules retain at least one live allocation at their reclaim boundary, then retained granules grow as R(t) ≥ p·m(t).

**Empirical validation:**
- **Mixed-lifetime**: 2.465 slabs/request growth rate (200K requests, perfectly linear)
- **Dual-epoch**: 0 growth after 10K requests (plateau at 2,143 slabs)

### Differential Summary

| Metric | Mixed (Violation) | Isolated (Satisfaction) | Ratio |
|--------|-------------------|-------------------------|-------|
| **Recycle rate** | 0.28% | 66.5% | **238×** |
| **Retained slabs** | ~493K (~1.9 GB) | ~2K (~8 MB) | 246× |
| **RSS behavior** | Ω(t) linear | O(1) bounded | Dichotomy confirmed |
| **Empty slabs/close** | ~7 | ~266 | 38× |

---

## Experimental Design

### Mixed-Lifetime Experiment (Violation)

**Configuration:**
- Single epoch pool shared by both session and request objects
- Session pool: 4,000 objects × 256B (long-lived, 2% churn)
- Request allocations: 200 objects/request × mixed sizes (short-lived)
- Workload: 200,000 requests

**Violation condition:**
```
∃a_session ∈ allocs(g_request): t_free(a_session) > t_reclaim(g_request)
```

Session objects outlive request-scoped granules, pinning granules past their reclaim boundaries.

**Results:**
- Recycle rate: **0.28%** (1,372 slabs recycled / 494,316 slabs scanned)
- Retained slabs: ~493K (1,925.56 MB)
- Growth slope: 2.465 slabs/request (constant across 200K requests)
- Empty slabs per close: ~7

**Data files:**
- `experimental/mixed-lifetime/mixed_timeseries.csv`: Net slab retention per 1K requests
- `experimental/mixed-lifetime/mixed_closes.csv`: Empty slabs found at each epoch close
- `experimental/mixed-lifetime/mixed_stats.txt`: Aggregate statistics

### Dual-Epoch Experiment (Satisfaction)

**Configuration:**
- Dual epoch mode: separate session epoch (persistent) and request epochs (rotating)
- Session objects → epoch 0 (never closed)
- Request objects → epochs 1-N (closed every 1K requests)
- Workload: 20,000 requests

**Satisfaction condition:**
```
∀g_request, ∀a ∈ allocs(g_request): t_free(a) ≤ t_reclaim(g_request)
```

No lifetime mixing; request epochs contain only short-lived allocations.

**Results:**
- Recycle rate: **66.5%** (4,258 slabs recycled / 6,401 total)
- Retained slabs: ~2K (steady-state plateau)
- Growth phase: 0-10K requests (working set builds up)
- Plateau phase: 10K-20K requests (stable at 2,143 slabs)
- Empty slabs per close: ~266

**Per-size-class breakdown** (demonstrates drainability at granular level):

| Size Class | New Slabs | Recycled | Recycle Rate |
|------------|-----------|----------|--------------|
| 128B       | 10        | 40       | 80.0%        |
| **256B**   | **20**    | **3,482**| **99.4%**    |
| 384B       | 2,047     | 512      | 20.0%        |
| 512B       | 66        | 224      | 77.2%        |

*Note: 256B class (session size) achieves 99.4% recycle rate because session churn migrates objects to persistent epoch, leaving request-epoch slabs fully drainable.*

**Data files:**
- `experimental/dual-epoch/dual_timeseries.csv`: Net slab retention (shows plateau)
- `experimental/dual-epoch/dual_closes.csv`: Empty slabs found at each epoch close
- `experimental/dual-epoch/dual_stats.txt`: Aggregate statistics with per-class breakdown

---

## Implementation Context

### temporal-slab Allocator

The experimental validation uses **temporal-slab**, a lifetime-aware slab allocator implementing the drainability principles as architectural constraints:

**Design principles:**
1. **Epoch-based routing**: Allocations are routed to epochs (granules) representing program phases
2. **Passive reclamation**: `epoch_close()` scans epoch-specific slab lists and recycles empty slabs
3. **Disjoint allocation phases**: Epochs 0-15 maintain separate partial/full lists (no cross-epoch mixing)
4. **Monotonic advancement**: Once advanced past epoch N, no new allocations enter epoch N
5. **Bounded recycling**: Empty slabs return to a bounded cache (32 slabs/class) preventing unbounded growth

**Key invariants that enforce drainability:**
- Each epoch maintains its own slab lists (structural isolation)
- `epoch_close(N)` only recycles slabs from epoch N's lists (no cross-epoch effects)
- Slab recycle condition: `free_count == capacity` (perfectly drainable)

**Relationship to theory:**
- **Routing function ρ**: Implemented as `slab_malloc_epoch(alloc, size, epoch_id)`
- **Granule lifecycle**: `epoch_advance()` opens new granule, `epoch_close(epoch_id)` designates reclaim boundary
- **Drainability check**: `empty_partial_count` in epoch state tracks drainable slabs

**Documentation:**
- `implementation/semantics.md`: Authoritative contracts for epoch_advance() and epoch_close()
- `implementation/ARCHITECTURE.md`: Technical design, invariants, and state machine

---

## Reproducing the Results

**Note**: The temporal-slab allocator and tslab-request-bench repositories are currently private. The raw experimental data in this directory enables verification of the paper's claims without requiring access to the implementation. For collaboration or reproduction requests, please contact the author.

### Prerequisites (if repositories become public)

1. **temporal-slab allocator**: Implementation repository (currently private)
2. **tslab-request-bench**: Benchmark harness (currently private)

### Running Mixed-Lifetime Experiment

```bash
cd tslab-request-bench
make clean && make ALLOCATOR=temporal-slab
./bin/server_tslab --workload=frag --requests=200000 --batch-size=1000 > out_mixed.log
python3 parse_slab_diagnostics.py out_mixed.log
```

Expected output: `mixed_stats.txt` with ~0.28% recycle rate

### Running Dual-Epoch Experiment

```bash
cd tslab-request-bench
make clean && make ALLOCATOR=temporal-slab CFLAGS=-DTSLAB_DUAL_EPOCH=1
./bin/server_tslab --workload=frag --requests=20000 --batch-size=1000 > out_dual.log
python3 parse_slab_diagnostics.py out_dual.log
```

Expected output: `dual_stats.txt` with ~66.5% recycle rate

### Plotting Results

```bash
python3 plot_drainability.py mixed_timeseries.csv dual_timeseries.csv
# Generates: figure_drainability_centerpiece.png
```

---

## Data Format Specifications

### Timeseries CSV Format

**Columns:**
- `req`: Request count (1000, 2000, ..., N)
- `net_slabs`: Cumulative net slab retention (allocated - recycled)

**Example:**
```csv
req,net_slabs
0,0
1000,2464
2000,4929
...
200000,492944
```

### Closes CSV Format

**Columns:**
- `req_close_at`: Request count when epoch close occurred
- `empty_slabs_found`: Number of empty slabs discovered and recycled at this close

**Example:**
```csv
req_close_at,empty_slabs_found
5000,7
6000,7
...
200000,7
```

### Stats TXT Format

**Structure:**
```
Temporal-slab epoch statistics:
  Total epoch advances: N
  Total epoch closes:   M

Slab pool diagnostics:
  Class <size>B: new=<allocated> recycled=<recycled> net=<retained> slowpath=<allocations_needing_new_slab>
  ...
  TOTAL: new=<total_allocated> recycled=<total_recycled> net=<total_retained> (<MB> retained)
```

---

## Additional Data Worth Including in Paper

Based on review of the raw data, the following additions would strengthen the empirical section:

### 1. Per-Size-Class Breakdown (HIGH PRIORITY)

**Location**: Section 6.3, after line 239

The per-class data shows drainability operates at slab-class granularity, not just aggregate:

| Class | New | Recycled | Recycle Rate |
|-------|-----|----------|--------------|
| 128B  | 10  | 40       | 80.0%        |
| 256B  | 20  | 3,482    | **99.4%**    |
| 384B  | 2,047 | 512    | 20.0%        |
| 512B  | 66  | 224      | 77.2%        |

The 256B class (session objects) achieving 99.4% recycle rate demonstrates that lifetime-aligned routing enables near-perfect reclamation at individual size-class level.

### 2. Exact Growth Slope (MEDIUM PRIORITY)

**Location**: Section 6.2, line 222

Current text: "~2.5 slabs per request"
Proposed: "2.465 slabs per request" (calculated from 492,944 slabs / 200,000 requests)

The constant slope across 200K requests confirms sustained violation, not transient effects.

### 3. Plateau Onset (LOW PRIORITY)

**Location**: Section 6.3, after line 240

From `dual_timeseries.csv`: plateau onset at request 10,000 (working set stabilization), then constant at 2,143±0 slabs through 20K requests. Demonstrates the O(1) bound is deterministic, not merely asymptotic.

---

## Citation

If using this data, please cite both the paper and the implementation:

**Paper:**
```bibtex
@techreport{blackwell2026drainability,
  title     = {Drainability: When Coarse-Grained Memory Reclamation Produces Bounded Retention},
  author    = {Blackwell, Dayna},
  year      = {2026},
  doi       = {10.5281/zenodo.18653776},
  note      = {Technical Report}
}
```

**Implementation:**
```bibtex
@software{blackwell2026temporalslab,
  author    = {Blackwell, Dayna},
  title     = {temporal-slab: Lifetime-Aware Slab Allocator},
  year      = {2026},
  url       = {https://github.com/blackwell-systems/temporal-slab}
}
```

---

## Contact

Questions about the data or reproduction:
- **GitHub Issues**: [drainability-framework/issues](https://github.com/blackwell-systems/drainability-framework/issues)
- **Email**: dayna@blackwell-systems.com

---

**Generated**: 2026-02-16
**Last Updated**: 2026-02-16
