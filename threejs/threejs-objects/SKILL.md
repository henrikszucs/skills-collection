---
name: threejs-objects
description: Three.js scene objects and helpers beyond basic meshes - Sprite, BatchedMesh, LOD, fat/thick lines (Line2, LineMaterial), HTML labels and DOM overlays (CSS2DRenderer, CSS3DRenderer), HTMLMesh UI in 3D, debug helpers (Grid, Axes, Arrow, Box, Plane, VertexNormals), clipping planes and section views, Lensflare, SkinnedMesh/Bone basics. Use when adding labels or tooltips to 3D objects, drawing thick lines, rendering many different meshes in one draw call, distance-based detail, billboards, debug visualization, or cutting/sectioning models.
---

# Three.js Objects, Labels, Lines, Helpers, Clipping

> **Version:** written for three.js **r186**. See `threejs-migration` for renamed APIs.

## Sprite (always faces the camera)

```javascript
import * as THREE from "three";

const map = await new THREE.TextureLoader().loadAsync("icon.png");
map.colorSpace = THREE.SRGBColorSpace;

const sprite = new THREE.Sprite(new THREE.SpriteMaterial({
  map,
  color: 0xffffff,
  sizeAttenuation: true, // false = constant screen size (scale is then in NDC-ish units)
  depthTest: true,
  transparent: true,
}));
sprite.scale.set(2, 2, 1);   // size in world units
sprite.center.set(0.5, 0);   // anchor at bottom center (default 0.5, 0.5)
sprite.material.rotation = Math.PI / 4;
scene.add(sprite);
```

Text sprite: draw text on a canvas, use it as a `CanvasTexture` map (see `threejs-textures`).
Thousands of sprites: use `THREE.Points` with a texture, or instanced sprites in WebGPU (`sprite.count`).

## BatchedMesh (many different geometries, one draw call)

`InstancedMesh` = one geometry many times. `BatchedMesh` = several geometries (and instances of each) sharing one material.

```javascript
const box = new THREE.BoxGeometry(1, 1, 1);
const sphere = new THREE.SphereGeometry(0.5, 16, 8);

const maxInstances = 1000;
const maxVertices = box.attributes.position.count + sphere.attributes.position.count;
const maxIndices = box.index.count + sphere.index.count;

const batched = new THREE.BatchedMesh(maxInstances, maxVertices, maxIndices, new THREE.MeshStandardMaterial());
const boxId = batched.addGeometry(box);
const sphereId = batched.addGeometry(sphere);

const m = new THREE.Matrix4();
const color = new THREE.Color();
for (let i = 0; i < maxInstances; i++) {
  const instanceId = batched.addInstance(i % 2 ? boxId : sphereId);
  m.makeTranslation(Math.random() * 40 - 20, 0, Math.random() * 40 - 20);
  batched.setMatrixAt(instanceId, m);
  batched.setColorAt(instanceId, color.setHSL(Math.random(), 0.6, 0.5));
}
scene.add(batched);

batched.setVisibleAt(instanceId, false);
batched.deleteInstance(instanceId);
batched.setGeometryIdAt(instanceId, sphereId); // swap shape
batched.perObjectFrustumCulled = true; // default
batched.sortObjects = true;            // default (helps transparency/overdraw)
batched.optimize();                    // compact after deletions
// Raycast hits report hit.batchId
```

All geometries need the same attributes (and all indexed or all non-indexed).

## LOD (level of detail)

```javascript
const lod = new THREE.LOD();
lod.addLevel(highMesh, 0);    // object, distance from camera
lod.addLevel(midMesh, 30);
lod.addLevel(lowMesh, 80, 5); // hysteresis avoids flicker at the boundary
lod.addLevel(new THREE.Object3D(), 200); // nothing beyond 200
scene.add(lod);
// lod.autoUpdate = true (default): updated during render. lod.getCurrentLevel()
```

## Thick Lines (Line2 / LineMaterial)

WebGL draws `THREE.Line` at 1px regardless of `linewidth`. For width control:

```javascript
import { Line2 } from "three/addons/lines/Line2.js";
import { LineMaterial } from "three/addons/lines/LineMaterial.js";
import { LineGeometry } from "three/addons/lines/LineGeometry.js";

const geometry = new LineGeometry();
geometry.setFromPoints(curve.getPoints(100)); // or setPositions([x,y,z, ...])
geometry.setColors(colorsArray);              // optional per-vertex colors (flat array)

const material = new LineMaterial({
  color: 0xffffff,
  linewidth: 4,          // pixels; with worldUnits: true it is in world units
  worldUnits: false,
  vertexColors: false,
  dashed: false,
  alphaToCoverage: true, // smoother edges with MSAA
});

const line = new Line2(geometry, material);
line.computeLineDistances(); // needed for dashes
scene.add(line);
// Resolution is set automatically from the renderer viewport each frame.
```

Segments (pairs): `LineSegments2` + `LineSegmentsGeometry` (`fromEdgesGeometry(new THREE.EdgesGeometry(g))`
for thick outlines). Thick wireframe: `Wireframe` + `WireframeGeometry2`.
WebGPU: `import { Line2 } from "three/addons/lines/webgpu/Line2.js"` with `THREE.Line2NodeMaterial`.

Alternative for tubes with lighting: `TubeGeometry` (see `threejs-geometry`).

## HTML Labels (CSS2DRenderer)

DOM elements that follow 3D positions. Crisp text, CSS styling, clickable.

```javascript
import { CSS2DRenderer, CSS2DObject } from "three/addons/renderers/CSS2DRenderer.js";

const labelRenderer = new CSS2DRenderer();
labelRenderer.setSize(window.innerWidth, window.innerHeight);
labelRenderer.domElement.style.position = "absolute";
labelRenderer.domElement.style.top = "0";
labelRenderer.domElement.style.pointerEvents = "none"; // let the canvas/OrbitControls get events
document.body.appendChild(labelRenderer.domElement);

const div = document.createElement("div");
div.className = "label"; // style with CSS
div.textContent = "Earth";
div.style.pointerEvents = "auto"; // re-enable for clickable labels

const label = new CSS2DObject(div);
label.position.set(0, 1.2, 0); // relative to the parent
label.center.set(0.5, 1);      // anchor: bottom center of the element
earthMesh.add(label);

renderer.setAnimationLoop(() => {
  renderer.render(scene, camera);
  labelRenderer.render(scene, camera); // same scene and camera
});

window.addEventListener("resize", () => {
  labelRenderer.setSize(window.innerWidth, window.innerHeight);
});
```

- Labels are not occluded by geometry. Hide them yourself (raycast from camera to label, or check distance).
- `label.visible = false` hides; layers work too (`label.layers.set(1)` + camera layers).
- When removing, call `label.removeFromParent()`: the renderer then removes the DOM element.
- Hundreds of labels can be slow (DOM updates); for thousands use sprites/SDF text.

## CSS3DRenderer (DOM elements transformed in 3D)

```javascript
import { CSS3DRenderer, CSS3DObject, CSS3DSprite } from "three/addons/renderers/CSS3DRenderer.js";

const cssRenderer = new CSS3DRenderer();
cssRenderer.setSize(window.innerWidth, window.innerHeight);
cssRenderer.domElement.style.position = "absolute";
cssRenderer.domElement.style.top = "0";
document.body.appendChild(cssRenderer.domElement);

const iframe = document.createElement("iframe");
iframe.src = "https://example.com";
iframe.style.width = "1024px";
iframe.style.height = "768px";
iframe.style.border = "0";

const panel = new CSS3DObject(iframe);
panel.scale.setScalar(0.005); // CSS pixels -> world units
cssScene.add(panel);          // usually a separate scene

renderer.setAnimationLoop(() => {
  renderer.render(scene, camera);
  cssRenderer.render(cssScene, camera);
});
```

CSS3D elements are always drawn on top of or below the WebGL canvas (no real depth mixing).
To mix, put a transparent-to-black "hole" mesh (`MeshBasicMaterial({ opacity: 0, blending: THREE.NoBlending })`) where the element is, and place the WebGL canvas above the CSS layer.

## HTML UI Inside the 3D Scene (HTMLMesh)

Renders a DOM element into a texture on a plane. Interactive through `InteractiveGroup` (great for VR menus).

```javascript
import { HTMLMesh } from "three/addons/interactive/HTMLMesh.js";
import { InteractiveGroup } from "three/addons/interactive/InteractiveGroup.js";

const group = new InteractiveGroup();
group.listenToPointerEvents(renderer, camera);
group.listenToXRControllerEvents(controller); // VR controllers (see threejs-webxr)
scene.add(group);

const mesh = new HTMLMesh(document.getElementById("panel")); // supports basic HTML/CSS, inputs, lil-gui
mesh.position.set(0, 1.5, -1);
group.add(mesh);
```

## Debug Helpers

```javascript
scene.add(new THREE.AxesHelper(5));                       // X red, Y green, Z blue
scene.add(new THREE.GridHelper(10, 10, 0x888888, 0x444444)); // size, divisions, centerColor, gridColor
scene.add(new THREE.PolarGridHelper(10, 16, 8, 64));
scene.add(new THREE.ArrowHelper(dir.normalize(), origin, 2, 0xffff00)); // dir must be normalized

const boxHelper = new THREE.BoxHelper(mesh, 0xffff00);  // world AABB of an object
scene.add(boxHelper);
boxHelper.update(); // after the object moves

scene.add(new THREE.Box3Helper(box3, 0x00ff00));        // a Box3 you computed
scene.add(new THREE.PlaneHelper(plane, 5, 0xff0000));   // a THREE.Plane
scene.add(new THREE.CameraHelper(shadowCamera));
scene.add(new THREE.SkeletonHelper(characterRoot));

import { VertexNormalsHelper } from "three/addons/helpers/VertexNormalsHelper.js";
import { VertexTangentsHelper } from "three/addons/helpers/VertexTangentsHelper.js";
scene.add(new VertexNormalsHelper(mesh, 0.1, 0xff0000));

import { ViewHelper } from "three/addons/helpers/ViewHelper.js"; // axis gizmo in a corner
const viewHelper = new ViewHelper(camera, renderer.domElement);
renderer.autoClear = false;
renderer.setAnimationLoop(() => {
  renderer.clear();
  renderer.render(scene, camera);
  viewHelper.render(renderer);
});
```

Light helpers: see `threejs-lighting`. Stats: `import Stats from "three/addons/libs/stats.module.js"`.
GUI: `import GUI from "three/addons/libs/lil-gui.module.min.js"`.

## Clipping Planes (cut-away / section views)

```javascript
renderer.localClippingEnabled = true; // required for per-material planes

// Keeps the side the normal points to. Plane(normal, constant): normal . p + constant = 0
const plane = new THREE.Plane(new THREE.Vector3(0, -1, 0), 0.8); // keep everything below y = 0.8

const material = new THREE.MeshStandardMaterial({
  color: 0x80ee10,
  side: THREE.DoubleSide, // show the inside through the cut
  clippingPlanes: [plane],
  clipShadows: true,
  clipIntersection: false, // true: clip only where ALL planes agree (corner cut)
});

// Global planes affect every object
renderer.clippingPlanes = [new THREE.Plane(new THREE.Vector3(-1, 0, 0), 0.1)];

// Move the cut: change plane.constant (or plane.normal) each frame
plane.constant = Math.sin(time) * 0.5;
```

Closed-looking caps on cut solids need the stencil technique (official example
`webgl_clipping_stencil`). WebGPU: `THREE.ClippingGroup` (`clippingPlanes`, `clipIntersection`,
`clipShadows`) applies planes to all of its children.

## Lens Flare

```javascript
import { Lensflare, LensflareElement } from "three/addons/objects/Lensflare.js";
const flareTexture = await new THREE.TextureLoader().loadAsync("lensflare0.png");
const lensflare = new Lensflare();
lensflare.addElement(new LensflareElement(flareTexture, 512, 0, new THREE.Color(0xffffff)));
pointLight.add(lensflare);
// WebGPU: LensflareMesh (three/addons/objects/LensflareMesh.js)
```

## SkinnedMesh and Bones (basics)

Usually loaded from glTF. Manual construction:

```javascript
const bones = [];
let prev = null;
for (let i = 0; i < 3; i++) {
  const bone = new THREE.Bone();
  bone.position.y = i === 0 ? 0 : 1;
  if (prev) prev.add(bone);
  bones.push(bone);
  prev = bone;
}
// geometry needs skinIndex (Uint16 x4) and skinWeight (Float32 x4) attributes
const skinned = new THREE.SkinnedMesh(geometry, material);
skinned.add(bones[0]);
skinned.bind(new THREE.Skeleton(bones));
bones[1].rotation.z = 0.5; // pose
```

Animating, cloning and retargeting skinned characters: see `threejs-animation`.

## See Also

- `threejs-geometry` - InstancedMesh, Points, basic lines
- `threejs-interaction` - Picking, world-to-screen
- `threejs-math` - Box3, Plane, bounding volumes
- `threejs-webxr` - VR controllers with InteractiveGroup
