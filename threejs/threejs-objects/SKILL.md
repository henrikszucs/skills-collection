---
name: threejs-objects
description: Three.js scene objects and helpers beyond basic meshes - Sprite, BatchedMesh, LOD, fat/thick lines (Line2, LineMaterial), HTML labels and DOM overlays (CSS2DRenderer, CSS3DRenderer), HTMLMesh UI in 3D, debug helpers (Grid, Axes, Arrow, Box, Plane, VertexNormals), clipping planes and section views, Lensflare, SkinnedMesh/Bone basics. Use when adding labels or tooltips to 3D objects, drawing thick lines, rendering many different meshes in one draw call, distance-based detail, billboards, debug visualization, or cutting/sectioning models.
---

# Three.js Objects, Labels, Lines, Helpers, Clipping

> three.js **r187**. See `threejs-migration` for renamed APIs.

## Sprite (faces the camera)

```javascript
const sprite = new THREE.Sprite(new THREE.SpriteMaterial({ map, sizeAttenuation: true, transparent: true }));
sprite.scale.set(2, 2, 1);  // world units
sprite.center.set(0.5, 0);  // anchor bottom-center
```

Text sprite: draw on a canvas and use a `CanvasTexture`. Thousands: `Points` with a texture, or
instanced sprites on WebGPU (`sprite.count`).

## BatchedMesh (different geometries, one draw call)

```javascript
const batched = new THREE.BatchedMesh(maxInstances, maxVertices, maxIndices, material);
const boxId = batched.addGeometry(boxGeometry);
const sphereId = batched.addGeometry(sphereGeometry);
const id = batched.addInstance(boxId);
batched.setMatrixAt(id, matrix);
batched.setColorAt(id, color);
batched.setVisibleAt(id, false); batched.deleteInstance(id); batched.setGeometryIdAt(id, sphereId);
batched.optimize(); // compact after deletions; raycast hits report hit.batchId
```

All geometries need the same attributes and must be all indexed or all non-indexed. Size
`maxVertices`/`maxIndices` for the sum of all added geometries.

## LOD

```javascript
const lod = new THREE.LOD();
lod.addLevel(high, 0); lod.addLevel(mid, 30); lod.addLevel(low, 80, 5); // distance, hysteresis
lod.addLevel(new THREE.Object3D(), 200); // nothing beyond 200
```

## Thick Lines

`THREE.Line` is always 1px. For width:

```javascript
import { Line2 } from "three/addons/lines/Line2.js";
import { LineMaterial } from "three/addons/lines/LineMaterial.js";
import { LineGeometry } from "three/addons/lines/LineGeometry.js";

const geometry = new LineGeometry().setFromPoints(curve.getPoints(100)); // or setPositions([...])
const line = new Line2(geometry, new LineMaterial({ color: 0xffffff, linewidth: 4, worldUnits: false, alphaToCoverage: true }));
line.computeLineDistances(); // for dashed: true
```

- `linewidth` is pixels, or world units with `worldUnits: true`. Resolution is set automatically.
- Pairs: `LineSegments2` + `LineSegmentsGeometry().fromEdgesGeometry(new THREE.EdgesGeometry(g))`.
- WebGPU: `Line2` from `three/addons/lines/webgpu/Line2.js` + `Line2NodeMaterial` (color via
  `colorNode`; `lineColorNode` is deprecated since r185).
- Lit tubes: `TubeGeometry`.

## HTML Labels (CSS2DRenderer)

```javascript
import { CSS2DRenderer, CSS2DObject } from "three/addons/renderers/CSS2DRenderer.js";

const labelRenderer = new CSS2DRenderer();
labelRenderer.setSize(innerWidth, innerHeight);
Object.assign(labelRenderer.domElement.style, { position: "absolute", top: "0", pointerEvents: "none" });
document.body.appendChild(labelRenderer.domElement);

const div = document.createElement("div");
div.className = "label";
div.textContent = "Earth";
const label = new CSS2DObject(div);
label.position.set(0, 1.2, 0); // relative to parent
label.center.set(0.5, 1);
earth.add(label);

// loop: renderer.render(scene, camera); labelRenderer.render(scene, camera);
// resize: labelRenderer.setSize(w, h)
```

- Labels are **not occluded** by geometry; hide them yourself (raycast or distance).
- `label.removeFromParent()` also removes the DOM element. Re-enable `pointerEvents: "auto"` per label to click it.
- Hundreds of labels are fine; thousands need sprites/SDF text.

**CSS3DRenderer** (`CSS3DObject`, `CSS3DSprite`) transforms DOM elements (iframes, panels) in 3D;
scale them down (`0.005`) and render a separate `cssScene` with the same camera. It cannot
depth-mix with WebGL except with a "hole" mesh trick.

**HTMLMesh** (`three/addons/interactive/HTMLMesh.js`) renders a DOM element onto a plane; put it in an
`InteractiveGroup` with `listenToPointerEvents(renderer, camera)` or
`listenToXRControllerEvents(controller)` for clickable UI in 3D/VR.

## Helpers

```javascript
scene.add(new THREE.AxesHelper(5));            // X red, Y green, Z blue
scene.add(new THREE.GridHelper(10, 10));
scene.add(new THREE.ArrowHelper(dir.normalize(), origin, length, 0xffff00));
const bh = new THREE.BoxHelper(mesh); scene.add(bh); // bh.update() after moving
scene.add(new THREE.Box3Helper(box3), new THREE.PlaneHelper(plane, 5), new THREE.SkeletonHelper(root));
// addons: VertexNormalsHelper, VertexTangentsHelper, ViewHelper (axis gizmo: viewHelper.render(renderer) with autoClear = false)
```

Light helpers: `threejs-lighting`. Stats: `three/addons/libs/stats.module.js`. GUI: `three/addons/libs/lil-gui.module.min.js`.

## Clipping Planes

```javascript
renderer.localClippingEnabled = true;
const plane = new THREE.Plane(new THREE.Vector3(0, -1, 0), 0.8); // keeps the side the normal points to (y < 0.8)
const mat = new THREE.MeshStandardMaterial({
  side: THREE.DoubleSide,  // see the inside through the cut
  clippingPlanes: [plane],
  clipShadows: true,
  clipIntersection: false, // true: clip only where all planes agree
});
renderer.clippingPlanes = [globalPlane]; // affects everything
plane.constant = Math.sin(t) * 0.5;      // animate the cut
```

Solid caps need the stencil technique (`webgl_clipping_stencil`). WebGPU: `THREE.ClippingGroup`
applies planes to its children.

## Lens Flare

```javascript
import { Lensflare, LensflareElement } from "three/addons/objects/Lensflare.js"; // WebGPU: LensflareMesh
const flare = new Lensflare();
flare.addElement(new LensflareElement(flareTexture, 512, 0));
pointLight.add(flare);
```

## SkinnedMesh Basics

Usually from glTF. Manually: build a `Bone` chain, give the geometry `skinIndex` (Uint16 ×4) and
`skinWeight` (Float32 ×4) attributes, then `skinned.add(rootBone); skinned.bind(new THREE.Skeleton(bones))`.
Animating, cloning, retargeting: `threejs-animation`.

See also: `threejs-geometry`, `threejs-interaction`, `threejs-webxr`.
