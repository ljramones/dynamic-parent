# Dynamis Performance Constitution

**Non-Negotiable Performance Rules for the Dynamis Engine Ecosystem**

This document defines the hard performance rules for the Dynamis engine ecosystem.

It exists to prevent architectural drift, preserve deterministic behaviour, and maintain a production-grade performance culture across:

- Vectrix
- MeshForge
- DynamisGPU
- DynamisLightEngine
- future subsystems built on the same core

If a design conflicts with this document, the design must be justified by measurement and consciously accepted as an exception.

---

## 1. Measurements Win

No subsystem may claim a performance improvement without evidence.

Accepted evidence includes:

- benchmark results
- JFR profiling
- controlled workload comparison
- frame-time or latency measurements on real workloads

Anecdotes, intuition, and "this should be faster" are not sufficient.

## 2. Hot Paths Must Be Allocation-Free

No per-frame or per-operation heap allocation is permitted in established hot paths unless explicitly approved.

This applies to:

- transform loops
- render submission paths
- geometry packing loops
- upload scheduling paths
- visibility/culling loops
- animation sampling loops
- physics integration loops
- audio mixing paths

Allowed strategies:

- persistent arenas
- pools
- pre-sized reusable buffers
- immutable prebuilt runtime structures

If allocation appears in a hot loop, treat it as a defect.

## 3. Data Layout Beats Abstraction

In performance-critical code, memory layout is more important than API elegance.

Preferred:

- contiguous arrays
- SoA layouts
- tightly packed structs/buffers
- bulk iteration
- stable traversal order

Disfavoured in hot paths:

- pointer chasing
- deep wrapper layers
- small temporary objects
- polymorphic dispatch in inner loops
- abstraction that hides memory movement

Readable code matters, but not at the cost of invisible performance failure.

## 4. The Runtime Must Operate on Runtime Data

Authoring structures are not runtime structures.

All hot systems must consume data that is already transformed into runtime-ready form.

Examples:

- MeshForge authoring mesh → packed runtime mesh
- scene descriptions → compact runtime world state
- content metadata → upload plans
- high-level graph structures → flattened execution order

The frame loop must not do authoring work.

## 5. Every Boundary Must Have a Cost Model

Subsystem boundaries are required, but every boundary must be understood in terms of:

- copies
- allocations
- synchronization
- cache disruption
- marshaling overhead
- latency added

A boundary that exists for cleanliness but whose cost is unknown is unfinished design.

## 6. Persistent Memory First

Before adding a new scratch allocation pattern, ask:

1. Can this memory be reused?
2. Can it live in an arena?
3. Can it be pooled?
4. Can we batch the work instead?

Default preference:

```
persistent > pooled > stack/local reuse > fresh allocation
```

Fresh allocation should be the exception, not the default.

## 7. Concurrency Must Be Measured, Not Assumed

More concurrency is not automatically better.

Concurrency is only justified when it improves one or more of:

- throughput
- latency
- responsiveness
- frame stability

The DynamisGPU upload work already established an important precedent:

```
optimal inflight depth = 2
```

That rule stands until new measurements prove otherwise.

Never increase parallelism "because more threads should help."

## 8. Throughput, Latency, and Pressure Must Be Considered Together

Subsystems must be evaluated as a triangle, not a single metric.

Core triad:

- **Throughput** — how much work gets done
- **Latency** — how quickly a unit of work completes
- **Pressure** — memory, queue depth, in-flight work, or scheduler stress

A design that improves throughput while exploding latency or pressure is not automatically an improvement.

## 9. No Blind Native Escapes

Do not jump to JNI, custom C/C++, or low-level native rewrites without evidence that the JVM path is genuinely insufficient.

Before considering native escape hatches, prove that:

- algorithm choice is already sound
- allocation behaviour is controlled
- data layout is correct
- Vector API / VarHandles / modern JVM tools are insufficient
- LWJGL / FFM / existing bindings are actually the bottleneck

Native code is a last resort, not a badge of seriousness.

## 10. Profile First, Then Target the Largest Real Cost

Optimization workflow is mandatory:

```
benchmark → profile → find dominant cost → fix dominant cost → re-measure
→ stop when dominance shifts or gain becomes marginal
```

Do not optimize the second-order effect while the first-order bottleneck remains.

Do not continue micro-optimizing once the hotspot has moved elsewhere.

## 11. Stable Frame Behaviour Matters More Than Peak Numbers

A subsystem that occasionally produces a great benchmark but causes erratic latency is not production-ready.

Prefer:

- bounded queues
- bounded inflight work
- predictable completion
- stable p95/p99
- controlled backpressure

Peak throughput is interesting. Stable behaviour is shippable.

## 12. Per-Frame Work Must Be Bounded

Each subsystem must be designed so its per-frame work can be limited or deferred.

Examples:

- capped upload work
- capped streaming work
- capped AI evaluation tiers
- capped background resource processing
- capped shader/material compilation work

The engine must never allow an "unbounded catch-up" pattern on the frame-critical path.

## 13. Bulk Work Beats Tiny Work

Whenever possible, process work in batches.

Prefer:

- batched uploads
- bulk transforms
- array-based culling passes
- grouped state changes
- packed submission buffers

Avoid turning naturally bulk work into thousands of tiny operations.

The CPU and memory hierarchy reward bulk work.

## 14. Copying Must Be Intentional

Copies are not forbidden, but every nontrivial copy must have a reason.

Valid reasons include:

- representation change
- packing/compression
- upload to device memory
- lifetime isolation
- synchronization simplification

Invalid reason:

- "it was easier this way"

If data is copied multiple times, the design should explain why each copy exists.

## 15. Telemetry Is Part of the System

Important subsystems must expose measurable state.

Examples:

- queue depth
- inflight count
- bytes transferred
- upload completion latency
- frame graph execution time
- culling counts
- draw/dispatch counts
- memory pool utilization

A system without telemetry is harder to optimize and harder to trust.

## 16. The Engine Must Favour Locality

Locality is a first-class performance feature.

Design for:

- contiguous memory
- stable iteration order
- cache-friendly grouping
- reduced working-set size
- minimal branch unpredictability

Locality is not an implementation detail. It is part of the architecture.

## 17. Runtime Math Must Be Allocation-Free and Bulk-Friendly

Vectrix-backed math in hot paths must favour:

- packed representations
- bulk transforms
- SIMD-friendly layouts
- reusable output buffers
- minimal wrapper overhead

Matrix-heavy fallback paths may exist for compatibility, but compact bulk transforms should be the preferred runtime path where validated.

This matches the current engine direction already established around packed-affine and locality-first bulk processing.

## 18. Scheduling Policy Is a Design Surface

Queues, backlog limits, and pull/push behaviour are not implementation trivia.

They materially affect:

- time-to-first-use
- memory pressure
- responsiveness
- completion fairness

Scheduling policy should be explicit, documented, and measured.

The system must not silently accumulate work in uncontrolled ways.

## 19. New Features Do Not Get a Free Performance Pass

Every new subsystem or major feature must answer:

1. What is the runtime cost model?
2. What memory does it retain?
3. What work happens every frame?
4. What work can be deferred?
5. What telemetry proves it behaves correctly?

A feature is not "done" when it works. It is done when its performance behaviour is understood.

## 20. Exceptions Must Be Explicit

Sometimes a rule will be broken for good reasons.

That is acceptable only when documented clearly:

- what rule is being broken
- why it is necessary
- what measurements justify it
- what guardrails limit the damage
- whether it is temporary or permanent

Silent exceptions are how performance cultures decay.

---

## Operational Checklist

Before merging performance-sensitive work, ask:

- [ ] Does this add heap allocation to a hot path?
- [ ] Does this increase copies or marshaling?
- [ ] Does this worsen locality?
- [ ] Does this add unpredictable synchronization?
- [ ] Is the cost measured?
- [ ] Is telemetry available?
- [ ] Is the per-frame work bounded?
- [ ] Is there a simpler design with the same effect?

If any answer is unclear, the work is not ready.

---

## Current Locked Precedents

These are currently treated as established until contradicted by new measurement:

- pull-based upload scheduling is preferred
- target upload inflight depth is 2
- persistent staging memory is justified
- Java-side upload prep micro-optimization has reached diminishing returns
- LWJGL replacement or patching is not currently justified
- subsystem boundaries between MeshForge, DynamisGPU, and LightEngine should remain explicit

---

## Closing Principle

The Dynamis engine does not pursue performance as folklore.

It pursues performance as an engineering discipline built on:

- evidence
- bounded behaviour
- clear cost models
- locality
- deterministic execution
- architectural restraint

That discipline is a competitive advantage.
