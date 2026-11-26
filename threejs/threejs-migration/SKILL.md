---
name: threejs-migration
description: Three.js version checks and migration - how to find the installed revision, renamed/removed/deprecated APIs by release (r125-r187), fixes for console deprecation warnings, outdated tutorial patterns (Geometry, encoding, uv2, legacy lights, Clock, RGBELoader, PCFSoftShadowMap, PostProcessing, iridescenceThicknessRange), and upgrade steps. Use when Three.js code throws "is not a constructor"/"undefined", logs a deprecation warning, comes from an old tutorial or LLM answer, or when upgrading three versions.
---

# Three.js Version Checks and Migration

The `threejs-*` skills target **r187** (`three@0.187.0`, October 2026). three.js releases about
monthly and renames or removes APIs often, so old code breaks or warns.

## 1. Find the Installed Version First

```bash
node -e "console.log(require('three/package.json').version)"   # 0.187.0 = r187
```

In code: `THREE.REVISION` (`"187"`). CDN/import map: the version is in the URL; always pin it.

When this skill and the installed version disagree, **the installed source wins**: search
`node_modules/three/src/` and `examples/jsm/` for `@deprecated` and `warn(`.

## 2. New in r187 (from r186)

| Change | Do this |
| --- | --- |
| `MeshPhysicalMaterial.iridescenceThicknessRange` deprecated | `iridescenceThicknessMinimum` / `iridescenceThicknessMaximum` (nm, default 100 / 400) |
| `renderer.library.addLight(LightNode, Light)` deprecated (WebGPU) | `CustomLight.registerNode(CustomLightNode)` |
| TSL struct storage buffers are always arrays | `storage.get('m')` -> `storage.element(0).get('m')` |
| TSL `pixelationPass()` `pixelSize` | plain number only, no node |
| `CubeUVReflectionMapping` removed; PMREM uses cube render targets | nothing to call; env-lit materials look slightly different |
| `WebGLRenderer.setViewport()/setScissor()` | no pixel-ratio scaling while a render target is bound |
| TSL math operands auto-promote to a common type | explicit casts are no longer needed for mixed `float`/`vec*` math |
| XR camera matrices come from the first sub-camera (left eye) | only matters if you read `renderer.xr.getCamera()` transforms |
| New: `MeshPhysicalMaterial.diffuseRoughness` (EON rough diffuse), `Texture.mipmapsAutoUpdate`, `emissiveNode` on Lambert/Phong/Toon node materials | optional |

Coming in **r188** (unreleased, per the migration guide): `LightProbe`, `LightProbeGenerator`
and `LightProbeHelper` are deprecated (use `scene.environment` or `LightProbeGrid`), ambient and
hemisphere lights add specular to PBR materials, and AO becomes multi-bounce.

## 3. Fix by Console Message

| Message / symptom | Since | Fix |
| --- | --- | --- |
| `Clock: This module has been deprecated` | r183 | `new THREE.Timer()`; `timer.connect(document)`; per frame `timer.update(t)` then `getDelta()` |
| `RGBELoader has been deprecated` | r180 | `HDRLoader` from `three/addons/loaders/HDRLoader.js` (same usage) |
| `PCFSoftShadowMap has been removed. Using PCFShadowMap` | r186 | `PCFShadowMap` + `light.shadow.radius` |
| `"PostProcessing" has been renamed to "RenderPipeline"` | r183 | `new THREE.RenderPipeline(renderer)` |
| `"renderAsync()" has been deprecated` | r181 | `await renderer.init()` once, then `render()` |
| `.render() called before the backend is initialized` | WebGPU | `await renderer.init()` first |
| `Material "ShaderMaterial" is not compatible` | WebGPU | rewrite as node material + TSL |
| TransformControls "not an instance of THREE.Object3D" | r169 | `scene.add(controls.getHelper())` |
| `THREE.Geometry` / `Face3` is not a constructor | r125 | `BufferGeometry` |
| `THREE.WebGL1Renderer` is not a constructor | r163 | `WebGLRenderer` (WebGL 2 only) |
| `SVGLoader createShapes() is deprecated` | r185 | `shapePath.toShapes()` |
| `Source has been renamed to TextureSource` | r186 | `THREE.TextureSource` |
| `Matrix3 .rotate()/.scale()/.translate()` deprecated | r185 | `makeRotation()` / `makeScale()` / `makeTranslation()` |
| `Line2NodeMaterial: "lineColorNode"` deprecated | r185 | `colorNode` |
| TSL `directionToFaceDirection()` renamed | r185 | `negateOnBackSide()` |
| TSL `premultipliedGaussianBlur()` deprecated | r180 | `gaussianBlur(node, dir, sigma, { premultipliedAlpha: true })` |
| TSL `label()` / `cache()` / `PI2` deprecated | r179-r181 | `setName()` / `isolate()` / `TWO_PI` |
| TSL `transformedNormalView/World` | r178 | `normalView` / `normalWorld` |
| `DRACOLoader setDecoderConfig` (removed in r194) | - | delete the call |
| `FirstPersonControls handleResize()` removed | r184 | delete the call |
| `USDZLoader` deprecated | r179 | `USDLoader` |
| `LWOLoader` / `VTKLoader` deprecated | r185 / r184 | convert assets to glTF |
| `KTX2Loader detectSupportAsync()` deprecated | r181 | `await renderer.init(); loader.detectSupport(renderer)` |
| GLSL `inverseTransformDirection` | r185 | `transformNormalByInverseViewMatrix()` / `transformDirectionByInverseViewMatrix()` |
| `onBeforeCompile` replace finds no `output_fragment` | r154 | chunk is `opaque_fragment`; `encodings_fragment` -> `colorspace_fragment` |

## 4. Old Tutorial -> Current

| Old | Current |
| --- | --- |
| `renderer.outputEncoding = sRGBEncoding` | `outputColorSpace = SRGBColorSpace` (default, r152) |
| `texture.encoding = sRGBEncoding` | `texture.colorSpace = SRGBColorSpace` |
| `LinearEncoding` | `LinearSRGBColorSpace`, or `NoColorSpace` for data textures |
| `physicallyCorrectLights`, `gammaOutput` | removed: always physical (r165); raise point/spot intensities (candela) |
| `WebGLMultisampleRenderTarget` | `new WebGLRenderTarget(w, h, { samples: 4 })` |
| `requestAnimationFrame(loop)` | `renderer.setAnimationLoop(loop)` |
| `BoxBufferGeometry` etc. | `BoxGeometry` etc. |
| `mergeBufferGeometries` | `mergeGeometries` |
| `TextGeometry({ height })` | `{ depth }` (r163) |
| `applyMatrix(m)` | `applyMatrix4(m)` |
| `uv2` attribute for aoMap/lightMap | maps use `uv`; second set is `uv1` + `texture.channel = 1` (r151) |
| `vertexColors: THREE.VertexColors` | `vertexColors: true` |
| `extensions: { derivatives: true }` | remove (always on in WebGL 2) |
| `THREE.ImageUtils.loadTexture()` | `new TextureLoader().load()` |
| `THREE.Math`, `getInverse(m)`, `quaternion.inverse()` | `MathUtils`, `copy(m).invert()`, `invert()` |
| `examples/js` globals (`THREE.OrbitControls`) | ES modules from `three/addons/...` (r148) |
| `three/addons/nodes/Nodes.js`, `three/nodes` | `three/tsl` (functions) + `three/webgpu` (classes) |
| `new FilmPass(noise, scanlines, ...)` | `new FilmPass(intensity, grayscale)` |
| `new SMAAPass(w, h)` / `new HalftonePass(w, h, p)` | `new SMAAPass()` / `new HalftonePass(p)` |
| `ShaderPass(GammaCorrectionShader)` | `new OutputPass()` (r153) |
| `ShaderPass(FXAAShader)` + resolution | `new FXAAPass()` |
| `ContactShadows` | does not exist (drei); see `threejs-lighting` |
| sync `LightProbeGenerator.fromCubeRenderTarget()` | `await` it |
| `SimplifyModifier.modify()` sync | `await` it (r186, meshoptimizer-based) |
| `LightProbeGrid` (WebGL) | `LightProbeGridWebGL` (r186) |

## 5. Still Valid but Often Wrong

- Lights too dark: pre-r155 point-light intensities (`1`) are far too low now.
- Washed-out textures: missing `colorSpace = SRGBColorSpace` on color maps.
- Dark EffectComposer output: missing `OutputPass`.
- Clones of skinned models share one skeleton: use `SkeletonUtils.clone()`.
- Mixed `three` and `three/webgpu` renderers in one app, or two copies of `three` in the
  bundle (`npm ls three`): breaks `instanceof` checks.

## 6. Upgrade Procedure

1. Read the release notes (https://github.com/mrdoob/three.js/releases) and the migration guide
   (https://github.com/mrdoob/three.js/wiki/Migration-Guide) for every skipped version.
2. Upgrade a few releases at a time; fix every console warning (they name the replacement).
3. Search your code for the "Old" columns above.
4. Re-copy decoder folders (`libs/draco/gltf/`, `libs/basis/`) so they match the new version.
5. Bump `@types/three` to match.
6. Re-check visuals: tone mapping, light intensity, shadow softness and PMREM changes are silent.
