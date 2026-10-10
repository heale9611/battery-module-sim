# Battery Assembly Base Simulation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver a locally runnable Three.js battery-module assembly base with a procedural site overview, observable equipment and inspection cycles, safe simulated traffic, camera tours, and a live local preview.

**Architecture:** Implement the overview first, then build simulation systems on the stable entity layout and scene interfaces. A single simulation engine and clock provide immutable snapshots to model controllers and HUD; camera control remains separate, with manual input taking precedence over tours.

**Tech Stack:** TypeScript, Vite, Three.js, OrbitControls, Vitest, npm.

**Spec:** `docs/superpowers/specs/2026-10-08-battery-assembly-simulation-design.md`

## Global Constraints

- The scene is a full-screen 3D battery assembly base with factories, roads, storage and port.
- Use white/gray surfaces with industrial orange accents and lightweight floating data panels.
- Build the main facilities programmatically and keep scene, models, interaction, simulation and UI in independent modules.
- Use OrbitControls for orbit, zoom and pan; use Raycaster selection and camera focus.
- Simulation uses local mock data and does not connect to PLC, vehicle systems, MES/WMS, cloud APIs or a backend.
- The simulation snapshot is the single source of truth for device, vehicle, alarm and HUD state.
- Pausing freezes elapsed simulation time and all state-driven motion; resuming continues the current phase.
- Robot phases are waiting, grasping, transporting, placing and returning; gripper and workpiece attachment must agree with state.
- Module layers and individual parts can be exploded, restored, hidden and shown.
- Inspection phases are clamping, probing, testing and feedback.
- Vehicles follow explicit routes with intersection exclusion, safe following distance, fault stops and linked alarms.
- Manual camera input cancels an automatic tour.
- Reuse geometries and materials, instance repeated props, bound shadow work and cap renderer pixel ratio.
- After acceptance, keep the local preview server running at `http://127.0.0.1:5173/`.

## Review Focus

- Large, negative or non-finite frame deltas must not skip device phases; pin clock clamping and invalid-delta behavior in `src/simulation/SimulationClock.test.ts`.
- Grasp/release transitions must neither duplicate nor lose a workpiece; pin state transitions and parent changes in `src/simulation/RobotCycle.test.ts`.
- Faults or pauses inside an intersection must preserve the reservation and block conflicting vehicles; pin lock lifecycle in `src/simulation/VehicleSystem.test.ts`.
- Every route segment must avoid facility footprints and cyclic following must maintain the minimum gap; pin route validation and adjacent-edge following in `src/simulation/trafficNetwork.test.ts` and `src/simulation/VehicleSystem.test.ts`.
- A manual orbit/drag during a camera interpolation must permanently take control until the user starts another tour; pin this in `src/interaction/TourController.test.ts`.

---

## File Structure and Existing Phase Plans

The repository currently contains design and implementation documents but no application source. Use the existing detailed phase plans as the task-level implementation source:

- `docs/superpowers/plans/2026-10-08-battery-assembly-overview.md` defines the Vite scaffold, site layout, procedural facilities, renderer, picking, focus and overview HUD.
- `docs/superpowers/plans/2026-10-08-battery-assembly-simulation.md` defines the simulation clock, robot/module, inspection, traffic, camera tour, unified engine/HUD, and acceptance.
- `src/main.ts` is the integration entry point; `src/styles.css` contains the full-screen scene and floating-panel layout.
- `src/models/` owns the stable site layout, reusable materials/geometries and programmatic facility/equipment models.
- `src/scene/` owns renderer lifecycle, viewport metrics and render-loop updates.
- `src/interaction/` owns entity picking, camera focus and the automatic tour.
- `src/simulation/` owns typed snapshots, the clock, equipment cycles, route coordination, vehicles and the engine.
- `src/ui/` owns the HUD and controls, consuming simulation snapshots rather than creating separate business state.

## Task 1: Build the runnable site overview

**Files:** Follow Task 1–4 of `docs/superpowers/plans/2026-10-08-battery-assembly-overview.md`.

**Interfaces:** Establish `SITE_ENTITIES`, `buildBase(scene)`, `createScene(canvas)`, `pickEntity(...)`, `CameraFocus`, `RenderLoop` and `mountHud(host)` exactly as defined in that phase plan. These stable interfaces are consumed by Tasks 2–6.

Defer the overview plan's development-server/browser check to Task 7; run the overview's unit tests, typecheck and production build here, but start the preview only after the integrated simulation is ready.

- [ ] **Step 1: Scaffold the Vite/TypeScript application and add the viewport, layout, picking, focus and HUD tests** using the exact files and cases in overview-plan Tasks 1–4.
- [ ] **Step 2: Run each new test before its implementation** using the focused commands in the overview plan and confirm it fails for the missing behavior.
- [ ] **Step 3: Implement the overview modules** with shared geometry/materials, tagged facility roots, bounded shadows, capped pixel ratio and resize/disposal handling.
- [ ] **Step 4: Run overview tests, typecheck and production build** using the commands in overview-plan Task 4; confirm the scene and HUD are wired through `src/main.ts`.
- [ ] **Step 5: Commit the runnable overview** with the overview-plan Task 4 commit command.

## Task 2: Add the simulation clock, robot cycle and exploded module

**Files:** Follow Tasks 1–2 of `docs/superpowers/plans/2026-10-08-battery-assembly-simulation.md`.

**Interfaces:** Produce `SimulationClock`, readonly snapshot types, `RobotCycle`, `createRobotRig` and `ModuleController`, as specified in simulation-plan Tasks 1–2.

- [ ] **Step 1: Add failing tests** for pause/resume, invalid and capped deltas, robot phase order, exact workpiece attachment/release, repeated cycles, layer offsets and part visibility.
- [ ] **Step 2: Run the focused clock, robot and module test commands** and confirm the missing implementations fail.
- [ ] **Step 3: Implement the clock and deterministic state machines**; preserve remainder across phase boundaries and reparent the workpiece only at grasp and placement transitions.
- [ ] **Step 4: Run the focused tests and `npm run typecheck`**; confirm pause/resume preserves progress and model identity/visibility.
- [ ] **Step 5: Commit the clock, robot and module deliverables** using the simulation-plan Task 1–2 commit messages and file lists.

## Task 3: Add the inspection station sequence

**Files:** Follow Task 3 of `docs/superpowers/plans/2026-10-08-battery-assembly-simulation.md`.

**Interfaces:** Produce `InspectionCycle` and `createInspectionStation(scene)` with the stable inspection entity ID.

- [ ] **Step 1: Add a failing phase/result test** for clamping → probing → testing → feedback, deterministic pass/fail feedback and reset to pending.
- [ ] **Step 2: Run the focused inspection test** and confirm it fails before implementation.
- [ ] **Step 3: Implement the cycle and station visuals** with synchronized clamp travel, probe movement, active testing indication and visible result feedback.
- [ ] **Step 4: Run the inspection test and `npm run typecheck`**.
- [ ] **Step 5: Commit the inspection cycle and station model** using the simulation-plan Task 3 commit command.

## Task 4: Add safe routes, vehicle behavior and fault alarms

**Files:** Follow Task 4 of `docs/superpowers/plans/2026-10-08-battery-assembly-simulation.md`.

**Interfaces:** Produce typed road nodes/edges, `IntersectionController`, `validateRoutes(...)` and `VehicleSystem`, as specified in that task.

- [ ] **Step 1: Add failing route and intersection tests** for facility-footprint avoidance, crossing-segment detection and exclusive junction reservation.
- [ ] **Step 2: Run the focused route test** and confirm it fails before implementation.
- [ ] **Step 3: Add failing vehicle tests** for four-unit minimum following distance across adjacent edges, junction entry, route wrap, zero-delta stability, fault alarms, and reservation retention during fault/pause.
- [ ] **Step 4: Run the focused vehicle tests** and confirm they fail before implementation.
- [ ] **Step 5: Implement the route graph and vehicle system**; cap movement at edge ends, intersection boundaries and following distance, and derive alarms from fault state.
- [ ] **Step 6: Run route and vehicle tests plus `npm run typecheck`**; confirm route segments avoid plant, warehouse, yard and port footprints.
- [ ] **Step 7: Commit the safe traffic system** using the simulation-plan Task 4 commit command.

## Task 5: Add detail shots, automatic tour and manual camera takeover

**Files:** Follow Task 5 of `docs/superpowers/plans/2026-10-08-battery-assembly-simulation.md`.

**Interfaces:** Produce the four `CameraShot` definitions and `TourController.start/stop/update/takeControl()` without replacing OrbitControls.

- [ ] **Step 1: Add failing tests** for workshop → robot → inspection → port order, interpolation, manual takeover and no automatic restart after takeover.
- [ ] **Step 2: Run the focused tour test** and confirm it fails before implementation.
- [ ] **Step 3: Implement camera interpolation and bind OrbitControls start plus pointer/touch input to `takeControl()`**; dispose all added listeners.
- [ ] **Step 4: Run the tour test and `npm run typecheck`**.
- [ ] **Step 5: Commit the tour controller and scene bindings** using the simulation-plan Task 5 commit command.

## Task 6: Integrate one simulation snapshot with models and HUD

**Files:** Follow Task 6 of `docs/superpowers/plans/2026-10-08-battery-assembly-simulation.md`.

**Interfaces:** Produce `SimulationEngine` commands and `SimulationSnapshot`; `Hud.setSimulation(snapshot)` consumes the returned immutable snapshot and exposes pause, module, visibility, fault and camera controls.

- [ ] **Step 1: Add failing integration tests** for shared clock delta, pause/resume continuity, production count, alarm linkage, module state, and `RangeError` for unknown vehicle/part IDs.
- [ ] **Step 2: Run the focused integration test** and confirm it fails before implementation.
- [ ] **Step 3: Implement the engine and connect all model controllers and HUD controls** from one render-loop callback using the same snapshot.
- [ ] **Step 4: Run `npm test`, `npm run typecheck` and `npm run build`**; confirm all focused and integration tests pass.
- [ ] **Step 5: Commit the integrated simulation** using the simulation-plan Task 6 commit command.

## Task 7: Document, validate and leave the local preview running

**Files:** Follow Task 7 of `docs/superpowers/plans/2026-10-08-battery-assembly-simulation.md`; update `README.md` and only modify source files for observed defects.

- [ ] **Step 1: Document Node/npm prerequisites, install/start commands, controls, four detail shots, simulation stages, pause/resume, module controls and simulated-data scope.**
- [ ] **Step 2: Run `npm test`, `npm run typecheck` and `npm run build`; confirm each exits successfully.**
- [ ] **Step 3: Start `npm run dev` at `http://127.0.0.1:5173/` and verify the app responds with HTTP 200. Do not stop the server after the check.**
- [ ] **Step 4: Inspect the overview and four detail shots; observe robot grasp/release, module explode/restore and part visibility, inspection feedback, pause/resume during activity, tour takeover and vehicle fault/recovery. Check for model overlap, depth fighting, shadow flicker, vehicle collision/building traversal and workpiece detachment.**
- [ ] **Step 5: Fix only defects demonstrated by acceptance checks, rerun tests/typecheck/build, and leave the preview server running for the user.**
- [ ] **Step 6: Commit README and evidence-based corrections** with the simulation-plan Task 7 commit command.

## Execution Order

Complete Tasks 1–7 sequentially. Tasks 2–6 depend on the stable site entities and scene lifecycle from Task 1; Task 6 integrates all previously implemented controllers, and Task 7 validates the assembled application. Do not launch the preview server until the final acceptance phase.
