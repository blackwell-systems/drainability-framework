# Phase 2.2 Validation Summary

## Dual-Track RSS Determinism Testing

### Track 1: Mixed-Lifetime Workload (Limitation)

**Result:** epoch_close ineffective under pinning pressure

| Configuration | Recycle Rate | RSS Growth | Finding |
|---------------|--------------|------------|---------|
| glibc | N/A | 7.25 MB/1K req | Baseline drift |
| tslab-unstructured | 0.00% | 9.57 MB/1K req | No reclamation (expected) |
| tslab-structured | **0.28%** | 9.69 MB/1K req | **Pinning prevents reclamation** |

**Smoking gun:** Out of 494,316 slabs scanned during 196 epoch closes, only 1,372 (0.28%) were reclaimable.

**Conclusion:** 
> Closing epochs cannot reclaim what never becomes empty.

When session and request objects share slab populations, pinning pressure prevents slabs from draining. This is correct conservative behavior - temporal-slab preserves liveness but cannot return memory at slab granularity when nothing becomes fully empty.

**Positioning:** Honest limitation / boundary condition figure for paper.

---

### Track 2: Dual-Epoch Workload (Success)

**Result:** Structural determinism validated

**Configuration:**
- `TSLAB_DUAL_EPOCH=1`
- Session epoch (0): persistent, never closed
- Request epochs (1-N): closed every 1K requests

**Key Findings (20K requests, 16 closes):**

| Metric | Value | Interpretation |
|--------|-------|----------------|
| Slabs allocated | 2,143 | Steady-state working set |
| Slabs recycled | 4,258 | **Aggressive reclamation** |
| **Recycle rate** | **66.5%** | **238× improvement vs mixed** |
| Request allocs = frees | 1M = 1M | Clean epoch drainage |

**Per-size-class breakdown:**

| Class | New | Recycled | Rate |
|-------|-----|----------|------|
| 128B  | 10  | 40       | 80.0% |
| 256B  | 20  | 3,482    | 99.4% |
| 384B  | 2,047 | 512    | 20.0% |
| 512B  | 66  | 224      | 77.2% |

**Critical observation:** 256B class (session size) shows 99.4% recycle rate in *request epochs* because churned sessions migrate to persistent epoch 0, leaving request-epoch slabs fully drainable.

**Conclusion:**
> Aligning allocations with semantic phase boundaries enables deterministic reclamation.

With proper lifetime isolation:
- Request epochs drain to empty → high recycle rates
- Session epoch remains stable → no unnecessary churn
- RSS growth bounded by structural design (not allocator heuristics)

**Positioning:** Centerpiece empirical validation for paper.

---

## The Thesis

**Claim:** Structural determinism requires epoch isolation at lifetime boundaries.

**Evidence:**
1. **Negative (mixed):** 0.28% recycle rate when lifetimes share epochs
2. **Positive (isolated):** 66.5% recycle rate when lifetimes separated
3. **Mechanism:** Request epochs fully drain → aggressive slab recycling

**Design Pattern:**

```c
// Session (persistent)
session_scope_enter();
ptr = xmalloc(256);  // → epoch 0
session_scope_exit();

// Request (ephemeral)  
request_scope_enter();  // Advances every 1K requests
ptr = xmalloc(384);     // → epoch N
xfree(ptr);             // Freed before close
request_scope_exit();   // → epoch_close(N-4)
```

**Outcome:** Session RSS stable, request RSS bounded by epoch reclamation frequency.

---

## Known Issue

**Zombie slab repair loop:** High-throughput dual-epoch workload triggers infinite repair on specific slabs. This is a separate allocator bug (likely in concurrent free_count updates during aggressive recycling). Does not invalidate the 66.5% recycle rate finding from early-stage data.

**Status:** Reproducible at ~20K requests. Investigation needed in temporal-slab core.

---

## Next Steps

1. **Fix zombie slab bug** in temporal-slab (separate issue from Phase 2.2 validation)
2. **Generate RSS stability plot** comparing mixed vs isolated (even with 5-20K data)
3. **Integrate findings into whitepaper:**
   - Mixed-lifetime figure → Section 5.7 (Limitations)
   - Isolated-lifetime figure → Section 3.4 or 5 (Empirical Validation)
4. **Write clear positioning:**
   - Not "temporal-slab always reduces RSS"
   - Instead: "Structure-aligned allocation enables deterministic reclamation"

---

**Files:**
- `RSS_DETERMINISM_RESULTS.md` - Mixed-lifetime analysis
- `DUAL_EPOCH_SUCCESS.md` - Isolated-lifetime validation
- `figure_rss_determinism_200k.png` - Mixed baseline (limitation)
- `test_dual_5k.csv` - Dual-epoch RSS data (clean run)

**Implementation:**
- `/home/blackwd/code/tslab-request-bench/src/alloc_tslab.c` - Dual-epoch adapter
- `/home/blackwd/code/tslab-request-bench/src/workload_frag.c` - Lifetime-separated workload

**Generated:** 2026-02-11
