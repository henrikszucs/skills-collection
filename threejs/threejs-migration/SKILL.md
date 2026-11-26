---
name: threejs-migration
description: Three.js version checks and migration - how to find the installed revision, renamed/removed/deprecated APIs by release (r125-r186), fixes for console deprecation warnings, outdated tutorial patterns (Geometry, encoding, uv2, legacy lights, Clock, RGBELoader, PCFSoftShadowMap, PostProcessing), and upgrade steps. Use when Three.js code throws "is not a constructor"/"undefined", logs a deprecation warning, comes from an old tutorial or LLM answer, or when upgrading three versions.
---

# Three.js Version Checks and Migration

The other `threejs-*` skills target **r186** (npm `three@0.186.x`, September 2026). three.js
ships a release roughly monthly and renames or removes APIs regularly, so older code
often breaks or warns.

## 1. Find the Installed Version First

```bash
node -e "console.log(require('three/package.json').version)"   # 0.186.1 means r186
grep '"version"' node_modules/three/package.json
```

```javascript
import * as THREE from "three";
console.log(THREE.REVISION); // "186"
```

No local install (CDN/import map)? The version is in the URL (`three@0.186.1`). Pin an exact
version; never use an unversioned `three` CDN URL in production.

When the code here and the installed version disagree, **the installed source is the truth**:
read `node_modules/three/src/...` or `node_modules/three/examples/jsm/...` and look for
`@deprecated` comments and `warn(` messages.

## 2. Fix by Console Message

| Message / symptom | Cause | Fix |
| --- | --- | --- |
| `Clock: This module has been deprecated. Please use THREE.Timer` | r183 | `const timer = new THREE.Timer(); timer.connect(document);` then `timer.update(t)` + `timer.getDelta()` each frame |
| `RGBELoader has been deprecated. Please use HDRLoader` | r180 | `import { HDRLoader } from "three/addons/loaders/HDRLoader.js"` (same usage) |
| `PCFSoftShadowMap has been removed. Using PCFShadowMap instead` | deprecated r186 | `renderer.shadowMap.type = THREE.PCFShadowMap; light.shadow.radius = 4;` |
| `"PostProcessing" has been renamed to "RenderPipeline"` | r183 (WebGPU) | `new THREE.RenderPipeline(renderer)` |
| `"renderAsync()" has been deprecated` | r181 (WebGPU) | `await renderer.init()` once, then `render()` (or use `setAnimationLoop`) |
| `.render() called before the backend is initialized` | WebGPU | `await renderer.init()` before the first manual render |
| `Material "ShaderMaterial" is not compatible` | WebGPU | Rewrite as a node material with TSL (`threejs-webgpu-tsl`) |
| `TransformControls` / "object not an instance of THREE.Object3D" | r169 | `scene.add(transformControls.getHelper())` |
| `X is not a constructor` for `THREE.Geometry`, `THREE.Face3` | removed r125 | Use `BufferGeometry` (see `threejs-geometry`) |
| `THREE.WebGL1Renderer is not a constructor` | removed r163 | `WebGLRenderer` (WebGL 2 only) |
| `SVGLoader createShapes() is deprecated` | r185 | `shapePath.toShapes()` |
| `Source has been renamed to TextureSource` | r186 | `THREE.TextureSource` |
| `Matrix3 .rotate()/.scale()/.translate() is deprecated` | r185 | `makeRotation()` / `makeScale()` / `makeTranslation()` |
| `.fromWorkingColorSpace() / .toWorkingColorSpace()` renamed | r177 | `ColorManagement.workingToColorSpace()` / `.colorSpaceToWorking()` |
| TSL `label()` deprecated | r179 | `.setName()` |
| TSL `cache()` deprecated | r181 | `isolate()` |
| TSL `PI2` deprecated | r181 | `TWO_PI` |
| TSL `transformedNormalView/World` deprecated | r178 | `normalView` / `normalWorld` |
| `DRACOLoader: setDecoderConfig ... removed in r194` | deprecated | Remove the call |
| `FirstPersonControls handleResize() has been removed` | r184 | Delete the call |
| `USDZLoader` deprecated | r179 | `USDLoader` |
| `LWOLoader` / `VTKLoader` deprecated | r185 / r184 | Convert assets to glTF |
| `KTX2Loader detectSupportAsync()` deprecated | r181 | `await renderer.init(); ktx2Loader.detectSupport(renderer)` |
| GLSL `inverseTransformDirection` (deprecated define) | r185 | `transformNormalByInverseViewMatrix(normal, viewMatrix)` / `transformDirectionByInverseViewMatrix(dir, viewMatrix)` |
| `onBeforeCompile` replace has no effect, `output_fragment` not found | r154 | Chunk renamed `opaque_fragment` (`encodings_fragment` became `colorspace_fragment`) |

## 3. Renamed and Removed APIs (old tutorial -> current)

### Renderer and color

| Old | Current |
| --- | --- |
| `renderer.outputEncoding = THREE.sRGBEncoding` | `renderer.outputColorSpace = THREE.SRGBColorSpace` (default; r152) |
| `texture.encoding = THREE.sRGBEncoding` | `texture.colorSpace = THREE.SRGBColorSpace` (r152) |
| `THREE.LinearEncoding` | `THREE.LinearSRGBColorSpace` or `THREE.NoColorSpace` for data textures |
| `renderer.gammaOutput`, `gammaFactor` | removed: color management handles it |
| `renderer.physicallyCorrectLights = true` | always physical now (`useLegacyLights` removed r165): raise point/spot intensities (candela) |
| `THREE.WebGLMultisampleRenderTarget` | `new THREE.WebGLRenderTarget(w, h, { samples: 4 })` (r138) |
| `requestAnimationFrame(loop)` | `renderer.setAnimationLoop(loop)` (required for WebXR/WebGPU) |
| `new THREE.Clock()` | `new THREE.Timer()` (r183) |

### Geometry

| Old | Current |
| --- | --- |
| `THREE.Geometry`, `Face3`, `geometry.vertices/faces` | `BufferGeometry` + attributes (r125) |
| `BoxBufferGeometry`, `SphereBufferGeometry`, ... | `BoxGeometry`, `SphereGeometry`, ... (aliases removed) |
| `BufferGeometryUtils.mergeBufferGeometries` | `mergeGeometries` |
| `BufferGeometryUtils.computeTangents(g)` | `geometry.computeTangents()` or `computeMikkTSpaceTangents` |
| `TextGeometry({ height })` | `TextGeometry({ depth })` (r163) |
| `CapsuleGeometry(radius, length, ...)` | `CapsuleGeometry(radius, height, capSegments, radialSegments, heightSegments)` |
| `geometry.applyMatrix(m)` / `object.applyMatrix(m)` | `applyMatrix4(m)` |
| `SVGLoader.createShapes(path)` | `path.toShapes()` (r185) |

### Materials and textures

| Old | Current |
| --- | --- |
| `geometry.setAttribute('uv2', ...)` for aoMap/lightMap | not needed; maps use `uv` by default. Second UV set: attribute `uv1` + `texture.channel = 1` (r151) |
| `material.vertexColors = THREE.VertexColors` | `material.vertexColors = true` |
| `ShaderMaterial({ extensions: { derivatives: true } })` | remove: always available in WebGL 2 (only `clipCullDistance`, `multiDraw` remain) |
| GLSL `#include <output_fragment>` | `#include <opaque_fragment>` (r154) |
| GLSL `#include <encodings_fragment>` | `#include <colorspace_fragment>` (r154) |
| `THREE.ImageUtils.loadTexture()` | `new THREE.TextureLoader().load()` |
| `RGBELoader` | `HDRLoader` (r180) |

### Math

| Old | Current |
| --- | --- |
| `THREE.Math` | `THREE.MathUtils` |
| `matrix.getInverse(m)` | `matrix.copy(m).invert()` |
| `quaternion.inverse()` | `quaternion.invert()` |
| `Matrix3.rotate/scale/translate` | `makeRotation/makeScale/makeTranslation` (r185) |

### Addons and post-processing

| Old | Current |
| --- | --- |
| `examples/js/...` global scripts (`THREE.OrbitControls`) | ES modules: `import { OrbitControls } from "three/addons/controls/OrbitControls.js"` (examples/js removed r148) |
| `three/examples/jsm/...` | still works; `three/addons/...` is the documented alias |
| `scene.add(transformControls)` | `scene.add(transformControls.getHelper())` (r169) |
| `new FilmPass(noise, scanlines, count, grayscale)` | `new FilmPass(intensity, grayscale)` |
| `new SMAAPass(w, h)` | `new SMAAPass()` (sized by `composer.setSize`) |
| `new HalftonePass(w, h, params)` | `new HalftonePass(params)` |
| `ShaderPass(FXAAShader)` + manual resolution | `new FXAAPass()` (manual ShaderPass still works) |
| `ShaderPass(GammaCorrectionShader)` at the end | `new OutputPass()` (tone mapping + color space; r153) |
| `ContactShadows` (does not exist in three.js; it is from drei) | `ShadowMaterial` floor / blob shadow / `webgl_shadow_contact` technique |
| `LightProbeGenerator.fromCubeRenderTarget()` sync | `await` it (async) |
| `PostProcessing` (WebGPU) | `RenderPipeline` (r183) |
| `three/addons/nodes/Nodes.js`, `three/nodes` | `three/tsl` (functions) and `three/webgpu` (classes) |

## 4. Patterns That Are Still Valid but Often Wrong in Old Code

- **Lights too dark:** intensities from pre-r155 tutorials (point light `1`) are far too low now.
- **Washed-out textures:** missing `texture.colorSpace = THREE.SRGBColorSpace` on color maps.
- **EffectComposer looks dark:** missing `OutputPass`.
- **Model clones share animation:** use `SkeletonUtils.clone()` for skinned meshes.
- **Mixing `three` and `three/webgpu` imports:** pick one entry point per app.
- **Addon import from a different three copy:** a bundler resolving two `three` versions breaks `instanceof` checks. Dedupe (`npm ls three`).

## 5. Upgrade Procedure

1. Read the release notes for every version you skip: https://github.com/mrdoob/three.js/releases
   and the migration guide: https://github.com/mrdoob/three.js/wiki/Migration-Guide
2. Upgrade one or a few releases at a time; run the app and fix every console warning (they name the replacement).
3. Search your code for the "Old" column entries above.
4. Copy decoder/transcoder folders again (Draco `libs/draco/gltf/`, Basis `libs/basis/`): they must match the three version.
5. If you use `@types/three`, upgrade it to the matching version.
6. Re-check visuals: tone mapping, light intensity and shadow softness changes are silent.

## See Also

- `threejs-fundamentals` - Current setup patterns
- `threejs-webgpu-tsl` - Moving from WebGL/GLSL to WebGPU/TSL
- `threejs-postprocessing` - OutputPass and pass ordering
