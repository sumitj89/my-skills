# Object Serialization & Cloning (`additionalProps`)

Single source of truth for safely cloning Fabric objects and serializing canvas state without dropping Designer custom properties.

---

## 1. The Native Fabric Cloning Trap

By default, native Fabric methods (`obj.clone()`, `obj.toObject()`, and `canvas.toJSON()`) only serialize standard Fabric properties (such as `left`, `top`, `scaleX`, `scaleY`, `angle`, `fill`, `stroke`). 

**The Bug**: Any custom property added to an object in Designer V2 is **silently stripped** during native cloning or serialization unless explicitly requested. If an object is cloned with `obj.clone()`, its `designerObjectType`, `layerType`, `cmykFill`, and spot color tags are wiped clean.

---

## 2. Central Property Registry: `additionalProps`

`src/App/Consts.js` exports `additionalProps`, an array listing every custom property that must survive serialization, cloning, and state persistence:

```javascript
import Consts, { additionalProps } from 'src/App/Consts';
```

Key properties preserved by `additionalProps`:
* **Domain Identity**: `designerObjectType`, `shapeType`, `origId`, `name`
* **Layers & Organization**: `layerType`, `addStructuralGroup`, `addStructuralObjId`
* **Color & CMYK**: `cmykFill`, `cmykStroke`, `colorProfile`, `rdgFillColorValue`, `rdgStrokeColorValue`
* **Special Hardware Inks**: `autoGenerateWhite`, `autoGenerateWhiteObj`, `autoGeneratePrimer`, `autoGenerateGloss`, `applyWhite`, `maskOverlays`
* **Contour & Cuts**: `contourOffset`, `baseContour`, `imageOutlineChoke`
* **Images**: `imageAttr`, `customSrc`, `imageEditorId`

---

## 3. Cloning Objects Safely

### Always Use `cloneFabricObject()`
Never call native `obj.clone()` directly. Import and use `cloneFabricObject()` from `src/App/modules/Editor/objectUtils.js`, which defaults to including `additionalProps`:

```javascript
import { cloneFabricObject } from 'src/App/modules/Editor/objectUtils';

// Good - clones cleanly while preserving all custom metadata
const clonedObj = await cloneFabricObject(sourceObj);

// If extra temporary properties are needed:
const clonedObj = await cloneFabricObject(sourceObj, [...additionalProps, 'tempTag']);
```

### Cloning Nested / Clipboard Objects
For deep cloning where child objects must receive unique IDs (e.g. duplicating or copying to clipboard):
```javascript
import { cloneWithNested } from 'src/App/modules/Editor/ActionPanels/CopyDeletePanel';

// Good - deeply clones groups and assigns fresh IDs to all children
const duplicatedObject = await cloneWithNested(sourceObj);
```

### Anti-Pattern
```javascript
// Bad - native clone strips designerObjectType, cmykFill, and special ink metadata!
sourceObj.clone((cloned) => {
  canvas.addToContentsLayer({ obj: cloned }); // Metadata lost!
});
```

---

## 4. Canvas & Object JSON Serialization

Whenever serializing canvas state for auto-save, export, or history snapshots, **always pass `additionalProps`**:

```javascript
import { additionalProps } from 'src/App/Consts';

// Good - full canvas JSON export with all custom metadata intact
const canvasJson = canvas.toJSON(additionalProps);

// Good - single object serialization with custom metadata intact
const objectData = obj.toObject(additionalProps);

// Bad - custom metadata dropped!
const corruptedJson = canvas.toJSON();
const corruptedObj = obj.toObject();
```
