---
name: threejs-addons
description: Catalog of Three.js addons (three/addons/...) with import paths - sky, water, mirrors, exporters (GLTF, USDZ, STL, OBJ), extra geometries, modifiers, BufferGeometryUtils/SkeletonUtils, CSM shadows, IK, GPGPU, physics wrappers, bundled libs (stats, lil-gui, tween). Use when you need a feature not in core three.js, are looking for an addon or its import path, export a scene to a file, or add sky, water or mirrors.
---

# Three.js Addons Catalog

> three.js **r187**. Addons live in `node_modules/three/examples/jsm/` and import as
> `three/addons/<folder>/<File>.js` (ES modules only). Unsure an addon exists? `ls` the folder or
> read the file's JSDoc (`@three_import` shows the exact import).

Covered elsewhere: controls (`threejs-interaction`), loaders (`threejs-loaders`), post-processing
(`threejs-postprocessing`), lines/labels/helpers (`threejs-objects`), math (`threejs-math`),
WebXR (`threejs-webxr`), TSL effect nodes (`threejs-webgpu-tsl`).

## Environments

`RoomEnvironment` (studio IBL without files: `pmrem.fromScene(new RoomEnvironment(), 0.04).texture`),
`DebugEnvironment`, `ColorEnvironment(color)` in `three/addons/environments/`.

## Sky, Water, Mirrors

```javascript
import { Sky } from "three/addons/objects/Sky.js"; // WebGPU: SkyMesh (same uniform names, as nodes)
const sky = new Sky();
sky.scale.setScalar(450000);
const u = sky.material.uniforms;
u.turbidity.value = 10; u.rayleigh.value = 3; u.mieCoefficient.value = 0.005; u.mieDirectionalG.value = 0.7;
u.sunPosition.value.setFromSphericalCoords(1, THREE.MathUtils.degToRad(80), THREE.MathUtils.degToRad(180));
scene.add(sky);
renderer.toneMapping = THREE.ACESFilmicToneMapping; // sky needs tone mapping (exposure ~0.5)
// Always +Y up (the "up" uniform was removed in r186)
```

- `Water` (`objects/Water.js`, WebGPU `WaterMesh`): needs a tiling `waterNormals` texture; advance
  `water.material.uniforms.time.value` each frame. `Water2` = flow-map water.
- `Reflector` (flat mirror; WebGPU `reflector()` TSL node) and `Refractor`: each re-renders the
  scene, so keep few and size the texture to the viewport × pixel ratio.
- `GroundedSkybox`: `threejs-textures`.

## Exporters (`three/addons/exporters/`)

```javascript
import { GLTFExporter } from "three/addons/exporters/GLTFExporter.js";
const glb = await new GLTFExporter().parseAsync(scene, { binary: true, onlyVisible: true, animations: clips });
const a = Object.assign(document.createElement("a"), {
  href: URL.createObjectURL(new Blob([glb], { type: "model/gltf-binary" })), download: "scene.glb",
});
a.click(); URL.revokeObjectURL(a.href);
```

Also `USDZExporter` (`await parseAsync(scene)`, iOS AR Quick Look), `STLExporter`
(`parse(mesh, { binary: true })`), `OBJExporter`, `PLYExporter`, `DRACOExporter` (`parseAsync`),
`EXRExporter`, `KTX2Exporter`. Screenshots: `renderer.domElement.toBlob()` right after `render()`.

## Geometries (`three/addons/geometries/`)

- `RoundedBoxGeometry(w, h, d, segments, radius)`
- `ConvexGeometry(points)`: convex hull
- `DecalGeometry(mesh, position, orientation: Euler, size: Vector3)`: project a decal at a
  raycast hit; material with `polygonOffset: true, polygonOffsetFactor: -4, depthWrite: false`
- `ParametricGeometry(fn(u, v, target), slices, stacks)` + `ParametricFunctions` (klein, mobius)
- `TextGeometry`, `TeapotGeometry`, `BoxLineGeometry`

## Modifiers (`three/addons/modifiers/`)

```javascript
const simplified = await new SimplifyModifier().modify(geometry, removeCount); // async since r186 (meshoptimizer)
const flow = new Flow(mesh); flow.updateCurve(0, curve); scene.add(flow.object3D); // CurveModifier.js; loop: flow.moveAlongCurve(0.001)
```

Also `InstancedFlow`, `EdgeSplitModifier`, `TessellateModifier`.

## Utilities (`three/addons/utils/`)

| Module | Highlights |
| --- | --- |
| `BufferGeometryUtils` | `mergeGeometries`, `mergeVertices`, `toCreasedNormals`, `computeMikkTSpaceTangents` |
| `SkeletonUtils` | `clone` (skinned meshes), `retargetClip`, `retarget` |
| `SceneUtils` | `createMeshesFromInstancedMesh`, `createMeshesFromMultiMaterialMesh`, `sortInstancedMesh`, `reduceVertices` |
| `CameraUtils` | `frameCorners` (off-axis projection, portals) |
| `GeometryCompressionUtils` | quantize normals/positions/UVs |
| `SceneOptimizer` | experimental auto-batching into `BatchedMesh` |
| `ShadowMapViewer`, `UVsDebug`, `WorkerPool`, `SortUtils` | debugging and infrastructure |

## Shadows, IK, GPGPU, Effects

```javascript
import { CSM } from "three/addons/csm/CSM.js"; // cascaded shadows for big outdoor scenes; WebGPU: CSMShadowNode
const csm = new CSM({ maxFar: 250, cascades: 4, shadowMapSize: 2048, lightDirection: new THREE.Vector3(1, -1, 1).normalize(), camera, parent: scene });
scene.traverse((o) => o.material && csm.setupMaterial(o.material));
// loop: csm.update(); on camera/resize change: csm.updateFrustums()

import { CCDIKSolver } from "three/addons/animation/CCDIKSolver.js"; // IK chains; ikSolver.update() after mixer.update()

import { GPUComputationRenderer } from "three/addons/misc/GPUComputationRenderer.js"; // WebGL texture simulation
// WebGPU: use TSL compute instead; GPU sorting in three/addons/gpgpu/ (BitonicSort)
```

Effects (`three/addons/effects/`, WebGL): `OutlineEffect` (toon outlines), `StereoEffect`,
`AnaglyphEffect`, `AsciiEffect`. Use `effect.render(scene, camera)` instead of `renderer.render`.

## Physics Wrappers (`three/addons/physics/`)

`RapierPhysics` (Rapier 0.21 since r187), `JoltPhysics`, `AmmoPhysics`: set
`mesh.userData.physics = { mass }`, then `(await RapierPhysics()).addScene(scene)`. For games
use the engine directly. Simple static collisions: `Octree` + `Capsule` (`threejs-math`).

## Other Folders

- `generators/` (WebGPU): `TreeGenerator`, `ForestGenerator`, `TerrainGenerator`, `CityGenerator`,
  with fluent setters, e.g. `new TreeGenerator().setSeed(1).build()`.
- `objects/`: `MarchingCubes`, `ShadowMesh`, `Lensflare`, `GaussianSplat`.
- `misc/`: `ProgressiveLightMap` (runtime lightmap baking), `ConvexObjectBreaker`, `Volume`, `TubePainter`.
- `libs/`: `stats.module.js`, `lil-gui.module.min.js`, `tween.module.js`, `meshopt_decoder.module.js`,
  `fflate.module.js`, plus `draco/` and `basis/` decoder folders (copy to your public dir, same version).
