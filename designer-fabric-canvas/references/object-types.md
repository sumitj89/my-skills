# Object Types & Domain Entity Classification

Single source of truth for distinguishing Fabric primitive types (`obj.type`) from Designer domain types (`obj.designerObjectType`), and safely inspecting objects across groups and selections.

---

## 1. The Two Type Systems

In Designer V2, objects carry two distinct type properties. Understanding when to reach for each prevents critical inspection bugs:

```
Fabric Object
  ├── obj.type                  <-- Low-level Fabric geometry primitive ('group', 'path', 'rect', 'layer')
  └── obj.designerObjectType    <-- High-level Designer domain entity (Consts.OBJECT_TYPE_*)
```

### When to Use `obj.type` (Fabric Primitive Checks)
Use `obj.type` when you care about Fabric's underlying drawing mechanics, geometry class, or container structure:
- Checking if an object is a layer: `obj.type === 'layer'`
- Checking if a selection contains multiple items: `obj.type === 'activeSelection'`
- Checking if an object is a group container: `obj.type === 'group'`
- Handling raw SVG path data: `obj.type === 'path'`

### When to Use `obj.designerObjectType` (Domain Logic Checks)
Use `obj.designerObjectType` for all tool panels, inspectors, feature toggles, and business domain logic.

**Why?** In Designer, domain entities often wrap standard Fabric classes. For example:
- A **Clipart** is implemented as a `fabric.Group` (`obj.type === 'group'`), but its domain identity is `Consts.OBJECT_TYPE_CLIPART`.
- A **Shape** may be a `fabric.Rect`, `fabric.Circle`, or a grouped compound shape (`obj.type === 'group'`), but its domain identity is `Consts.OBJECT_TYPE_SHAPE`.
- Checking `obj.type === 'clipart'` will **always evaluate to false** because Fabric has no native clipart class.

---

## 2. Standard Domain Constants (`Consts.js`)

Always import `Consts` from `src/App/Consts` (or relative path). Never compare against arbitrary string literals.

```javascript
import Consts from 'src/App/Consts';

switch (obj.designerObjectType) {
  case Consts.OBJECT_TYPE_TEXT:
  case Consts.OBJECT_TYPE_CURVED_TEXT:
    // Handle text styling, font family, typography
    break;

  case Consts.OBJECT_TYPE_IMAGE:
    // Handle bitmap filters, crop, transparency
    break;

  case Consts.OBJECT_TYPE_CLIPART:
    // Handle multi-color clipart fills, knockout
    break;

  case Consts.OBJECT_TYPE_SHAPE:
    // Handle geometric shapes, borders, corner rounding
    break;

  case Consts.OBJECT_TYPE_PATH:
    // Handle vector bezier paths
    break;

  case Consts.OBJECT_TYPE_CONTOUR:
    // Handle cut lines and contour offsets
    break;

  default:
    break;
}
```

---

## 3. Auto-Assigning Domain Types (`assignDesignerObjectType`)

When creating new Fabric objects or importing raw SVGs that lack domain metadata, use `assignDesignerObjectType(obj)` from `src/App/modules/Editor/objectUtils.js`:

```javascript
import { assignDesignerObjectType } from 'src/App/modules/Editor/objectUtils';

const rect = new fabric.Rect({ width: 100, height: 100, fill: 'red' });
assignDesignerObjectType(rect);
// rect.designerObjectType is now Consts.OBJECT_TYPE_SHAPE
```

### Mapping Behavior in `assignDesignerObjectType`:
* `fabric.IText` ──> `Consts.OBJECT_TYPE_TEXT`
* `fabric.CurvedText` ──> `Consts.OBJECT_TYPE_CURVED_TEXT`
* `fabric.Path` ──> `Consts.OBJECT_TYPE_PATH`
* `fabric.Image` ──> `Consts.OBJECT_TYPE_IMAGE`
* `fabric.Rect`, `Circle`, `Triangle`, `Ellipse`, `Line`, `Polygon`, `Polyline` ──> `Consts.OBJECT_TYPE_SHAPE`
* `fabric.Group` ──> Preserved as-is (groups represent Cliparts, compound shapes, or multi-selections and must have explicit assignment).

---

## 4. Multi-Selection & Group Traversal (`selectionUtils`)

When users select multiple objects or a grouped entity, inspecting `activeObject` directly can fail or miss nested items. 

Use helper utilities from `src/App/modules/Editor/selectionUtils.js`:

```javascript
import { selectionUtils } from 'src/App/modules/Editor/selectionUtils';

// 1. Unpack all objects of specific domain types across active selection:
const textObjects = selectionUtils.getAllActiveObjects(
  activeObject,
  [Consts.OBJECT_TYPE_TEXT, Consts.OBJECT_TYPE_CURVED_TEXT]
);

// 2. Unpack all designer domain objects recursively from a group or selection:
const allDomainObjects = selectionUtils.getAllDesignerActiveObjects(activeObject);

// 3. Inspect target objects from useActiveFabricObject wrapper:
const [activeWrapper] = useActiveFabricObject();
const targetFabricObjects = activeWrapper.getTargetObjects();
```

### Anti-Patterns to Avoid
```javascript
// Bad - fails if clipart is a group or text is CurvedText
if (target.type === 'text') { ... }

// Bad - hardcoded magic string without Consts
if (target.designerObjectType === 'clipart') { ... }

// Good - uses Consts and domain property
if (target.designerObjectType === Consts.OBJECT_TYPE_CLIPART) { ... }

// Good - uses obj.type legitimately for low-level Fabric container checks
if (target.type === 'group' && !target.designerObjectType) { ... }
```
