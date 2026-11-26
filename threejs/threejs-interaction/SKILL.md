---
name: threejs-interaction
description: Three.js interaction - raycasting and object picking, pointer/touch input, hover and selection, camera controls (OrbitControls, MapControls, PointerLock, Fly, FirstPerson, Trackball, Arcball), TransformControls gizmo, DragControls, box selection, keyboard movement and world/screen coordinate conversion. Use when handling mouse/touch/keyboard input, implementing click or hover detection, adding camera controls, moving objects with a gizmo, or projecting between 3D and screen space.
---

# Three.js Interaction

> **Version:** written for three.js **r186**. All controls extend `THREE.Controls` (r169+):
> they are not `Object3D`s. See `threejs-migration` for renamed APIs.

## Quick Start

```javascript
import * as THREE from "three";
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;

const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();

function updatePointer(event) {
  const rect = renderer.domElement.getBoundingClientRect(); // works for any canvas size/position
  pointer.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
  pointer.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;
}

renderer.domElement.addEventListener("click", (event) => {
  updatePointer(event);
  raycaster.setFromCamera(pointer, camera);
  const hits = raycaster.intersectObjects(clickables, true);
  if (hits.length > 0) console.log("Clicked:", hits[0].object.name, hits[0].point);
});

renderer.setAnimationLoop(() => {
  controls.update(); // required for damping/autoRotate
  renderer.render(scene, camera);
});
```

## Raycaster

```javascript
const raycaster = new THREE.Raycaster();

raycaster.setFromCamera(pointerNDC, camera); // NDC: x, y in -1..1, y up
raycaster.set(origin, direction);            // any ray; direction must be normalized

const hits = raycaster.intersectObjects(objects, true); // recursive (default true)
const hit = raycaster.intersectObject(mesh, false);

// Each hit (sorted nearest first):
// {
//   distance,        // from ray origin
//   point,           // Vector3, world space
//   face,            // { a, b, c, normal, materialIndex }  (normal in object space)
//   faceIndex,
//   object,          // the Mesh/Points/Line hit (may be a child of what you passed)
//   uv, uv1,         // texture coordinates at the hit
//   normal,          // interpolated normal (object space)
//   instanceId,      // InstancedMesh
//   batchId,         // BatchedMesh
// }

const worldNormal = hit.face.normal.clone().transformDirection(hit.object.matrixWorld);
```

### Options

```javascript
raycaster.near = 0;
raycaster.far = 100;
raycaster.params.Points.threshold = 0.1; // world units around each point
raycaster.params.Line.threshold = 0.1;
raycaster.layers.set(1);                 // only objects on layer 1
raycaster.firstHitOnly = true;           // honored by three-mesh-bvh only
```

- Invisible objects (`visible = false`) are skipped. For invisible hit areas use
  `new THREE.MeshBasicMaterial({ visible: false })` on a visible mesh.
- `Sprite` raycasting needs `raycaster.camera`, which `setFromCamera` sets.
- Skinned/morphed meshes are raycast in their animated pose.
- Raycasting large meshes is slow (brute force over triangles). Use simpler proxy meshes or the
  `three-mesh-bvh` library (`computeBoundsTree`) for big models.

### Find the Logical Object (child mesh was hit)

```javascript
function findSelectable(object) {
  while (object && !object.userData.selectable) object = object.parent;
  return object;
}
const target = findSelectable(hits[0]?.object);
```

## Pointer and Touch Input

Use **pointer events**: one code path for mouse, touch and pen.

```javascript
const canvas = renderer.domElement;
canvas.style.touchAction = "none"; // stop the browser from scrolling/zooming on touch drags

canvas.addEventListener("pointerdown", onPointerDown);
canvas.addEventListener("pointermove", onPointerMove);
canvas.addEventListener("pointerup", onPointerUp);

// Distinguish click from drag (OrbitControls also uses pointerdown/up)
let downX = 0, downY = 0;
function onPointerDown(e) { downX = e.clientX; downY = e.clientY; }
function onPointerUp(e) {
  if (Math.hypot(e.clientX - downX, e.clientY - downY) > 5) return; // was a drag
  updatePointer(e);
  pick();
}
```

## Hover and Selection

```javascript
let hovered = null;
let selected = null;

function onPointerMove(event) {
  updatePointer(event);
  raycaster.setFromCamera(pointer, camera);
  const hit = raycaster.intersectObjects(selectables, false)[0];
  const object = hit ? hit.object : null;

  if (object !== hovered) {
    if (hovered) hovered.material.emissive.setHex(hovered.userData.baseEmissive);
    hovered = object;
    if (hovered) {
      hovered.userData.baseEmissive ??= hovered.material.emissive.getHex();
      hovered.material.emissive.setHex(0x333333);
    }
    canvas.style.cursor = hovered ? "pointer" : "default";
  }
}
```

Materials are often shared (especially in GLTF): clone before highlighting one object
(`mesh.material = mesh.material.clone()`), or use `OutlinePass` (see `threejs-postprocessing`).

Raycasting on every `pointermove` is fine for small scenes. For big scenes raycast once per
frame in the render loop using the latest pointer position, instead of in the event handler.

## Camera Controls

All controls share: `controls.enabled`, `controls.update(delta?)`, `controls.connect(element)`,
`controls.disconnect()`, `controls.dispose()`.

### OrbitControls

```javascript
import { OrbitControls } from "three/addons/controls/OrbitControls.js";

const controls = new OrbitControls(camera, renderer.domElement);
controls.target.set(0, 1, 0);      // orbit pivot
controls.update();                 // after changing target or camera position manually

controls.enableDamping = true;     // requires controls.update() every frame
controls.dampingFactor = 0.05;

controls.minDistance = 2;          // perspective zoom limits
controls.maxDistance = 50;
controls.minZoom = 0.5;            // orthographic zoom limits
controls.maxZoom = 4;
controls.minPolarAngle = 0;        // vertical limits (0 = top)
controls.maxPolarAngle = Math.PI / 2; // stop at the horizon
controls.minAzimuthAngle = -Math.PI / 4; // horizontal limits
controls.maxAzimuthAngle = Math.PI / 4;

controls.enableRotate = true;
controls.enableZoom = true;
controls.enablePan = true;
controls.screenSpacePanning = true; // false: pan parallel to the ground (map style)
controls.zoomToCursor = true;       // zoom toward the pointer

controls.autoRotate = true;
controls.autoRotateSpeed = 2.0;

controls.mouseButtons = { LEFT: THREE.MOUSE.ROTATE, MIDDLE: THREE.MOUSE.DOLLY, RIGHT: THREE.MOUSE.PAN };
controls.touches = { ONE: THREE.TOUCH.ROTATE, TWO: THREE.TOUCH.DOLLY_PAN };
controls.listenToKeyEvents(window); // arrow keys pan

controls.addEventListener("change", render); // render-on-demand
controls.addEventListener("start", () => {}); // user started interacting
controls.addEventListener("end", () => {});

controls.saveState();
controls.reset();
```

### MapControls

```javascript
import { MapControls } from "three/addons/controls/MapControls.js";
const controls = new MapControls(camera, renderer.domElement); // left = pan, right = rotate
controls.enableDamping = true;
controls.screenSpacePanning = false;
controls.maxPolarAngle = Math.PI / 2;
```

### PointerLockControls (first-person / FPS)

```javascript
import { PointerLockControls } from "three/addons/controls/PointerLockControls.js";

const controls = new PointerLockControls(camera, document.body);
scene.add(camera); // the camera itself is moved

startButton.addEventListener("click", () => controls.lock()); // needs a user gesture
controls.addEventListener("lock", () => (menu.style.display = "none"));
controls.addEventListener("unlock", () => (menu.style.display = "block"));

const keys = new Set();
document.addEventListener("keydown", (e) => keys.add(e.code));
document.addEventListener("keyup", (e) => keys.delete(e.code));

const velocity = new THREE.Vector3();
const speed = 5; // meters per second

renderer.setAnimationLoop((t) => {
  timer.update(t);
  const dt = timer.getDelta();
  if (controls.isLocked) {
    const forward = Number(keys.has("KeyW")) - Number(keys.has("KeyS"));
    const right = Number(keys.has("KeyD")) - Number(keys.has("KeyA"));
    velocity.x = THREE.MathUtils.damp(velocity.x, right * speed, 10, dt);
    velocity.z = THREE.MathUtils.damp(velocity.z, forward * speed, 10, dt);
    controls.moveRight(velocity.x * dt);
    controls.moveForward(velocity.z * dt); // horizontal plane, ignores pitch
  }
  renderer.render(scene, camera);
});
```

Collisions/gravity: use `Octree` + `Capsule` from addons (see `threejs-addons`) or a physics engine.

### FlyControls, FirstPersonControls

```javascript
import { FlyControls } from "three/addons/controls/FlyControls.js";
const fly = new FlyControls(camera, renderer.domElement);
fly.movementSpeed = 10;
fly.rollSpeed = Math.PI / 24;
fly.dragToLook = true;
// loop: fly.update(dt)  (needs delta)

import { FirstPersonControls } from "three/addons/controls/FirstPersonControls.js";
const fp = new FirstPersonControls(camera, renderer.domElement);
fp.movementSpeed = 10;
fp.lookSpeed = 0.1;
// loop: fp.update(dt)  (handleResize() was removed; nothing to call on resize)
```

### TrackballControls, ArcballControls

```javascript
import { TrackballControls } from "three/addons/controls/TrackballControls.js";
const tb = new TrackballControls(camera, renderer.domElement); // free rotation, no "up"
tb.rotateSpeed = 2.0;
// loop: tb.update(); on resize: tb.handleResize()

import { ArcballControls } from "three/addons/controls/ArcballControls.js";
const arc = new ArcballControls(camera, renderer.domElement, scene); // CAD-style with gizmo
arc.setGizmosVisible(false);
```

## TransformControls (move/rotate/scale gizmo)

```javascript
import { TransformControls } from "three/addons/controls/TransformControls.js";

const transform = new TransformControls(camera, renderer.domElement);
scene.add(transform.getHelper()); // add the HELPER, not the controls (r169+)

transform.attach(selectedMesh);
transform.setMode("translate");  // 'translate' | 'rotate' | 'scale'
transform.setSpace("local");     // 'local' | 'world'
transform.setSize(1);
transform.setTranslationSnap(0.5);
transform.setRotationSnap(THREE.MathUtils.degToRad(15));
transform.setScaleSnap(0.25);
transform.showX = true;

// Don't orbit while dragging the gizmo
transform.addEventListener("dragging-changed", (e) => {
  orbitControls.enabled = !e.value;
});
transform.addEventListener("objectChange", () => saveTransform(transform.object));
transform.addEventListener("change", render); // for render-on-demand

window.addEventListener("keydown", (e) => {
  if (e.key === "w") transform.setMode("translate");
  if (e.key === "e") transform.setMode("rotate");
  if (e.key === "r") transform.setMode("scale");
  if (e.key === "Escape") transform.detach();
});
```

Exclude the gizmo from your own picking: raycast only against your selectable list, not `scene.children`.

## DragControls

```javascript
import { DragControls } from "three/addons/controls/DragControls.js";

const drag = new DragControls(draggables, camera, renderer.domElement);
drag.recursive = true;       // child meshes count (default)
drag.transformGroup = false; // true: drag the top-level object instead of the child mesh

drag.addEventListener("dragstart", (e) => { orbitControls.enabled = false; });
drag.addEventListener("drag", (e) => { e.object.position.y = 0; }); // constrain to the ground
drag.addEventListener("dragend", (e) => { orbitControls.enabled = true; });
drag.addEventListener("hoveron", (e) => { canvas.style.cursor = "grab"; });
drag.addEventListener("hoveroff", () => { canvas.style.cursor = "auto"; });
```

Dragging moves objects on a plane facing the camera. For ground-plane dragging, raycast against a
`THREE.Plane` yourself (below).

## Box Selection

```javascript
import { SelectionBox } from "three/addons/interactive/SelectionBox.js";
import { SelectionHelper } from "three/addons/interactive/SelectionHelper.js";

const selectionBox = new SelectionBox(camera, scene);
const helper = new SelectionHelper(renderer, "selectBox"); // CSS class for the rectangle div
// CSS: .selectBox { border: 1px solid #55aaff; background: rgba(75,160,255,0.3); position: fixed; }

const toNDC = (e, z = 0.5) => {
  const r = renderer.domElement.getBoundingClientRect();
  return new THREE.Vector3(((e.clientX - r.left) / r.width) * 2 - 1, -((e.clientY - r.top) / r.height) * 2 + 1, z);
};

canvas.addEventListener("pointerdown", (e) => selectionBox.startPoint.copy(toNDC(e)));
canvas.addEventListener("pointermove", (e) => {
  if (helper.isDown) selectionBox.endPoint.copy(toNDC(e));
});
canvas.addEventListener("pointerup", (e) => {
  selectionBox.endPoint.copy(toNDC(e));
  const selected = selectionBox.select(); // meshes whose center is inside the box
  console.log(selected, selectionBox.instances); // instances: { [instancedMesh.uuid]: [instanceIds] }
});
// Disable OrbitControls while box-selecting (e.g. only when Shift is held)
```

## Keyboard Input

```javascript
const keys = new Set();
window.addEventListener("keydown", (e) => keys.add(e.code));
window.addEventListener("keyup", (e) => keys.delete(e.code));
window.addEventListener("blur", () => keys.clear()); // avoid stuck keys

function updatePlayer(dt) {
  const speed = 5 * dt;
  if (keys.has("KeyW")) player.translateZ(-speed);
  if (keys.has("KeyS")) player.translateZ(speed);
  if (keys.has("KeyA")) player.rotation.y += 2 * dt;
  if (keys.has("KeyD")) player.rotation.y -= 2 * dt;
}
```

Use `event.code` (physical key) for movement so WASD works on AZERTY keyboards.

## World and Screen Coordinates

### World to Screen (position HTML over 3D)

```javascript
const tmp = new THREE.Vector3();
function worldToScreen(object, camera, canvas) {
  object.getWorldPosition(tmp).project(camera); // NDC
  const rect = canvas.getBoundingClientRect();
  return {
    x: (tmp.x * 0.5 + 0.5) * rect.width + rect.left,
    y: (-tmp.y * 0.5 + 0.5) * rect.height + rect.top,
    visible: tmp.z > -1 && tmp.z < 1, // in front of the camera and within near/far
  };
}
```

For many labels prefer `CSS2DRenderer` (see `threejs-objects`): it does this for you.

### Pointer to a Point on a Plane (ground picking, dragging)

```javascript
const groundPlane = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0); // y = 0
const hitPoint = new THREE.Vector3();

function pointerToGround(event) {
  updatePointer(event);
  raycaster.setFromCamera(pointer, camera);
  return raycaster.ray.intersectPlane(groundPlane, hitPoint); // null if parallel/behind
}
```

### Screen Point at a Given Depth

```javascript
const v = new THREE.Vector3(pointer.x, pointer.y, 0.5).unproject(camera); // a point along the ray
const dir = v.sub(camera.position).normalize();
const pointAt10m = camera.position.clone().addScaledVector(dir, 10);
```

## Picking Helper Class

```javascript
class Picker {
  constructor(camera, canvas) {
    this.camera = camera;
    this.canvas = canvas;
    this.raycaster = new THREE.Raycaster();
    this.pointer = new THREE.Vector2();
    this.targets = [];
    this.onClick = this.onClick.bind(this);
    canvas.addEventListener("click", this.onClick);
  }
  add(object, callback) {
    object.userData.onClick = callback;
    this.targets.push(object);
  }
  remove(object) {
    this.targets = this.targets.filter((o) => o !== object);
  }
  onClick(event) {
    const rect = this.canvas.getBoundingClientRect();
    this.pointer.set(((event.clientX - rect.left) / rect.width) * 2 - 1, -((event.clientY - rect.top) / rect.height) * 2 + 1);
    this.raycaster.setFromCamera(this.pointer, this.camera);
    const hit = this.raycaster.intersectObjects(this.targets, true)[0];
    if (!hit) return;
    let o = hit.object;
    while (o && !o.userData.onClick) o = o.parent;
    o?.userData.onClick(hit);
  }
  dispose() {
    this.canvas.removeEventListener("click", this.onClick);
  }
}

const picker = new Picker(camera, renderer.domElement);
picker.add(mesh, (hit) => console.log("clicked at", hit.point));
```

The `InteractionManager` addon (`three/addons/interaction/InteractionManager.js`) is different:
it aligns `HTMLTexture` elements with meshes so the browser can deliver DOM events to them. For VR/AR
input see `threejs-webxr`; for HTML UI rendered in 3D (`HTMLMesh`, `InteractiveGroup`) see `threejs-objects`.

## Performance Tips

1. **Raycast a short list** of candidates, not `scene.children` recursively.
2. **Proxy geometry**: raycast against simple invisible boxes for complex models.
3. **Use layers** to filter raycast targets.
4. **three-mesh-bvh** for raycasting large meshes (orders of magnitude faster).
5. **Throttle hover** raycasts to once per frame.
6. **`controls.enabled = false`** when UI overlays need the pointer.

## See Also

- `threejs-fundamentals` - Camera and renderer setup
- `threejs-math` - Ray, Plane, projections
- `threejs-objects` - CSS2D labels, HTMLMesh
- `threejs-webxr` - Controller and hand input
