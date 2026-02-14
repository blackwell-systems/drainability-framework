[![Blackwell Systems™](https://raw.githubusercontent.com/blackwell-systems/blackwell-docs-theme/main/badge-trademark.svg)](https://github.com/blackwell-systems)

# Drainability: A Structural Condition for Granularity-Based Memory Reclamation

**Dayna Blackwell, 2026**

## Abstract

Many memory management systems reclaim memory at coarse granularity — slabs, regions, generations, or epochs — where a reclamation unit becomes reclaimable only when it contains no live allocations. We define *drainability* as a structural property of allocation lifetimes relative to the lifecycle of the granule to which they are routed, and use it to characterize when coarse-grained reclamation can succeed.

We establish two results:

**Alignment Theorem.** A granule is reclaimable at its designated reclaim boundary if and only if every allocation routed to it has completed by that boundary.

**Pinning Growth Theorem.** If a positive fraction of granules retain at least one live allocation at their reclaim boundary, and granules close at a sustained rate, then the number of retained granules grows at least linearly in the number of reclamation cycles.

These results yield a precise distinction between two failure modes: *logical leaks*, where an allocation is never freed (a correctness bug), and *structural leaks*, where all allocations are eventually freed but lifetime–granularity misalignment prevents granules from draining — producing unbounded memory growth that is invisible to conventional leak detectors.

We validate both results empirically in a slab-granularity allocator, observing a 238× differential in recycle rate between lifetime-mixed and lifetime-isolated routing under identical allocator logic and identical workload, differing only in allocation routing. This work does not introduce a new allocator; it isolates a structural condition governing when coarse-grained reclamation can succeed.

## 1. Introduction

Coarse-grained memory reclamation is fundamental to systems programming. Slab allocators, region-based allocators, arena allocators, epoch-based reclamation schemes, and non-compacting generational collectors all share a common structure: memory is organized into reclamation units (which we call *granules*), and a granule can be reclaimed only when it contains no live allocations.

Despite the ubiquity of this pattern, there is no standard framework for reasoning about when it works. Practitioners encounter a recurring failure mode: a system that passes Valgrind, contains no logical leaks, and yet exhibits unbounded RSS growth under sustained load. The usual diagnosis is "fragmentation," but this term conflates several distinct phenomena and offers no structural guidance for remediation.

This paper isolates the core structural condition: *drainability*. A granule is drainable if every allocation routed to it completes by the granule's reclaim boundary. When this condition holds, reclamation succeeds deterministically. When it is violated — typically because allocations with different lifetime characteristics are mixed within the same granule — reclamation degrades, and retained memory grows without bound.

The contribution is primarily one of framing. The alignment theorem is a direct consequence of the definitions, which is a feature rather than a limitation: it means the abstraction is well-chosen. The value lies in naming the condition precisely, distinguishing structural leaks from logical leaks, characterizing the growth behavior under violation, and demonstrating that allocation *routing* — not allocator *policy* — is the determining factor.

**Scope.** The drainability framework applies to allocators that reclaim memory at granularity boundaries without relocating live objects across granules. Compacting collectors, which can consolidate live objects into fewer granules through relocation, alter the reclaimability condition fundamentally and are outside the scope of this analysis. We discuss the relationship to compacting collection in Section 7.

## 2. Model

### 2.1 Allocations and Granules

Let **A** be the set of allocations and **G** be the set of reclamation units (*granules*: slabs, regions, generations, epochs).

Each allocation *a* ∈ **A** has a lifetime:

    lifetime(a) = [t_alloc(a), t_free(a)]

where t_free(a) < ∞ for all allocations we consider (the case t_free(a) = ∞ is a logical leak, treated separately in Section 4).

Each granule *g* ∈ **G** has a lifecycle with a designated reclaim boundary:

    lifecycle(g) = [t_open(g), t_reclaim(g)]

Each allocation is routed to exactly one granule:

    granule: A → G

### 2.2 Reclamation Rule

We consider *non-relocating* reclamation: a granule *g* is reclaimable at time *t* if and only if it contains no live allocations at *t*:

    Reclaimable(g, t) ⟺ ∀a: granule(a) = g ⇒ t ≥ t_free(a)

Equivalently:

    Reclaimable(g, t) ⟺ live(g, t) = ∅

This captures slab reclamation, region deallocation, non-compacting generation collection, and epoch close. The key constraint is that live objects are never moved between granules.

**Non-Relocating Retention Assumption.** We assume that a granule that is not reclaimable at its designated boundary remains retained until it becomes reclaimable at some later time. In particular, the allocator does not relocate live objects across granules, and granules are not merged across reclaim boundaries. This assumption connects the alignment theorem (Section 3.2) to the growth bounds (Section 5): a non-drainable granule at its boundary necessarily contributes to retained memory.

## 3. Drainability

### 3.1 Definition

A granule *g* is **drainable** if and only if every allocation routed to *g* completes by the reclaim boundary:

    Drainable(g) ⟺ ∀a ∈ allocs(g): t_free(a) ≤ t_reclaim(g)

Drainability is not purely an allocator property; it is a joint property of (1) the program's allocation lifetime structure and (2) the routing function that assigns allocations to granules. The same program may produce drainable granules under one routing and non-drainable granules under another. This is the central observation of the paper.

### 3.2 Alignment Theorem

**Theorem 1** (Reclaimability ⟺ Drainability at the Boundary). *For any allocator reclaiming memory at granularity G under the non-relocating rule, a granule g is reclaimable at its designated reclaim boundary t_reclaim(g) if and only if g is drainable.*

**Proof.**

(⇒) Suppose *g* is reclaimable at t_reclaim(g). Then live(g, t_reclaim(g)) = ∅, so for every *a* ∈ allocs(g), t_free(a) ≤ t_reclaim(g). Hence *g* is drainable.

(⇐) Suppose *g* is drainable. Then for every *a* ∈ allocs(g), t_free(a) ≤ t_reclaim(g), so live(g, t_reclaim(g)) = ∅, and *g* is reclaimable at t_reclaim(g). ∎

The biconditional follows directly from the definitions. This is by design: the purpose of the drainability abstraction is to name the exact condition that separates reclaimable from non-reclaimable granules, not to derive a surprising consequence.

The practical content of the theorem is that non-drainable granules induce *pinning pressure*: a single long-lived allocation can prevent reclaiming the entire granule and all the memory it contains.

## 4. Failure Modes

Two structurally distinct failure modes produce retained memory growth:

**Logical leak (object-level).**

    ∃a: t_free(a) = ∞

An allocation is never freed. This is a correctness bug, detectable by tools such as Valgrind and AddressSanitizer.

**Structural leak (granule-level).**

    ∀a: t_free(a) < ∞, but ∃g: live(g, t_reclaim(g)) ≠ ∅

Every allocation is eventually freed, but lifetime mixing prevents granules from draining at their reclaim boundaries.

The structural leak is the more insidious failure mode precisely because it is invisible to object-level leak detectors. A system exhibiting structural leaks will pass Valgrind cleanly: every allocation has a corresponding deallocation. Yet RSS grows without bound because granules accumulate faster than they drain.

**Structural Leak Principle.** In granularity-based reclamation systems, retained-granule growth (and therefore retained memory under non-relocating retention) may be asymptotically forced by lifetime–granularity misalignment, even in the complete absence of logical leaks.

## 5. Growth Bounds

We characterize retained memory growth under drainability satisfaction versus violation.

### 5.1 Slab-Granularity Refinement

For slab allocators with size classes, let:

- *k*: a size class
- *C_k*: objects per slab (capacity for class *k*)
- *L_k(g)*: live objects of class *k* remaining in granule *g* at its reclaim boundary
- *R_k(g)*: retained slabs of class *k* after the reclaim boundary of *g*

A slab is reclaimable if and only if it contains zero live objects, so:

    R_k(g) ≥ ⌈L_k(g) / C_k⌉

Any live object pins its entire slab. Total retained slabs by time *t*:

    R(t) = Σ_{g closed by t} Σ_k R_k(g)

### 5.2 Assumptions

We make two bounding assumptions to isolate structural effects from allocator-internal behaviors:

1. **Bounded recycling caches.** The number of empty slabs held in free lists or recycling caches is bounded by a constant *K*.
2. **Bounded pipeline depth.** The number of simultaneously open granules is bounded by a constant *G_open*.
3. **Non-merging.** Granules are not merged across reclaim boundaries. Each violating granule contributes a distinct set of retained slabs.

These assumptions are satisfied by practical allocators and ensure that any unbounded growth in R(t) is attributable to structural pinning rather than unbounded internal caching.

### 5.3 Bounded Growth Under Drainability (Theorem 2)

**Theorem 2.** *If every closed granule is drainable, then R(t) is bounded independent of time.*

**Proof.** Drainability implies L_k(g) = 0 for all size classes *k* and all closed granules *g*, hence R_k(g) = 0 at close. Retained slabs therefore consist only of slabs in currently open granules and slabs in bounded recycling caches. Let *peak_k* be the maximum number of live objects of class *k* in any single open granule. Then:

    R(t) ≤ G_open · Σ_k ⌈peak_k / C_k⌉ + K

This bound is constant in *t*. ∎

**Corollary.** Under drainability, RSS converges to a steady-state plateau determined by the pipeline depth, peak per-granule allocation density, and cache capacity — not by total allocation volume.

### 5.4 Linear Lower Bound Under Violation (Theorem 3)

**Theorem 3.** *Let m(t) be the number of granules closed by time t. If there exists p > 0 such that for at least a fraction p of closed granules, at reclaim time there exists some size class k with L_k(g) ≥ 1, then:*

    R(t) ≥ p · m(t)

**Proof.** Each such granule contributes at least one retained slab (since ⌈L_k(g)/C_k⌉ ≥ 1 when L_k(g) ≥ 1). With at least p · m(t) such granules by time *t*, the bound follows. ∎

**Instantiation for batch workloads.** For workloads processing requests of average size *B* allocations, granule closes scale as m(t) ≈ t/B, yielding:

    R(t) = Ω(t/B)

**Corollary.** Under sustained drainability violation, RSS grows without bound, at a rate proportional to the request processing rate.

**On "sustained" violation.** Theorem 3 requires that a positive fraction *p* of granules are non-drainable. In practice, this occurs whenever allocations from distinct lifetime classes (e.g., session-scoped and request-scoped objects) are persistently routed to the same granules. The violation is sustained as long as the routing policy and workload mix remain stable, which is the common case for server workloads under steady-state load. Transient mixing — a brief burst followed by clean routing — produces a bounded one-time cost rather than unbounded growth.

## 6. Empirical Validation

We validate the alignment and pinning growth theorems using a slab-granularity allocator with controlled routing.

### 6.1 Experimental Design

The allocator implementation is held constant across both experiments. The *only* variable is the allocation routing function: whether allocations with different lifetime characteristics are routed to the same granules (mixing) or to separate granules (isolation).

This controlled design isolates the structural effect of drainability from all other allocator behaviors (caching policy, size-class selection, slab sizing, etc.).

### 6.2 Lifetime Mixing (Drainability Violation)

**Workload.** Long-lived session objects and short-lived request objects are routed into the same rotating granules.

**Violation condition:**

    ∃a_session: lifetime(a_session) ⊈ lifecycle(g_request)

Session objects outlive the request-scoped granules they are placed in, pinning those granules past their reclaim boundaries.

**Results (200K requests):**

| Metric | Value |
|---|---|
| Recycle rate | 0.28% |
| Retained slabs | ~493K |
| RSS growth rate | ~9.7 MB / 1K requests |
| Empty slabs per close | ~7 |

**Observed behavior.** Retained slabs grew at approximately 2.5 slabs per request, confirming the linear growth predicted by Theorem 3. The observed slope remained stable over the full measurement window, indicating steady-state sustained violation rather than transient burst effects.

### 6.3 Lifetime Isolation (Drainability Satisfaction)

**Workload.** Session objects are routed to a persistent granule; request objects are routed to rotating granules. No cross-lifetime mixing occurs.

**Satisfaction condition:**

    ∀g_request, ∀a ∈ allocs(g_request): t_free(a) ≤ t_reclaim(g_request)

**Results (20K requests):**

| Metric | Value |
|---|---|
| Recycle rate | 66.5% |
| Retained slabs | ~2K |
| RSS plateau | ~8 MB |
| Empty slabs per close | ~266 |

**Observed behavior.** Retained slabs plateaued at approximately 2K regardless of total request volume, confirming the constant bound predicted by Theorem 2.

### 6.4 Comparative Summary

| Metric | Mixed (Violation) | Isolated (Satisfaction) | Theoretical Prediction |
|---|---|---|---|
| Recycle rate | 0.28% | 66.5% | — |
| Retained slabs | ~493K | ~2K | Ω(t) vs O(1) |
| RSS behavior | Linear growth | Plateau | Ω(t) vs O(1) |

The recycle-rate differential is 238× (66.5% / 0.28%). The allocator implementation is identical in both experiments, under identical workload; only routing changed.

### 6.5 Limitations

This empirical validation uses a single workload (session + request objects), a single allocator, and a single size-class distribution. The 238× figure is specific to this configuration's parameters — particularly the ratio of session to request object lifetimes and the slab capacity. The theoretical results (Theorems 2 and 3) are general, but validating them across a broader range of workloads — multiple lifetime classes, real application traces, varying size-class distributions — remains future work.

## 7. Discussion

### 7.1 Allocation Routing Is a Structural Decision

The drainability framework reveals that allocation routing is not merely an optimization concern but a structural one. Reclamation success depends on whether the routing function respects lifetime boundaries, not on how frequently granules are closed or how aggressively caches are managed.

A routing-aware allocation pattern separates lifetime classes explicitly:

```
// Long-lived allocations → persistent scope
session_scope_enter();
allocate_session_object();
session_scope_exit();

// Short-lived allocations → rotating scope
request_scope_enter();
allocate_request_object();
request_scope_exit();  // granule close triggers reclamation
```

### 7.2 The Limits of Policy

Under sustained drainability violation, no amount of allocator tuning can restore bounded memory growth within the non-relocating model. More frequent granule closes do not help — they simply create more non-drainable granules faster. Heuristic reordering of allocations within a granule does not help — the pinning object remains.

However, it is worth noting that practical mitigations can reduce the *constant factor* of the growth. Segregated free lists that probabilistically cluster allocations by lifetime, lazy coalescing strategies, and partial compaction within a granule can all slow the rate of growth. The drainability framework predicts that these approaches change the coefficient but not the asymptotic class: under sustained mixing, growth remains Ω(m(t)) regardless of mitigation effort.

The only *asymptotic* fix is architectural: route allocations so that each granule's contents are lifetime-homogeneous with respect to the granule's reclaim boundary.

### 7.3 Relationship to Compacting Collection

Compacting garbage collectors (and relocating generational collectors) operate under a fundamentally different reclamation rule: live objects can be *moved* from one granule to another, consolidating surviving objects into fewer granules and freeing the rest. This violates the non-relocating assumption of Section 2.2 and changes the reclaimability condition entirely.

In a compacting collector, a granule need not be drainable to be reclaimed — the collector simply evacuates its live objects first. Generational collectors exploit this directly: objects that survive a young-generation collection are *promoted* (relocated) to an older generation, allowing the young generation to be reclaimed even though it contained long-lived objects.

Drainability is therefore *not* a necessary condition for reclamation in general — it is a necessary condition for reclamation *in non-relocating systems*. The framework characterizes exactly the systems where architectural routing discipline is the only path to bounded memory, as opposed to systems where the runtime can compensate through relocation. This distinction helps practitioners choose between allocator designs: if relocation is acceptable (e.g., in a managed runtime), compaction can mask lifetime mixing. If relocation is not acceptable (e.g., in systems with raw pointers, foreign function interfaces, or real-time constraints), drainability becomes a hard requirement.

### 7.4 Relationship to Fragmentation

"Fragmentation" is commonly used to describe the symptom that drainability violation produces, but the term conflates several distinct phenomena: *internal* fragmentation (wasted space within an allocated block), *external* fragmentation (free memory that cannot satisfy a request due to non-contiguity), and what we call *structural* fragmentation (granules that cannot be reclaimed due to pinning). The drainability framework provides a precise characterization of the third category and separates it from the first two, which have different causes and different remedies.

## 8. Related Granularity-Based Systems

The drainability condition applies uniformly to non-relocating systems that reclaim at granularity boundaries, including:

- **Slab allocators** (Bonwick, 1994; Linux SLUB): granule = slab, reclaim boundary = all objects freed.
- **Region/arena allocators** (Tofte & Talpin, 1997; Rust's `bumpalo`): granule = region, reclaim boundary = region deallocation.
- **Epoch-based reclamation** (Fraser, 2004; crossbeam): granule = epoch, reclaim boundary = epoch quiescence.
- **Non-compacting generational schemes**: granule = generation, reclaim boundary = generation collection without relocation.

In each case, the alignment theorem applies: deterministic reclamation at the boundary requires drainability of the granule.

## 9. Conclusion

We establish two structural results for coarse-grained, non-relocating memory reclamation:

**Alignment Theorem:**

    Reclaimable(g, t_reclaim(g)) ⟺ ∀a ∈ allocs(g): t_free(a) ≤ t_reclaim(g)

**Pinning Growth Theorem:**

    Under sustained violation with fraction p > 0: R(t) ≥ p · m(t)

Empirically, drainability violation produces reclamation collapse (0.28% recycle rate, linear RSS growth) while drainability satisfaction produces predictable drainage (66.5% recycle rate, bounded RSS plateau), a 238× recycle-rate differential under identical allocator logic and identical workload, differing only in allocation routing.

**Core thesis.** Reclamation success in non-relocating, coarse-grained systems is determined by the structural alignment between allocation routing and lifetime semantics — not by allocator policy, tuning, or heuristics. The drainability framework characterizes when deterministic reclamation is possible and explains a class of memory growth failures that are invisible to conventional leak detection.

---

*If you use this framework, please cite:*

Blackwell, D. (2026). Drainability: A Structural Condition for Granularity-Based Memory Reclamation. Technical Report.

*Licensed under CC-BY-4.0.*
