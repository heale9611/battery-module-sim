# 电池模组装配基地总览实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建可本机启动的 Three.js 基地总览，展示厂房、道路、仓储和港口，并支持旋转、缩放、平移、选中聚焦和轻量 HUD。

**Architecture:** 使用 Vite + TypeScript + Three.js。模型数据、程序化设施、场景渲染、摄像机交互和 DOM HUD 分模块实现；本阶段提供稳定的实体标识和更新接口，供第二阶段仿真接入。

**Tech Stack:** Node.js、npm、Vite、TypeScript、Three.js、Vitest。

**Spec:** `docs/superpowers/specs/2026-10-08-battery-assembly-simulation-design.md`

## Global Constraints

- 场景以全屏 3D 为主，数据以轻量浮层呈现。
- 视觉采用白灰基调与工业橙强调色。
- 使用程序化几何体和共享材质构造基地设施。
- OrbitControls 支持旋转、缩放、平移；Raycaster 支持设施选择和镜头聚焦。
- 运行数据先由本地模拟状态提供。
- 使用独立模块管理场景、模型、交互、仿真和界面。
- 不引入真实设备联网或后端依赖。

## Review Focus

- **窗口缩放/高 DPI**：renderer 尺寸、相机比例和像素比保持正确；在 Task 3 测试 viewport 尺寸计算。
- **射线击中设施的子 Mesh**：选中必须解析到设施根实体；在 Task 3 测试祖先实体解析。
- **无可选对象的点击**：清除旧选择且不抛错；在 Task 4 测试空选中。
- **摄像机聚焦重叠/嵌套模型**：按被选实体根节点定位而非任意子网格；在 Task 3 测试根节点中心。
- **窄屏 HUD 遮挡**：浮层可换行并保持 canvas 可操作；在 Task 4 做窄屏本机浏览器检查。

---

## 文件结构

- `package.json`：依赖、开发/构建/测试脚本。
- `index.html`：应用挂载点和页面元信息。
- `vite.config.ts`、`tsconfig.json`：构建与类型检查配置。
- `src/main.ts`：创建场景、模型、交互和 HUD，并接线生命周期。
- `src/styles.css`：全屏画布、悬浮面板、窄屏布局和工业主题。
- `src/models/siteLayout.ts`：基地实体 ID、标签、种类、位置和占地尺寸。
- `src/models/materials.ts`：共享材质调色板。
- `src/models/primitives.ts`：共享基础几何体，按 Mesh 缩放复用。
- `src/models/buildBase.ts`：程序化生成地面、厂房、装配区、道路、仓储、车辆、港口及可选实体。
- `src/scene/createScene.ts`：renderer、scene、camera、灯光、OrbitControls 与尺寸监听。
- `src/scene/viewportMetrics.ts`、`src/scene/viewportMetrics.test.ts`：CSS 视口尺寸、高 DPI 像素比上限和安全宽高比。
- `src/scene/RenderLoop.ts`：可注册 update 回调的 requestAnimationFrame 生命周期。
- `src/interaction/entityPicking.ts`：实体标记和 Raycaster 命中到实体根节点的解析。
- `src/interaction/CameraFocus.ts`：平滑移动视点/目标并约束控制器。
- `src/interaction/CameraFocus.test.ts`：嵌套设施聚焦位置回归测试。
- `src/ui/Hud.ts`：总览状态卡片、选中详情和相机提示。
- `src/ui/Hud.test.ts`：状态摘要格式化和选择清除回归测试。
- `src/models/siteLayout.test.ts`、`src/interaction/entityPicking.test.ts`：布局与选取回归测试。

## Task 1: 初始化 Vite、TypeScript 与测试脚本

**Files:**
- Create: `package.json`
- Create: `index.html`
- Create: `vite.config.ts`
- Create: `tsconfig.json`
- Create: `src/main.ts`
- Create: `src/styles.css`

**Interfaces:**
- Produces: npm scripts `dev`、`build`、`test`、`typecheck`；`src/main.ts` 是唯一应用入口。

- [ ] **Step 1: Scaffold minimal files and scripts**

Create the Vite TypeScript package with dependency `three`, dev dependencies `vite`, `typescript`, `vitest`; set scripts to `vite --host 127.0.0.1 --port 5173 --strictPort`, `vite build`, `vitest run`, and `tsc --noEmit`. Configure `vite.config.ts` with that same host/port and Vitest `environment: "node"` plus `include: ["src/**/*.test.ts"]`. Add an HTML document containing `<main id="app"></main>` and a module script for `/src/main.ts`. Add a strict TypeScript config targeting ES2022 with bundler module resolution and DOM libraries. Add an empty `main.ts` that asserts `#app` exists and imports `styles.css`.

- [ ] **Step 2: Install dependencies and verify clean scaffold**

Run: `npm install`
Expected: lockfile is created and installation exits successfully.

Run: `npm run build`
Expected: Vite emits a production bundle without TypeScript or import errors.

- [ ] **Step 3: Commit the runnable scaffold**

```powershell
git add package.json package-lock.json index.html vite.config.ts tsconfig.json src/main.ts src/styles.css
git commit -m "chore: scaffold battery base overview" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

## Task 2: Define the site layout and build procedural facilities

**Files:**
- Create: `src/models/siteLayout.ts`
- Create: `src/models/siteLayout.test.ts`
- Create: `src/models/materials.ts`
- Create: `src/models/primitives.ts`
- Create: `src/models/buildBase.ts`

**Interfaces:**
- `SiteEntityDefinition`: `{ id: string; label: string; kind: "plant" | "road" | "warehouse" | "port" | "line" | "yard"; position: readonly [number, number, number]; footprint: readonly [number, number] }`.
- `SITE_ENTITIES: readonly SiteEntityDefinition[]`: stable definitions for plant, road network, assembly line, warehouse, storage yard, and port.
- `buildBase(scene: THREE.Scene): THREE.Object3D[]`: adds all visible facilities and returns selectable entity roots.

- [ ] **Step 1: Write failing tests for unique IDs and intended footprints**

In `siteLayout.test.ts`, assert IDs are unique, all footprints have positive width/depth, each required kind (`plant`, `road`, `warehouse`, `port`) is represented, and the warehouse/port/main plant rectangles do not overlap. Test helper:

```ts
function overlaps(a: SiteEntityDefinition, b: SiteEntityDefinition): boolean {
  const [ax, , az] = a.position;
  const [aw, ad] = a.footprint;
  const [bx, , bz] = b.position;
  const [bw, bd] = b.footprint;
  return Math.abs(ax - bx) * 2 < aw + bw && Math.abs(az - bz) * 2 < ad + bd;
}
```

- [ ] **Step 2: Run the layout tests and verify they fail**

Run: `npm test -- src/models/siteLayout.test.ts`
Expected: FAIL because `siteLayout.ts` does not exist.

- [ ] **Step 3: Implement the layout and procedural model builder**

Use a 160-by-110-unit site centered at the origin, with bounds `x=[-80,80]`, `z=[-55,55]`. Define these primary centered footprints: plant `(0, 12)` size `64x42`; warehouse `(-56, -15)` size `24x24`; storage yard `(56, -15)` size `24x24`; port `(0, -41)` size `100x14`. The perimeter road centerline runs `x=-74`, `x=74`, `z=48`, `z=-52`, with 3-unit road width. Use unit `BoxGeometry`, `CylinderGeometry`, and a `PlaneGeometry` from `primitives.ts`, scaling shared geometry instances for compatible parts; keep a small set of shared white/gray/orange materials. Keep true-volume facility faces non-coplanar; render road markings slightly above the road surface. Create repeated yard pallets and lights with `InstancedMesh`. Tag each root with its entity ID and label. The `buildBase` implementation iterates `SITE_ENTITIES`, calls focused facility/road builders and returns the tagged roots; do not introduce per-instance geometry/material copies.

- [ ] **Step 4: Run the layout tests and verify they pass**

Run: `npm test -- src/models/siteLayout.test.ts`
Expected: PASS for unique IDs, required facilities, positive dimensions and non-overlapping primary facility footprints.

- [ ] **Step 5: Commit the site model**

```powershell
git add src/models/siteLayout.ts src/models/siteLayout.test.ts src/models/materials.ts src/models/primitives.ts src/models/buildBase.ts
git commit -m "feat: build procedural battery assembly site" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

## Task 3: Create the renderer, controls, entity picking and focus

**Files:**
- Create: `src/scene/createScene.ts`
- Create: `src/scene/viewportMetrics.ts`
- Create: `src/scene/viewportMetrics.test.ts`
- Create: `src/scene/RenderLoop.ts`
- Create: `src/interaction/entityPicking.ts`
- Create: `src/interaction/entityPicking.test.ts`
- Create: `src/interaction/CameraFocus.ts`
- Create: `src/interaction/CameraFocus.test.ts`

**Interfaces:**
- `createScene(canvas: HTMLCanvasElement): { scene: THREE.Scene; camera: THREE.PerspectiveCamera; renderer: THREE.WebGLRenderer; controls: OrbitControls; dispose(): void }`.
- `getViewportMetrics(width: number, height: number, devicePixelRatio: number): { width: number; height: number; pixelRatio: number; aspect: number }`: width/height are at least 1 CSS pixel, pixel ratio is clamped to `[1, 1.5]`, aspect is the safe width/height ratio.
- `getEntityRoot(object: THREE.Object3D): THREE.Object3D | null`: walks ancestors and returns the first node whose `userData.entityId` is a string.
- `pickEntity(event: PointerEvent, canvas: HTMLCanvasElement, camera: THREE.Camera, roots: readonly THREE.Object3D[]): THREE.Object3D | null`.
- `CameraFocus.focus(root: THREE.Object3D, durationSeconds?: number): void`; `CameraFocus.update(deltaSeconds: number): void`.
- `RenderLoop.add(update: (deltaSeconds: number) => void): () => void`; `start(): void`; `dispose(): void`.

- [ ] **Step 1: Write failing viewport, entity-root/selection and focus tests**

Test `getViewportMetrics(1280, 720, 3)` returns a 1.5 pixel ratio and aspect `1280 / 720`, while zero CSS dimensions yield safe nonzero dimensions and finite aspect. Test a nested mesh returns its tagged ancestor, an untagged object returns `null`, and a ray through the center of a tagged box returns that tagged root while a ray outside it returns `null`. Test `CameraFocus.focus` uses the center of a translated group containing multiple child meshes as its target. Use real Three.js scene objects and `Raycaster` in Vitest; do not mock intersection behavior.

- [ ] **Step 2: Run the tests and verify they fail**

Run: `npm test -- src/scene/viewportMetrics.test.ts src/interaction/entityPicking.test.ts src/interaction/CameraFocus.test.ts`
Expected: FAIL because viewport metrics, entity picking and camera focus modules/tests do not exist.

- [ ] **Step 3: Implement scene lifecycle and picking**

Create an antialiased renderer, soft background, hemisphere/key lighting and bounded shadow casting. Use `getViewportMetrics` for initial size and every resize, then update renderer size, pixel ratio and camera aspect/projection. Dispose listeners, controls and renderer. Configure OrbitControls damping and finite min/max distance. Convert pointer coordinates using the canvas bounding rectangle before raycasting. Camera focus uses `Box3.setFromObject(root)` to determine a safe target and eases camera and target together.

- [ ] **Step 4: Run picking tests and typecheck**

Run: `npm test -- src/scene/viewportMetrics.test.ts src/interaction/entityPicking.test.ts src/interaction/CameraFocus.test.ts`
Expected: PASS for viewport bounds, nested/missing entities, ray hit/miss and group-center focus cases.

Run: `npm run typecheck`
Expected: PASS with no diagnostics.

- [ ] **Step 5: Commit scene and interaction primitives**

```powershell
git add src/scene src/interaction
git commit -m "feat: add scene controls and entity picking" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```

## Task 4: Add overview HUD and connect the runnable application

**Files:**
- Modify: `src/main.ts`
- Modify: `src/styles.css`
- Create: `src/ui/Hud.ts`

**Interfaces:**
- `mountHud(host: HTMLElement): { setSelection(entity: THREE.Object3D | null): void; setOverview(summary: OverviewSummary): void; dispose(): void }`.
- `formatOverviewSummary(summary: OverviewSummary): readonly { label: string; value: string }[]`; `OverviewSummary` is `{ productionCount: number; activeDeviceCount: number; vehicleCount: number; alarmCount: number }`.
- `main.ts` constructs the site once and wires `createScene`, `buildBase`, `pickEntity`, `CameraFocus`, `RenderLoop`, and `mountHud`.

- [ ] **Step 1: Add a deterministic HUD state formatter test**

Create `src/ui/Hud.test.ts` and test that `formatOverviewSummary` converts a known summary into the expected Chinese metric/value pairs. Test that setting a selected entity exposes its label/ID and setting `null` clears selection. Keep DOM event/rendering logic small; pure formatter and selection-state tests do not require jsdom.

- [ ] **Step 2: Run the HUD test and verify it fails**

Run: `npm test -- src/ui/Hud.test.ts`
Expected: FAIL because the HUD module/formatter does not exist.

- [ ] **Step 3: Implement the HUD and application wiring**

Render one compact top status strip and one right-side selection card in ordinary DOM positioned above the canvas. Include Chinese labels for production, active devices, vehicles, alarms, and controls. Use fixed mock values `{ productionCount: 128, activeDeviceCount: 5, vehicleCount: 3, alarmCount: 0 }`, never random values. Make the layout reflow below 760px. Use one canvas covering the viewport; on click, pick and focus the entity and update the HUD; if no entity is hit, clear the previous selection. Always call `controls.update()` and render from `RenderLoop`. Register page unload cleanup.

- [ ] **Step 4: Run focused tests and production build**

Run: `npm test -- src/ui/Hud.test.ts src/models/siteLayout.test.ts src/interaction/entityPicking.test.ts`
Expected: PASS for all selected tests.

Run: `npm run typecheck; npm run build`
Expected: both commands exit successfully.

Run: `npm run dev`
Expected: Vite reports `http://127.0.0.1:5173/`; load it in a local browser and verify the canvas, HUD, camera gestures, facility selection and focus on desktop and narrow viewports. Stop the dev server after this check unless the user explicitly requested it to remain running.

- [ ] **Step 5: Commit the overview**

```powershell
git add src/main.ts src/styles.css src/ui
git commit -m "feat: deliver interactive battery base overview" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
```
