# Graphics Cluster Tightening Plan — v2

## Purpose

Tighten the graphics cluster boundaries across:

- `DynamisLightEngine`
- `DynamisGPU`
- `DynamisVFX`
- `DynamisSky`
- `DynamisTerrain`

using **small, non-breaking slices** that stop further architectural drift without triggering a broad rewrite.

This plan assumes the following strict ownership decision is now in force:

- **DynamisLightEngine** is the sole owner of global render planning, phase ordering, and host-facing render orchestration.
- **DynamisGPU** is the sole owner of direct GPU/backend access, GPU resource lifecycle, upload/orchestration, and execution primitives.
- **Feature repos** (`DynamisVFX`, `DynamisSky`, `DynamisTerrain`) own only feature-local runtime state, feature-local simulation, feature-local render-data generation, and declarative feature requirements.

## Canonical Tightening Slices (Completed)

### A1 — Graphics footholds

Commits:

- Terrain: `8cb5f2b`
- Sky: `503234e`
- LightEngine: `9f3885d`

Purpose:

- Introduced minimal LightEngine phase contract.
- Added typed Terrain ↔ Sky seam.
- Added internal GPU adapter seams in Terrain and Sky.

### A2 — Adapter seam follow-through

Commits:

- Terrain: `81f7d60`
- Sky: `96218aa`

Purpose:

- Routed first real backend paths through GPU adapter seams.
- Began inversion of Terrain ↔ Sky typed seam.

### A3 / B1 — Raw-handle containment

Commits:

- Terrain: `1714558`
- Sky: `58cfc9b`
- VFX: `a40d026`

Purpose:

- Introduced typed replacements for raw GPU handles.
- Preserved compatibility paths.

### D1 — Backend exposure reduction

Commits:

- Terrain: `fc3eeb1`
- Sky: `3bae8f4`

Purpose:

- Reduced JPMS exports in feature Vulkan modules.

### Integration Seam Tightening (LightEngine ↔ Sky)

Canonical seam-only commits:

- Sky seam: `200f5afb9937cfbf9e61fa8b1f8c3aed4c4ad3b1`
- LightEngine seam: `73f8ecd9ee2fd124b5a479496a4c1f3111259c4f`

Separate unrelated follow-up:

- `3de5a81ce4886ecae9196ce53b96a0987b74d5fd`

Purpose:

- Removed LightEngine reflection into Sky LUT internals.
- Narrowed LightEngine dependency to `VulkanSkyIntegration`.
- Kept unrelated runtime/doc changes out of seam-only commits.

Future slices must preserve compatibility and remain narrow.
Do not combine unrelated architectural tightening work into a single commit.

## Core Architectural Rule

## DynamisGPU owns

- backend API access
- GPU resource allocation/lifetime
- uploads/staging
- descriptor/binding setup
- pipeline/backend execution details
- command recording/submission
- synchronization
- device-address/handle management

## DynamisLightEngine owns

- render planning
- global phase/pass ordering
- frame orchestration
- host-facing rendering contracts
- composition of feature-subsystem render participation

## Feature repos own

- feature-local state
- feature-local simulation
- feature-local data generation
- typed feature requests / render-data outputs
- declarative phase/resource requirements

## Feature repos must not own

- direct Vulkan/backend execution
- GPU resource lifecycle/orchestration
- global render planning
- global pass ordering

## Target Seam Model

## 1. LightEngine → DynamisGPU

Allowed:

- API-level execution/service contracts
- typed requests
- typed resource/result contracts

Forbidden:

- direct `dynamis-gpu-vulkan` usage in planning/orchestration code
- Vulkan handles in planning contracts
- backend-specific execution logic in LightEngine core APIs

## 2. Feature repo → DynamisGPU

Allowed:

- typed feature execution requests
- typed resource/result contracts
- internal adapter seams that shield backend usage

Forbidden:

- stable public APIs exposing raw backend handles
- direct backend helper usage as part of public feature contracts
- feature-owned GPU resource lifecycle logic as a stable cross-repo dependency

## 3. LightEngine → Feature repo

Allowed:

- typed feature service/update hooks
- typed feature output consumption
- declarative phase/resource requirements

Forbidden:

- feature-owned global phase ordering
- LightEngine absorbing feature-local simulation/state
- LightEngine depending on feature backend internals

## 4. Feature repo → LightEngine

Allowed:

- feature-local render-data outputs
- declarative phase requirements
- feature metadata needed for orchestration

Forbidden:

- feature repos imposing global render pass order
- feature repos owning host-wide render policy
- feature repos encoding global phase sequencing assumptions as stable contracts

## Non-Goals

This tightening plan does **not** do the following:

- no broad rewrite of the graphics cluster
- no migration of feature-local logic into DynamisGPU
- no creation of a large new generic graphics abstraction framework
- no public API deletions in the first wave
- no consolidation of repos
- no attempt to solve all feature/backend leakage in one pass

## Problem Categories

## A. Dependency / Authority Problems

1. direct coupling to `dynamis-gpu-vulkan` internals
2. feature-side phase-order assumptions leaking render policy out of LightEngine
3. LightEngine overlapping with DynamisGPU or MeshForge concerns

## B. API Surface Problems

1. raw backend handle leakage into public feature APIs
2. weakly typed feature seams
3. over-broad backend package exports

These should be treated as related but distinct problems.

## Phase Structure

## Phase A — Boundary Declaration and Seam Shielding

### Purpose

Define the seam model clearly and stop further backend leakage without changing feature behavior.

### What it changes

- A1: minimal LightEngine phase contract (declarative authority declaration)
- A2: internal-only GPU adapter seams in feature repos (shielding layers)
- A3: typed Terrain↔Sky seam replacing weak `Object` coupling

### What it does not change

- no broad behavior rewrites
- no public API deletions
- no feature logic migration into DynamisGPU

### Expected outcome

- global phase authority is explicit in LightEngine
- no new direct backend leakage outside internal feature adapter seams
- Terrain/Sky seam is typed and explicit

### Phase A1 — Minimal LightEngine phase contract

Add a minimal explicit render-phase contract owned by `DynamisLightEngine`.

Scope:

- phase identifiers / phase model
- declarative participation requirements
- no large runtime behavior rewrite

Success condition:

- LightEngine becomes the explicit documented owner of global render phase authority.

### Phase A2 — Internal GPU adapter seams in feature repos

In `DynamisVFX`, `DynamisSky`, and `DynamisTerrain`, add **internal-only** adapter seams for GPU execution/backend interaction.

Important:

- these are anti-corruption layers
- they are **not** new public SPIs
- they shield the feature code from direct backend leakage

Success condition:

- no new direct `dynamis-gpu-vulkan` usage appears outside these internal adapter seams.

### Phase A3 — Typed Terrain ↔ Sky seam

Replace the weak Terrain↔Sky `Object` seam with a typed contract.

Reason:

- narrow
- high-signal
- low-risk
- ideal exemplar for the broader tightening style

Success condition:

- Terrain/Sky coupling becomes typed and explicit without behavior change.

## Phase B — Public API Hygiene

### Purpose

Stop freezing bad boundaries into public APIs.

### What it changes

- B1: typed replacements for raw-handle public API paths
- B2: typed replacements for weakly typed feature seams

### What it does not change

- no first-wave public API deletions
- no backend implementation rewrites
- no new giant framework layer

### Expected outcome

- stable feature APIs become backend-agnostic and typed
- legacy raw-handle paths remain only as compatibility bridges

### Phase B1 — Raw handle containment

For VFX, Sky, and Terrain:

- introduce typed replacements for raw GPU handles in public APIs
- keep compatibility paths temporarily

Critical rule:

> once typed replacements exist, **no new or modified code** may use the old raw-handle public paths.

### Phase B2 — Weak typing cleanup

Replace weakly typed public feature seams with typed contracts where justified.

Success condition:

- public feature APIs stop leaking backend assumptions and ambiguous coupling.

## Phase C — LightEngine Authority Reassertion

### Purpose

Ensure LightEngine is the only global render-planning authority in practice.

### What it changes

- C1: feature participation becomes declarative (requirements, not global order)
- C2: LightEngine explicitly resolves cross-feature composition/ordering

### What it does not change

- no LightEngine takeover of feature-local simulation/state
- no migration of feature-local render-data generation into LightEngine

### Expected outcome

- feature repos no longer act as mini render planners
- global pass-order policy is centralized in LightEngine

### Phase C1 — Declarative feature phase participation

Feature repos may declare:

- needed phase(s)
- required resources
- local ordering constraints

Feature repos may not:

- own host/global pass order
- encode engine-wide render phase policy

### Phase C2 — LightEngine resolves composition

LightEngine becomes the single point that resolves:

- cross-feature ordering
- phase placement
- orchestration/composition policy

Success condition:

- feature repos no longer act like mini render planners.

## Phase D — Backend Exposure Reduction

### Purpose

Reduce technical debt only after replacement seams are proven.

### What it changes

- D1: narrow backend package exports
- D2: route residual backend/internal reach through shielding seams

### What it does not change

- no speculative broad abstraction rewrite
- no immediate deletion of still-needed compatibility paths

### Expected outcome

- backend details become implementation details instead of cluster-wide assumptions

### Phase D1 — Narrow backend package exports

Reduce public exports in:

- `dynamisvfx-vulkan`
- `dynamissky-vulkan`
- `dynamisterrain-vulkan`

### Phase D2 — Shrink direct GPU-internal reach

Route remaining backend/internal calls through the internal seams introduced earlier.

Success condition:

- backend details become implementation details rather than cluster-wide assumptions.

## Phase E — Convergence and Cleanup

### Purpose

Finish the tightening in a controlled way.

### What it changes

- E1: compatibility-path review/deprecation hardening
- E2: cluster-level contract tests and guardrails

### What it does not change

- no broad cross-cluster rewrite
- no out-of-scope architecture reorganization

### Expected outcome

- tightened boundaries are enforceable in CI and review process
- legacy paths have a controlled retirement plan

### Phase E1 — Compatibility path review

Review legacy raw-handle paths and old seams to decide what can be more strongly deprecated.

### Phase E2 — Cluster contract tests

Add/strengthen tests ensuring:

- no new direct backend leakage outside approved seams
- LightEngine remains sole phase authority
- typed seams stay stable

Success condition:

- the tightened architecture is enforceable, not merely documented.

## Immediate Priorities

## Priority 1

**LightEngine-owned phase contract**

- minimal
- explicit
- no behavior rewrite

## Priority 2

**Typed Terrain ↔ Sky seam**

- best first exemplar
- low risk
- high architectural value

## Priority 3

**Internal GPU adapter seams in feature repos**

- shielding only
- internal only
- blocks further leakage

## Implementation Order

Recommended repo order:

1. **DynamisLightEngine**
   - define minimal phase contract only
2. **DynamisTerrain**
   - typed Terrain↔Sky seam
   - internal GPU adapter seam stub
3. **DynamisSky**
   - align to typed seam
   - internal GPU adapter seam stub
4. **DynamisVFX**
   - same shielding pattern after seam model is proven
5. **DynamisGPU**
   - adapt/support stabilized upstream seam shapes as needed

### Why this order

- LightEngine must declare global phase authority first
- Terrain is the safest first feature seam to clean up
- Sky follows naturally because of the Terrain↔Sky coupling
- VFX is likely the messiest feature repo, so tackle it after the seam style is proven
- DynamisGPU is the stable execution authority and should adapt to clarified upstream seam shapes rather than being forced first into speculative abstraction changes

## Rules for Phase A Implementation

1. **No behavior rewrites**
2. **No public API deletions**
3. **No new public SPIs unless explicitly justified**
4. **Internal GPU adapter seams are shielding layers first, not frozen new contracts**
5. **Any typed replacement added must become the preferred path for all new/modified code**
6. **LightEngine may define phase contracts, but must not absorb feature-local simulation/state**
7. **DynamisGPU remains the only owner of backend execution/resource logic**

## Explicit Anti-Goals for LightEngine

Because “LightEngine is sole render-planning authority” can be misread, make this explicit:

LightEngine should **not** absorb:

- feature-local simulation/state
- feature-local render-data generation
- backend-specific GPU setup
- asset shaping / geometry preparation
- scene ownership / world ownership

LightEngine should coordinate, not centralize everything.

## First Implementation Slice

## Recommended Slice A1

### Scope

- minimal LightEngine phase contract
- typed Terrain↔Sky seam
- internal GPU adapter seam stubs in Terrain and Sky

### Deliverables

- explicit phase contract in LightEngine
- typed cross-feature Terrain↔Sky seam
- internal backend-shielding seams in Terrain and Sky
- no meaningful runtime behavior change
- no public API deletions

### Why this slice first

- establishes architecture without large risk
- proves the tightening style on a narrow seam
- avoids opening the entire VFX/GPU problem on day one
- prevents further drift while remaining non-breaking

## Success Criteria for Phase A

After Phase A, the following should be true:

- LightEngine has an explicit phase contract
- Terrain no longer uses a weakly typed Sky seam
- Terrain and Sky have internal GPU adapter boundaries
- no new backend-internal usage is added outside those seams
- feature behavior is materially unchanged

## One-Line Guiding Principle

> **Move backend execution authority downward into DynamisGPU, keep feature-local logic inside feature repos, and keep global render planning solely inside LightEngine.**

## Completed Integration Tightening (Canonicalized)

### LightEngine ↔ Sky seam tightening

Canonical seam-only commits:

- Sky seam: `200f5afb9937cfbf9e61fa8b1f8c3aed4c4ad3b1`
- LightEngine seam: `73f8ecd9ee2fd124b5a479496a4c1f3111259c4f`

Separate unrelated follow-up:

- `3de5a81ce4886ecae9196ce53b96a0987b74d5fd`

Seam files:

- `VulkanSkyIntegration.java`
- `VulkanSkyRuntimeBridge.java`

Result:

- LightEngine bridge narrowed to Sky integration facade
- direct LUT reflection calls removed from LightEngine seam
- unrelated runtime/doc changes split out of seam commits

### Untracked / non-canonical leftovers

These are outside the canonical seam slice and are not part of the tightening work unless intentionally handled later:

- `DynamisSky/docs/arch_notes.md`
- `DynamisSky/dynamissky-bench/dependency-reduced-pom.xml`
