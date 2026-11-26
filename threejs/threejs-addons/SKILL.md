---
name: threejs-addons
description: Catalog of Three.js addons (three/addons/...) with import paths and usage - environments (RoomEnvironment), sky/water/mirrors (Sky, Water, Reflector, Refractor, GroundedSkybox), exporters (GLTFExporter, USDZ, STL, OBJ, PLY), extra geometries (RoundedBox, Convex, Decal, Parametric), modifiers (Simplify, curve Flow), utilities (BufferGeometryUtils, SkeletonUtils, SceneUtils, CameraUtils), CSM shadows, IK, GPGPU, stereo/ASCII/outline effects, physics wrappers, procedural generators and bundled libs (stats, lil-gui, tween, meshopt). Use when you need a feature not in core three.js, are looking for the right addon or its import path, export a scene to a file, or add sky, water or mirrors.
---

# Three.js Addons Catalog

> **Version:** written for three.js **r186**. Addons live in `node_modules/three/examples/jsm/`
> and are imported as `three/addons/<folder>/<File>.js`. They are ES modules only (the old
> global `examples/js` scripts were removed in r148). If unsure whether an addon exists or
> how it is called, list the folder or read the file's JSDoc header (`@three_import` shows the exact import).

```bash
ls node_modules/three/examples/jsm/            # folders
ls node_modules/three/examples/jsm/objects/    # files in one folder
```

Covered in depth elsewhere: controls (`threejs-interaction`), loaders (`threejs-loaders`),
post-processing (`threejs-postprocessing`), lines/labels/helpers (`threejs-objects`),
math addons (`threejs-math`), WebXR (`threejs-webxr`), TSL effect nodes (`threejs-webgpu-tsl`).

## Environments (instant IBL, no files)

```javascript
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";
const pmrem = new THREE.PMREMGenerator(renderer);
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;

// Also: DebugEnvironment (colored walls to inspect reflections), ColorEnvironment(color)
```

## Sky, Water, Mirrors

```javascript
// Physical sky (WebGL). WebGPU: SkyMesh from three/addons/objects/SkyMesh.js (uniform nodes, same names)
import { Sky } from "three/addons/objects/Sky.js";
const sky = new Sky();
sky.scale.setScalar(450000);
scene.add(sky);
const u = sky.material.uniforms;
u.turbidity.value = 10;
u.rayleigh.value = 3;
u.mieCoefficient.value = 0.005;
u.mieDirectionalG.value = 0.7;
const sun = new THREE.Vector3().setFromSphericalCoords(1, THREE.MathUtils.degToRad(90 - 10), THREE.MathUtils.degToRad(180));
u.sunPosition.value.copy(sun);
renderer.toneMapping = THREE.ACESFilmicToneMapping; // sky needs tone mapping
renderer.toneMappingExposure = 0.5;
// Use it as IBL too: scene.environment = pmrem.fromScene(skyOnlyScene).texture

// Ocean (WebGL). WebGPU: WaterMesh
import { Water } from "three/addons/objects/Water.js";
const waterNormals = await new THREE.TextureLoader().loadAsync("waternormals.jpg");
waterNormals.wrapS = waterNormals.wrapT = THREE.RepeatWrapping;
const water = new Water(new THREE.PlaneGeometry(10000, 10000), {
  textureWidth: 512,
  textureHeight: 512,
  waterNormals,
  sunDirection: sun.clone(),
  sunColor: 0xffffff,
  waterColor: 0x001e0f,
  distortionScale: 3.7,
  fog: scene.fog !== undefined,
});
water.rotation.x = -Math.PI / 2;
scene.add(water);
// loop: water.material.uniforms.time.value += dt;
// Water2 (three/addons/objects/Water2.js): flow-map water for rivers/pools.

// Flat mirror (WebGL). WebGPU: reflector() TSL node
import { Reflector } from "three/addons/objects/Reflector.js";
const mirror = new Reflector(new THREE.PlaneGeometry(4, 4), {
  textureWidth: window.innerWidth * window.devicePixelRatio,
  textureHeight: window.innerHeight * window.devicePixelRatio,
  color: 0xb5b5b5,
  clipBias: 0.003,
});
mirror.position.y = 0.001;
mirror.rotation.x = -Math.PI / 2;
scene.add(mirror);
// Refractor: see-through distortion (glass panes). Each mirror re-renders the scene: keep few.

// HDR skybox with a ground plane: GroundedSkybox (see threejs-textures)
```

## Exporters

```javascript
import { GLTFExporter } from "three/addons/exporters/GLTFExporter.js";
const glb = await new GLTFExporter().parseAsync(scene, {
  binary: true,          // ArrayBuffer (.glb); false = JSON (.gltf)
  onlyVisible: true,
  animations: clips,     // AnimationClip[] to include
  // trs: true, maxTextureSize: 2048, includeCustomExtensions: false
});
download(new Blob([glb], { type: "model/gltf-binary" }), "scene.glb");

function download(blob, filename) {
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = filename;
  a.click();
  URL.revokeObjectURL(a.href);
}

import { USDZExporter } from "three/addons/exporters/USDZExporter.js"; // iOS AR Quick Look
const usdz = await new USDZExporter().parseAsync(scene);

import { STLExporter } from "three/addons/exporters/STLExporter.js";  // 3D printing
const stl = new STLExporter().parse(mesh, { binary: true });

import { OBJExporter } from "three/addons/exporters/OBJExporter.js";
const objText = new OBJExporter().parse(scene);

import { PLYExporter } from "three/addons/exporters/PLYExporter.js";
new PLYExporter().parse(mesh, (result) => download(new Blob([result]), "mesh.ply"), { binary: true });

// Also: DRACOExporter, EXRExporter (HDR render targets), KTX2Exporter
```

Screenshots: `renderer.domElement.toBlob(cb)` right after `renderer.render()` (see `threejs-fundamentals`).

## Extra Geometries (`three/addons/geometries/`)

```javascript
import { RoundedBoxGeometry } from "three/addons/geometries/RoundedBoxGeometry.js";
new RoundedBoxGeometry(1, 1, 1, 4, 0.1); // width, height, depth, segments, radius

import { ConvexGeometry } from "three/addons/geometries/ConvexGeometry.js";
new ConvexGeometry(arrayOfVector3);      // convex hull mesh

import { DecalGeometry } from "three/addons/geometries/DecalGeometry.js";
// Project a decal onto a mesh at a raycast hit
const orientation = new THREE.Euler().setFromRotationMatrix(
  new THREE.Matrix4().lookAt(hit.point, hit.point.clone().add(hit.face.normal), THREE.Object3D.DEFAULT_UP),
);
const decal = new THREE.Mesh(
  new DecalGeometry(hit.object, hit.point, orientation, new THREE.Vector3(0.3, 0.3, 0.3)),
  new THREE.MeshStandardMaterial({ map: decalTexture, transparent: true, depthWrite: false, polygonOffset: true, polygonOffsetFactor: -4 }),
);
scene.add(decal);

import { ParametricGeometry } from "three/addons/geometries/ParametricGeometry.js";
import { klein, mobius } from "three/addons/geometries/ParametricFunctions.js"; // also plane, mobius3d
new ParametricGeometry(klein, 25, 25); // func(u, v, target), slices, stacks
new ParametricGeometry((u, v, target) => target.set(u * 2 - 1, Math.sin(u * 6) * 0.2, v * 2 - 1), 32, 32);

// Also: TextGeometry (threejs-geometry), TeapotGeometry, BoxLineGeometry (grid room lines)
```

## Modifiers (`three/addons/modifiers/`)

```javascript
import { SimplifyModifier } from "three/addons/modifiers/SimplifyModifier.js";
const simplified = await new SimplifyModifier().modify(geometry, Math.floor(vertexCount * 0.5)); // async (r186): vertices to REMOVE

import { Flow } from "three/addons/modifiers/CurveModifier.js"; // bend a mesh along a curve
const flow = new Flow(textMesh);
flow.updateCurve(0, curve);
scene.add(flow.object3D);
// loop: flow.moveAlongCurve(0.001);   InstancedFlow for many copies

// Also: EdgeSplitModifier (split normals at hard edges), TessellateModifier
```

## Utilities (`three/addons/utils/`)

| Module | Highlights |
| --- | --- |
| `BufferGeometryUtils` | `mergeGeometries`, `mergeVertices`, `toCreasedNormals`, `computeMikkTSpaceTangents`, `interleaveAttributes` (see `threejs-geometry`) |
| `SkeletonUtils` | `clone` (skinned meshes), `retargetClip`, `retarget` (see `threejs-animation`) |
| `SceneUtils` | `createMeshesFromInstancedMesh`, `createMeshesFromMultiMaterialMesh`, `createMultiMaterialObject`, `reduceVertices`, `sortInstancedMesh`, `traverseGenerator` |
| `CameraUtils` | `frameCorners` (off-axis projection: portals, CAVE displays) |
| `GeometryUtils` | `hilbert2D/3D`, `gosper` curves |
| `GeometryCompressionUtils` | quantize normals/positions/UVs to save memory |
| `SortUtils` | `radixSort` |
| `WorkerPool` | small Web Worker pool |
| `TextureUtils` (core `THREE.TextureUtils`) | `contain`, `cover`, `fill` |
| `UVsDebug` | draw a geometry's UV layout to a canvas |
| `ShadowMapViewer` | show a light's shadow map on screen |
| `SceneOptimizer` | experimental auto-batching of meshes into `BatchedMesh` |
| `ColorUtils`, `GaussianSplatUtils`, `LDrawUtils` | specialized helpers |

## Cascaded Shadow Maps (large outdoor scenes)

```javascript
import { CSM } from "three/addons/csm/CSM.js";
const csm = new CSM({
  maxFar: 250,
  cascades: 4,
  mode: "practical",            // 'uniform' | 'logarithmic' | 'practical' | 'custom'
  parent: scene,
  shadowMapSize: 2048,
  lightDirection: new THREE.Vector3(1, -1, 1).normalize(),
  camera,
});
scene.traverse((o) => { if (o.material) csm.setupMaterial(o.material); }); // every receiving material
// loop: csm.update();   on resize/camera change: csm.updateFrustums();
// WebGPU: CSMShadowNode (three/addons/csm/CSMShadowNode.js); debug: CSMHelper
```

## Animation Addons

```javascript
import { CCDIKSolver, CCDIKHelper } from "three/addons/animation/CCDIKSolver.js";
// iks: [{ target: boneIndex, effector: boneIndex, links: [{ index: boneIndex, limitation?, rotationMin?, rotationMax? }], iteration: 10 }]
const ikSolver = new CCDIKSolver(skinnedMesh, iks);
// loop, after mixer.update(): ikSolver.update();

import { AnimationClipCreator } from "three/addons/animation/AnimationClipCreator.js";
const spin = AnimationClipCreator.CreateRotationAnimation(2, "y"); // period, axis
```

## GPGPU (WebGL)

```javascript
import { GPUComputationRenderer } from "three/addons/misc/GPUComputationRenderer.js";
const gpu = new GPUComputationRenderer(128, 128, renderer); // texture-based simulation
const pos0 = gpu.createTexture();
const posVar = gpu.addVariable("texturePosition", positionFragmentShader, pos0);
gpu.setVariableDependencies(posVar, [posVar]);
const error = gpu.init();
// loop: gpu.compute(); material.uniforms.tPos.value = gpu.getCurrentRenderTarget(posVar).texture;
```

With WebGPURenderer use TSL compute instead (`threejs-webgpu-tsl`). Sorting on the GPU: `three/addons/gpgpu/` (`BitonicSort`, `CountingSort`, WebGPU).

## Render Effects (`three/addons/effects/`, WebGL)

```javascript
import { StereoEffect } from "three/addons/effects/StereoEffect.js";     // side-by-side (cardboard)
import { AnaglyphEffect } from "three/addons/effects/AnaglyphEffect.js"; // red/cyan 3D glasses
import { OutlineEffect } from "three/addons/effects/OutlineEffect.js";   // toon outlines (inverted hull)
import { AsciiEffect } from "three/addons/effects/AsciiEffect.js";       // ASCII art in the DOM

const effect = new OutlineEffect(renderer, { defaultThickness: 0.004 });
effect.setSize(w, h);
// loop: effect.render(scene, camera)  (instead of renderer.render)
```

## Physics Wrappers (`three/addons/physics/`)

`RapierPhysics`, `JoltPhysics`, `AmmoPhysics`: small helpers that load the engine and sync
meshes with `userData.physics = { mass }`.

```javascript
import { RapierPhysics } from "three/addons/physics/RapierPhysics.js";
const physics = await RapierPhysics();
floor.userData.physics = { mass: 0 }; // static
box.userData.physics = { mass: 1 };
physics.addScene(scene);              // adds every mesh with userData.physics
```

For games, use an engine directly (Rapier, cannon-es, Jolt) for joints, character controllers and queries.
Simple static collisions: `Octree` + `Capsule` (`threejs-math`).

## Procedural Generators (`three/addons/generators/`, WebGPU)

`TreeGenerator`, `ForestGenerator`, `TerrainGenerator`, `CityGenerator` and street furniture
generators build meshes with node materials (they import `three/webgpu`):

```javascript
import { TreeGenerator } from "three/addons/generators/TreeGenerator.js";
const generator = new TreeGenerator();
const oak = generator.setSeed(1).setLevels(4).build(); // fluent setters, build() returns a new mesh
scene.add(oak);
```

## Other Objects (`three/addons/objects/`)

`MarchingCubes` (metaballs), `ShadowMesh` (planar projected shadows), `Lensflare` / `LensflareMesh`,
`GroundedSkybox`, `GaussianSplat` (3D Gaussian splatting with the splat loaders), `ReflectorForSSRPass`.

## Misc (`three/addons/misc/`)

`ProgressiveLightMap` (bake soft GI/shadows into a lightmap at runtime), `ConvexObjectBreaker`
(fracture meshes), `Volume`/`VolumeSlice` (medical NRRD volumes), `Gyroscope` (keep world
orientation under a rotating parent), `TubePainter` (3D drawing strokes, XR), `RollerCoaster*` geometries.

## Bundled Libraries (`three/addons/libs/`)

```javascript
import Stats from "three/addons/libs/stats.module.js";          // FPS panel
const stats = new Stats(); document.body.appendChild(stats.dom); // loop: stats.update()

import GUI from "three/addons/libs/lil-gui.module.min.js";       // tweak panel
const gui = new GUI(); gui.add(params, "speed", 0, 10);

import TWEEN from "three/addons/libs/tween.module.js";           // tweening
import { MeshoptDecoder } from "three/addons/libs/meshopt_decoder.module.js"; // GLTF meshopt
import * as fflate from "three/addons/libs/fflate.module.js";    // zip/unzip
// Folders: libs/draco/ (Draco decoders), libs/basis/ (KTX2 transcoder) - copy them to your public dir
```

## See Also

- `threejs-loaders` - All loader addons
- `threejs-postprocessing` - Pass and shader addons
- `threejs-objects` - Lines, CSS renderers, helpers, interactive HTML
- `threejs-webgpu-tsl` - WebGPU versions of Sky/Water/Reflector/CSM/Line2
