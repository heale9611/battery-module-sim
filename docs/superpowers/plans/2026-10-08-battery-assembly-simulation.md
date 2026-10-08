# 电池模组装配基地仿真实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在基地总览上增加可暂停的机械臂/模组/检测动画、遵守路口与跟车规则的故障联动车辆、细节镜头和自动巡游。

**Architecture:** 使用统一仿真时钟和类型化状态作为设备动画、车辆移动、告警和 HUD 的唯一状态源。每个设备控制器独立更新可视模型；交通系统在确定的路网和路口仲裁器上运行；摄像机巡游被人工输入立即中断。

**Tech Stack:** TypeScript、Three.js、OrbitControls、Vitest、项目已有的 Vite 开发/构建命令。

**Spec:** `docs/superpowers/specs/2026-10-08-battery-assembly-simulation-design.md`

## Global Constraints

- 仿真状态是设备动画、车辆动画、告警和 HUD 的唯一数据源。
- 暂停时仿真增量为零，恢复时从原状态连续执行，不重置阶段。
- 机械臂按等待、抓取、搬运、落位、回位运行，夹爪开合及工件附着同步。
- 模组支持逐层拆解、复原和零件显隐。
- 检测台按夹紧、探针接触、检测、结果反馈顺序运行。
- 车辆沿显式定义的路网和路线移动；路口采用互斥通行控制；车辆依据前车间距减速或停车。
- 故障车辆停在安全状态并同步产生告警。
- 自动巡游和交互共用镜头控制权，手动输入优先级更高。
- 重复对象使用共享几何体/材质和 InstancedMesh；限制阴影区域及 renderer 像素比。
- 不连接真实 PLC、车辆控制系统、MES/WMS、云端 API 或后端服务。

## Review Focus

- **浏览器失焦后出现超大帧增量**：不能让机械臂、检测或车辆跳过阶段；Task 1 测试时钟对负值及大增量的边界。
- **工件恰逢阶段切换时被重复抓取/丢失**：工件只在夹爪下附着一次，且落位时释放；Task 2 测试两次完整工序和父子层级。
- **暂停或故障发生在路口占用期间**：暂停时锁持有者和进度不变；故障车辆保留通行权并阻止冲突车辆进入，清故障后继续通行；Task 4 测试暂停/恢复及故障期间锁生命周期。
- **路线定义与建筑占地相交或路段末端越界**：每条路线均通过占地避让验证且车辆循环回到有效节点；Task 4 测试路径安全和循环边界。
- **手动拖动恰逢自动巡游摄像机插值**：首个用户输入取消自动镜头目标，不得再次拉回巡游路径；Task 5 测试输入接管。

---

## 文件结构

- `src/simulation/types.ts`：设备阶段、车辆、告警和统一只读快照类型。
- `src/simulation/SimulationClock.ts`、`SimulationClock.test.ts`：暂停/恢复及帧增量约束。
- `src/simulation/RobotCycle.ts`、`RobotCycle.test.ts`：可独立测试的机械臂阶段状态机。
- `src/models/createRobotRig.ts`：关节组、夹爪和可附着工件模型。
- `src/models/ModuleController.ts`、`ModuleController.test.ts`：模组层级拆解与零件显隐。
- `src/models/createInspectionStation.ts`：夹紧、探针、状态灯和检测结果显示。
- `src/models/createVehicle.ts`：车辆视觉模型和路网位置更新。
- `src/simulation/InspectionCycle.ts`、`InspectionCycle.test.ts`：检测工位阶段状态机及反馈。
- `src/simulation/trafficNetwork.ts`、`trafficNetwork.test.ts`：节点/边/路线数据、路口互斥控制及建筑避让验证。
- `src/simulation/VehicleSystem.ts`、`VehicleSystem.test.ts`：跟车、行驶、故障停车和告警。
- `src/simulation/SimulationEngine.ts`、`SimulationEngine.test.ts`：驱动控制器并发布单一状态快照。
- `src/interaction/TourController.ts`、`TourController.test.ts`：四类预设镜头、巡游插值和用户接管。
- `src/ui/Hud.ts`：消费统一快照、详情、暂停/恢复、模组拆解与巡游控制。
- `src/main.ts`：接线模型/仿真/相机/UI 更新及资源销毁。

## Task 1: 添加仿真时钟与状态类型

**Files:**
- Create: `src/simulation/types.ts`
- Create: `src/simulation/SimulationClock.ts`
- Create: `src/simulation/SimulationClock.test.ts`

**Interfaces:**
- `SimulationClock.setPaused(paused: boolean): void`.
- `SimulationClock.advance(realDeltaSeconds: number): number`: 暂停或无效负增量返回 `0`；运行时将单次增量限制到 `0.05` 秒。
- `SimulationSnapshot`: `{ elapsedSeconds: number; deltaSeconds: number; paused: boolean; productionCount: number; robot: RobotSnapshot; inspection: InspectionSnapshot; module: ModuleSnapshot; vehicles: readonly VehicleSnapshot[]; alarms: readonly AlarmSnapshot[] }`.
- `RobotPhase = "waiting" | "grasping" | "transporting" | "placing" | "returning"`.
- `InspectionPhase = "clamping" | "probing" | "testing" | "feedback"`.
- `RobotSnapshot`: `{ phase: RobotPhase; progress: number; holdingPartId: string | null; completedCycles: number }`.
- `InspectionSnapshot`: `{ phase: InspectionPhase; progress: number; result: "pending" | "pass" | "fail" }`.
- `VehicleSnapshot`: `{ id: string; edgeId: string; progress: number; speed: number; faulted: boolean }`.
- `AlarmSnapshot`: `{ id: string; severity: "warning" | "critical"; message: string; sourceId: string }`.
- `ModuleSnapshot`: `{ exploded: boolean; visiblePartIds: readonly string[] }`; module controls are included in the unified snapshot so HUD and model consume the same state.

- [ ] **Step 1: Write clock behavior tests**

Test new clock starts running, `setPaused(true)` makes `advance(0.04)` return `0`, resuming advances by exactly the supplied positive delta when at most `0.05`, negative and non-finite values return `0`, and `advance(8)` returns at most `0.05`.

- [ ] **Step 2: Run the tests and verify they fail**

Run: `npm test -- src/simulation/SimulationClock.test.ts`
Expected: FAIL because `SimulationClock.ts` does not exist.

- [ ] **Step 3: Implement clock and state types**

Implement a clock with an explicit paused flag and no wall-clock accumulation while paused. Sanitize non-finite and negative deltas to zero; clamp valid deltas to 50 ms so tab-resume frames cannot skip discrete phases. Define all snapshot types as plain data with readonly collections.

- [ ] **Step 4: Verify tests and typecheck**

Run: `npm test -- src/simulation/SimulationClock.test.ts`
Expected: PASS for pause, resume, negative, non-finite and large deltas.

Run: `npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit clock and types**

```powershell
git add src/simulation/types.ts src/simulation/SimulationClock.ts src/simulation/SimulationClock.test.ts
git commit -m "feat: add simulation clock and state types" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

## Task 2: Implement robot cycle, gripper attachment and module explosion

**Files:**
- Create: `src/simulation/RobotCycle.ts`
- Create: `src/simulation/RobotCycle.test.ts`
- Create: `src/models/createRobotRig.ts`
- Create: `src/models/ModuleController.ts`
- Create: `src/models/ModuleController.test.ts`

**Interfaces:**
- `RobotCycle.update(deltaSeconds: number): RobotSnapshot`; fixed phases each advance only by nonnegative simulation delta.
- `createRobotRig(scene: THREE.Scene): { root: THREE.Group; gripper: THREE.Group; fixture: THREE.Group; workpiece: THREE.Object3D; update(state: RobotSnapshot): void }`.
- `ModuleController.setExploded(exploded: boolean): void`; `setPartVisible(partId: string, visible: boolean): void`; `update(deltaSeconds: number): void`.

- [ ] **Step 1: Write failing tests for phase order, held workpiece and module controls**

Advance `RobotCycle` using small increments and assert order `waiting → grasping → transporting → placing → returning → waiting`. Assert `holdingPartId` becomes non-null only after grasp and returns to null at placement; completing two cycles yields two acquisitions, two releases and `completedCycles === 2` (increment once when placement completes). For the rig, assert the workpiece is a child of the gripper while `holdingPartId` is set and is reparented to the placement group on release. For the module, assert explosion changes layer offsets without changing part identity and hidden parts set `visible === false`.

- [ ] **Step 2: Run tests and verify they fail**

Run: `npm test -- src/simulation/RobotCycle.test.ts src/models/ModuleController.test.ts`
Expected: FAIL because the state machine and model controller do not exist.

- [ ] **Step 3: Implement state machine and visual controllers**

Use fixed positive phase durations: waiting `1.2s`, grasping `0.8s`, transporting `1.5s`, placing `0.8s`, returning `1.5s`; carry phase remainder across updates so a normal update cannot lose time at a transition. At grasp, attach the workpiece to the gripper exactly once; during transport compute the arm joint pose from bounded interpolation; at placement, move it to the fixture and reparent it before entering return. Build the module from named layer groups with stored closed positions; explosion interpolates each layer along a deterministic offset vector, and visibility changes target a named part only.

- [ ] **Step 4: Verify robot/module tests and integration typecheck**

Run: `npm test -- src/simulation/RobotCycle.test.ts src/models/ModuleController.test.ts`
Expected: PASS for ordered cycles, parent transitions, visibility and layer restoration.

Run: `npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit the robot and module behavior**

```powershell
git add src/simulation/RobotCycle.ts src/simulation/RobotCycle.test.ts src/models/createRobotRig.ts src/models/ModuleController.ts src/models/ModuleController.test.ts
git commit -m "feat: animate robot and exploded battery module" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

## Task 3: Implement inspection sequence and result feedback

**Files:**
- Create: `src/simulation/InspectionCycle.ts`
- Create: `src/simulation/InspectionCycle.test.ts`
- Modify: `src/models/buildBase.ts`
- Create: `src/models/createInspectionStation.ts`

**Interfaces:**
- `InspectionCycle.update(deltaSeconds: number): InspectionSnapshot`.
- `createInspectionStation(scene: THREE.Scene): { root: THREE.Group; update(state: InspectionSnapshot): void }`.

- [ ] **Step 1: Write a failing detector-order test**

Assert the cycle visits exactly `clamping → probing → testing → feedback`, reports `pending` before feedback, then returns pass for the first cycle and fail for the second at feedback; the next cycle resets to pending and the third returns pass.

- [ ] **Step 2: Run the test and verify it fails**

Run: `npm test -- src/simulation/InspectionCycle.test.ts`
Expected: FAIL because the detector cycle is missing.

- [ ] **Step 3: Implement detector controller and station model**

Give phases fixed durations: clamping `0.8s`, probing `0.7s`, testing `1.2s`, feedback `0.8s`. In `createInspectionStation`, use a translating clamp pair, probe group, visible status lamp and result display. `update` maps phase progress to clamp closure and probe travel; testing illuminates the active indicator, feedback selects green PASS or orange FAIL. Tag the station root with the existing stable entity ID for Raycaster details.

- [ ] **Step 4: Verify detector tests and typecheck**

Run: `npm test -- src/simulation/InspectionCycle.test.ts`
Expected: PASS for phase order, reset and deterministic results.

Run: `npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit inspection behavior**

```powershell
git add src/simulation/InspectionCycle.ts src/simulation/InspectionCycle.test.ts src/models/buildBase.ts src/models/createInspectionStation.ts
git commit -m "feat: animate inspection station sequence" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

## Task 4: Add safe routes, intersection control, following and fault alarms

**Files:**
- Create: `src/simulation/trafficNetwork.ts`
- Create: `src/simulation/trafficNetwork.test.ts`
- Create: `src/simulation/VehicleSystem.ts`
- Create: `src/simulation/VehicleSystem.test.ts`
- Create: `src/models/createVehicle.ts`

**Interfaces:**
- `RoadNode`: `{ id: string; position: readonly [number, number, number]; intersectionId?: string }`.
- `RoadEdge`: `{ id: string; from: string; to: string; speedLimit: number; length: number }`.
- `IntersectionController.request(intersectionId: string, vehicleId: string): boolean`; `release(intersectionId: string, vehicleId: string): void`.
- `validateRoutes(nodes, edges, forbiddenFootprints): readonly string[]`: returns route-segment violations.
- `VehicleSystem.update(deltaSeconds: number): void`; `setFault(vehicleId: string, faulted: boolean): void`; `snapshot(): readonly VehicleSnapshot[]`; `alarms(): readonly AlarmSnapshot[]`.

- [ ] **Step 1: Write failing route and intersection tests**

Use world coordinates `[x, 0, z]`. Define route nodes NW `(-74,48)`, NE `(74,48)`, SE `(74,-52)`, SW `(-74,-52)` for the one-way perimeter loop. Add service pads outside facility footprints: warehouse `(-70,-15)`, yard `(70,-15)`, plant entrance `(-34,12)`, port dock `(0,-50)`; connect them with spokes that terminate outside the building footprint. Test all configured route segments remain outside plant `(0,12)` size `64x42`, warehouse `(-56,-15)` size `24x24`, yard `(56,-15)` size `24x24`, and port `(0,-41)` size `100x14`. Assert two vehicles cannot own one intersection at once, the owner can release it, and a segment crossing a forbidden footprint is reported.

- [ ] **Step 2: Run the tests and verify they fail**

Run: `npm test -- src/simulation/trafficNetwork.test.ts`
Expected: FAIL because traffic network definitions and validator do not exist.

- [ ] **Step 3: Implement route graph and mutex-style intersection reservations**

Define explicit waypoint coordinates matching the overview plan's Task 2 layout, each with `y=0`: a one-way perimeter loop NW `(-74,48)` → NE `(74,48)` → SE `(74,-52)` → SW `(-74,-52)` → NW, plus short service-pad spokes to warehouse `(-70,-15)`, yard `(70,-15)`, plant entrance `(-34,12)`, and port dock `(0,-50)`. Validate every segment against the four facility rectangles. Grant a junction to at most one vehicle; only the matching vehicle releases its reservation.

- [ ] **Step 4: Write failing vehicle spacing, collision, fault and junction-pause tests**

Test vehicles maintain a minimum 4-unit gap over the cyclic route even when leader and follower are on adjacent edges; a vehicle waits before a busy intersection without entering it; edge progress remains within its edge; one full route cycle returns the vehicle to a valid node; advancing by zero seconds preserves vehicle progress and intersection reservations; a faulted vehicle stops with a critical alarm sourced to its ID; a faulted vehicle already holding an intersection keeps that reservation and blocks conflicting traffic; clearing the fault removes the alarm and permits it to exit/release the intersection.

- [ ] **Step 5: Run the vehicle tests and verify they fail**

Run: `npm test -- src/simulation/VehicleSystem.test.ts`
Expected: FAIL because `VehicleSystem.ts` does not exist.

- [ ] **Step 6: Implement vehicle movement and verify traffic tests**

Compute each vehicle's forward distance to the next vehicle across the entire cyclic route, not just its current edge. Advance by `speed * deltaSeconds`, cap each step at the minimum of edge end, junction boundary, and available following distance minus 4 units. Acquire a junction before entering its conflict region and release it only after exiting. A fault sets speed to zero and preserves edge/progress; if already inside an intersection, retain its reservation until recovery and safe exit so conflicting traffic remains stopped. Derive alarms from fault state rather than separately toggling them.

Run: `npm test -- src/simulation/trafficNetwork.test.ts src/simulation/VehicleSystem.test.ts`
Expected: PASS for route bounds, forbidden footprints, junction exclusivity, cross-edge following, fault stopping/reservation and alarm recovery.

- [ ] **Step 7: Commit traffic behavior**

```powershell
git add src/simulation/trafficNetwork.ts src/simulation/trafficNetwork.test.ts src/simulation/VehicleSystem.ts src/simulation/VehicleSystem.test.ts src/models/createVehicle.ts
git commit -m "feat: simulate safe site vehicle traffic" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

## Task 5: Add detail shots and automatic camera tour

**Files:**
- Create: `src/interaction/TourController.ts`
- Create: `src/interaction/TourController.test.ts`
- Modify: `src/interaction/CameraFocus.ts`
- Modify: `src/scene/createScene.ts`

**Interfaces:**
- `CameraShot`: `{ id: "workshop" | "robot" | "inspection" | "port"; position: THREE.Vector3; target: THREE.Vector3; durationSeconds: number }`.
- `TourController.start(): void`; `stop(): void`; `update(deltaSeconds: number): void`; `takeControl(): void`.
- `TourController` consumes a callback to set camera position and OrbitControls target; it does not own or replace OrbitControls.

- [ ] **Step 1: Write tests for shot sequence and manual takeover**

Assert four named shots are present in workshop/robot/inspection/port order, interpolation reaches each destination, `takeControl()` stops interpolation and preserves the current user-controlled camera position, and later `update()` calls do not restart the tour.

- [ ] **Step 2: Run the test and verify it fails**

Run: `npm test -- src/interaction/TourController.test.ts`
Expected: FAIL because `TourController.ts` does not exist.

- [ ] **Step 3: Implement tour and bind controls**

Create the four shot coordinates from the established base layout. Interpolate camera and target smoothly with bounded progress. Listen for OrbitControls `start` and pointer/touch interaction on the canvas to call `takeControl`; a manual selection focus is allowed to run after the tour relinquishes control. Dispose added listeners when the controller is destroyed.

- [ ] **Step 4: Verify tour tests and interaction typecheck**

Run: `npm test -- src/interaction/TourController.test.ts`
Expected: PASS for sequence, interpolation and persistent user takeover.

Run: `npm run typecheck`
Expected: PASS.

- [ ] **Step 5: Commit camera tour**

```powershell
git add src/interaction/TourController.ts src/interaction/TourController.test.ts src/interaction/CameraFocus.ts src/scene/createScene.ts
git commit -m "feat: add guided facility camera tour" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

## Task 6: Integrate a single simulation snapshot with HUD and controls

**Files:**
- Create: `src/simulation/SimulationEngine.ts`
- Create: `src/simulation/SimulationEngine.test.ts`
- Modify: `src/ui/Hud.ts`
- Modify: `src/ui/Hud.test.ts`
- Modify: `src/main.ts`
- Modify: `src/styles.css`

**Interfaces:**
- `SimulationEngine.setPaused(paused: boolean): void`.
- `SimulationEngine.setModuleExploded(exploded: boolean): void`.
- `SimulationEngine.setPartVisible(partId: string, visible: boolean): void`.
- `SimulationEngine.setVehicleFault(vehicleId: string, faulted: boolean): void`.
- `SimulationEngine.update(realDeltaSeconds: number): SimulationSnapshot`.
- `Hud.setSimulation(snapshot: SimulationSnapshot): void`; controls expose pause/resume, explode/restore, per-part visibility, camera shot selection and tour start/stop.

- [ ] **Step 1: Write failing integration tests**

Assert one update advances robot, inspection and vehicles by the same clock delta; pausing freezes their snapshots, module interpolation delta and elapsed time; resuming advances from preserved phase/progress; `productionCount` starts at 128 and increments once per completed robot placement, not while paused; a vehicle fault appears in the same returned snapshot's alarm list; toggling module explosion changes `module.exploded` and `visiblePartIds` without mutating unrelated simulation state. Assert unknown vehicle and part IDs throw a `RangeError` from their command methods instead of silently succeeding.

- [ ] **Step 2: Run the integration test and verify it fails**

Run: `npm test -- src/simulation/SimulationEngine.test.ts`
Expected: FAIL because `SimulationEngine.ts` does not exist.

- [ ] **Step 3: Implement engine, HUD bindings and render-loop wiring**

`SimulationEngine` owns one `SimulationClock`, `RobotCycle`, `InspectionCycle` and `VehicleSystem`. `update` advances the clock once, updates every subsystem with that exact delta, and returns it as `snapshot.deltaSeconds`; `productionCount` starts at 128 and increases once for each completed robot placement. In `main.ts`, feed the same snapshot to robot, detector, vehicle, module model controllers and HUD from one render callback; use `snapshot.deltaSeconds` for any interpolated module explosion so it freezes while paused. Reject unknown vehicle/part IDs with `RangeError`. Add controls for pause/resume, module layers and part visibility, named detail shots, tour start/stop, and a demo fault toggle; every control invokes the engine/controller API, not DOM-local state. Dispose subscriptions/listeners/renderers at page teardown.

- [ ] **Step 4: Verify integration tests and all project checks**

Run: `npm test`
Expected: PASS for all clock, robot, module, inspection, traffic, tour and integration tests.

Run: `npm run typecheck; npm run build`
Expected: both commands exit successfully.

- [ ] **Step 5: Commit integrated simulation**

```powershell
git add src/simulation/SimulationEngine.ts src/simulation/SimulationEngine.test.ts src/ui/Hud.ts src/ui/Hud.test.ts src/main.ts src/styles.css
git commit -m "feat: integrate factory simulation controls" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

## Task 7: Perform local preview and visual/performance acceptance

**Files:**
- Modify only defect files discovered during the checks above.
- Modify: `README.md` with actual setup/run instructions and controls.

**Interfaces:**
- Developer workflow: `npm install`, `npm run dev`, `npm run typecheck`, `npm test`, `npm run build`.
- Local preview URL: `http://127.0.0.1:5173/`.

- [ ] **Step 1: Add README instructions and verify command accuracy**

Document Node/npm prerequisites, install/start commands, controls, four detail shots, simulation stages, pause/resume, module controls and simulated-data scope. Run each documented noninteractive check command exactly as written.

- [ ] **Step 2: Run the local preview**

Run: `npm run dev`
Expected: server reports `http://127.0.0.1:5173/`; verify with `Invoke-WebRequest http://127.0.0.1:5173/` that the response is HTTP 200 and contains the application document.

- [ ] **Step 3: Manually inspect the required view and simulation cases**

In a local browser inspect the base overview and all four detail shots. Observe two full robot cycles and verify workpiece attachment/release, expand and restore the module and toggle a part, observe one complete inspection result, pause/resume during each action, run tour then drag to take control, and fault/recover a vehicle. Confirm vehicles do not enter facility footprints or overlap each other, junctions remain exclusive, alarms track faults, road markings do not z-fight, and shadows do not flicker.

- [ ] **Step 4: Apply only evidence-based visual/performance corrections and rerun acceptance**

Fix route coordinates, object spacing, surface height, shadow camera bounds/bias, or repeated-object instancing only where the inspection demonstrates a defect. Re-run `npm test`, `npm run typecheck`, and `npm run build`; expected: all pass.

- [ ] **Step 5: Commit final acceptance and docs**

```powershell
git add README.md src
git commit -m "docs: document battery simulation preview" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```
