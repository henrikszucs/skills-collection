---
name: threejs-webgpu-tsl
description: Three.js WebGPURenderer and TSL (Three.js Shading Language) - setup and imports (three/webgpu, three/tsl), WebGL 2 fallback, node materials (colorNode, positionNode, ...), TSL syntax (Fn, uniform, If, Loop, toVar/assign), built-in nodes, uniforms, textures, instancing, compute shaders with storage buffers, RenderPipeline post-processing, WebGPU versions of addons, and what does not work (ShaderMaterial, onBeforeCompile, EffectComposer). Use when the user mentions WebGPU, TSL, node materials, compute shaders or GPU particles, or when writing shaders that should run on both WebGPU and WebGL.
---

# Three.js WebGPU Renderer and TSL

> three.js **r187**. The WebGPU/TSL API changes faster than the WebGL API; when in doubt, read
> `node_modules/three/src/nodes/` and the `examples/webgpu_*.html` of the installed version.

**Use it for** compute/GPU particles, one shader source for WebGPU **and** WebGL 2, and new
node-based projects. **Stay on WebGLRenderer** if you depend on GLSL `ShaderMaterial`,
`onBeforeCompile` or EffectComposer passes. `WebGPURenderer` falls back to a WebGL 2 backend
automatically; TSL compiles to WGSL or GLSL.

## Setup

```javascript
import * as THREE from "three/webgpu"; // core + WebGPU classes; use instead of "three" in this app
import { color, positionLocal, sin, time, vec3 } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true }); // forceWebGL: true to test the fallback
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);
await renderer.init(); // required before render()/compute() outside setAnimationLoop

const material = new THREE.MeshStandardNodeMaterial({ roughness: 0.4 });
material.colorNode = color(0x2266ff).mul(sin(time).mul(0.5).add(0.5));
material.positionNode = positionLocal.add(vec3(0, sin(time.add(positionLocal.x.mul(4))).mul(0.1), 0));

renderer.setAnimationLoop(() => renderer.render(scene, camera));
console.log(renderer.backend.isWebGPUBackend ? "WebGPU" : "WebGL 2");
```

Import map (no bundler): map `three` **and** `three/webgpu` to `build/three.webgpu.js`, `three/tsl`
to `build/three.tsl.js` (tree-shakeable since r187) and `three/addons/` to `examples/jsm/`, all at
`three@0.187.0`. Never import `WebGPURenderer` from `"three"`.

Same renderer API as WebGL for `setSize`, `setPixelRatio`, tone mapping, `outputColorSpace`,
`shadowMap`, `compileAsync`, `info`, `xr`, `dispose`.

| Doesn't work with WebGPU | Use |
| --- | --- |
| `ShaderMaterial`, `RawShaderMaterial`, `onBeforeCompile` | node material slots + TSL |
| `EffectComposer` | `RenderPipeline` (`threejs-postprocessing`) |
| `Reflector`, `Sky`, `Water` | `reflector()` node, `SkyMesh`, `WaterMesh` |
| `CSM` | `CSMShadowNode` |
| `Line2` + `LineMaterial` | `three/addons/lines/webgpu/Line2.js` + `Line2NodeMaterial` |
| `RectAreaLightUniformsLib.init()` | `THREE.RectAreaLightNode.setLTC(RectAreaLightTexturesLib.init())` |
| `GPUComputationRenderer` | TSL compute |
| `renderer.library.addLight(Node, Light)` | `CustomLight.registerNode(CustomLightNode)` (r187) |

Classic materials (`MeshStandardMaterial`, ...) are converted automatically. Use the
`*NodeMaterial` classes when you need slots.

## Node Material Slots

`colorNode` (**replaces** color × map), `opacityNode`, `alphaTestNode`, `normalNode` (view space),
`positionNode` (**local**, vertex displacement), `emissiveNode` (Standard/Physical, and Lambert,
Phong and Toon since r187), `roughnessNode`, `metalnessNode`, `aoNode`, `envNode`, `lightsNode`
(`lights([...])`), `maskNode` (discard incl. shadows), `castShadowNode`, `depthNode`,
`backdropNode`, `mrtNode`, `outputNode`, `fragmentNode` / `vertexNode` (replace a whole stage).
Points: `sizeNode`; sprites: `scaleNode`, `rotationNode`. Physical adds clearcoat, sheen,
iridescence, transmission, thickness, ior, dispersion and anisotropy nodes.

```javascript
import { materialColor, texture, uv } from "three/tsl";
m.colorNode = materialColor.mul(texture(detail, uv().mul(8))); // keep color/map and build on them
```

Re-assigning nodes after the first render needs `material.needsUpdate = true`; changing
uniform **values** does not.

## TSL Essentials

Operations are **methods**; JS operators like `+` don't work on nodes. Numbers convert
automatically, and since r187 mixed operand types auto-promote (float × vec3 just works).

```javascript
import { Fn, If, Loop, float, vec3, color, uniform, mix, smoothstep, uv, time } from "three/tsl";

const t = smoothstep(0.2, 0.8, uv().y);
const c = mix(vec3(1, 0, 0), color(0x0000ff), t);
uv().y.greaterThan(0.5).select(vec3(1), vec3(0)); // ternary

const uIntensity = uniform(1.0);           // uIntensity.value = 2 from JS: no recompile
const uScroll = uniform(0).onFrameUpdate((f) => f.time * 0.1);

const shade = Fn(([p, freq]) => p.length().mul(freq).sin()); // positional args arrive as an array
const tint = Fn(({ base, amount }) => base.mul(amount));     // named args arrive as an object
material.colorNode = vec3(shade(uv().sub(0.5), 40));

material.colorNode = Fn(() => {          // a Fn without params must still be CALLED: Fn(...)()
  const col = vec3(0).toVar();           // mutable variable
  If(uv().x.greaterThan(0.5), () => { col.assign(vec3(1, 0.5, 0)); }).Else(() => { col.assign(vec3(0.1)); });
  Loop(8, ({ i }) => { col.addAssign(float(i).mul(0.01)); });
  return col;
})();
```

- `If`, `Loop`, `Discard()`, `Return()`, `Break()`, `Switch()` only work inside `Fn`. A JS `if`
  runs once while building the graph; it is not a shader branch.
- `.setLayout({ name, type, inputs })` on a reused `Fn` emits a real shader function (smaller output).
- `varying(node)` / `vertexStage(node)` compute per vertex.
- `wgslFn` / `glslFn` run on one backend only: prefer plain TSL.

**Built-ins:** `positionLocal/World/View`, `normalLocal/World/View`, `positionViewDirection`,
`cameraPosition`, `uv(i)`, `screenUV`, `screenSize`, `time`, `deltaTime`, `instanceIndex`,
`vertexIndex`, `attribute("name", "vec3")`, `materialColor`/`materialRoughness`/...,
`texture(tex, uv?, level?)` (swap with `node.value = otherTex`), `cubeTexture`, `hash`,
`mx_noise_float`, `mx_fractal_noise_float`, `triNoise3D`, `oscSine`, `rotate`, `billboarding`,
`triplanarTexture`, `viewportSharedTexture()`, `viewportLinearDepth`, `pmremTexture`.
Full list: `node_modules/three/src/nodes/TSL.js`.

## Recipes

```javascript
// Fresnel rim
material.emissiveNode = color(0x44aaff).mul(float(1).sub(dot(normalView, positionViewDirection).saturate()).pow(3));

// Dissolve (masks shadows too)
const progress = uniform(0);
const n = mx_noise_float(positionLocal.mul(3)).mul(0.5).add(0.5);
material.maskNode = n.greaterThan(progress);

// Per-instance color on an InstancedMesh
material.colorNode = vec3(hash(instanceIndex), hash(instanceIndex.add(1)), hash(instanceIndex.add(2)));

// Gradient background
scene.backgroundNode = mix(color(0x87ceeb), color(0xffffff), normalWorld.y.mul(0.5).add(0.5));
```

## Compute (GPU particles)

```javascript
import { Fn, If, instancedArray, instanceIndex, uniform, vec3, hash } from "three/tsl";

const COUNT = 100_000;
const pos = instancedArray(COUNT, "vec3");   // GPU storage buffers
const vel = instancedArray(COUNT, "vec3");
const dt = uniform(0);

const init = Fn(() => {
  pos.element(instanceIndex).assign(vec3(hash(instanceIndex), hash(instanceIndex.add(1)), hash(instanceIndex.add(2))).sub(0.5).mul(10));
})().compute(COUNT);

const update = Fn(() => {
  const p = pos.element(instanceIndex), v = vel.element(instanceIndex);
  v.y.addAssign(dt.mul(-9.8));
  p.addAssign(v.mul(dt));
  If(p.y.lessThan(-5), () => { p.y.assign(-5); v.y.assign(v.y.negate().mul(0.6)); });
})().compute(COUNT);

const mat = new THREE.SpriteNodeMaterial({ transparent: true, depthWrite: false });
mat.positionNode = pos.toAttribute();
mat.scaleNode = uniform(0.05);
const particles = new THREE.Sprite(mat);
particles.count = COUNT;          // instanced sprites
particles.frustumCulled = false;  // positions live on the GPU
scene.add(particles);

await renderer.init();
renderer.compute(init);
renderer.setAnimationLoop((t) => {
  timer.update(t);
  dt.value = Math.min(timer.getDelta(), 1 / 30);
  renderer.compute(update);
  renderer.render(scene, camera);
});
```

- `instancedArray` = per instance, `attributeArray` = per vertex, `storage(attr, type, count)` wraps your own buffer.
- **r187:** struct storage buffers are always arrays: `storage.element(0).get("member")`.
- CPU readback: `await renderer.getArrayBufferAsync(attr)` (slow, not per frame).
- The WebGL 2 fallback runs compute through transform feedback: no atomics, workgroup memory or storage textures.

## Lights

Same light and shadow API as WebGL. Extras: `IESSpotLight` (`iesMap` from `IESLoader`),
`ProjectorLight` (`map`), rect area lights via `RectAreaLightNode.setLTC(...)` (above).
Custom light types: `MyLight.registerNode(MyLightNode)`.

## Debugging and Performance

```javascript
import { Inspector } from "three/addons/inspector/Inspector.js";
renderer.inspector = new Inspector();                // stats, node previews, parameters
renderer.inspector.createParameters("Settings").add(uIntensity, "value", 0, 3);
material.colorNode = someNode.toInspector("My Node");
```

- Output values as color to debug; `forceWebGL: true` tells you whether a bug is backend-specific.
- `await renderer.compileAsync(scene, camera)` before showing the scene (pipelines compile in parallel since r187).
- Reuse node materials; graph changes recompile, uniform values don't. `THREE.BundleGroup` records static draws once.

See also: `threejs-postprocessing`, `threejs-materials`, `threejs-shaders`, `threejs-migration`.
