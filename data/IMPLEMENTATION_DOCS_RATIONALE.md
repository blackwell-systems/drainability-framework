# Implementation Documentation Rationale

## Documents Included in data/implementation/

### semantics.md (670 lines)
**Purpose**: Authoritative reference for epoch and domain contracts

**Why included for drainability paper:**
- Defines formal contracts for `epoch_advance()` and `epoch_close()`
- Maps directly to paper's mathematical model:
  - Granule lifecycle → Epoch states (ACTIVE, CLOSING)
  - t_reclaim(g) → `epoch_close(epoch_id)` call
  - Allocation rejection → `EPOCH_CLOSING` state check
- Documents invariants that enforce drainability:
  - "At most 1 ACTIVE epoch at any time"
  - "Allocations rejected when epoch state is CLOSING"
  - "Drain and reclaim empties: free_count == capacity"
- Validates Alignment Theorem empirically:
  - Precondition: epoch MUST be CLOSING
  - Postcondition: empty slabs recycled

**Key sections:**
- §Epoch Lifecycle States: ACTIVE vs CLOSING
- §epoch_advance() Contract: How granules are created
- §epoch_close() Contract: How reclaim boundary is implemented
- §Invariants: Structural guarantees

### ARCHITECTURE.md (546 lines)
**Purpose**: Design overview and system invariants

**Why included:**
- Memory model: Per-epoch slab lists (isolation)
- State machine: Allocation routing and recycling
- Invariants: "Slabs never unmapped during runtime", "Conservative deferred recycling"
- RSS bounds: Cache + overflow = known maximum

**Key sections:**
- §Memory Model: 16-epoch ring buffer, per-class state
- §Core Allocator Design: Fast path, slow path, recycling
- §Epoch State: How routing function ρ is implemented

---

## Documents NOT Included (and why)

### LIFETIME_ALGEBRA.md (173 lines) - REMOVED
**Why excluded:**
- Marketing/philosophical focus ("The Third Way", "Why This Changes Everything")
- Too high-level for research validation
- Repeats motivation that paper already covers
- Commercial applications examples not relevant to theory

**Replacement:** semantics.md provides the technical contracts needed

### foundations.md (~400+ lines)
**Why excluded:**
- Explains RSS drift problem (paper Section 1 already covers this)
- Focuses on "why temporal-slab exists" (motivation, not mechanism)
- Overlaps with paper's introduction

### EPOCH_DOMAINS.md (573 lines)
**Why excluded:**
- Documents high-level API (RAII-style domains)
- The paper uses raw epochs, not domains API
- API details not relevant to theoretical framework
- Refcount semantics are implementation details

### RSS_INSTRUMENTATION_SUMMARY.md (437 lines)
**Why excluded:**
- Narrowly focused on instrumentation methodology
- Explains how RSS was decomposed (allocator vs harness)
- Useful for implementation debugging, not theory validation
- The paper cites final results, not measurement methodology

---

## Mapping: Paper Concepts → Implementation Docs

| Paper Concept | semantics.md Reference | ARCHITECTURE.md Reference |
|---------------|------------------------|---------------------------|
| **Granule g** | Epoch (0-15 ring slot) | §Memory Model: epochs[16] |
| **Routing ρ** | implicit in epoch_id param | §Allocation: alloc_obj_epoch() |
| **t_reclaim(g)** | §epoch_close() Contract | §Epoch state: CLOSING transitions |
| **Drainable(g)** | "Drain and reclaim empties" | §Recycling: free_count == capacity |
| **live(g,t)** | Allocation rejection when CLOSING | §Invariants: per-epoch partial lists |
| **R(t) bound** | Cache capacity limits | §Bounded RSS: cache + overflow |

---

## For Paper Reviewers

If you want to understand how the experimental validation works:

1. **Start with semantics.md**: Understand epoch_advance() and epoch_close() contracts
2. **Read ARCHITECTURE.md §Memory Model**: See how per-epoch slab lists prevent mixing
3. **Check experimental data**: See how epoch_close() recycling matches theory

The implementation enforces drainability structurally:
- Separate per-epoch slab lists (no cross-epoch mixing)
- CLOSING state rejects new allocations (prevents post-boundary allocations)
- Conservative recycling (only reclaim when free_count == capacity)

These are not heuristics or optimizations—they're architectural invariants that make the Alignment Theorem hold by construction.

---

**Generated**: 2026-02-16
**Context**: Supporting drainability framework paper empirical validation
