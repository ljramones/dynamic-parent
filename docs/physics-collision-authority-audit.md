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

### P2 — First legacy contact flow migration (next)

Target:

- Migrate one representative legacy contact-resolution path from collision-transitional flow to the Physics-owned seam.

Constraints:

- Additive and compatibility-preserving.
- No broad API deletions or package moves.
- No behavior rewrite; only authority-direction correction for one flow.
