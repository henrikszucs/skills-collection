---
name: threejs-textures
description: Three.js textures - loading, color space, wrapping/repeat, filtering, mipmaps, UV channels, data/canvas/video/compressed (KTX2) textures, environment maps (HDR, EXR, cube maps, PMREM, skyboxes), render targets and texture memory. Use when working with images, fixing washed-out or dark colors, tiling textures, UV coordinates, cubemaps, HDR environments, render-to-texture, or texture optimization.
---

# Three.js Textures

> three.js **r187**. The canonical reference for **environment maps** (HDR/EXR/cube/PMREM).

## Loading

```javascript
const loader = new THREE.TextureLoader();
const tex = await loader.loadAsync("brick.jpg");
tex.colorSpace = THREE.SRGBColorSpace; // REQUIRED for color images, else washed out
tex.anisotropy = renderer.capabilities.getMaxAnisotropy(); // sharp at grazing angles

material.map = loader.load("x.jpg");   // sync form: fills in when loaded
```

**Color space rule:** `map`, `emissiveMap`, `sheenColorMap`, `specularColorMap`, matcaps and
backgrounds are sRGB. Normal, roughness, metalness, AO, displacement, alpha and bump maps keep the
default `NoColorSpace`. GLTFLoader sets this for you.

`ImageBitmapLoader` decodes off the main thread but ignores `flipY`: pass
`setOptions({ imageOrientation: "flipY" })`.

## Configuration

```javascript
tex.wrapS = tex.wrapT = THREE.RepeatWrapping; // default ClampToEdgeWrapping
tex.repeat.set(4, 4);
tex.offset.set(0.5, 0);
tex.center.set(0.5, 0.5); tex.rotation = Math.PI / 4;

// Pixel art
tex.minFilter = tex.magFilter = THREE.NearestFilter;
tex.generateMipmaps = false;

tex.flipY = false;        // TextureLoader images default true, glTF textures false
tex.needsUpdate = true;   // after changing wrap/flip/format on an uploaded texture
tex.channel = 1;          // read uv1 instead of uv (0 = uv, 1 = uv1, ...)
tex.mipmapsAutoUpdate = false; // r187: stop regenerating mips (write your own levels)
```

NPOT textures get mipmaps in WebGL 2; power-of-two is still slightly better for compression.

`THREE.TextureUtils.cover(tex, aspect)` / `contain(tex, aspect)` fit an image like CSS `object-fit`.

## Texture Types

```javascript
// Raw pixels. Defaults differ from images: NearestFilter, no mipmaps, flipY false.
const data = new Uint8Array(w * h * 4);
const dt = new THREE.DataTexture(data, w, h, THREE.RGBAFormat); // FloatType for float data
dt.needsUpdate = true;
// Also Data3DTexture, DataArrayTexture (sampler3D / sampler2DArray in GLSL 3)

const ct = new THREE.CanvasTexture(canvas); // set colorSpace; needsUpdate = true after redrawing

video.muted = true; video.playsInline = true; await video.play(); // autoplay needs muted
const vt = new THREE.VideoTexture(video);   // updates itself; set colorSpace
```

### Compressed (KTX2 / Basis)

4-8x less VRAM and faster upload than JPG/PNG. Set up `KTX2Loader` as in `threejs-loaders`
(transcoder path + `detectSupport(renderer)`), then `await ktx2.loadAsync("texture.ktx2")`.
Create with `toktx` or `gltf-transform etc1s/uastc`.

## Environment Maps

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js"; // RGBELoader is deprecated (r180)
const env = await new HDRLoader().loadAsync("studio.hdr");
env.mapping = THREE.EquirectangularReflectionMapping;
scene.environment = env;  // pre-filtered (PMREM) automatically
scene.background = env;   // backgroundBlurriness, backgroundIntensity
```

- **EXR:** `EXRLoader`, same usage. **UltraHDR:** `UltraHDRLoader` (HDR as a small JPEG + gain map).
- **Cube maps:** `new THREE.CubeTextureLoader().setPath(p).loadAsync(["px","nx","py","ny","pz","nz"].map(f => f + ".jpg"))`
  (sets sRGB for you). HDR faces: `HDRCubeTextureLoader`.
- Free HDRs: Poly Haven. 1-2k is enough for lighting; use a separate image for a sharp background.

**Explicit PMREM** (per-material `envMap`, reuse, or generate from a scene):

```javascript
const pmrem = new THREE.PMREMGenerator(renderer);
material.envMap = pmrem.fromEquirectangular(hdr).texture; // or fromCubemap(cube)
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture; // no files
pmrem.dispose();
```

PMREMs are cube render targets since r187 (`CubeUVReflectionMapping` was removed). Use the
generator; don't build that mapping by hand.

**Grounded skybox:** `new GroundedSkybox(hdr, height, radius)` from `three/addons/objects/GroundedSkybox.js`,
`skybox.position.y = height - 0.01`. Dynamic reflections: `CubeCamera` (`threejs-fundamentals`);
flat mirrors: `Reflector` (`threejs-addons`).

## Render Targets

```javascript
const rt = new THREE.WebGLRenderTarget(512, 512, { type: THREE.HalfFloatType, samples: 4 });
renderer.setRenderTarget(rt);
renderer.render(otherScene, otherCamera);
renderer.setRenderTarget(null);
screenMaterial.map = rt.texture;
// rt.setSize(w, h) on resize; rt.dispose() when done

rt.depthTexture = new THREE.DepthTexture(512, 512);            // read depth
const mrt = new THREE.WebGLRenderTarget(w, h, { count: 2 });   // MRT: mrt.textures[0..1]
await renderer.readRenderTargetPixelsAsync(rt, x, y, 1, 1, buf); // CPU readback (slow)
```

## Sprite Sheet

```javascript
atlas.repeat.set(1 / cols, 1 / rows);
const show = (i) => atlas.offset.set((i % cols) / cols, 1 - (Math.floor(i / cols) + 1) / rows);
// every material using this texture shows the same frame: clone() the texture per sprite
```

## Memory

- `texture.dispose()` frees GPU memory (re-uploads if used again). `renderer.info.memory.textures` counts them.
- 2048 max for most web uses, 1024 on mobile; check `renderer.capabilities.maxTextureSize`.
- Production: KTX2. Atlases/array textures cut draw calls. Reuse textures across materials.

See also: `threejs-materials`, `threejs-loaders` (caching), `threejs-lighting`, `threejs-shaders`.
