# Drainability as a Structural Condition for Granularity-Based Reclamation

## Abstract

We study allocators that reclaim memory at coarse granularity-slabs, regions, generations, or epochs-where a reclamation unit is reclaimable only when it contains no live allocations. We define **drainability** as a property of allocation lifetimes relative to the lifecycle of the granule to which they are routed.

We state two laws:

**Granularity–Lifetime Alignment Law.** Deterministic reclamation at granularity G is possible iff allocations routed to a granule have lifetimes bounded by that granule's reclaim boundary.

**Pinning Growth Law.** Under sustained lifetime mixing, if a non-zero fraction of reclamation units close while retaining at least one live allocation, then retained granules grow at least linearly in the number of reclamation cycles.

This yields a practical distinction:

* **Logical leaks**: an allocation is never freed.
* **Structural leaks**: all allocations are freed, but lifetime–granularity misalignment prevents granules from draining at reclaim boundaries, making retained-memory growth inevitable under churn.

We validate both laws empirically in a slab-granularity allocator with controlled lifetime mixing versus lifetime isolation, observing a 238× differential in recycle rate under identical allocator logic. This work does not introduce a new allocator; it isolates a structural condition governing when coarse-grained reclamation can succeed.

---

## 1. Model

### 1.1 Allocations and Granules

Let:

* **A** be the set of allocations.
* **G** be the set of reclamation units ("granules": slabs, regions, generations, epochs).

Each allocation a ∈ A has a lifetime:

```
lifetime(a) = [t_alloc(a), t_free(a)]
```

Each granule g ∈ G has a lifecycle with a designated reclaim boundary:

```
lifecycle(g) = [t_open(g), t_reclaim(g)]
```

Each allocation is routed to exactly one granule:

```
granule(a) ∈ G
```

---

### 1.2 Reclamation Rule (Non-relocating)

A granule g is reclaimable at time t iff it contains no live allocations at t:

```
Reclaimable(g,t) ⟺ ∀a: granule(a) = g ⇒ t ≥ t_free(a)
```

Equivalently:

```
Reclaimable(g,t) ⟺ live(g,t) = ∅
```

This captures slab reclamation, region free, generation collection (without relocation), and epoch close.

---

## 2. Drainability

### 2.1 Definition

A granule g is **drainable** iff every allocation routed to g completes by the reclaim boundary:

```
Drainable(g) ⟺ ∀a ∈ allocs(g): t_free(a) ≤ t_reclaim(g)
```

Drainability is therefore not purely an allocator property. It is a property of allocation routing relative to program lifetime structure.

---

## 3. Granularity–Lifetime Alignment

### Theorem (Reclaimability ⇔ Drainability at the Boundary)

For any allocator reclaiming memory at granularity G under the non-relocating rule, a granule g can be reclaimed at its designated reclaim boundary t_reclaim(g) iff g is drainable.

### Proof sketch

**Necessity.** If g is not drainable, then ∃a ∈ allocs(g) with t_free(a) > t_reclaim(g). At t_reclaim(g), a is live, so live(g, t_reclaim(g)) ≠ ∅, and g is not reclaimable.

**Sufficiency.** If ∀a ∈ allocs(g), t_free(a) ≤ t_reclaim(g), then no allocations are live at the boundary, so live(g, t_reclaim(g)) = ∅ and reclamation succeeds.

Thus:

```
Reclaimable(g, t_reclaim(g)) ⟺ Drainable(g)
```

Non-drainable granules induce **pinning pressure**: one long-lived allocation can prevent reclaiming the entire granule.

---

### Granularity–Lifetime Alignment Law

> Deterministic reclamation at granularity G is possible iff allocations routed to each granule have lifetimes bounded by that granule's reclaim boundary.

---

### Pinning Growth Law

> If there exists ε > 0 such that at least an ε fraction of closed granules retain at least one live allocation at close time, then retained granules grow as Ω(m(t)), where m(t) is the number of reclamation cycles.

---

### 3.1 Structural Leaks

Two failure modes are fundamentally different:

**Logical leak** (object-level):

```
∃a: t_free(a) = ∞
```

An allocation is never freed. This is a correctness bug and is detectable by tools like Valgrind/ASan.

**Structural leak** (granule-level):

```
∀a: t_free(a) < ∞
but
∃g: live(g, t_reclaim(g)) ≠ ∅
```

All allocations are eventually freed, but lifetime mixing prevents drainage at reclaim boundaries.

**Structural Leak Principle**:

> In granularity-based reclamation systems, retained-memory growth may be asymptotically forced by lifetime–granularity misalignment even in the absence of logical leaks.

This explains "leaking" systems that are Valgrind-clean: reclamation events occur, objects are freed, yet RSS grows because granules cannot drain.

---

## 4. Formal Growth Bounds

We characterize retained memory growth under drainability satisfaction vs violation.

---

### 4.1 Slab Refinement (Pinning at Slab Granularity)

For slab allocators with size classes:

* **k**: size class
* **C_k**: objects per slab (capacity)
* **L_k(g)**: live objects of class k remaining in granule g at reclaim boundary
* **R_k(g)**: retained slabs of class k after reclaim boundary

A slab is reclaimable iff it contains zero live objects, so:

```
R_k(g) ≥ ⌈L_k(g) / C_k⌉
```

Any live object pins its slab.

Total retained slabs by time t:

```
R(t) = Σ_{g closed by t} Σ_k R_k(g)
```

---

### 4.2 Assumptions

* Recycling caches are bounded by constant K.
* The number of simultaneously open granules is bounded by pipeline depth.

These isolate structural effects from allocator-internal unbounded caching.

---

### 4.3 Bounded Growth Under Drainability

If every closed granule is drainable, then retained slabs are bounded independent of time.

Drainability implies L_k(g) = 0, hence R_k(g) = 0 at close. Retained slabs then consist only of:

* slabs in currently open granules
* slabs in bounded recycling caches

Let:

* **G_open**: max simultaneously open granules
* **peak_k**: max live objects of class k per open granule

Then:

```
R(t) ≤ G_open · Σ_k ⌈peak_k / C_k⌉ + K
```

This is constant in t.

**Corollary**: RSS converges to a steady-state plateau.

---

### 4.4 Linear Lower Bound Under Violation

Suppose that for a fraction p > 0 of closed granules, at reclaim time there exists some size class k with at least one live object:

```
∃k: L_k(g) ≥ 1
```

Then R_k(g) ≥ 1, so each such granule contributes at least one retained slab. Let m(t) be the number of closed granules by time t. Then:

```
R(t) ≥ p·m(t)
```

For batch workloads with batch size B, closes scale as m(t) ≈ t/B, hence:

```
R(t) = Ω(t/B)
```

**Corollary**: RSS grows unboundedly under sustained violation.

---

## 5. Empirical Validation

We validate the laws using slab-granularity reclamation.

---

### 5.1 Lifetime Mixing (Violation)

**Workload.** Long-lived session objects and short-lived request objects are routed into the same rotating granules.

**Violation:**

```
∃a_session: lifetime(a_session) ⊈ lifecycle(g_request)
```

**Measured** (200K requests):

* Recycle rate: 0.28%
* Retained slabs: ~493K
* RSS growth: ~9.7 MB / 1K requests
* Empty slabs per close: ~7

**Observed behavior:**

```
R(t) ≈ 2.5 slabs/request (linear slope)
```

Matches Proposition 4.4 (Ω(t/B) growth).

---

### 5.2 Lifetime Isolation (Satisfaction)

**Workload.** Session objects are routed to a persistent granule; request objects are routed to rotating granules. No cross-lifetime mixing.

**Property:**

```
∀g_request, ∀a ∈ allocs(g_request): t_free(a) ≤ t_reclaim(g_request)
```

**Measured** (20K requests):

* Recycle rate: 66.5%
* Retained slabs: ~2K
* Empty slabs per close: ~266
* RSS plateau: ~8 MB

**Observed behavior:**

```
R(t) ≤ ~2K slabs (plateau independent of total requests)
```

Matches Proposition 4.3 (O(1) bound).

---

### 5.3 Differential

| Metric | Mixed | Isolated | Asymptotic Match |
|--------|-------|----------|------------------|
| Recycle rate | 0.28% | 66.5% | Violation vs Satisfaction |
| Retained slabs | ~493K | ~2K | Linear vs Bounded |
| RSS behavior | Linear | Plateau | Ω(t) vs O(1) |

Relative recycle-rate differential: 238×.

The allocator implementation is identical. Only allocation routing changed. The observed slopes and plateaus match the theoretical bounds from Proposition 4.4 (Ω) and Proposition 4.3 (O(1)), respectively. Together, **the Alignment Law and Pinning Growth Law define the Drainability Framework for coarse-grained reclamation**.

---

## 6. Architectural Implications

### 6.1 Allocation Routing Is Structural

To enable reclamation at granularity G, routing must align with lifetime classes.

```c
// Long-lived allocations
session_scope_enter();
allocate_session_object();
session_scope_exit();

// Short-lived allocations
request_scope_enter();
allocate_request_object();
request_scope_exit();  // granule close
```

Reclamation success depends on routing, not close frequency.

---

### 6.2 Policy Cannot Repair Structural Violation

Structural leaks cannot be fixed by allocator tuning:

* More frequent closes do not help.
* Heuristics do not help.
* Compaction cannot eliminate cross-granule pinning in non-relocating schemes.

The only fix is architectural: route allocations to align with their lifetimes.

---

## 7. Generalization

The Drainability Framework applies to allocators that reclaim memory strictly at granularity boundaries without relocating live objects across granules, including:

* slab allocators
* region/arena allocators
* epoch systems
* non-compacting generational schemes

Compacting collectors are excluded because relocation can consolidate live objects into fewer granules, changing the reclaimability condition.

Granularity–lifetime alignment is therefore a necessary condition for deterministic reclamation in non-relocating, coarse-grained systems.

---

## 8. Conclusion

We establish two structural laws for coarse-grained reclamation:

**Granularity–Lifetime Alignment Law:**

```
Reclaimable(g, t_reclaim(g)) ⟺ ∀a ∈ allocs(g): t_free(a) ≤ t_reclaim(g)
```

**Pinning Growth Law:**

```
Under sustained violation: R(t) = Ω(m(t))
```

Empirically:

* violation → reclamation collapse (0.28% recycle, linear growth)
* satisfaction → predictable drainage (66.5% recycle, bounded plateau)
* 238× differential under identical allocator logic

**Core thesis:**

> Reclamation success is not a property of policy; it is a property of structural alignment between allocation routing and lifetime semantics.

Coarse-grained memory reclamation is therefore a structural alignment problem, not solely an allocator problem. The Drainability Framework characterizes when deterministic reclamation is possible.
