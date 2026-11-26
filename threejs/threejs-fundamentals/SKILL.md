---
name: threejs-fundamentals
description: Three.js scene setup, cameras, WebGLRenderer, render loop, Object3D hierarchy, coordinate system, color management, tone mapping, resizing and disposal. Use when starting a Three.js project, creating a scene/camera/renderer, writing the animation loop, handling resize, managing object hierarchies and transforms, or choosing between WebGL and WebGPU.
---

# Three.js Fundamentals

> three.js **r187** (`three@0.187.x`). Check `node_modules/three/package.json`; if an API here
> throws or warns, see `threejs-migration`.

## Quick Start

```javascript
import * as THREE from "three";

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(50, innerWidth / innerHeight, 0.1, 1000);
camera.position.set(0, 1, 5);

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.setSize(innerWidth, innerHeight);
renderer.toneMapping = THREE.ACESFilmicToneMapping; // or NeutralToneMapping / AgXToneMapping
document.body.appendChild(renderer.domElement);

const cube = new THREE.Mesh(new THREE.BoxGeometry(), new THREE.MeshStandardMaterial({ color: 0x44aa88 }));
scene.add(cube);
scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 1));
const sun = new THREE.DirectionalLight(0xffffff, 3); // physical units, see threejs-lighting
sun.position.set(5, 5, 5);
scene.add(sun);

const timer = new THREE.Timer(); // Clock is deprecated (r183)
timer.connect(document);         // no huge delta after the tab was hidden

renderer.setAnimationLoop((t) => { // not requestAnimationFrame: required for WebXR and WebGPU
  timer.update(t);
  const dt = timer.getDelta();     // seconds; also getElapsed(), setTimescale()
  cube.rotation.y += dt;
  renderer.render(scene, camera);
});

addEventListener("resize", () => {
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth, innerHeight);
});
```

**Imports:** core from `"three"`, addons from `"three/addons/..."` (alias of `three/examples/jsm/`).
Without a bundler use an import map with a **pinned** version:

```html
<script type="importmap">
{ "imports": {
  "three": "https://cdn.jsdelivr.net/npm/three@0.187.0/build/three.module.js",
  "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.187.0/examples/jsm/"
} }
</script>
```

**WebGL or WebGPU?** `WebGLRenderer` is the mature default (GLSL `ShaderMaterial`, `EffectComposer`).
`WebGPURenderer` (`"three/webgpu"`, auto-falls back to WebGL 2) uses node materials + TSL.
Pick one per app; see `threejs-webgpu-tsl`.

## Scene

```javascript
scene.background = new THREE.Color(0x111111); // or a texture / cube texture
scene.environment = envMap;                   // IBL for all PBR materials (see threejs-textures)
scene.backgroundBlurriness = 0.3;             // also backgroundIntensity, backgroundRotation
scene.environmentIntensity = 1;               // also environmentRotation
scene.fog = new THREE.Fog(0xffffff, 1, 100);  // or FogExp2(color, density)
```

## Cameras

- `PerspectiveCamera(fov, aspect, near, far)`: vertical fov in degrees. Call
  `updateProjectionMatrix()` after changing fov/aspect/near/far/zoom. Keep `near` as large as
  possible; tiny values cause z-fighting. Huge scenes: `reversedDepthBuffer: true` (or
  `logarithmicDepthBuffer`) on the renderer.
- `OrthographicCamera(left, right, top, bottom, near, far)`: on resize update the four frustum
  planes (not `.aspect`), then `updateProjectionMatrix()`.
- `CubeCamera(near, far, cubeRenderTarget)`: dynamic reflections; `cubeCamera.update(renderer, scene)`
  (6 renders, hide the reflective object while capturing).
- `ArrayCamera(cameras)`: sub-cameras with `viewport` in **physical** pixels (CSS × devicePixelRatio).

## Renderer

```javascript
const renderer = new THREE.WebGLRenderer({
  canvas,                       // optional existing <canvas>
  antialias: true,
  alpha: true,                  // transparent background
  powerPreference: "high-performance",
  preserveDrawingBuffer: false, // true only for toDataURL() outside the render call
});
renderer.setClearColor(0x000000, 1);
renderer.outputColorSpace = THREE.SRGBColorSpace; // default
renderer.toneMappingExposure = 1;
renderer.shadowMap.enabled = true;                // type PCFShadowMap (default) or VSMShadowMap
renderer.localClippingEnabled = true;             // per-material clipping planes

renderer.info.render.calls;                // draw calls; info.memory.geometries/textures for leaks
await renderer.compileAsync(scene, camera); // pre-compile shaders, avoids first-frame stutter
```

Multiple views in one canvas: `setScissorTest(true)`, then `setViewport`/`setScissor` (CSS pixels,
origin bottom-left) per view. Since r187 they no longer scale by pixel ratio while a render
target is bound: pass render-target pixels there.

## Color Management

- Hex/CSS colors (`0xff0000`, `"#f00"`) are sRGB and converted to linear for you.
- `color.setRGB(r, g, b)` / `setHSL` take **linear** values; pass `THREE.SRGBColorSpace` as the
  last argument for sRGB numbers.
- Color textures (`map`, `emissiveMap`, ...) need `texture.colorSpace = THREE.SRGBColorSpace`;
  data textures (normal, roughness, AO) keep `NoColorSpace`. GLTFLoader does this for you.

## Object3D Essentials

```javascript
obj.position.set(x, y, z);
obj.rotation.set(x, y, z);        // Euler radians; quaternion stays in sync
obj.lookAt(target);               // world space
obj.getWorldPosition(new THREE.Vector3()); // world queries always take a target
parent.attach(child);             // re-parent keeping the world transform
obj.traverse((o) => { if (o.isMesh) o.castShadow = true; });
obj.layers.set(1); camera.layers.enable(1); // selective rendering/raycasting
obj.renderOrder = 10;             // transparent sorting
obj.frustumCulled = false;        // for objects moved in a vertex shader
obj.matrixAutoUpdate = false; obj.updateMatrix(); // static objects
```

Right-handed, **Y-up**, cameras look down -Z, angles in radians, **1 unit = 1 meter** (lights
and physics assume it). Math API: `threejs-math`.

## Resizing a Container (not the window)

```javascript
new ResizeObserver(() => {
  const { clientWidth: w, clientHeight: h } = container;
  camera.aspect = w / h;
  camera.updateProjectionMatrix();
  renderer.setSize(w, h);
}).observe(container);
```

Static scenes: render on demand (`controls.addEventListener("change", render)`) instead of a loop.

## Cleanup

`scene.remove()` does not free GPU memory. Dispose explicitly:

```javascript
function disposeObject(root) {
  root.traverse((o) => {
    o.geometry?.dispose();
    for (const m of [o.material].flat()) {
      if (!m) continue;
      for (const v of Object.values(m)) if (v?.isTexture) v.dispose();
      m.dispose();
    }
    if (o.isSkinnedMesh) o.skeleton.dispose();
  });
  root.removeFromParent();
}
// whole app: renderer.setAnimationLoop(null); disposeObject(scene); controls?.dispose(); renderer.dispose();
```

Only dispose shared geometries/materials/textures when nothing else uses them. Custom
`Object3D` subclasses with their own `dispose()` must call `super.dispose()` (r186).

## Performance

1. Draw calls first: `mergeGeometries`, `InstancedMesh`, `BatchedMesh`.
2. Cap pixel ratio at 2 (lower on mobile).
3. Keep bounding volumes valid after editing vertices (`computeBoundingSphere()`).
4. `LOD` for distant detail; `matrixAutoUpdate = false` for static objects.
5. Reuse scratch `Vector3`/`Matrix4` objects in the loop; no allocations per frame.

See also: `threejs-migration`, `threejs-webgpu-tsl`, `threejs-math`, `threejs-objects`.
