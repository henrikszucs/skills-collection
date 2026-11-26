---
name: threejs-textures
description: Three.js textures - loading, color space, wrapping/repeat, filtering, mipmaps, UV channels, data/canvas/video/compressed (KTX2) textures, environment maps (HDR, EXR, cube maps, PMREM, skyboxes), render targets and texture memory. Use when working with images, fixing washed-out or dark colors, tiling textures, UV coordinates, cubemaps, HDR environments, render-to-texture, or texture optimization.
---

# Three.js Textures

> **Version:** written for three.js **r186**. See `threejs-migration` for renamed/removed APIs.
> This skill is the canonical reference for **environment maps** (HDR/EXR/cube/PMREM).

## Quick Start

```javascript
import * as THREE from "three";

const loader = new THREE.TextureLoader();
const texture = await loader.loadAsync("brick.jpg");
texture.colorSpace = THREE.SRGBColorSpace; // REQUIRED for color images, otherwise washed out
texture.anisotropy = renderer.capabilities.getMaxAnisotropy();

const material = new THREE.MeshStandardMaterial({ map: texture });
```

## Texture Loading

```javascript
const loader = new THREE.TextureLoader();

// Promise
const tex = await loader.loadAsync("texture.jpg");

// Callbacks (onProgress is not supported for images)
loader.load("texture.jpg", (t) => {}, undefined, (err) => console.error(err));

// Immediate: returns a Texture that fills in when loaded (fine for materials)
material.map = loader.load("texture.jpg");

// Several at once
const [color, normal, rough] = await Promise.all(
  ["color.jpg", "normal.jpg", "roughness.jpg"].map((u) => loader.loadAsync(u)),
);

// Off-main-thread decoding. ImageBitmaps ignore texture.flipY, so flip while decoding instead.
const bitmapLoader = new THREE.ImageBitmapLoader();
bitmapLoader.setOptions({ imageOrientation: "flipY" });
const bitmap = await bitmapLoader.loadAsync("big.jpg");
const bitmapTexture = new THREE.Texture(bitmap);
bitmapTexture.needsUpdate = true;
```

## Texture Configuration

### Color Space (most common mistake)

```javascript
// Images that hold COLOR: map, emissiveMap, sheenColorMap, specularColorMap, matcap, backgrounds
colorTexture.colorSpace = THREE.SRGBColorSpace;

// DATA textures: normalMap, roughnessMap, metalnessMap, aoMap, displacementMap, alphaMap, bumpMap
// leave the default THREE.NoColorSpace
```

Textures loaded by GLTFLoader already have the right color space.

### Wrapping, Repeat, Offset, Rotation

```javascript
texture.wrapS = THREE.RepeatWrapping; // U (default ClampToEdgeWrapping)
texture.wrapT = THREE.RepeatWrapping; // V
// MirroredRepeatWrapping also available
texture.repeat.set(4, 4);    // tiles (needs RepeatWrapping)
texture.offset.set(0.5, 0);  // 0-1
texture.center.set(0.5, 0.5); // rotation pivot
texture.rotation = Math.PI / 4;
// Changing wrap mode after first render needs texture.needsUpdate = true
```

### Filtering and Mipmaps

```javascript
texture.minFilter = THREE.LinearMipmapLinearFilter; // default, smooth at distance
texture.magFilter = THREE.LinearFilter;             // default
// Pixel art:
texture.minFilter = THREE.NearestFilter;
texture.magFilter = THREE.NearestFilter;
texture.generateMipmaps = false;

texture.anisotropy = renderer.capabilities.getMaxAnisotropy(); // sharp at grazing angles (floors)
```

WebGL 2 supports mipmaps for non-power-of-two images, so NPOT textures are fine. Power-of-two
sizes are still slightly better for compression and memory.

### Flip, Alpha

```javascript
texture.flipY = true;            // default for TextureLoader images; GLTF textures use false
texture.premultiplyAlpha = false;
texture.needsUpdate = true;      // after changing these on an already uploaded texture
```

### UV Channel

```javascript
// Which UV attribute a texture reads: 0 = uv, 1 = uv1, 2 = uv2, 3 = uv3
aoTexture.channel = 1;
geometry.setAttribute("uv1", new THREE.BufferAttribute(secondUVs, 2));
```

The old "aoMap/lightMap need a `uv2` attribute" rule is obsolete (r151). By default every map uses `uv`.

## Texture Types

### DataTexture (raw pixels)

```javascript
const size = 256;
const data = new Uint8Array(size * size * 4);
for (let i = 0; i < size * size; i++) {
  const v = Math.random() * 255;
  data.set([v, v, v, 255], i * 4);
}
const tex = new THREE.DataTexture(data, size, size, THREE.RGBAFormat);
tex.needsUpdate = true;
// Defaults differ from images: NearestFilter, no mipmaps, flipY false.
tex.magFilter = THREE.LinearFilter;
tex.minFilter = THREE.LinearFilter;
// Float data: new THREE.DataTexture(float32Array, w, h, THREE.RGBAFormat, THREE.FloatType)
```

3D and array textures: `Data3DTexture`, `DataArrayTexture` (sample with `sampler3D` / `sampler2DArray` in GLSL 3).

### CanvasTexture

```javascript
const canvas = document.createElement("canvas");
canvas.width = 512;
canvas.height = 256;
const ctx = canvas.getContext("2d");
ctx.fillStyle = "#222";
ctx.fillRect(0, 0, 512, 256);
ctx.fillStyle = "white";
ctx.font = "64px sans-serif";
ctx.fillText("Hello", 40, 150);

const tex = new THREE.CanvasTexture(canvas);
tex.colorSpace = THREE.SRGBColorSpace;
// After drawing again:
tex.needsUpdate = true;
```

### VideoTexture

```javascript
const video = document.createElement("video");
video.src = "video.mp4";
video.loop = true;
video.muted = true;       // autoplay requires muted (or a user gesture)
video.playsInline = true;
await video.play();

const tex = new THREE.VideoTexture(video); // updates itself every frame
tex.colorSpace = THREE.SRGBColorSpace;
```

Webcam: `video.srcObject = await navigator.mediaDevices.getUserMedia({ video: true })`.

### Compressed Textures (KTX2 / Basis)

GPU-compressed: far less VRAM and faster upload than JPG/PNG.

```javascript
import { KTX2Loader } from "three/addons/loaders/KTX2Loader.js";

const ktx2Loader = new KTX2Loader()
  .setTranscoderPath("/basis/") // copy node_modules/three/examples/jsm/libs/basis/ (same version as three!)
  .detectSupport(renderer);

const tex = await ktx2Loader.loadAsync("texture.ktx2");
tex.colorSpace = THREE.SRGBColorSpace; // if it is a color texture
```

Create KTX2 files with `toktx` or `gltf-transform etc1s/uastc`. Other formats: `DDSLoader`, `KTXLoader`, `PVRLoader`.

### FramebufferTexture

```javascript
// Copy the current framebuffer into a texture (e.g. for a frozen frame / minimap)
const fbTex = new THREE.FramebufferTexture(width, height);
renderer.copyFramebufferToTexture(fbTex, new THREE.Vector2(0, 0));
```

## Environment Maps

### Equirectangular HDR (most common)

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js"; // RGBELoader is deprecated since r180

const env = await new HDRLoader().loadAsync("studio.hdr");
env.mapping = THREE.EquirectangularReflectionMapping;
scene.environment = env;  // renderer pre-filters (PMREM) automatically
scene.background = env;
scene.backgroundBlurriness = 0.1;
scene.backgroundIntensity = 1;
scene.environmentIntensity = 1;
```

Free HDRs: Poly Haven. 1k-2k is enough for lighting; use a separate JPG for a sharp background.

### EXR and UltraHDR

```javascript
import { EXRLoader } from "three/addons/loaders/EXRLoader.js";
const exr = await new EXRLoader().loadAsync("env.exr");
exr.mapping = THREE.EquirectangularReflectionMapping;
scene.environment = exr;

// UltraHDR: HDR stored as a small JPEG with gain map (much smaller downloads)
import { UltraHDRLoader } from "three/addons/loaders/UltraHDRLoader.js";
const uhdr = await new UltraHDRLoader().loadAsync("env.jpg");
uhdr.mapping = THREE.EquirectangularReflectionMapping;
scene.environment = uhdr;
```

### Cube Maps (6 images)

```javascript
const cube = await new THREE.CubeTextureLoader()
  .setPath("textures/cube/")
  .loadAsync(["px.jpg", "nx.jpg", "py.jpg", "ny.jpg", "pz.jpg", "nz.jpg"]);
// CubeTextureLoader already sets colorSpace = SRGBColorSpace
scene.background = cube;
scene.environment = cube;
```

HDR cube faces: `HDRCubeTextureLoader` (`three/addons/loaders/HDRCubeTextureLoader.js`).

### Explicit PMREM (pre-filtered environment)

Needed when you assign an env map to a single material via `material.envMap` with blur, reuse
the result elsewhere, or generate from a scene:

```javascript
const pmrem = new THREE.PMREMGenerator(renderer);

const envRT = pmrem.fromEquirectangular(hdrTexture); // or pmrem.fromCubemap(cubeTexture)
material.envMap = envRT.texture;
hdrTexture.dispose(); // the source is no longer needed

// From a scene (procedural studio lighting, no files)
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;

pmrem.dispose(); // when done generating
```

### Ground-Projected Skybox

```javascript
import { GroundedSkybox } from "three/addons/objects/GroundedSkybox.js";
const skybox = new GroundedSkybox(hdrTexture, 15, 100); // map, camera height, radius
skybox.position.y = 15 - 0.01;
scene.add(skybox);
```

### Dynamic Reflections

`CubeCamera` + `WebGLCubeRenderTarget` (see `threejs-fundamentals`). Flat mirrors: `Reflector` (see `threejs-addons`).

## Render Targets (Render to Texture)

```javascript
const rt = new THREE.WebGLRenderTarget(512, 512, {
  type: THREE.HalfFloatType,   // HDR-capable; default UnsignedByteType
  samples: 4,                  // MSAA
  depthBuffer: true,
});

renderer.setRenderTarget(rt);
renderer.render(otherScene, otherCamera);
renderer.setRenderTarget(null); // back to the canvas

screenMaterial.map = rt.texture;

rt.setSize(w, h); // on resize
rt.dispose();
```

```javascript
// Depth texture
const depthRT = new THREE.WebGLRenderTarget(512, 512);
depthRT.depthTexture = new THREE.DepthTexture(512, 512);
depthRT.depthTexture.type = THREE.UnsignedIntType;

// Multiple render targets (MRT): one draw writes several textures
const mrt = new THREE.WebGLRenderTarget(512, 512, { count: 2 });
// mrt.textures[0], mrt.textures[1]; GLSL 3: layout(location = 1) out vec4 gNormal;

// Read pixels back to the CPU (slow)
const pixels = new Uint8Array(4);
renderer.readRenderTargetPixels(rt, x, y, 1, 1, pixels);
// or: await renderer.readRenderTargetPixelsAsync(rt, x, y, 1, 1, pixels);
```

## UV Mapping

```javascript
const uv = geometry.attributes.uv;
uv.getX(i); uv.getY(i);
uv.setXY(i, u, v);
uv.needsUpdate = true;
```

UV transforms in a shader: see `threejs-shaders`.

### Sprite Sheet / Atlas

```javascript
const atlas = await loader.loadAsync("atlas.png");
atlas.colorSpace = THREE.SRGBColorSpace;
atlas.magFilter = THREE.NearestFilter;
atlas.repeat.set(1 / cols, 1 / rows);

function showFrame(index) {
  const col = index % cols;
  const row = Math.floor(index / cols);
  atlas.offset.set(col / cols, 1 - (row + 1) / rows);
}
// Each material using this texture shows the same frame; clone() the texture per sprite.
```

### Fit an Image Like CSS object-fit

```javascript
import { TextureUtils } from "three";
TextureUtils.cover(texture, planeWidth / planeHeight);   // fills, crops
TextureUtils.contain(texture, planeWidth / planeHeight); // fits, letterboxes
```

## PBR Texture Set

```javascript
const material = new THREE.MeshStandardMaterial({
  map: colorTex,          // sRGB
  normalMap: normalTex,   // linear; DirectX-style maps: normalScale = (1, -1)
  roughnessMap: ormTex,   // G channel
  metalnessMap: ormTex,   // B channel (an ORM texture can feed AO/roughness/metalness)
  aoMap: ormTex,          // R channel
  displacementMap: heightTex,
  displacementScale: 0.05,
  emissiveMap: emissiveTex, // sRGB
  emissive: 0xffffff,       // multiplies emissiveMap, so must be non-black
});
material.normalMapType = THREE.TangentSpaceNormalMap; // default; ObjectSpaceNormalMap also exists
```

## Procedural Gradient

```javascript
function gradientTexture(c1, c2, size = 256) {
  const canvas = document.createElement("canvas");
  canvas.width = size;
  canvas.height = 1;
  const ctx = canvas.getContext("2d");
  const g = ctx.createLinearGradient(0, 0, size, 0);
  g.addColorStop(0, c1);
  g.addColorStop(1, c2);
  ctx.fillStyle = g;
  ctx.fillRect(0, 0, size, 1);
  const tex = new THREE.CanvasTexture(canvas);
  tex.colorSpace = THREE.SRGBColorSpace;
  return tex;
}
```

## Texture Memory Management

```javascript
texture.dispose(); // frees GPU memory; the texture re-uploads if used again

function disposeMaterialTextures(material) {
  for (const value of Object.values(material)) {
    if (value && value.isTexture) value.dispose();
  }
  material.dispose();
}

renderer.info.memory.textures; // live texture count
```

A cache keyed by URL (to avoid loading the same file twice) is shown in `threejs-loaders`.

## Performance Tips

1. **KTX2/Basis** for production: 4-8x less VRAM than JPG/PNG.
2. **Size limits**: 2048 is enough for most web uses; 1024 on mobile. Check `renderer.capabilities.maxTextureSize`.
3. **Atlases / texture arrays** reduce material count and draw calls.
4. **Mipmaps on** for anything seen at a distance.
5. **Reuse** textures between materials; `clone()` shares the image data.
6. **Dispose** textures you no longer need.

## See Also

- `threejs-materials` - Texture slots on materials
- `threejs-loaders` - GLTF, Draco/KTX2/Meshopt setup, caching
- `threejs-lighting` - Environment lighting
- `threejs-shaders` - Sampling textures in GLSL
