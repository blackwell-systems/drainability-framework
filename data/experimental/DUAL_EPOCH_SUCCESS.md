# Dual-Epoch Mode: Structural Determinism Validated

## Summary

**Thesis:** Aligning allocations with semantic phase boundaries enables deterministic reclamation.

**Result:** ✅ VALIDATED

With proper lifetime separation:
- **66.5% slab recycle rate** (20K requests, 16 epoch closes)
- **238× improvement** over mixed-lifetime baseline (0.28%)
- Request epochs drain to empty, enabling aggressive reclamation
- Session epoch remains stable (never closed)

---

## Test Configuration

**Mode:** TSLAB_DUAL_EPOCH=1, EPOCH_POLICY=batch, BATCH_SIZE=1000

**Lifetime Separation:**
- **Session epoch (epoch 0):** Long-lived objects, 2% churn, never closed
- **Request epochs (1-N):** Short-lived objects, fully freed, closed every 1K requests

**Workload:**
- 20,000 requests
- Session pool: 4,000 objects × 256B = 1MB persistent
- Request allocations: 200 objects × mixed sizes (96-480B) = ~50KB per request
- All request objects freed before epoch close

---

## Results

### Epoch Close Statistics

| Metric | Value |
|--------|-------|
| Epoch advances | 20 |
| Epoch closes | 16 |
| Slabs allocated | 2,143 |
| Slabs recycled | 4,258 |
| **Recycle rate** | **66.5%** |

### Per-Size-Class Breakdown

| Size Class | New | Recycled | Recycle Rate |
|------------|-----|----------|--------------|
| 128B | 10 | 40 | 80.0% |
| 256B | 20 | 3,482 | 99.4% |
| 384B | 2,047 | 512 | 20.0% |
| 512B | 66 | 224 | 77.2% |

**Key observation:** 256B class (session size) shows 99.4% recycle rate because churned sessions go to persistent epoch 0, while their old slabs in request epochs drain completely.

---

## Comparison: Mixed vs Isolated Lifetimes

| Configuration | Recycle Rate | Slabs Retained | Interpretation |
|---------------|--------------|----------------|----------------|
| **Mixed lifetimes** (single epoch pool) | 0.28% | ~493K slabs (1.9 GB) | Pinning pressure prevents reclamation |
| **Isolated lifetimes** (dual epoch) | 66.5% | ~2K slabs (8 MB) | Clean phase boundaries enable aggressive reclamation |

**Improvement factor:** 238×

---

## Why This Works

### Before (Mixed):
- Session and request objects allocated in same epochs
- Session churn creates long-lived objects in "request" epochs
- Request epochs cannot drain to empty → no reclamation
- Result: 0.28% recycle rate

### After (Isolated):
- Session objects: `session_scope_enter()` → epoch 0 → never closed
- Request objects: `request_scope_enter()` → epochs 1-N → closed aggressively
- Request epochs contain ONLY short-lived allocations
- All request objects freed before close → epochs drain to empty
- Result: 66.5% recycle rate

---

## Code Pattern

```c
// Session operations (persistent epoch)
session_scope_enter();
xmalloc(SESSION_SIZE);  // Goes to epoch 0
session_scope_exit();

// Request operations (rotating epochs)
request_scope_enter();  // Advances epoch every 1K requests
xmalloc(REQUEST_SIZE);  // Goes to current request epoch
xfree(...);            // All request objects freed
request_scope_exit();   // Triggers epoch close (4-batch lag)
```

**Critical property:** No lifetime mixing within request epochs.

---

## Implications

1. **Structural determinism requires epoch isolation at lifetime boundaries**
   - Not merely periodic closing
   - Not merely temporal allocation
   - Requires: semantic alignment between allocation domains and object lifetimes

2. **Slab-granularity reclamation works when slabs can drain**
   - With isolation: 66.5% of slabs become reclaimable
   - Without isolation: 0.28% of slabs become reclaimable
   - Drainage is a property of workload structure, not allocator policy

3. **RSS determinism emerges from program+allocator co-design**
   - Allocator provides: epoch domains, deterministic close
   - Program provides: lifetime-aligned allocation routing
   - Result: predictable memory behavior

---

## Next Steps

1. **Run 200K test** to generate centerpiece RSS stability figure
2. **Compare RSS curves:**
   - Mixed: linear drift (~9.7 MB/1K requests)
   - Isolated: stable band with sawtooth (epoch boundaries visible)
3. **Integrate into whitepaper** as primary empirical validation
4. **Use mixed-lifetime result** as honest limitation / boundary condition

---

**Generated:** 2026-02-11  
**Test data:** `test_dual_20k.csv`  
**Implementation:** `/home/blackwd/code/tslab-request-bench/src/workload_frag.c` (dual-epoch mode)
