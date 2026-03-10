# Graphics Cluster Tightening Plan (Non-Breaking)

Date: 2026-03-10
Scope: `DynamisLightEngine`, `DynamisGPU`, `DynamisVFX`, `DynamisSky`, `DynamisTerrain`
Input baseline: `docs/graphics-cluster-architecture-synthesis.md`

This plan defines staged, non-breaking boundary tightening slices. It is implementation-planning only.

## 1. Target Seam Model

## 1.1 LightEngine -> GPU seam
Allowed surface:
- stable execution-facing interfaces from `dynamis-gpu-api`
- resource/capability contracts that are backend-agnostic

Not allowed as stable contract:
- direct `dynamis-gpu-vulkan` internals in planning-layer contracts
- backend utility types in host-facing LightEngine APIs

## 1.2 Feature -> GPU seam (VFX/Sky/Terrain)
Allowed surface:
- typed feature execution inputs/outputs mapped to API-level GPU seams
- feature-local backend adapters behind internal module boundaries

Not allowed as stable contract:
- raw Vulkan handles in public feature APIs
- feature API dependence on `dynamis-gpu-vulkan` internals

## 1.3 LightEngine -> Feature seam
Allowed surface:
- typed feature service/update/record contracts
- declarative feature phase capability contracts

Not allowed:
- LightEngine depending on feature backend internals
- feature contracts exposing backend handles as required inputs

## 1.4 Feature -> LightEngine seam
Allowed surface:
- feature-local data generation and explicit phase requirements
- optional callbacks/capabilities indicating required pass slots

Not allowed:
- feature repos owning global pass ordering policy
- feature repos encoding hard global frame ordering assumptions as authority

## 1.5 Cross-cutting seam rules
- Public APIs: typed, backend-agnostic, minimal.
- Backend specifics: internal implementation only.
- Render-planning authority: `DynamisLightEngine` only.
- GPU execution/resource authority: `DynamisGPU` only.

## 2. Prioritized Tightening Work Items

## 2.1 Problem: direct `dynamis-gpu-vulkan` internal coupling
Where:
- LightEngine backend
- VFX Vulkan backend
- Sky Vulkan backend
- Terrain Vulkan backend

Tightened end state:
- planning and feature public seams use API-level contracts
- backend-internal dependencies are isolated to implementation modules

Minimal non-breaking first slice:
- add explicit “GPU adapter boundary” interfaces per repo (internal package) and route new call sites through adapter facades while preserving existing code paths.

## 2.2 Problem: raw GPU handle leakage in public feature APIs
Where:
- VFX frame/draw context
- Sky/Terrain frame/resource contracts

Tightened end state:
- public feature APIs expose typed handles/resources or opaque tokens
- raw backend handles restricted to backend/internal adapters

Minimal non-breaking first slice:
- introduce parallel typed API fields/methods (additive), keep existing raw-handle methods deprecated but functional.

## 2.3 Problem: backend package over-export
Where:
- Sky Vulkan module exports broad backend packages
- Terrain Vulkan module exports broad backend packages

Tightened end state:
- only intentional integration packages exported
- pass/lut/material/renderer internals remain internal

Minimal non-breaking first slice:
- classify exports into required/public vs internal/deprecate; add usage audit and deprecation notice before export reduction.

## 2.4 Problem: weakly typed feature seams
Where:
- Terrain sky source seam (`Object`)

Tightened end state:
- typed feature integration contracts for Sky->Terrain and similar crossings

Minimal non-breaking first slice:
- add typed overload/interface and route internal use to typed seam; keep old method as compatibility wrapper.

## 2.5 Problem: feature-side phase assumptions leaking render policy
Where:
- VFX/Sky/Terrain integration docs and runtime assertions

Tightened end state:
- LightEngine owns canonical phase ordering contract
- features declare phase requirements, not global order rules

Minimal non-breaking first slice:
- define shared phase contract spec in LightEngine API/SPI and map existing feature assertions to that spec.

## 3. Staged Non-Breaking Phases

## Phase A — Seam Definition & Shielding
Goal:
- define canonical seam contracts and adapter boundaries without behavior change.

Affected repos:
- LightEngine, DynamisGPU, VFX, Sky, Terrain

Benefit:
- stops further drift while code remains stable.

Risk:
- Low

Order dependencies:
- Must precede deeper API cleanup and export reduction.

## Phase B — Feature API Cleanup (Typed/Public)
Goal:
- reduce raw-handle leakage and weak typing in feature public APIs.

Affected repos:
- VFX, Sky, Terrain (with LightEngine integration touchpoints)

Benefit:
- stronger API boundaries and backend independence.

Risk:
- Medium (compatibility management needed)

Order dependencies:
- After Phase A contracts exist.

## Phase C — Backend Exposure Reduction
Goal:
- reduce over-exported backend packages and isolate implementation-only surfaces.

Affected repos:
- Sky Vulkan, Terrain Vulkan, VFX Vulkan (as needed)

Benefit:
- prevents accidental external coupling.

Risk:
- Medium (may break internal consumers if audit is incomplete)

Order dependencies:
- After Phase B migration paths are in place.

## Phase D — LightEngine Authority Reassertion
Goal:
- make LightEngine the single source of render phase ordering and feature orchestration policy.

Affected repos:
- LightEngine (primary), VFX/Sky/Terrain adapters

Benefit:
- removes distributed pass-policy ownership.

Risk:
- Medium

Order dependencies:
- Can begin late Phase B; complete after Phase C.

## Phase E — Cluster Cleanup & Validation
Goal:
- remove deprecated shims when safe; verify boundaries with tests/docs.

Affected repos:
- all five

Benefit:
- boundary debt reduction without broad rewrite.

Risk:
- Medium

Order dependencies:
- last phase.

## 4. Immediate vs Deferred

## 4.1 Immediate tightening targets
1. Define and ratify seam contracts (Phase A).
2. Add typed replacement seams for raw-handle and weakly typed APIs (Phase B start).
3. Publish LightEngine-owned phase contract and align feature integrations (Phase D start).

## 4.2 Near-term follow-up
4. Backend export reduction after usage audit and migration (Phase C).
5. Adapter hardening to minimize direct `dynamis-gpu-vulkan` references in non-backend packages.

## 4.3 Watch/defer
6. Full replacement of all backend handle usage in implementation internals.
7. Deep geometry-shaping overlap cleanup between LightEngine and MeshForge (execute after cluster seams stabilize).
8. Broader non-graphics repo boundary work.

## 5. Recommended Repo Execution Order

Recommended order for implementation work:
1. `DynamisLightEngine` — define canonical phase/orchestration contract and integration seam expectations.
2. `DynamisVFX` — implement typed API shielding + phase contract adoption.
3. `DynamisSky` — implement typed API shielding + export classification.
4. `DynamisTerrain` — implement typed sky seam replacement + API shielding + export classification.
5. `DynamisGPU` — finalize narrowed adapter seam support and remove residual cross-repo internal leakage where practical.

Rationale:
- start with authority contract (`LightEngine`),
- migrate feature repos to compliant typed seams,
- then tighten GPU coupling and exposed backend surfaces.

## 6. Recommended First Implementation Slice

First concrete slice (smallest high-value step):
- **Phase A1: Cluster seam contract package + docs + adapter stubs**

Deliver in one non-breaking pass:
- LightEngine publishes explicit feature phase contract (declarative ordering model).
- VFX/Sky/Terrain add internal GPU adapter boundary interfaces and mark raw-handle API points as legacy entry points.
- Terrain adds typed sky-source seam alongside existing `Object` seam.

This yields immediate drift control with minimal behavioral risk and sets up all later slices.

## 7. Plan Guardrails

- No broad rewrites.
- No forced immediate breaking API changes.
- Each phase must have explicit compatibility strategy and rollback path.
- Validate each slice with repo-local parity/contract tests before moving to next phase.
