# Fabric.js Architecture & Override Lifecycle

Single source of truth for where Fabric.js is located, how it is developed locally, and how `designer_ui` overrides and initializes it.

---

## 1. Fabric.js Package Location & Local Development

Fabric.js in Designer V2 does not come from the standard public npm `fabric` package. It is wrapped inside a corporate Roland DG package named `react-fabric-page`.

### Source Repositories
* **Git Package Dependency**: `package.json` resolves `react-fabric-page` from the corporate Azure DevOps git repository (`#v1.0.63`).
* **Local Source Path**: `D:\perforce\react-fabric-page\rdgconnect-designer-v2-react-fabric-page`

### Local Development Workflow (`dev-fabric.js`)
When developing or debugging Fabric engine changes alongside `designer_ui`:
1. In `designer_ui/.env` (or `.env.local`), configure the local repository path:
   ```env
   FABRIC_PATH=D:\perforce\react-fabric-page\rdgconnect-designer-v2-react-fabric-page
   ```
2. Running `npm run dev` executes `scripts/dev-fabric.js`. This script inspects `FABRIC_PATH` and links the live build of `react-fabric-page` into `designer_ui/node_modules/react-fabric-page/dist/index.js`, allowing real-time testing of engine changes without publishing packages.

---

## 2. Two-Phase Override Lifecycle

`designer_ui` overrides and configures Fabric using a two-phase architecture:

```
App / Editor Boot
  │
  ├─ Phase 1: Static Monkey-Patching (Module Load Time)
  │    ├─ fabricMemory.js  ──> Global memory helpers on fabric.util & disposal patches
  │    └─ fabricCustom.js  ──> Prototype extensions (fabric.Object, fabric.Canvas, PatternImage)
  │
  └─ Phase 2: Runtime Canvas Initialization (Component Mount Time)
       └─ customCallback(editor) ──> Passed to <PageCanvas customCallback={customCallback} />
```

### Phase 1: Static Monkey-Patching (Module Evaluation Time)
`src/App/modules/Editor/fabricCustom.js` and `fabricMemory.js` run immediately when imported. They patch the Fabric namespace before any canvas is created:

1. **Custom Classes**:
   - `fabric.PatternImage`: Custom image class supporting pattern fills, color replacements, and white ink handling (`applyWhite`).
2. **Prototype Extensions**:
   - `fabric.Object.prototype`: Patches `toSVG()`, `_set()`, coordinate transforms, custom overlay rendering (`applyOverlayRenderLogic`), and low-res image computation.
   - `fabric.Canvas.prototype` & `fabric.StaticCanvas.prototype`: Patches layer integration, touch handling, custom viewport rendering.
   - `fabric.Group.prototype`: Patches nested object iteration and coordinate mapping.
3. **Deep Disposal**:
   - Monkey-patches `dispose()` on `Canvas`, `StaticCanvas`, `Group`, `Image`, and `Object` to zero `<canvas>` elements to `0x0`, preventing mobile Safari memory exhaustion.

### Phase 2: Runtime Canvas Initialization (`customCallback`)
`fabricCustom.js` exports an async initialization function:
```javascript
export default async function customCallback(editor) {
  // editor is the CanvasEditor instance
}
```

In `src/App/modules/Editor/Editor.js`, this function is passed to `<PageCanvas>`:
```jsx
<PageCanvas
  page={page}
  customCallback={customCallback}
  onPageMount={handlePageMount}
/>
```

When `<PageCanvas>` mounts:
- It creates the `CanvasEditor` instance and active Fabric `canvas`.
- It invokes `customCallback(editor)`.
- `customCallback` sets up editor-level event bindings (snapping guides, custom cursor updates, keyboard shortcuts, selection boundary constraints).

---

## 3. Where New Code Belongs (Decision Matrix)

When extending canvas functionality, place code according to this matrix:

| Task | Where It Belongs | Why |
| :--- | :--- | :--- |
| **New Fabric object class** or subclassing | `fabricCustom.js` (or a sub-module imported by it) | Must extend `fabric.*` before canvases or layers instantiate. |
| **Monkey-patching native Fabric method** (e.g. `toSVG`, `_render`) | `fabricCustom.js` | Keeps all prototype alterations co-located and tracked in one module. |
| **Canvas/Editor-wide event or interaction** (e.g. guide snapping, ruler lines) | `customCallback(editor)` in `fabricCustom.js` | Requires access to the live `editor` and `canvas` instances upon canvas mount. |
| **Reusable object helper** (e.g. alignment, cloning, bounds calculation) | `src/App/modules/Editor/objectUtils.js` | Pure helper functions that operate on Fabric objects without modifying prototypes. |
| **Selection inspection & traversal** | `src/App/modules/Editor/selectionUtils.js` | Standard place for reading or filtering active selection objects. |
| **Memory reclamation & temporary canvas** | `src/App/modules/Editor/fabricMemory.js` | Low-level GPU memory release and buffer recycling. |
