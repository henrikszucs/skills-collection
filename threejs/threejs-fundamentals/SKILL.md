---
name: threejs-fundamentals
description: Three.js scene setup, cameras, WebGLRenderer, render loop, Object3D hierarchy, coordinate system, color management, tone mapping, resizing and disposal. Use when starting a Three.js project, creating a scene/camera/renderer, writing the animation loop, handling resize, managing object hierarchies and transforms, or choosing between WebGL and WebGPU.
---

# Three.js Fundamentals

> **Version:** written for three.js **r186** (npm `three@0.186.x`). Check the installed
> version in `node_modules/three/package.json` before relying on an API. If something here
> is missing, throws, or logs a deprecation warning, see `threejs-migration`.

## Quick Start

```javascript
import * as THREE from "three";

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
camera.position.z = 5;

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const cube = new THREE.Mesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial({ color: 0x00ff00 }),
);
scene.add(cube);

// Lights use physical units (see threejs-lighting): directional ~1-5, point/spot in candela (10-1000+)
scene.add(new THREE.AmbientLight(0xffffff, 0.5));
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.position.set(5, 5, 5);
scene.add(dirLight);

// THREE.Clock is deprecated (r183) - use THREE.Timer
const timer = new THREE.Timer();
timer.connect(document); // avoids a huge delta after the tab was hidden

// setAnimationLoop instead of requestAnimationFrame (required for WebXR and WebGPU)
renderer.setAnimationLoop((timestamp) => {
  timer.update(timestamp);
  const delta = timer.getDelta(); // seconds since last frame
  cube.rotation.x += delta;
  cube.rotation.y += delta;
  renderer.render(scene, camera);
});

window.addEventListener("resize", () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

**Imports:** core classes come from `"three"`. Addons (controls, loaders, post-processing,
etc.) come from `"three/addons/..."` (an alias of `three/examples/jsm/...`). Without a
bundler, use an import map:

```html
<script type="importmap">
  {
    "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@0.186.1/build/three.module.js",
      "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.186.1/examples/jsm/"
    }
  }
</script>
```

**WebGL or WebGPU?** `WebGLRenderer` (this skill) is the mature default and supports
`ShaderMaterial`/GLSL and `EffectComposer`. `WebGPURenderer` (imported from `"three/webgpu"`,
falls back to WebGL 2 automatically) uses node materials and TSL instead. Never mix the two
APIs in one app. See `threejs-webgpu-tsl`.

## Core Classes

### Scene

```javascript
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x000000); // solid color
scene.background = texture;                   // equirect (set texture.mapping) or CubeTexture
scene.environment = envMap;                   // IBL for all PBR materials
scene.backgroundBlurriness = 0.3;             // 0-1, needs a PMREM/equirect background
scene.backgroundIntensity = 1;
scene.backgroundRotation.y = Math.PI / 2;     // Euler
scene.environmentIntensity = 1;
scene.environmentRotation.y = Math.PI / 2;    // Euler
scene.fog = new THREE.Fog(0xffffff, 1, 100);   // linear (near, far)
scene.fog = new THREE.FogExp2(0xffffff, 0.02); // exponential (density)
scene.overrideMaterial = null;                // force one material for everything
```

How to build `envMap`: see `threejs-textures` (Environment Maps).

### Cameras

**PerspectiveCamera(fov, aspect, near, far)**: fov is vertical, in degrees.

```javascript
const camera = new THREE.PerspectiveCamera(50, width / height, 0.1, 1000);
camera.position.set(0, 5, 10);
camera.lookAt(0, 0, 0);
camera.zoom = 1;
camera.updateProjectionMatrix(); // REQUIRED after changing fov, aspect, near, far, zoom
```

Keep `far / near` as small as possible; a tiny `near` (e.g. 0.0001) causes z-fighting. For huge
scenes use `new THREE.WebGLRenderer({ logarithmicDepthBuffer: true })` or `reversedDepthBuffer: true`.

**OrthographicCamera(left, right, top, bottom, near, far)**: no perspective, for 2D/isometric/CAD.

```javascript
const frustumSize = 10;
let aspect = window.innerWidth / window.innerHeight;
const camera = new THREE.OrthographicCamera(
  (-frustumSize * aspect) / 2, (frustumSize * aspect) / 2,
  frustumSize / 2, -frustumSize / 2,
  0.1, 1000,
);

// On resize, update the frustum, not .aspect
function onResize() {
  aspect = window.innerWidth / window.innerHeight;
  camera.left = (-frustumSize * aspect) / 2;
  camera.right = (frustumSize * aspect) / 2;
  camera.top = frustumSize / 2;
  camera.bottom = -frustumSize / 2;
  camera.updateProjectionMatrix();
}
```

**ArrayCamera**: several sub-cameras rendered in one pass. `viewport` is in **physical pixels**
(CSS pixels × devicePixelRatio), not 0-1.

```javascript
const AMOUNT = 2;
const W = (window.innerWidth / AMOUNT) * window.devicePixelRatio;
const H = (window.innerHeight / AMOUNT) * window.devicePixelRatio;
const cameras = [];
for (let y = 0; y < AMOUNT; y++) {
  for (let x = 0; x < AMOUNT; x++) {
    const sub = new THREE.PerspectiveCamera(40, W / H, 0.1, 100);
    sub.viewport = new THREE.Vector4(Math.floor(x * W), Math.floor(y * H), Math.ceil(W), Math.ceil(H));
    sub.position.set(x - 0.5, 0.5 - y, 3);
    sub.lookAt(0, 0, 0);
    sub.updateMatrixWorld();
    cameras.push(sub);
  }
}
const arrayCamera = new THREE.ArrayCamera(cameras);
```

**CubeCamera**: renders 6 directions into a cube render target (dynamic reflections).

```javascript
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256, { type: THREE.HalfFloatType });
const cubeCamera = new THREE.CubeCamera(0.1, 1000, cubeRenderTarget);
scene.add(cubeCamera);
reflectiveMaterial.envMap = cubeRenderTarget.texture;

// each frame (expensive: 6 renders). Hide the reflective object while capturing.
reflectiveMesh.visible = false;
cubeCamera.position.copy(reflectiveMesh.position);
cubeCamera.update(renderer, scene);
reflectiveMesh.visible = true;
```

**StereoCamera**: `stereo.update(camera)` gives `stereo.cameraL` / `stereo.cameraR`. For real VR use `threejs-webxr`.

### WebGLRenderer

```javascript
const renderer = new THREE.WebGLRenderer({
  canvas: document.querySelector("#canvas"), // optional existing canvas
  antialias: true,                // MSAA on the default framebuffer
  alpha: true,                    // transparent canvas background
  powerPreference: "high-performance",
  preserveDrawingBuffer: false,   // true only if you need canvas.toDataURL() after rendering
  logarithmicDepthBuffer: false,
});

renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // cap for performance
renderer.setSize(width, height);           // also sets canvas CSS size (pass false as 3rd arg to skip)
renderer.setClearColor(0x000000, 1);       // color, alpha

// Color management. These are the defaults; you rarely need to set them.
renderer.outputColorSpace = THREE.SRGBColorSpace;
THREE.ColorManagement.enabled = true;

// Tone mapping (default NoToneMapping). Use one for HDR/PBR scenes.
renderer.toneMapping = THREE.ACESFilmicToneMapping;
// Others: NeutralToneMapping (keeps product colors accurate), AgXToneMapping (filmic, good with
// bright lights), ReinhardToneMapping, CineonToneMapping, LinearToneMapping
renderer.toneMappingExposure = 1.0;

// Shadows
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap; // default. PCFSoftShadowMap is deprecated (r186, falls back to PCF): use PCFShadowMap + light.shadow.radius
// Also: THREE.VSMShadowMap, THREE.BasicShadowMap

// Clipping planes (see threejs-objects)
renderer.localClippingEnabled = true;

renderer.render(scene, camera);
```

Useful extras:

```javascript
renderer.info.render.calls;       // draw calls last frame (set renderer.info.autoReset as needed)
renderer.info.render.triangles;
renderer.info.memory.geometries;  // live GPU geometries
renderer.info.memory.textures;
renderer.capabilities.getMaxAnisotropy();
renderer.capabilities.maxTextureSize;
await renderer.compileAsync(scene, camera); // pre-compile shaders to avoid first-frame stutter

// Multiple viewports in one canvas
renderer.setScissorTest(true);
renderer.setViewport(x, y, w, h);  // CSS pixels, origin bottom-left
renderer.setScissor(x, y, w, h);
renderer.render(scene, cameraA);
renderer.setScissorTest(false);

// Screenshot
renderer.render(scene, camera);
const dataURL = renderer.domElement.toDataURL("image/png"); // same frame as render, or preserveDrawingBuffer

// Stop and free everything
renderer.setAnimationLoop(null);
renderer.dispose();
```

### Color Management (important)

- Colors from hex/CSS (`0xff0000`, `"#ff0000"`, `"red"`) are treated as **sRGB** and converted to linear.
- `color.setRGB(r, g, b)` and `setHSL` take **linear** values by default. Pass
  `THREE.SRGBColorSpace` as the last argument if your numbers are sRGB (e.g. from a design tool).
- Color textures (`map`, `emissiveMap`, `sheenColorMap`, `specularColorMap`) need
  `texture.colorSpace = THREE.SRGBColorSpace`. Data textures (normal, roughness, metalness, AO) keep the default `NoColorSpace`.
- GLTFLoader sets texture color spaces for you.

### Object3D

Base class of Mesh, Group, Light, Camera, Points, Line, Sprite, etc.

```javascript
obj.position.set(x, y, z);
obj.rotation.set(x, y, z);              // Euler, radians, order obj.rotation.order = 'XYZ'
obj.quaternion.setFromAxisAngle(axis, angle); // rotation and quaternion stay in sync
obj.scale.set(1, 1, 1);
obj.lookAt(target);                     // Vector3 or x, y, z (world space)
obj.rotateY(angle);                     // relative, local axis
obj.translateZ(distance);               // move along local axis

// World-space queries (always pass a target)
obj.getWorldPosition(new THREE.Vector3());
obj.getWorldQuaternion(new THREE.Quaternion());
obj.getWorldScale(new THREE.Vector3());
obj.getWorldDirection(new THREE.Vector3()); // +Z axis of the object (cameras: -Z, the view direction)
obj.localToWorld(vec);                  // mutates vec
obj.worldToLocal(vec);

// Hierarchy
parent.add(child);
parent.remove(child);
child.removeFromParent();
parent.attach(child);   // re-parent while keeping the world transform
parent.clear();         // remove all children
scene.getObjectByName("Head");
scene.getObjectByProperty("uuid", id);
obj.traverse((o) => { if (o.isMesh) o.castShadow = true; });
obj.traverseVisible(cb);

// Misc
obj.visible = false;
obj.name = "player";
obj.userData.hp = 100;     // your own data (serialized by toJSON/GLTFExporter)
obj.renderOrder = 10;      // higher = later (mainly for transparent sorting)
obj.frustumCulled = true;  // default. Set false for objects moved in a vertex shader
obj.castShadow = true;
obj.receiveShadow = true;

// Layers (32 bits) for selective rendering/raycasting/lighting
obj.layers.set(1);        // only layer 1
obj.layers.enable(2);     // also layer 2
camera.layers.enable(1);  // camera renders objects that share a layer with it

// Matrices
obj.matrixAutoUpdate = false; // static objects: update manually
obj.updateMatrix();
obj.updateMatrixWorld(true);  // force world matrices now (e.g. before reading them outside the loop)
```

### Group and Mesh

```javascript
const group = new THREE.Group(); // empty transform container
group.add(mesh1, mesh2);
scene.add(group);

const mesh = new THREE.Mesh(geometry, material);
const multi = new THREE.Mesh(geometry, [matA, matB]); // one material per geometry group
```

For thousands of copies use `InstancedMesh` or `BatchedMesh` (see `threejs-geometry`, `threejs-objects`).

## Coordinate System

Right-handed, **Y-up**: +X right, +Y up, +Z toward the viewer. Cameras look down their local -Z.
Units are arbitrary but lights/physics assume **1 unit = 1 meter**. Angles are radians.

```javascript
scene.add(new THREE.AxesHelper(5)); // red=X, green=Y, blue=Z
scene.add(new THREE.GridHelper(10, 10));
```

## Math Essentials

Full reference (Box3, Sphere, Ray, Plane, Frustum, Spherical...) in `threejs-math`.

```javascript
const v = new THREE.Vector3(1, 2, 3);
v.add(b).sub(c).multiplyScalar(2).normalize(); // all mutate v and return it (chainable)
const copy = v.clone();
v.length(); v.distanceTo(b); v.dot(b);
v.cross(b);                // mutates v
v.lerp(target, alpha);
v.applyMatrix4(m); v.applyQuaternion(q);
v.project(camera);         // world -> NDC (-1..1)
v.unproject(camera);       // NDC -> world

const q = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI / 2);
q.slerp(targetQ, t);

const m = new THREE.Matrix4().compose(position, quaternion, scale);
m.decompose(position, quaternion, scale);

const c = new THREE.Color("#ff8800");
c.setHSL(0.5, 1, 0.5);
c.getHexString(); // "ff8800"

THREE.MathUtils.degToRad(90);
THREE.MathUtils.clamp(x, 0, 1);
THREE.MathUtils.damp(current, target, lambda, delta); // frame-rate independent smoothing
```

**Avoid allocations in the render loop:** create scratch `Vector3`/`Matrix4` objects once and reuse them.

## Common Patterns

### Timer (replaces Clock)

```javascript
const timer = new THREE.Timer();
timer.connect(document);

renderer.setAnimationLoop((timestamp) => {
  timer.update(timestamp);         // call once per frame, first
  const dt = timer.getDelta();     // seconds
  const t = timer.getElapsed();    // seconds
  timer.setTimescale(1);           // slow motion / pause with 0
  renderer.render(scene, camera);
});
```

### Responsive Canvas (container, not window)

```javascript
const container = document.getElementById("app");
const observer = new ResizeObserver(() => {
  const { clientWidth: w, clientHeight: h } = container;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
});
observer.observe(container);
```

### Render on Demand (static scenes, saves battery)

```javascript
function render() { renderer.render(scene, camera); }
controls.addEventListener("change", render); // OrbitControls without damping
render();
```

### Proper Cleanup

`scene.remove()` does NOT free GPU memory. Dispose geometries, materials and textures yourself:

```javascript
function disposeObject(root) {
  root.traverse((obj) => {
    if (obj.geometry) obj.geometry.dispose();
    const materials = Array.isArray(obj.material) ? obj.material : [obj.material];
    for (const mat of materials) {
      if (!mat) continue;
      for (const value of Object.values(mat)) {
        if (value && value.isTexture) value.dispose();
      }
      mat.dispose();
    }
    if (obj.isSkinnedMesh) obj.skeleton.dispose();
  });
  root.removeFromParent();
}

// Tear down the whole app
renderer.setAnimationLoop(null);
disposeObject(scene);
controls?.dispose();
renderer.dispose();
renderer.domElement.remove();
```

Shared geometries/materials/textures should only be disposed when nothing uses them anymore.

## Performance Tips

1. **Draw calls matter most**: merge static meshes (`mergeGeometries`), use `InstancedMesh` / `BatchedMesh`.
2. **Cap pixel ratio** at 2 (or lower on mobile).
3. **Frustum culling** is automatic; keep bounding volumes correct (`geometry.computeBoundingSphere()` after editing vertices).
4. **LOD**: `THREE.LOD` swaps meshes by distance (see `threejs-objects`).
5. **Static objects**: `matrixAutoUpdate = false` + one `updateMatrix()`.
6. **Pre-compile**: `renderer.compileAsync(scene, camera)` before showing the scene.
7. **Check `renderer.info`** for draw calls and leaks.

```javascript
import { mergeGeometries } from "three/addons/utils/BufferGeometryUtils.js";
const merged = mergeGeometries([geo1, geo2, geo3]); // same attributes required
```

## See Also

- `threejs-migration` - Renamed/removed APIs, version checks
- `threejs-webgpu-tsl` - WebGPURenderer, node materials, TSL
- `threejs-geometry`, `threejs-materials`, `threejs-lighting`, `threejs-textures`
- `threejs-math` - Vectors, matrices, bounding volumes, rays, planes
- `threejs-objects` - Sprites, BatchedMesh, LOD, fat lines, HTML labels, helpers, clipping
