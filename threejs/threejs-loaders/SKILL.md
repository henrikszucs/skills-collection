---
name: threejs-loaders
description: Three.js asset loading - GLTF/GLB with Draco, Meshopt and KTX2 compression, LoadingManager progress, loadAsync/Promise patterns, textures and HDR, OBJ/FBX/STL/PLY/USDZ and other formats, caching, cloning loaded models, parse from ArrayBuffer, error handling. Use when loading 3D models, textures or HDR environments, showing a loading bar, fixing models that load too big/small/dark, or managing asset reuse.
---

# Three.js Loaders

> **Version:** written for three.js **r186**. `RGBELoader` is deprecated (use `HDRLoader`).
> See `threejs-migration` for other renamed APIs.

## Quick Start

```javascript
import * as THREE from "three";
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const gltf = await new GLTFLoader().loadAsync("model.glb");
scene.add(gltf.scene);

const texture = await new THREE.TextureLoader().loadAsync("texture.jpg");
texture.colorSpace = THREE.SRGBColorSpace;
```

Every loader has `load(url, onLoad, onProgress, onError)` and `loadAsync(url, onProgress)`.

## LoadingManager (progress bar)

```javascript
const manager = new THREE.LoadingManager();
manager.onStart = (url, loaded, total) => console.log(`start ${url}`);
manager.onProgress = (url, loaded, total) => updateBar(loaded / total); // counts files, not bytes
manager.onLoad = () => hideLoadingScreen(); // all files queued on this manager finished
manager.onError = (url) => console.error(`failed: ${url}`);

const gltfLoader = new GLTFLoader(manager);
const textureLoader = new THREE.TextureLoader(manager);

// Rewrite URLs (CDN, blob URLs for drag-and-drop)
manager.setURLModifier((url) => (url.startsWith("http") ? url : `https://cdn.example.com/${url}`));
```

Byte-level progress for one big file: `loader.loadAsync(url, (e) => e.lengthComputable && updateBar(e.loaded / e.total))`.

## GLTF / GLB (preferred format)

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const loader = new GLTFLoader();
const gltf = await loader.loadAsync("model.glb");

gltf.scene;      // THREE.Group - add this
gltf.scenes;     // all scenes in the file
gltf.animations; // AnimationClip[] (see threejs-animation)
gltf.cameras;    // cameras defined in the file
gltf.asset;      // generator/version info
gltf.parser;     // advanced: gltf.parser.getDependency('material', 0)
gltf.userData;   // glTF "extras"

scene.add(gltf.scene);
```

GLTFLoader handles PBR materials, texture color spaces, KHR material extensions (clearcoat,
transmission, sheen, iridescence, volume, ior, specular, anisotropy, dispersion, emissive
strength), lights (`KHR_lights_punctual`), instancing (`EXT_mesh_gpu_instancing`), quantization, and texture transforms.

### Compressed GLTF: Draco, Meshopt, KTX2

Set up all decoders once and reuse the loader:

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";
import { DRACOLoader } from "three/addons/loaders/DRACOLoader.js";
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";
import { MeshoptDecoder } from "three/addons/libs/meshopt_decoder.module.js";

// Draco: copy node_modules/three/examples/jsm/libs/draco/gltf/ to /draco/ (or use a CDN of the SAME three version)
const dracoLoader = new DRACOLoader().setDecoderPath("/draco/");
dracoLoader.preload();

// KTX2/Basis: copy node_modules/three/examples/jsm/libs/basis/ to /basis/ (must match the three version)
const ktx2Loader = new KTX2Loader().setTranscoderPath("/basis/").detectSupport(renderer);

const gltfLoader = new GLTFLoader()
  .setDRACOLoader(dracoLoader)
  .setKTX2Loader(ktx2Loader)
  .setMeshoptDecoder(MeshoptDecoder);

const gltf = await gltfLoader.loadAsync("compressed.glb");
```

CDN alternative (pin the exact installed version):

```javascript
dracoLoader.setDecoderPath("https://cdn.jsdelivr.net/npm/three@0.186.1/examples/jsm/libs/draco/gltf/");
ktx2Loader.setTranscoderPath("https://cdn.jsdelivr.net/npm/three@0.186.1/examples/jsm/libs/basis/");
```

Optimize assets with `gltf-transform optimize in.glb out.glb --texture-compress ktx2` (Meshopt +
KTX2) or `gltfpack`. Typical savings: 5-20x.

### Prepare a Loaded Model

```javascript
const model = gltf.scene;

model.traverse((child) => {
  if (child.isMesh) {
    child.castShadow = true;
    child.receiveShadow = true;
    // child.material is shared between meshes that used the same glTF material
  }
});

const head = model.getObjectByName("Head"); // names come from the DCC tool

// Center and scale to fit ~1 unit
const box = new THREE.Box3().setFromObject(model);
const size = box.getSize(new THREE.Vector3());
const center = box.getCenter(new THREE.Vector3());
model.position.sub(center);                         // center at origin
model.scale.setScalar(1 / Math.max(size.x, size.y, size.z));
// To rest it on the floor afterwards: recompute the box and set model.position.y -= box.min.y

scene.add(model);
```

**Model looks black/dark?** Add an environment map (`scene.environment`, see `threejs-lighting`)
and/or lights. Metallic glTF materials reflect the environment and are black without one.

**Model invisible?** Check scale (log the bounding box size), camera near/far, and that you added `gltf.scene`.

### Fit the Camera to a Model

```javascript
function frameObject(object, camera, controls, offset = 1.5) {
  const box = new THREE.Box3().setFromObject(object);
  const size = box.getSize(new THREE.Vector3()).length();
  const center = box.getCenter(new THREE.Vector3());
  camera.near = size / 100;
  camera.far = size * 100;
  camera.updateProjectionMatrix();
  camera.position.copy(center).add(new THREE.Vector3(size, size * 0.5, size).multiplyScalar(offset * 0.5));
  camera.lookAt(center);
  if (controls) { controls.target.copy(center); controls.update(); }
}
```

## Textures and HDR

```javascript
const textureLoader = new THREE.TextureLoader();
const color = await textureLoader.loadAsync("color.jpg");
color.colorSpace = THREE.SRGBColorSpace; // only for color images (not normal/roughness maps)

import { HDRLoader } from "three/addons/loaders/HDRLoader.js";
const env = await new HDRLoader().loadAsync("env.hdr");
env.mapping = THREE.EquirectangularReflectionMapping;
scene.environment = env;
```

Configuration (wrapping, filtering, KTX2, EXR, cube maps, PMREM): see `threejs-textures`.

## Other Formats

| Format | Loader (`three/addons/loaders/...`) | Result |
| ------ | ----------------------------------- | ------ |
| OBJ (+MTL) | `OBJLoader`, `MTLLoader` | `Group` |
| FBX | `FBXLoader` | `Group` with `.animations` |
| STL | `STLLoader` | `BufferGeometry` |
| PLY | `PLYLoader` | `BufferGeometry` |
| USDZ / USD | `USDLoader` | `Group` |
| Collada (.dae) | `ColladaLoader` | `{ scene }` |
| 3MF | `ThreeMFLoader` | `Group` |
| 3DS | `TDSLoader` | `Group` |
| VRML | `VRMLLoader` | `Scene` |
| PCD / XYZ (point clouds) | `PCDLoader`, `XYZLoader` | `Points` / `BufferGeometry` |
| Gaussian splats | `GaussianSplatPLYLoader`, `SPZLoader`, `SPLATLoader`, `KSPLATLoader` | splat geometry |
| SVG | `SVGLoader` | paths -> shapes (see `threejs-geometry`) |
| Fonts | `FontLoader`, `TTFLoader` | `Font` for TextGeometry |
| Rhino 3dm | `Rhino3dmLoader` | `Object3D` |
| LDraw | `LDrawLoader` | `Group` |
| BVH mocap | `BVHLoader` | `{ skeleton, clip }` |
| IES profiles | `IESLoader` | texture (WebGPU IESSpotLight) |
| LUTs | `LUTCubeLoader`, `LUT3dlLoader`, `LUTImageLoader` | 3D LUT for color grading |

```javascript
import { OBJLoader } from "three/addons/loaders/OBJLoader.js";
import { MTLLoader } from "three/addons/loaders/MTLLoader.js";
const materials = await new MTLLoader().loadAsync("model.mtl");
materials.preload();
const obj = await new OBJLoader().setMaterials(materials).loadAsync("model.obj");
scene.add(obj);

import { FBXLoader } from "three/addons/loaders/FBXLoader.js";
const fbx = await new FBXLoader().loadAsync("model.fbx");
fbx.scale.setScalar(0.01); // FBX is often in centimeters
scene.add(fbx);

import { STLLoader } from "three/addons/loaders/STLLoader.js";
const stlGeometry = await new STLLoader().loadAsync("part.stl");
stlGeometry.computeVertexNormals();
scene.add(new THREE.Mesh(stlGeometry, new THREE.MeshStandardMaterial({ color: 0x888888 })));

import { PLYLoader } from "three/addons/loaders/PLYLoader.js";
const plyGeometry = await new PLYLoader().loadAsync("scan.ply");
plyGeometry.computeVertexNormals();
scene.add(new THREE.Mesh(plyGeometry, new THREE.MeshStandardMaterial({ vertexColors: plyGeometry.hasAttribute("color") })));
```

Convert other formats to GLB (Blender, `gltf-transform`, FBX2glTF) whenever you can: GLB loads fastest and most completely.

## Async Patterns

```javascript
async function loadAll() {
  const [gltf, env, color] = await Promise.all([
    gltfLoader.loadAsync("model.glb"),
    new HDRLoader().loadAsync("env.hdr"),
    new THREE.TextureLoader().loadAsync("color.jpg"),
  ]);
  env.mapping = THREE.EquirectangularReflectionMapping;
  color.colorSpace = THREE.SRGBColorSpace;
  scene.environment = env;
  scene.add(gltf.scene);
}

try {
  await loadAll();
} catch (err) {
  console.error("Asset failed to load", err); // 404, CORS, or a parse error
}
```

## Reusing Loaded Models

Load once, clone many times. Use `SkeletonUtils.clone` for anything with bones (a plain
`clone()` makes all copies share one skeleton):

```javascript
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";

class AssetManager {
  constructor(gltfLoader) {
    this.gltfLoader = gltfLoader;
    this.textureLoader = new THREE.TextureLoader();
    this.models = new Map();   // url -> Promise<gltf>
    this.textures = new Map(); // url -> Promise<Texture>
  }

  loadGLTF(url) {
    if (!this.models.has(url)) this.models.set(url, this.gltfLoader.loadAsync(url));
    return this.models.get(url); // concurrent requests share one download
  }

  async instance(url) {
    const gltf = await this.loadGLTF(url);
    return { scene: SkeletonUtils.clone(gltf.scene), animations: gltf.animations };
  }

  loadTexture(url, srgb = true) {
    if (!this.textures.has(url)) {
      this.textures.set(url, this.textureLoader.loadAsync(url).then((t) => {
        if (srgb) t.colorSpace = THREE.SRGBColorSpace;
        return t;
      }));
    }
    return this.textures.get(url);
  }
}

const assets = new AssetManager(gltfLoader);
const tree1 = await assets.instance("tree.glb");
const tree2 = await assets.instance("tree.glb"); // no second download
```

Clones share geometries and materials. Clone the material too (`mesh.material = mesh.material.clone()`) before changing it on one copy.
Many identical static copies: convert to `InstancedMesh` (see `threejs-geometry`).

### THREE.Cache

```javascript
THREE.Cache.enabled = true; // FileLoader/ImageLoader keep raw responses in memory by URL
THREE.Cache.clear();
```

It caches file data, not parsed objects. The AssetManager pattern above is usually more useful.

## Loading From Other Sources

```javascript
// ArrayBuffer (fetch, IndexedDB, file input)
const buffer = await (await fetch("model.glb")).arrayBuffer();
const parsed = await gltfLoader.parseAsync(buffer, ""); // path for external resources

// Drag-and-drop File
input.addEventListener("change", async (e) => {
  const file = e.target.files[0];
  const url = URL.createObjectURL(file);
  try {
    const gltf = await gltfLoader.loadAsync(url);
    scene.add(gltf.scene);
  } finally {
    URL.revokeObjectURL(url);
  }
});

// Data URL
const tex = await new THREE.TextureLoader().loadAsync("data:image/png;base64,iVBORw0KGgo...");

// Paths
gltfLoader.setPath("assets/models/");          // prefix for load() urls
gltfLoader.setResourcePath("assets/textures/"); // where .gltf finds .bin/textures
gltfLoader.setCrossOrigin("anonymous");
gltfLoader.setRequestHeader({ Authorization: "Bearer ..." });
```

Multi-file `.gltf` (gltf + bin + textures) from drag-and-drop: map each file to a blob URL and
resolve them in `manager.setURLModifier`.

## Error Handling and Retries

```javascript
async function loadWithRetry(loader, url, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      return await loader.loadAsync(url);
    } catch (err) {
      if (i === retries - 1) throw err;
      await new Promise((r) => setTimeout(r, 1000 * (i + 1)));
    }
  }
}
```

Common failures: **CORS** (serve assets from the same origin or with CORS headers), opening
`index.html` via `file://` (use a local dev server such as `npx vite`), wrong relative paths for `.bin`/textures.

## Performance Tips

1. **GLB + Meshopt/Draco + KTX2**: smallest downloads and VRAM.
2. **Reuse loaders**, especially DRACOLoader/KTX2Loader (they keep worker pools).
3. **Show something early**: load a low-poly placeholder or the hero asset first.
4. **Pre-compile** after loading: `await renderer.compileAsync(scene, camera)`.
5. **Dispose** what you remove (see `threejs-fundamentals`), and `dracoLoader.dispose()` / `ktx2Loader.dispose()` when finished loading.

## See Also

- `threejs-textures` - Texture settings, environment maps
- `threejs-animation` - Playing loaded animations
- `threejs-materials` - Materials on loaded models
- `threejs-addons` - Exporters (GLTFExporter)
