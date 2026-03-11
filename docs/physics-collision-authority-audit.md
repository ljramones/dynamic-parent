# Physics ↔ Collision Authority Audit

Date: 2026-03-10
Scope: Focused authority-seam audit only (no refactor)

## Current Authority Map

### Physics (intended simulation authority)

`PhysicsWorld` in `DynamisPhysics` exposes simulation ownership surface:

- stepping and time control (`step`, pause/resume)
- body mutation (`applyImpulse`, `applyForce`, `applyTorque`, `setVelocity`, `teleport`)
- constraint lifecycle
- character/vehicle/ragdoll runtime lifecycle
- simulation queries and event drain

Reference: `DynamisPhysics/dynamisphysics-api/src/main/java/org/dynamisphysics/api/world/PhysicsWorld.java:26-85`

### Collision (currently mixed substrate + simulation authority)

`DynamisCollision` includes clean substrate areas (bounds/shapes/broadphase/narrowphase/contact generation), but currently also owns simulation-style runtime concerns.

## Exact Seam Pressure Points

### 1. Collision world stepping and integration lives in Collision

`CollisionWorld3D.step(...)` performs:

- gravity velocity integration
- position prediction
- constraint iteration
- contact response application

Reference: `DynamisCollision/collision_detection/src/main/java/org/dynamiscollision/world/CollisionWorld3D.java:178-227`

### 2. Contact solver/resolution authority lives in Collision

`ContactSolver3D` performs:

- position correction
- restitution/friction impulse solve
- warm-start handling
- direct velocity/position writes via `RigidBodyAdapter3D`

Reference: `DynamisCollision/collision_detection/src/main/java/org/dynamiscollision/contact/ContactSolver3D.java:30-209`

### 3. Fixed timestep utility is in Collision world package

`PhysicsStep3D` provides deterministic fixed-step accumulation utility under collision world runtime namespace.

Reference: `DynamisCollision/collision_detection/src/main/java/org/dynamiscollision/world/PhysicsStep3D.java:22-83`

### 4. Collision public surface currently exports simulation-oriented packages

`org.dynamiscollision` exports `world`, `contact`, and `constraints` as public module surface.

Reference: `DynamisCollision/collision_detection/src/main/java/module-info.java:11-16`

## Responsibility Classification

### Collision-query substrate (appropriate in Collision)

- shape/bounds primitives
- broadphase and narrowphase detection
- filtering and pair pipeline
- contact manifold generation utilities

Reference example: `ContactGenerator3D` in `DynamisCollision/collision_detection/src/main/java/org/dynamiscollision/contact/ContactGenerator3D.java:27-194`

### Simulation authority (should be Physics-owned)

- simulation stepping/integration loop
- constraint solving loop policy
- contact position/velocity resolution
- warm-start impulse lifecycle as simulation runtime behavior

Current location: `CollisionWorld3D`, `ContactSolver3D`, `PhysicsStep3D`

## Seam Verdict

Verdict: **needs boundary tightening (authority correction)**

Reason:

- Physics API clearly declares simulation authority.
- Collision still executes simulation/resolution logic and exposes it publicly.
- This is not just naming ambiguity; it is executable authority overlap.

## Recommended Next Narrow Slice

Execute a targeted **Physics ↔ Collision authority re-seam slice** (no broad rewrite):

1. Define an explicit seam contract: Collision returns detection/query/manifold results; Physics applies integration/resolution decisions.
2. Mark collision-side simulation constructs (`CollisionWorld3D.step`, `ContactSolver3D`, `PhysicsStep3D`) as transitional in docs/contracts.
3. Introduce Physics-owned adapter consumption path for one constrained contact-resolution flow, leaving compatibility paths intact.
4. Defer package moves/API deletions; keep first implementation slice additive and behavior-preserving.

## Recommendation

Proceed with a narrow implementation slice for seam correction before broader repo-by-repo reviews. This seam is now the highest-signal authority hotspot outside the graphics cluster.

## Canonical Tightening Slices

### P1 — Physics ↔ Collision authority foothold (completed)

Commits:

- `f001d6f` (`dynamis-parent`): added audit baseline
- `ef90c76` (`DynamisPhysics`): added Physics-owned collision contact seam
- `94ed32a` (`DynamisCollision`): marked collision-side simulation paths as transitional

Outcome:

- Physics now has an explicit contact-consumption/resolution seam (`DetectedCollisionContact`, `PhysicsContactResolver`, `PhysicsContactResolutionStrategy`).
- Collision overlap constructs are explicitly transitional (`CollisionWorld3D.step`, `ContactSolver3D`, `PhysicsStep3D`).
- Compatibility is preserved; no broad solver/runtime rewrite was performed.

### P2 — First legacy contact flow migration (completed)

Commit:

- `dc74e53` (`DynamisPhysics`): routed legacy `CollisionEvent<T>` flow through `PhysicsContactResolver`

Outcome:

- Added `CollisionEventContactAdapter` as a narrow transitional adapter from legacy collision events into `DetectedCollisionContact`.
- Added `PhysicsContactResolver.resolveFromCollisionEvents(...)` to consume one representative legacy flow through the Physics-owned seam.
- Preserved compatibility and behavior while shifting authority direction toward Physics-owned resolution path.

### P3 — Legacy solver-authority migration (completed)

Commit:

- `6b266cf` (`DynamisPhysics`): migrated normal-impulse velocity-resolution responsibility into Physics-owned strategy path

Before/After/Fallback:

- Before: `ContactSolver3D` owned normal-impulse velocity resolution behavior.
- After: Physics-owned `NormalImpulseContactResolutionStrategy` now owns that responsibility for the migrated seam path.
- Fallback: legacy `ContactSolver3D` flow remains present and compatibility-safe.

Outcome:

- Added `PhysicsContactBodyAdapter` and `NormalImpulseContactResolutionStrategy` under Physics-owned seam package.
- Added parity coverage comparing migrated strategy output with legacy `ContactSolver3D` for a representative head-on contact.
- Preserved behavior and kept migration scope narrow.

### P4 — Position-correction migration (completed)

Commit:

- `362d0d6` (`DynamisPhysics`): migrated position-correction responsibility into Physics-owned strategy path

Before/After/Fallback:

- Before: `ContactSolver3D` owned position-correction responsibility via `solvePosition(...)`.
- After: Physics-owned `PositionCorrectionContactResolutionStrategy` now owns position-correction on the migrated seam path.
- Fallback: legacy `ContactSolver3D` path remains present and compatibility-safe.

Outcome:

- Extended `PhysicsContactBodyAdapter` with position read/write operations needed for Physics-owned position-correction.
- Added `PositionCorrectionContactResolutionStrategy` under Physics-owned seam package.
- Added parity coverage comparing migrated strategy output with legacy `ContactSolver3D.solvePosition(...)` for a representative penetration case.

### P5 — Warm-start application policy migration (completed)

Commit:

- `9c109fa` (`DynamisPhysics`): migrated warm-start application policy into Physics-owned strategy path

Before/After/Fallback:

- Before: `ContactSolver3D.solveVelocity(..., WarmStartImpulse)` owned warm-start impulse application behavior.
- After: Physics-owned `WarmStartApplicationContactResolutionStrategy` now owns warm-start application on the migrated seam path.
- Fallback: legacy `ContactSolver3D` path remains present and compatibility-safe.

Outcome:

- Added Physics-owned `PhysicsWarmStartImpulse` value type.
- Added `WarmStartApplicationContactResolutionStrategy` under Physics-owned seam package.
- Added parity coverage comparing migrated strategy output with legacy `ContactSolver3D.solveVelocity(..., warmStart)` for a representative warm-start case.

### P6 — Step-path orchestration preference migration (completed)

Commit:

- `9618d20` (`DynamisPhysics`): added Physics-owned responder bridge preferring seam resolution before legacy collision fallback

Before/After/Fallback:

- Before: legacy collision-side responder/solver flow remained the implicit primary resolution path.
- After: Physics-owned `PhysicsPreferredCollisionResponder` provides a narrow orchestration bridge that prefers Physics seam strategies for resolvable events.
- Fallback: legacy `CollisionResponder3D` flow remains available and is invoked when seam conditions do not apply.

Outcome:

- Added `PhysicsPreferredCollisionResponder` as a Physics-owned step-path bridge over `CollisionEvent<T>`.
- Added focused behavior tests proving seam-first preference and fallback invocation behavior.
- Preserved compatibility and avoided broad step-loop rewrite.

### P7 — Warm-start persistence/cache policy migration (completed)

Commit:

- `cdcef10` (`DynamisPhysics`): migrated warm-start persistence/cache policy ownership into Physics-owned seam control

Before/After/Fallback:

- Before: warm-start persistence/invalidation behavior remained implicitly tied to collision-transitional cache flow.
- After: Physics-owned cache policy seam (`PhysicsWarmStartCachePolicy`) and map-backed policy (`MapBackedWarmStartCachePolicy`) now own warm-start load/store/invalidation policy for seam-based handling.
- Fallback: legacy collision-side warm-start cache path remains present and compatibility-safe.

Outcome:

- Added Physics-owned warm-start cache policy contract and default policy implementation.
- Added policy-backed warm-start application strategy (`PolicyBackedWarmStartApplicationStrategy`).
- Extended `PhysicsPreferredCollisionResponder` to notify Physics-owned cache policy per collision event, enabling explicit EXIT invalidation.
- Added focused behavior coverage proving seam-first warm-start policy use and cache invalidation fallback behavior.

### P8 — Warm-start persistence update preference migration (completed)

Commit:

- `ba79f93` (`DynamisPhysics`): moved post-resolution warm-start persistence update preference into Physics-owned orchestration bridge

Before/After/Fallback:

- Before: post-resolution warm-start persistence update behavior was not explicitly Physics-owned at the step-path bridge level.
- After: `PhysicsPreferredCollisionResponder` now owns seam-path post-resolution persistence update preference via `PhysicsWarmStartPersistenceStrategy`.
- Fallback: legacy collision-side cache update behavior remains available as compatibility fallback.

Outcome:

- Added Physics-owned `PhysicsWarmStartPersistenceStrategy` contract.
- Extended `PhysicsPreferredCollisionResponder` with seam-path persistence-update orchestration (`load before`, `update after`, `store preferred`).
- Kept policy-backed warm-start strategy focused on application behavior, with persistence update preference moved to orchestration boundary.
- Added focused behavior coverage proving seam-path load/store/update preference on resolvable events.

### P9 — Fixed-step responder ordering preference migration (completed)

Commit:

- `9151147` (`DynamisPhysics`): moved fixed-step seam strategy ordering preference into Physics-owned policy contract

Before/After/Fallback:

- Before: seam strategy execution order was implicit base-list order in `PhysicsPreferredCollisionResponder`.
- After: Physics-owned `PhysicsFixedStepResponderOrderingPolicy` now controls preferred strategy ordering for seam-resolvable events.
- Fallback: legacy collision ordering/flow remains explicit fallback when seam conditions do not apply.

Outcome:

- Added Physics-owned ordering policy contract (`PhysicsFixedStepResponderOrderingPolicy`).
- Extended `PhysicsPreferredCollisionResponder` to consume ordering policy and execute seam strategies in policy-defined order.
- Added focused behavior coverage proving policy-controlled execution ordering for one representative seam flow.

### P10 — Seam-vs-fallback selection policy migration (completed)

Commit:

- `fd39e68` (`DynamisPhysics`): moved seam-vs-fallback applicability decision from hardcoded predicate to Physics-owned policy contract

Before/After/Fallback:

- Before: `PhysicsPreferredCollisionResponder` used a hardcoded seam-applicability predicate.
- After: Physics-owned `PhysicsSeamSelectionPolicy` controls seam-vs-fallback selection for representative flows.
- Fallback: legacy collision fallback path remains explicit and compatibility-safe.

Outcome:

- Added Physics-owned selection policy contract (`PhysicsSeamSelectionPolicy`).
- Extended `PhysicsPreferredCollisionResponder` to delegate seam selection to policy.
- Added focused behavior coverage proving policy can force fallback even for otherwise seam-resolvable events.

### P11 — Collision-side response-path policy gate migration (completed)

Commit:

- `dfae62e` (`DynamisCollision`): made legacy `ContactSolver3D` special-case handling in `CollisionWorld3D.applyResponses(...)` policy-gated

Before/After/Fallback:

- Before: collision-transitional `applyResponses(...)` used an unconditional `ContactSolver3D` type special-case branch.
- After: `CollisionResponsePathPolicy3D` now gates legacy special-path usage, enabling explicit representative-flow preference of non-legacy responder handling.
- Fallback: legacy `ContactSolver3D` special-path remains default-compatible via default policy behavior.

Outcome:

- Added `CollisionResponsePathPolicy3D` policy contract.
- Extended `CollisionWorld3D` with configurable response-path policy (`setResponsePathPolicy(...)`).
- Added focused tests proving default legacy special-path behavior and policy-disabled representative-flow behavior.

### P12 — Representative preferred-flow integration wiring (completed)

Commits:

- `dfae62e` (`DynamisCollision`): collision-side response-path policy gate
- `cdda2dc` (`DynamisPhysics`): representative Physics-preferred integration-flow coverage

Before/After/Fallback:

- Before: policy components existed, but representative integration proof of composed preferred flow was fragmented.
- After: representative Physics-preferred integration flow is covered with configured seam policies and explicit fallback behavior, while collision-side policy gate behavior is validated in Collision tests.
- Fallback: legacy paths remain intact and explicitly test-covered.

Outcome:

- Added representative integration-flow test coverage in Physics (`PhysicsCollisionPreferredFlowIntegrationTest`) proving configured seam-policy execution and fallback routing.
- Kept collision-side response-path policy gate validated in Collision tests from P11.
- Documented cross-repo API-version constraint: Physics currently compiles against released `org.dynamiscollision` API, so direct `setResponsePathPolicy(...)` wiring in Physics code remains pending aligned dependency adoption.

### P13 — Dependency-aligned representative direct wiring (completed)

Commits / alignment action:

- `5d79308` (`DynamisPhysics`): representative Physics integration flow now directly configures `CollisionWorld3D.setResponsePathPolicy(...)`.
- Local dependency alignment action: installed updated `collision_detection` artifact (`1.1.1`) from `DynamisCollision` so Physics compiles against the policy-gate API.

Before/After/Fallback:

- Before: representative Physics integration test could not directly wire `setResponsePathPolicy(...)` due cross-repo API version mismatch.
- After: representative Physics flow directly wires collision response-path policy gate along with Physics seam policies.
- Fallback: legacy fallback branches remain unchanged and explicit.

Outcome:

- Closed the representative direct-wiring gap identified in P12.
- Preserved bounded scope (test/integration wiring only; no broad behavior rewrite).

### P14 — Production-facing representative entrypoint wiring (completed)

Commit:

- `69fc223` (`DynamisPhysics`): added production-facing representative integration entrypoint `PhysicsCollisionPreferredFlowConfigurator` and routed representative integration coverage through it

Before/After/Fallback:

- Before: representative preferred-path composition was demonstrated in tests, but no dedicated production-facing entrypoint existed for that composed wiring.
- After: `PhysicsCollisionPreferredFlowConfigurator.configure(...)` provides an opt-in production-facing entrypoint that applies composed preferred-path wiring (`setResponsePathPolicy(...)` + preferred responder assignment) for representative flows.
- Fallback: legacy/default behavior remains unchanged when configurator is not applied; fallback routing remains explicit inside preferred responder policies.

Outcome:

- Added bounded production-facing entrypoint without changing global defaults.
- Updated representative integration tests to exercise the production-facing configurator path.
- Preserved additive/non-breaking scope.

### P15 — Production-facing preferred-flow convenience preset (completed)

Commit:

- `e8ec3d5` (`DynamisPhysics`): added `PhysicsCollisionPreferredFlowPresets` with one representative default policy bundle around `PhysicsCollisionPreferredFlowConfigurator`

Before/After/Fallback:

- Before: production code could enable the preferred flow via configurator, but had to assemble policy/responder wiring manually.
- After: `PhysicsCollisionPreferredFlowPresets.configureDefault(...)` provides one bounded production-facing convenience preset for a representative adapter shape with default seam-selection/ordering/persistence behavior.
- Fallback: legacy/default behavior remains unchanged when preset is not used; explicit seam-selection override and fallback responder behavior remain available when preset is used.

Outcome:

- Added one narrow preset entrypoint without changing global defaults.
- Added focused coverage proving default preset preferred-path behavior and explicit seam-selection override to fallback.
- Preserved additive/non-breaking scope.

### P16 — Production-entrypoint preset wiring (completed)

Commit:

- `a96cbbb` (`DynamisPhysics`): wired the production-facing configurator entrypoint to consume the P15 default preset bundle

Before/After/Fallback:

- Before: the convenience preset existed, but production callers had to choose between low-level configurator wiring or calling preset helpers directly.
- After: `PhysicsCollisionPreferredFlowConfigurator.configureDefault(...)` is a production-facing runtime assembly entrypoint that applies the representative default preset wiring for one adapter shape.
- Fallback: legacy/default behavior remains unchanged when configurator is not applied; explicit seam-selection override and fallback responder routing remain available via configurator overloads.

Outcome:

- Added one bounded production assembly surface that now routes through preset-based preferred-flow wiring.
- Updated focused integration coverage to exercise configurator default + override entrypoints.
- Preserved additive/non-breaking scope with no global default flip.

### P17 — Non-test runtime composition adoption (completed)

Commit:

- `c735076` (`DynamisPhysics`): added `PhysicsCollisionWorldAssemblies` as a concrete non-test runtime composition callsite using `PhysicsCollisionPreferredFlowConfigurator.configureDefault(...)`

Before/After/Fallback:

- Before: preferred-flow default wiring was exposed via configurator/preset entrypoints, but no concrete non-test runtime composition helper consumed that path.
- After: `PhysicsCollisionWorldAssemblies.createWithPreferredDefaults(...)` constructs `CollisionWorld3D` and applies configurator default wiring in one production-facing composition callsite.
- Fallback: explicit opt-out remains available via seam-selection override overload; legacy/default behavior remains unchanged for non-adopting flows.

Outcome:

- Added one bounded, non-test runtime assembly adoption callsite for the preferred path.
- Added focused integration coverage for assembly default-path behavior and explicit opt-out fallback routing.
- Preserved additive/non-breaking scope with no global default flip.

### P18 — Next candidate (bounded)

Target:

- Adopt `PhysicsCollisionWorldAssemblies.createWithPreferredDefaults(...)` in one concrete cross-repo runtime consumer (outside `DynamisPhysics`) and verify opt-out path remains available.

Constraints:

- Additive and compatibility-preserving.
- One consumer integration only; no broad rollout across callsites.
- No solver/runtime rewrite and no global default behavior change.



