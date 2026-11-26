---
name: threejs-webgpu-tsl
description: Three.js WebGPURenderer and TSL (Three.js Shading Language) - setup and imports (three/webgpu, three/tsl), WebGL 2 fallback, node materials (colorNode, positionNode, ...), TSL syntax (Fn, uniform, If, Loop, toVar/assign), built-in nodes, uniforms, textures, instancing, compute shaders with storage buffers, RenderPipeline post-processing, WebGPU versions of addons, and what does not work (ShaderMaterial, onBeforeCompile, EffectComposer). Use when the user mentions WebGPU, TSL, node materials, compute shaders or GPU particles, or when writing shaders that should run on both WebGPU and WebGL.
---

# Three.js WebGPU Renderer and TSL

> **Version:** written for three.js **r186**. The WebGPU/TSL API still changes between
> releases more than the WebGL API; check `node_modules/three/src/nodes/` and the official
> `examples/webgpu_*.html` for your version. See `threejs-migration`.

## When to Use

| Use `WebGPURenderer` + TSL when | Stay with `WebGLRenderer` when |
| --- | --- |
| You need compute shaders / GPU particles / GPGPU | You rely on `ShaderMaterial`/GLSL or `onBeforeCompile` |
| You want one shader source for WebGPU **and** WebGL 2 | You use `EffectComposer` passes |
| You are starting a new project with node-based effects | You need maximum stability/compatibility with older addons |

`WebGPURenderer` uses WebGPU where available and **falls back to a WebGL 2 backend
automatically**. TSL compiles to WGSL or GLSL accordingly.

## Quick Start

```javascript
import * as THREE from "three/webgpu"; // core + WebGPU classes (replaces "three" in this app)
import { color, positionLocal, sin, time, vec3 } from "three/tsl";

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(50, innerWidth / innerHeight, 0.1, 100);
camera.position.z = 4;

const renderer = new THREE.WebGPURenderer({ antialias: true });
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);
await renderer.init(); // required before render()/compute() outside setAnimationLoop

const material = new THREE.MeshStandardNodeMaterial({ roughness: 0.4 });
material.colorNode = color(0x2266ff).mul(sin(time).mul(0.5).add(0.5));     // animated color
material.positionNode = positionLocal.add(vec3(0, sin(time.add(positionLocal.x.mul(4))).mul(0.1), 0)); // wobble

scene.add(new THREE.Mesh(new THREE.TorusKnotGeometry(1, 0.3, 128, 32), material));
scene.add(new THREE.DirectionalLight(0xffffff, 3));
scene.add(new THREE.AmbientLight(0xffffff, 0.3));

renderer.setAnimationLoop(() => renderer.render(scene, camera)); // setAnimationLoop also awaits init
```

### Imports

- `three/webgpu`: everything from `three` **plus** `WebGPURenderer`, node materials, `RenderPipeline`, nodes classes.
- `three/tsl`: TSL functions (`Fn`, `uniform`, `vec3`, `texture`, `positionLocal`, ...).
- `three/addons/...`: addons. WebGPU-specific ones live in e.g. `three/addons/tsl/display/`, `three/addons/lines/webgpu/`.
- Do not import `WebGPURenderer` from `"three"`. With a bundler, `three` and `three/webgpu` share one core, so
  addons importing `"three"` still work.
- Import map (no bundler):

```html
<script type="importmap">
  {
    "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@0.186.1/build/three.webgpu.js",
      "three/webgpu": "https://cdn.jsdelivr.net/npm/three@0.186.1/build/three.webgpu.js",
      "three/tsl": "https://cdn.jsdelivr.net/npm/three@0.186.1/build/three.tsl.js",
      "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.186.1/examples/jsm/"
    }
  }
</script>
```

### Renderer Options and Checks

```javascript
const renderer = new THREE.WebGPURenderer({
  antialias: true,
  alpha: false,
  forceWebGL: false,           // true: always use the WebGL 2 backend (testing/compat)
  powerPreference: "high-performance",
});
await renderer.init();
console.log(renderer.backend.isWebGPUBackend ? "WebGPU" : "WebGL 2");

import WebGPU from "three/addons/capabilities/WebGPU.js";
if (!WebGPU.isAvailable()) console.info("No WebGPU: running on the WebGL 2 fallback");
```

Same API as WebGLRenderer for: `setSize`, `setPixelRatio`, `toneMapping`, `toneMappingExposure`,
`outputColorSpace`, `shadowMap.enabled/type`, `setAnimationLoop`, `compileAsync`, `info`, `dispose`, `xr`.

### What Does Not Work With WebGPURenderer

| WebGL feature | WebGPU replacement |
| --- | --- |
| `ShaderMaterial`, `RawShaderMaterial` | Node material + TSL (`colorNode`, `positionNode`, `fragmentNode`, ...) |
| `material.onBeforeCompile` | Node slots on the built-in node material |
| `EffectComposer` + passes | `RenderPipeline` + `pass()` + TSL effect nodes |
| `Reflector` addon | `reflector()` TSL node |
| `Sky`, `Water` | `SkyMesh`, `WaterMesh` (`three/addons/objects/`) |
| `CSM` | `CSMShadowNode` (`three/addons/csm/CSMShadowNode.js`) |
| `Line2` + `LineMaterial` | `three/addons/lines/webgpu/Line2.js` + `Line2NodeMaterial` |
| `RectAreaLightUniformsLib.init()` | `THREE.RectAreaLightNode.setLTC(RectAreaLightTexturesLib.init())` |
| `GPUComputationRenderer` | TSL compute (`Fn().compute(count)`) |

Built-in classic materials (`MeshStandardMaterial`, etc.) are converted to node materials
automatically, so plain scenes work unchanged. Use the `*NodeMaterial` classes when you need node slots.

## Node Materials

`MeshBasicNodeMaterial`, `MeshLambertNodeMaterial`, `MeshPhongNodeMaterial`,
`MeshStandardNodeMaterial`, `MeshPhysicalNodeMaterial`, `MeshToonNodeMaterial`,
`MeshMatcapNodeMaterial`, `MeshNormalNodeMaterial`, `MeshSSSNodeMaterial` (subsurface),
`PointsNodeMaterial`, `SpriteNodeMaterial`, `LineBasicNodeMaterial`, `LineDashedNodeMaterial`,
`Line2NodeMaterial`, `ShadowNodeMaterial`, `VolumeNodeMaterial`, and the base `NodeMaterial`.

They accept the same constructor parameters as their classic counterparts, plus node slots:

```javascript
const m = new THREE.MeshPhysicalNodeMaterial();

// Common (all node materials)
m.colorNode;        // vec3/vec4 base color. REPLACES color * map (see below)
m.opacityNode;      // float
m.alphaTestNode;    // float threshold
m.normalNode;       // vec3, view-space normal
m.positionNode;     // vec3, LOCAL position (vertex displacement)
m.emissiveNode;     // vec3 (Standard/Physical/Phong/Lambert)
m.aoNode;           // float
m.envNode;          // environment for this material
m.lightsNode;       // custom set of lights: lights([light1, light2])
m.maskNode;         // bool: discard where false (also affects shadows)
m.castShadowNode;   // vec4 color/opacity for shadows cast (colored shadows)
m.receivedShadowNode;
m.depthNode;        // custom depth
m.backdropNode;     // what is behind the object (refraction/water/glass)
m.mrtNode;          // extra outputs for multiple render targets
m.outputNode;       // final vec4 after lighting
m.fragmentNode;     // replace the whole fragment stage (no lighting)
m.vertexNode;       // replace the whole vertex stage (clip-space vec4)

// Standard / Physical
m.roughnessNode; m.metalnessNode;
m.clearcoatNode; m.clearcoatRoughnessNode; m.sheenNode; m.iridescenceNode;
m.transmissionNode; m.thicknessNode; m.iorNode; m.dispersionNode; m.anisotropyNode;

// Points / Sprites
pointsMaterial.sizeNode;
spriteMaterial.scaleNode; spriteMaterial.rotationNode;
```

**colorNode replaces the material's color/map.** To keep them, build on the material inputs:

```javascript
import { materialColor, texture, uv } from "three/tsl";
m.colorNode = materialColor.mul(texture(detailMap, uv().mul(8)));
```

Changing which nodes are assigned after the first render requires `material.needsUpdate = true`.
Changing **uniform values** does not.

## TSL Fundamentals

TSL is JavaScript that builds a node graph. Operations are **methods** (JS operators like `+`
do not work on nodes). Plain numbers are converted automatically.

```javascript
import { float, int, vec2, vec3, vec4, color, uniform, mix, smoothstep, sin, abs, time, uv } from "three/tsl";

const a = float(1.5);
const v = vec3(1, 0, 0);
const c = color(0xff8800);           // or color("#ff8800"), color(new THREE.Color())
const b = a.mul(2).add(1).sin();     // method chaining: sin(a * 2 + 1)
const d = sin(a.mul(2).add(1));      // same, function style

v.x; v.xy; v.zyx; c.rgb; c.r;        // swizzles
const len = v.length();
const n = v.normalize();
const t = smoothstep(0.2, 0.8, uv().y);
const m = mix(vec3(1, 0, 0), vec3(0, 0, 1), t);

// Comparisons return bool nodes
const isAbove = uv().y.greaterThan(0.5);
const picked = isAbove.select(vec3(1), vec3(0)); // ternary, or select(cond, a, b)
```

Math methods: `add sub mul div mod pow sqrt abs sign floor ceil fract min max clamp saturate
mix step smoothstep sin cos tan atan length distance dot cross normalize reflect refract negate
oneMinus remap remapClamp`. Comparison/logic: `equal notEqual lessThan greaterThan lessThanEqual greaterThanEqual and or not`.

### Uniforms

```javascript
const uIntensity = uniform(1.0);               // float
const uTint = uniform(new THREE.Color(0xff0000)); // color
const uOffset = uniform(new THREE.Vector2());

material.colorNode = uTint.mul(uIntensity);

uIntensity.value = 2;      // update from JS anytime: no recompile
uTint.value.set(0x00ff00);

// Automatic per-frame/per-object updates
const uScroll = uniform(0).onFrameUpdate((frame) => frame.time * 0.1);
const uObjY = uniform(0).onObjectUpdate(({ object }) => object.position.y);
```

Built-in time: `time` (seconds) and `deltaTime` update automatically.

### Functions with Fn

```javascript
import { Fn, vec3, float, If, Loop, Break } from "three/tsl";

// Positional parameters arrive as an array
const ripple = Fn(([p, freq]) => {
  return p.length().mul(freq).sin().mul(0.5).add(0.5);
});
material.colorNode = vec3(ripple(uv().sub(0.5), 40));

// Named parameters arrive as an object
const tintFn = Fn(({ base, amount }) => base.mul(amount));
material.colorNode = tintFn({ base: color(0xffffff), amount: 0.5 });

// Typed layout (emits a real shader function, reusable and smaller)
const remap01 = Fn(([x]) => x.mul(0.5).add(0.5)).setLayout({
  name: "remap01",
  type: "float",
  inputs: [{ name: "x", type: "float" }],
});

// A Fn without parameters must still be CALLED: Fn(() => ...)()
material.colorNode = Fn(() => vec3(1, 0, 0))();
```

### Variables, Assignment, Control Flow

Inside `Fn`, use `.toVar()` for mutable variables and `.assign()` / `.addAssign()` to mutate.

```javascript
material.colorNode = Fn(() => {
  const col = vec3(0).toVar();          // mutable
  const p = uv().sub(0.5).toVar();

  If(p.length().lessThan(0.25), () => {
    col.assign(vec3(1, 0.5, 0));
  }).ElseIf(p.x.greaterThan(0), () => {
    col.assign(vec3(0, 0.5, 1));
  }).Else(() => {
    col.assign(vec3(0.1));
  });

  Loop(8, ({ i }) => {                  // also Loop({ start: 0, end: 8, type: 'int' }, ...)
    col.addAssign(sin(float(i).mul(p.x)).mul(0.01));
    If(col.r.greaterThan(2), () => { Break(); });
  });

  return col;
})();
```

`If`, `Loop`, `Discard()`, `Return()`, `Break()`, `Continue()` and `Switch()` only work inside `Fn`. A
JS `if` runs once while building the graph; it is not a shader branch.

### Varyings (vertex -> fragment)

```javascript
import { varying, positionLocal, vertexStage } from "three/tsl";
const vHeight = varying(positionLocal.y);          // computed per vertex, interpolated
material.colorNode = mix(color("blue"), color("white"), vHeight.add(1).mul(0.5));
// vertexStage(node) forces any node to be computed in the vertex stage
```

### Common Built-in Nodes

| Node | Meaning |
| --- | --- |
| `positionLocal`, `positionWorld`, `positionView` | position in object / world / view space |
| `normalLocal`, `normalWorld`, `normalView` | normals (already include normal maps where relevant) |
| `positionViewDirection`, `cameraPosition` | view direction, camera world position |
| `uv(index = 0)` | UV set |
| `screenUV`, `screenSize`, `screenCoordinate` | 0-1 screen position, size in pixels |
| `time`, `deltaTime` | seconds |
| `modelWorldMatrix`, `cameraViewMatrix`, `cameraProjectionMatrix` | matrices |
| `instanceIndex`, `vertexIndex` | instance / vertex id |
| `attribute('name', 'vec3')` | custom geometry attribute |
| `materialColor`, `materialRoughness`, `materialOpacity`, ... | the classic material property values |
| `texture(tex, uv?, level?)`, `cubeTexture(tex, dir)` | sampling |
| `hash(x)`, `mx_noise_float(p)`, `mx_fractal_noise_float(p)`, `triNoise3D(p, speed, t)` | noise |
| `oscSine(t)`, `oscSquare`, `oscTriangle`, `oscSawtooth` | 0-1 oscillators |
| `rotate(vec, angle)`, `billboarding()`, `rotateUV`, `spherizeUV`, `triplanarTexture` | helpers |
| `viewportSharedTexture()`, `viewportDepthTexture()`, `viewportLinearDepth` | what is already rendered (refraction, soft particles) |

Full list: https://threejs.org/docs/ (TSL section) and `node_modules/three/src/nodes/TSL.js`.

### Textures

```javascript
import { texture, uv, vec2, time } from "three/tsl";

material.colorNode = texture(colorMap);                              // default uv()
material.colorNode = texture(colorMap, uv().mul(4));                 // tiling (set RepeatWrapping)
material.colorNode = texture(colorMap, uv().add(vec2(time.mul(0.1), 0))); // scrolling
material.colorNode = texture(colorMap, uv(), 3);                     // explicit mip level

const texNode = texture(mapA);
texNode.value = mapB; // swap the texture without recompiling
```

### Raw WGSL / GLSL Snippets

```javascript
import { wgslFn, glslFn } from "three/tsl";
const desaturate = wgslFn(`
  fn desaturate(color: vec3<f32>) -> vec3<f32> {
    return vec3<f32>(dot(vec3<f32>(0.299, 0.587, 0.114), color));
  }
`);
material.colorNode = desaturate({ color: texture(map) });
// wgslFn only runs on the WebGPU backend, glslFn only on the WebGL backend: prefer plain TSL.
```

## Recipes

### Fresnel Rim

```javascript
import { normalView, positionViewDirection, dot, float, color } from "three/tsl";
const fresnel = float(1).sub(dot(normalView, positionViewDirection).saturate()).pow(3);
material.emissiveNode = color(0x44aaff).mul(fresnel);
```

### Dissolve

```javascript
import { mx_noise_float, positionLocal, uniform, smoothstep, color } from "three/tsl";
const progress = uniform(0);
const noise = mx_noise_float(positionLocal.mul(3)).mul(0.5).add(0.5);
material.maskNode = noise.greaterThan(progress); // discards pixels AND their shadows
material.emissiveNode = color(0xff6600).mul(smoothstep(progress.add(0.05), progress, noise));
// animate: progress.value = 0..1
```

### Per-Instance Colors/Offsets

```javascript
import { instanceIndex, hash, vec3, attribute } from "three/tsl";
const mesh = new THREE.InstancedMesh(geometry, material, 1000); // setMatrixAt as usual
material.colorNode = vec3(hash(instanceIndex), hash(instanceIndex.add(1)), hash(instanceIndex.add(2)));

// Or a custom InstancedBufferAttribute named 'aOffset':
// material.positionNode = positionLocal.add(attribute('aOffset', 'vec3'));
```

### Background and Environment Nodes

```javascript
import { color, normalWorld, mix, texture, pmremTexture } from "three/tsl";
scene.backgroundNode = mix(color(0x87ceeb), color(0xffffff), normalWorld.y.mul(0.5).add(0.5));
scene.environmentNode = pmremTexture(hdrTexture);   // or just scene.environment = hdrTexture
```

## Compute Shaders (GPU particles, simulations)

```javascript
import { Fn, If, instancedArray, instanceIndex, uniform, vec3, hash } from "three/tsl";

const COUNT = 100_000;
const positions = instancedArray(COUNT, "vec3"); // GPU storage buffers
const velocities = instancedArray(COUNT, "vec3");
const gravity = uniform(-9.8);
const dt = uniform(0); // set from JS each frame (explicit and backend-independent)

// Init (run once)
const computeInit = Fn(() => {
  const p = positions.element(instanceIndex);
  p.assign(vec3(hash(instanceIndex), hash(instanceIndex.add(1)), hash(instanceIndex.add(2))).sub(0.5).mul(10));
})().compute(COUNT);

// Update (run every frame)
const computeUpdate = Fn(() => {
  const p = positions.element(instanceIndex);
  const v = velocities.element(instanceIndex);
  v.y.addAssign(gravity.mul(dt));
  p.addAssign(v.mul(dt));
  If(p.y.lessThan(-5), () => {
    p.y.assign(-5);
    v.y.assign(v.y.negate().mul(0.6));
  });
})().compute(COUNT);

// Render the buffer as sprites (one draw call)
const material = new THREE.SpriteNodeMaterial({ transparent: true, depthWrite: false });
material.positionNode = positions.toAttribute();
material.colorNode = vec3(1, 0.6, 0.2);
material.scaleNode = uniform(0.05);
const particles = new THREE.Sprite(material);
particles.count = COUNT;          // instanced sprites
particles.frustumCulled = false;  // positions live on the GPU
scene.add(particles);

await renderer.init();
renderer.compute(computeInit);

const timer = new THREE.Timer();
renderer.setAnimationLoop((t) => {
  timer.update(t);
  dt.value = Math.min(timer.getDelta(), 1 / 30); // clamp big frame gaps
  renderer.compute(computeUpdate);
  renderer.render(scene, camera);
});
```

- `instancedArray(count, type)` = per-instance storage; read in materials with `.toAttribute()`
  or `.element(instanceIndex)`. `attributeArray` = per-vertex storage.
- `storage(bufferAttribute, type, count)` wraps your own `StorageBufferAttribute`/`StorageInstancedBufferAttribute`.
- Reading results back on the CPU: `await renderer.getArrayBufferAsync(storageAttribute)` (slow; avoid per frame).
- Compute also runs on the WebGL 2 fallback (via transform feedback), with fewer features (no atomics, workgroup memory, storage textures).

## Post-Processing: RenderPipeline

`PostProcessing` was renamed `RenderPipeline` in r183.

```javascript
import { pass } from "three/tsl";
import { bloom } from "three/addons/tsl/display/BloomNode.js";

const renderPipeline = new THREE.RenderPipeline(renderer);
const scenePass = pass(scene, camera);
const sceneColor = scenePass.getTextureNode("output");
const sceneDepth = scenePass.getTextureNode("depth");
const bloomNode = bloom(sceneColor, 1, 0.4, 0.85);
renderPipeline.outputNode = sceneColor.add(bloomNode);

renderer.setAnimationLoop(() => renderPipeline.render()); // instead of renderer.render
```

Effects: `three/addons/tsl/display/*` (bloom, fxaa, smaa, traa, ao/gtao, ssr, ssgi, dof, outline,
film, chromaticAberration, lut3D, gaussianBlur, pixelationPass, godrays, motionBlur, ...). MRT,
selective bloom and AA ordering are shown in `threejs-postprocessing`.

## Lights and Shadows

Same light classes and shadow API as WebGL (`castShadow`, `shadow.mapSize`, `shadow.radius`, ...).
WebGPU extras:

```javascript
// Rect area lights
import { RectAreaLightTexturesLib } from "three/addons/lights/RectAreaLightTexturesLib.js";
THREE.RectAreaLightNode.setLTC(RectAreaLightTexturesLib.init());

// Photometric spot light (IES profile)
import { IESLoader } from "three/addons/loaders/IESLoader.js";
const iesTexture = await new IESLoader().loadAsync("profile.ies");
const ies = new THREE.IESSpotLight(0xffffff, 500);
ies.iesMap = iesTexture;

// Texture-projecting light
const projector = new THREE.ProjectorLight(0xffffff, 100);
projector.map = slideTexture;
```

## Debugging

```javascript
import { Inspector } from "three/addons/inspector/Inspector.js";
renderer.inspector = new Inspector();            // overlay with stats, node previews, parameters
const gui = renderer.inspector.createParameters("Settings");
gui.add(uIntensity, "value", 0, 3).name("intensity");

material.colorNode = someNode.toInspector("My Node"); // preview a node in the inspector
```

- Output intermediate values as color: `material.colorNode = vec3(normalWorld.mul(0.5).add(0.5))`.
- Shader errors appear in the console with the generated WGSL/GLSL.
- Try `forceWebGL: true` to see whether a bug is backend-specific.

## Performance Tips

1. **`await renderer.compileAsync(scene, camera)`** before showing the scene (node graphs compile lazily).
2. **Reuse node materials**; changing node *graphs* recompiles, changing *uniform values* does not.
3. **Compute for particles** instead of updating BufferAttributes from JS.
4. **`setLayout`** on Fns used many times keeps generated shaders small.
5. **Static groups of objects**: `THREE.BundleGroup` records draw commands once (WebGPU backend).

## See Also

- `threejs-postprocessing` - RenderPipeline details, MRT selective bloom
- `threejs-materials` - Classic material parameters (same names on node materials)
- `threejs-shaders` - GLSL equivalent for WebGLRenderer
- `threejs-migration` - TSL renames (label -> setName, cache -> isolate, PI2 -> TWO_PI)
