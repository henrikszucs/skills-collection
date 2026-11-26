---
name: threejs-loaders
description: Three.js asset loading - GLTF/GLB with Draco, Meshopt and KTX2 compression, LoadingManager progress, loadAsync/Promise patterns, textures and HDR, OBJ/FBX/STL/PLY/USDZ and other formats, caching, cloning loaded models, parse from ArrayBuffer, error handling. Use when loading 3D models, textures or HDR environments, showing a loading bar, fixing models that load too big/small/dark, or managing asset reuse.
---

# Three.js Loaders

> three.js **r187**. Every loader has `load(url, onLoad, onProgress, onError)` and
> `loadAsync(url, onProgress)`. `RGBELoader` is deprecated: use `HDRLoader`.

## GLTF / GLB (preferred)

Set up all decoders once and reuse the loader:

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/addons/loaders/DRACOLoader.js";
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";
import { MeshoptDecoder } from "three/addons/libs/meshopt_decoder.module.js";

// Decoder folders must match the installed three version:
// copy node_modules/three/examples/jsm/libs/draco/gltf/ -> /draco/ and libs/basis/ -> /basis/
// or use https://cdn.jsdelivr.net/npm/three@0.187.0/examples/jsm/libs/draco/gltf/ (and .../libs/basis/)
const draco = new DRACOLoader().setDecoderPath("/draco/");
const ktx2 = new KTX2Loader().setTranscoderPath("/basis/").detectSupport(renderer); // WebGPU: await renderer.init() first
const gltfLoader = new GLTFLoader().setDRACOLoader(draco).setKTX2Loader(ktx2).setMeshoptDecoder(MeshoptDecoder);

const gltf = await gltfLoader.loadAsync("model.glb");
scene.add(gltf.scene); // also gltf.animations, gltf.cameras, gltf.userData (extras)
```

GLTFLoader handles PBR, texture color spaces, KHR material extensions (clearcoat, transmission,
sheen, iridescence, volume, ior, specular, anisotropy, dispersion), punctual lights, GPU
instancing, quantization and texture transforms. Optimize assets with
`gltf-transform optimize in.glb out.glb --texture-compress ktx2` or `gltfpack` (5-20x smaller).

### Prepare a Model

```javascript
const model = gltf.scene;
model.traverse((o) => { if (o.isMesh) { o.castShadow = o.receiveShadow = true; } });

const box = new THREE.Box3().setFromObject(model);           // center and scale to ~1 unit
const size = box.getSize(new THREE.Vector3());
const s = 1 / Math.max(size.x, size.y, size.z);
model.scale.multiplyScalar(s);
model.position.sub(box.getCenter(new THREE.Vector3())).multiplyScalar(s); // scale the offset too
```

Frame the camera: put it at `center + size * k`, set `near = size / 100`, `far = size * 100`,
`updateProjectionMatrix()`, and point `controls.target` at the center.

- **Black or dark model:** no environment map. Metals reflect `scene.environment` and render
  black without one (`threejs-lighting`).
- **Invisible model:** log the bounding-box size (scale), check near/far, and make sure you added `gltf.scene`.
- Materials are shared between meshes that used the same glTF material: clone before editing one.

## Progress (LoadingManager)

```javascript
const manager = new THREE.LoadingManager();
manager.onProgress = (url, loaded, total) => bar(loaded / total); // counts FILES, not bytes
manager.onLoad = () => hideLoader();
manager.onError = (url) => console.error(url);
manager.setURLModifier((url) => mapToBlobOrCdn(url)); // drag-and-drop multi-file .gltf, CDNs
const loader = new GLTFLoader(manager);
```

Byte progress for one file: `loadAsync(url, (e) => e.lengthComputable && bar(e.loaded / e.total))`.

## Other Formats (`three/addons/loaders/`)

| Format | Loader | Result |
| --- | --- | --- |
| OBJ + MTL | `OBJLoader` + `MTLLoader` (`materials.preload(); objLoader.setMaterials(materials)`) | `Group` |
| FBX | `FBXLoader` (often cm: `scale.setScalar(0.01)`) | `Group` + `.animations` |
| STL / PLY | `STLLoader` / `PLYLoader` (`computeVertexNormals()`) | `BufferGeometry` |
| USD / USDZ | `USDLoader` (`USDZLoader` is deprecated) | `Group` |
| Collada, 3MF, 3DS, VRML, Rhino, LDraw | `ColladaLoader`, `ThreeMFLoader`, `TDSLoader`, `VRMLLoader`, `Rhino3dmLoader`, `LDrawLoader` | scene graph |
| Point clouds, splats | `PCDLoader`, `XYZLoader`, `GaussianSplatPLYLoader`, `SPZLoader`, `SPLATLoader` | points / splats |
| SVG, fonts | `SVGLoader`, `FontLoader`, `TTFLoader` | `threejs-geometry` |
| BVH, IES, LUTs | `BVHLoader`, `IESLoader`, `LUTCubeLoader` / `LUT3dlLoader` | clip / texture / LUT |
| Textures, HDR | `TextureLoader`, `HDRLoader`, `EXRLoader`, `KTX2Loader` | `threejs-textures` |

`LWOLoader` and `VTKLoader` are deprecated (removal in r195/r194): convert to glTF. Convert
anything you can to GLB; it loads fastest and most completely.

## Reuse: Load Once, Clone Many

```javascript
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";

const cache = new Map(); // url -> Promise<gltf>; concurrent requests share one download
const loadGLTF = (url) => cache.get(url) ?? cache.set(url, gltfLoader.loadAsync(url)).get(url);

async function instance(url) {
  const gltf = await loadGLTF(url);
  return { scene: SkeletonUtils.clone(gltf.scene), animations: gltf.animations };
}
```

Use `SkeletonUtils.clone` for anything with bones (plain `clone()` shares the skeleton). Clones
share geometry and materials. Many identical static copies: `InstancedMesh`.
`THREE.Cache.enabled = true` caches raw file data only, not parsed objects.

## Other Sources

```javascript
await gltfLoader.parseAsync(arrayBuffer, "");      // fetch, IndexedDB, file input
const url = URL.createObjectURL(file);             // drag-and-drop File (revokeObjectURL after)
gltfLoader.setPath("assets/").setResourcePath("assets/textures/");
gltfLoader.setRequestHeader({ Authorization: "Bearer ..." });
```

## Errors

Wrap `loadAsync` in try/catch. Common causes: **CORS** (serve from the same origin or send CORS
headers), opening the page via `file://` (use a dev server such as `npx vite`), and wrong relative
paths for `.bin` files and textures. Retry with backoff for flaky networks.

## Performance

1. GLB + Meshopt/Draco + KTX2.
2. Reuse loaders (Draco/KTX2 keep worker pools); `dispose()` them when loading is finished.
3. Load the hero asset or a low-poly placeholder first.
4. `await renderer.compileAsync(scene, camera)` after loading.

See also: `threejs-textures`, `threejs-animation`, `threejs-addons` (exporters).
