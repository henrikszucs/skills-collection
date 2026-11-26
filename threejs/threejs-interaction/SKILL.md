---
name: threejs-interaction
description: Three.js interaction - raycasting and object picking, pointer/touch input, hover and selection, camera controls (OrbitControls, MapControls, PointerLock, Fly, FirstPerson, Trackball, Arcball), TransformControls gizmo, DragControls, box selection, keyboard movement and world/screen coordinate conversion. Use when handling mouse/touch/keyboard input, implementing click or hover detection, adding camera controls, moving objects with a gizmo, or projecting between 3D and screen space.
---

# Three.js Interaction

> three.js **r187**. All controls extend `THREE.Controls` (r169+): they are not `Object3D`s.
> Shared API: `enabled`, `update(delta?)`, `connect(el)`, `disconnect()`, `dispose()`.

## Picking

```javascript
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();

function setPointer(e) {
  const r = renderer.domElement.getBoundingClientRect(); // works for any canvas size/position
  pointer.set(((e.clientX - r.left) / r.width) * 2 - 1, -((e.clientY - r.top) / r.height) * 2 + 1);
}

canvas.addEventListener("click", (e) => {
  setPointer(e);
  raycaster.setFromCamera(pointer, camera);
  const hit = raycaster.intersectObjects(clickables, true)[0]; // nearest first
  if (hit) console.log(hit.object.name, hit.point, hit.instanceId, hit.batchId);
});
```

- Hit fields: `distance`, `point` (world), `face.normal` (object space; world:
  `.clone().transformDirection(hit.object.matrixWorld)`), `uv`, `instanceId`, `batchId`.
- Options: `raycaster.far`, `params.Points.threshold`, `params.Line.threshold`, `layers.set(n)`.
- Raycasting ignores `visible`: hidden objects are still hit, so keep them out of the list (or on
  another layer). Invisible hit areas: a mesh with `MeshBasicMaterial({ visible: false })`.
- The hit is often a child mesh: walk up `parent` until you find your logical object (`userData.selectable`).
- Large meshes are slow (brute force): raycast simple proxies or use `three-mesh-bvh`.
- `raycaster.setFromXRController(controller)` for VR (`threejs-webxr`).

## Pointer Input

Use **pointer events** (mouse, touch, pen in one path) and set `canvas.style.touchAction = "none"`.
Tell a click from an orbit drag by comparing pointerdown/pointerup positions (> ~5 px = drag).

**Hover:** raycast on `pointermove` (or once per frame for big scenes), swap `emissive` on
the hovered mesh, and set `canvas.style.cursor`. glTF materials are shared, so clone the material
before highlighting one mesh, or use `OutlinePass`.

## Camera Controls

```javascript
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
const controls = new OrbitControls(camera, renderer.domElement);
controls.target.set(0, 1, 0);
controls.enableDamping = true;          // then controls.update() EVERY frame
controls.minDistance = 2; controls.maxDistance = 50;
controls.maxPolarAngle = Math.PI / 2;   // stop at the horizon
controls.zoomToCursor = true;
controls.autoRotate = true;             // also needs update()
controls.addEventListener("change", render); // render on demand (no damping)
controls.update();                      // after changing target/position manually
```

| Controls | Notes |
| --- | --- |
| `MapControls` | left = pan, right = rotate; `screenSpacePanning = false` |
| `PointerLockControls(camera, document.body)` | FPS; `lock()` needs a user gesture; `moveForward/moveRight(dist)`; events `lock`/`unlock` |
| `FlyControls` | `update(delta)` required; `dragToLook` |
| `FirstPersonControls` | `update(delta)`; `handleResize()` was removed (r184) |
| `TrackballControls` | free rotation, no fixed up; `update()` each frame, `handleResize()` on resize |
| `ArcballControls(camera, el, scene)` | CAD-style with gizmo; `setGizmosVisible(false)` |

FPS collisions and gravity: `Octree` + `Capsule` (`threejs-math`) or a physics engine.

## TransformControls (gizmo)

```javascript
import { TransformControls } from "three/addons/controls/TransformControls.js";
const gizmo = new TransformControls(camera, renderer.domElement);
scene.add(gizmo.getHelper()); // the HELPER, not the controls (r169+)
gizmo.attach(mesh);
gizmo.setMode("rotate");     // translate | rotate | scale
gizmo.setSpace("local");
gizmo.setTranslationSnap(0.5); gizmo.setRotationSnap(THREE.MathUtils.degToRad(15));
gizmo.addEventListener("dragging-changed", (e) => (orbit.enabled = !e.value)); // don't orbit while dragging
gizmo.addEventListener("objectChange", save);
```

Raycast only your selectable list so the gizmo is never picked.

## DragControls and Box Selection

```javascript
import { DragControls } from "three/addons/controls/DragControls.js";
const drag = new DragControls(draggables, camera, renderer.domElement);
drag.transformGroup = false; // true: drag the top-level object
drag.addEventListener("dragstart", () => (orbit.enabled = false));
drag.addEventListener("drag", (e) => (e.object.position.y = 0)); // constrain
drag.addEventListener("dragend", () => (orbit.enabled = true));
```

DragControls moves on a camera-facing plane; for ground dragging intersect a `THREE.Plane` (below).

Box selection: `SelectionBox(camera, scene)` + `SelectionHelper(renderer, "selectBox")` from
`three/addons/interactive/`. Set `startPoint`/`endPoint` (NDC, z = 0.5) on pointerdown/up, then
`select()`; instanced hits are in `selectionBox.instances`.

## Keyboard

```javascript
const keys = new Set();
addEventListener("keydown", (e) => keys.add(e.code)); // event.code = physical key (WASD on AZERTY too)
addEventListener("keyup", (e) => keys.delete(e.code));
addEventListener("blur", () => keys.clear());          // no stuck keys
// loop: if (keys.has("KeyW")) player.translateZ(-speed * dt);
```

## World <-> Screen

```javascript
// World to screen pixels (for HTML overlays; many labels: CSS2DRenderer)
const v = obj.getWorldPosition(new THREE.Vector3()).project(camera);
const r = canvas.getBoundingClientRect();
const x = (v.x * 0.5 + 0.5) * r.width + r.left;
const y = (-v.y * 0.5 + 0.5) * r.height + r.top;
const onScreen = v.z > -1 && v.z < 1;

// Pointer to ground point
const ground = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
raycaster.setFromCamera(pointer, camera);
const p = raycaster.ray.intersectPlane(ground, new THREE.Vector3()); // null if parallel/behind
```

The `InteractionManager` addon (`three/addons/interaction/`) is for `HTMLTexture` DOM events, not
picking. HTML UI in 3D: `HTMLMesh` + `InteractiveGroup` (`threejs-objects`).

## Performance

Raycast a short candidate list (not `scene.children`), use proxies and layers, throttle hover
to once per frame, and use `three-mesh-bvh` for big meshes.

See also: `threejs-math`, `threejs-objects`, `threejs-webxr`.
