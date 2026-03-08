# Dynamis Engine Review Checklist

**Practical Review Rules for Performance-Sensitive Engine Work**

Use this checklist for pull requests and design reviews that affect:

- Vectrix
- MeshForge
- DynamisGPU
- DynamisLightEngine
- future runtime-critical subsystems

This document is the practical companion to:

- [docs/dynamis-engine-doctrine.md](dynamis-engine-doctrine.md)
- [docs/dynamis-performance-constitution.md](dynamis-performance-constitution.md)

It is intended to catch performance drift, architectural leakage, and hidden runtime costs before they land.

---

## 1. Boundary Review

**Questions**

- Does this change preserve subsystem boundaries?
- Is responsibility still in the correct layer?
- Does this introduce cross-layer leakage?

**Rules**

- MeshForge prepares geometry
- DynamisGPU executes uploads and owns GPU transfer mechanics
- LightEngine orchestrates rendering and frame work
- Vectrix provides math / SIMD substrate

**Red flags**

- MeshForge touching Vulkan objects
- LightEngine doing geometry packing work
- GPU upload code absorbing renderer policy
- runtime systems depending on authoring-time structures

## 2. Cost Model Review

**Questions**

- What does this change cost in CPU time?
- What does it cost in memory?
- Does it add synchronization?
- Does it add copies?
- Is per-frame work increased?

**Required**

Every meaningful runtime change should be explainable in terms of:

- allocations
- copies
- queueing/backlog
- synchronization
- cache/locality effects
- frame-time or latency impact

**Red flags**

- "Should be cheap"
- "Probably negligible"
- "Cleaner this way" with no cost explanation

## 3. Allocation Review

**Questions**

- Does this add heap allocation in a hot path?
- Does this create temporary wrappers, collections, lambdas, or buffers?
- Can the memory be reused instead?

**Rules**

Hot paths should remain allocation-free.

Preferred strategies:

- persistent arenas
- object reuse
- pools
- pre-sized reusable arrays/buffers

**Red flags**

- new allocations inside loops
- per-frame list/map creation
- repeated boxing/unboxing
- fresh direct buffers in upload paths
- convenience allocations in math-heavy code

## 4. Data Layout Review

**Questions**

- Does this improve or worsen locality?
- Is the runtime data layout still bulk-friendly?
- Are we preserving SoA / packed structures where intended?

**Rules**

Prefer:

- contiguous memory
- stable iteration order
- packed representations
- batch-friendly traversal

**Red flags**

- replacing arrays with object graphs
- splitting tightly related data across many objects
- adding abstraction layers that obscure memory layout
- pointer-chasing in hot loops

## 5. Copying Review

**Questions**

- Does this add a new data copy?
- Why does the copy exist?
- Can one copy be removed or fused?

**Rules**

Copies must be intentional and justified.

Valid reasons include:

- representation transformation
- runtime packing
- device upload
- lifetime isolation
- simplified synchronization

**Red flags**

- duplicate staging of the same content
- repeated marshaling between internal formats
- "copy for convenience"
- hidden array duplication

## 6. Scheduling Review

**Questions**

- Does this alter queueing, backlog, or inflight policy?
- Is work bounded?
- Does this affect throughput, latency, or pressure?

**Rules**

Scheduling is a first-class design surface.

Established precedent:

- pull-based upload scheduling is preferred
- target inflight depth for GPU upload is 2

**Red flags**

- unbounded queues
- push-driven work accumulation where pull control was intentional
- concurrency increase without measurement
- fairness or responsiveness regressions
- "more threads should help"

## 7. Telemetry Review

**Questions**

- Can this behaviour be observed at runtime?
- What telemetry proves this subsystem is healthy?
- Are key counters exposed?

**Required where relevant**

Examples:

- backlog depth
- inflight count
- bytes transferred
- completion latency
- frame timing
- memory pool usage
- draw/dispatch counts
- culling counts

**Red flags**

- runtime-critical logic with no counters
- inability to inspect queue pressure
- no way to validate success/regression in production-like runs

## 8. Benchmark / Profiling Review

**Questions**

- Was this measured?
- What workload was used?
- Was before/after comparison done?
- Did the hotspot actually move?

**Rules**

Accepted evidence:

- benchmark runs
- JFR
- controlled scenario comparison
- latency distribution comparison
- throughput measurements

**Red flags**

- change merged on intuition alone
- no baseline
- benchmark only for best case
- quoting average only while hiding p95/p99

## 9. Bounded Work Review

**Questions**

- Is the work introduced per-frame bounded?
- Can the system fall behind uncontrollably?
- Is there backpressure?

**Rules**

Per-frame and frame-adjacent work must be capped or controllable.

Examples:

- upload caps
- bounded streaming work
- bounded async processing
- capped background rebuilds

**Red flags**

- catch-up loops
- drain-until-empty frame logic
- uncontrolled producer/consumer imbalance
- "process everything available now"

## 10. API Surface Review

**Questions**

- Is the API minimal?
- Does it expose policy too early?
- Does it force future complexity into the present design?

**Rules**

Start with the smallest boundary that supports the measured need.

Example: UploadManager should currently provide:

- backlog control
- inflight control
- telemetry
- completion tracking

It should not yet absorb:

- residency policy
- streaming strategy
- renderer integration policy
- complex prioritization frameworks

**Red flags**

- speculative extensibility
- generic abstractions without a real second use case
- policy baked into low-level plumbing
- interfaces designed for imagined future systems

## 11. Runtime Representation Review

**Questions**

- Is runtime operating on runtime-ready data?
- Are authoring structures still leaking into live systems?
- Is the frame loop paying preparation cost?

**Rules**

The runtime must consume compact, execution-ready structures.

**Red flags**

- semantic/editor-friendly structures used directly in hot paths
- repeated runtime derivation of already-known data
- parsing, validation, or reshaping on the frame path

## 12. JVM Strategy Review

**Questions**

- Are modern JVM features being used appropriately?
- Is someone proposing native escape too early?
- Is the real bottleneck actually in Java/JVM?

**Rules**

Prefer solving performance problems with:

- better algorithms
- better layout
- reduced allocation
- Vector API
- VarHandles
- better batching
- better scheduling

Do not jump to JNI/native rewrites without proof.

**Red flags**

- "Java is the problem" without evidence
- replacing LWJGL prematurely
- native rewrite proposals before profiling

## 13. Stability Review

**Questions**

- Does this improve peak numbers but hurt predictability?
- What happens to p95/p99?
- Does this create bursts, stalls, or queue spikes?

**Rules**

Stable behaviour is more important than isolated peaks.

Prefer:

- bounded backlog
- predictable completion
- controlled memory pressure
- stable latency distributions

**Red flags**

- benchmark win with worse tail latency
- occasional great throughput but unstable completion
- memory pressure spikes
- large inflight bursts with no control

## 14. Documentation Review

**Questions**

- Does this change update the relevant design docs?
- Are new invariants or precedents recorded?
- Will a future session understand why this was done?

**Required when relevant**

Update:

- doctrine docs
- performance constitution
- subsystem plan docs
- experiment results docs
- comments where invariants matter

**Red flags**

- important measurement result not recorded
- new architectural rule exists only in chat or memory
- code changes that silently alter subsystem policy

## 15. Exception Review

**Questions**

- Does this intentionally break a doctrine/constitution rule?
- Is that exception explicit?
- Is the tradeoff documented?

**Required for exceptions**

State clearly:

- which rule is being bent or broken
- why
- what evidence supports it
- whether it is temporary or permanent
- what guardrails limit risk

**Red flags**

- "just this once" with no note
- silent degradation of a hard rule
- temporary workaround with no exit condition

---

## Review Decision Template

For performance-sensitive changes, reviewers should be able to summarize the change like this:

### Change Summary

- **What changed:**
- **Why it changed:**
- **Expected benefit:**
- **Cost added:**
- **Evidence provided:**

### Review Outcome

- Boundaries preserved: yes/no
- Hot-path allocations introduced: yes/no
- Data locality improved/preserved/worsened
- Copies added/removed:
- Scheduling impact:
- Telemetry present: yes/no
- Measured: yes/no
- Docs updated: yes/no

### Final Decision

- Approve
- Approve with follow-up
- Needs measurement
- Needs redesign

---

## Fast PR Checklist

For quick reviews, ask:

- [ ] Does this violate subsystem boundaries?
- [ ] Does this add hot-path allocation?
- [ ] Does this worsen locality?
- [ ] Does this add copies?
- [ ] Does this add unpredictable synchronization?
- [ ] Is work still bounded?
- [ ] Is telemetry present?
- [ ] Was it measured?
- [ ] Are docs updated?

If any answer is unclear, stop and inspect further.

---

## Current Locked Precedents

Until new measurements contradict them, reviewers should treat these as established:

- pull-based upload scheduling is preferred
- GPU upload target inflight depth is 2
- persistent staging memory is justified
- Java-side upload micro-optimization has diminishing returns
- LWJGL replacement/patching is not currently justified
- subsystem boundaries between MeshForge, DynamisGPU, and LightEngine are intentional

If a change conflicts with one of these, the review should explicitly state:

> This contradicts a previously measured decision and should be validated before implementation.

---

## Closing Rule

The goal of review is not merely correctness.

The goal is to preserve:

- performance clarity
- subsystem discipline
- bounded runtime behaviour
- long-term engine integrity

A change that works but obscures cost is incomplete.
