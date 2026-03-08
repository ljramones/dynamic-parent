# Dynamis Engine Doctrine

**Architectural Principles for the Dynamis Engine Ecosystem**

The Dynamis engine ecosystem is a high-performance Java engine architecture designed for modern GPU-driven rendering and large-scale simulation.

**Primary repositories:**

- **Vectrix** – SIMD math and compute substrate
- **MeshForge** – geometry processing and packing
- **DynamisGPU** – GPU resources, transfers, synchronization
- **DynamisLightEngine** – renderer orchestration and frame planning

The doctrine below defines non-negotiable architectural principles intended to maintain performance, predictability, and subsystem clarity as the engine evolves.

---

## 1. Evidence-Driven Performance

Performance decisions must be supported by:

- benchmarks
- profiling (JFR)
- hardware measurements

Avoid speculative optimization.

When measurements contradict assumptions, measurements win.

## 2. Deterministic Hot Paths

Engine hot paths must prioritize:

- deterministic execution time
- predictable memory access
- minimal allocation

Hot loops should strive for:

- 0 allocations
- contiguous memory access
- SIMD-friendly layouts

## 3. Data-Oriented Architecture

The engine favors data-oriented design over object-oriented design in performance-critical systems.

Preferred characteristics:

- Structure of Arrays (SoA)
- cache-friendly layouts
- bulk processing
- predictable iteration

Avoid deep object graphs in hot paths.

## 4. Explicit Representation Boundaries

Subsystems must maintain clear boundaries between authoring data and runtime data.

Example from MeshForge:

```
MeshData (mutable, semantic, debug-friendly)
        ↓
PackedMesh (immutable, runtime optimized)
```

The runtime must never operate directly on authoring structures.

## 5. Geometry Pipeline Doctrine

Geometry flows through the following pipeline:

```
Authoring Geometry
      ↓
MeshForge Processing
      ↓
Packed Geometry
      ↓
GpuGeometryUploadPlan
      ↓
DynamisGPU Upload
      ↓
GPU-resident buffers
```

Rules:

- MeshForge does not interact with Vulkan
- DynamisGPU does not manipulate geometry
- LightEngine orchestrates rendering

## 6. Persistent Memory Strategies

Transient allocation in hot paths is prohibited.

Preferred strategies:

- persistent arenas
- pooled buffers
- reusable staging memory

Example: The DynamisGPU upload pipeline uses a persistent staging arena to eliminate direct-buffer churn.

## 7. GPU-Driven Rendering Philosophy

The engine is designed to support GPU-driven rendering.

This implies:

- large batch submission
- GPU-visible geometry structures
- minimal CPU draw-call overhead
- indirect draw pipelines

Future systems should assume GPU-driven pipelines as the primary model.

## 8. Scheduling and Concurrency

Subsystem concurrency should prefer predictable scheduling over maximal parallelism.

Example: Measured results in DynamisGPU show optimal upload overlap at inflight depth = 2. Increasing concurrency beyond this point increases latency without improving throughput.

Concurrency decisions must be measurement-driven.

## 9. Minimal Early Abstraction

New subsystems should begin with minimal architecture.

Avoid:

- premature generalization
- speculative streaming systems
- complex policy layers

Subsystems should evolve only after real workloads justify expansion.

## 10. Clear Subsystem Responsibilities

Subsystems have strict responsibilities.

| Subsystem | Responsibility |
|-----------|---------------|
| Vectrix | math, SIMD compute |
| MeshForge | geometry preparation |
| DynamisGPU | GPU resources and transfers |
| LightEngine | renderer orchestration |

Cross-layer coupling is discouraged.

## 11. Allocation-Free Math Layer

Vectrix provides the compute substrate for:

- geometry operations
- transforms
- physics interaction
- SIMD acceleration

Math operations in hot paths should be allocation-free. Temporary object creation in math loops is prohibited.

## 12. Profiling Before Optimization

Optimization should follow this workflow:

```
benchmark → profile → identify hotspot → target fix → re-measure
```

If a hotspot disappears after measurement, optimization stops.

## 13. JVM-Native Performance

The engine intentionally leverages modern JVM capabilities.

Key technologies:

- JDK 25
- Vector API
- VarHandles
- Loom (virtual threads)
- JFR profiling

Where possible, performance improvements should use modern JVM primitives rather than JNI or custom native code.

## 14. Platform-Neutral Rendering Core

The engine targets:

- macOS (MoltenVK)
- Linux
- Windows

Platform-specific work must remain isolated. Rendering architecture must remain Vulkan-centric.

## 15. Measured System Boundaries

The Dynamis architecture evolves through measured subsystem passes.

Example workflow:

```
Subsystem baseline → benchmark → profiling → targeted improvements → stabilization
```

Once stabilized, the subsystem becomes a performance baseline for future systems.

## 16. Simplicity Over Cleverness

The engine favors:

- simple, measurable systems
- clear data flow
- explicit responsibilities

Avoid clever designs that obscure performance characteristics.

## 17. Engine Evolution Model

The engine evolves in phases:

```
baseline implementation → performance stabilization → feature expansion → architectural integration
```

Major subsystems should reach a stable performance baseline before expanding features.

---

## Conclusion

The Dynamis engine architecture prioritizes:

- deterministic performance
- data-oriented design
- GPU-driven rendering
- clear subsystem boundaries

These principles ensure the engine remains scalable, maintainable, and high-performance as it evolves.
