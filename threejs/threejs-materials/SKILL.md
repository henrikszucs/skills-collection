---
name: threejs-materials
description: Three.js materials - MeshStandardMaterial/MeshPhysicalMaterial PBR, basic/lambert/phong/toon/matcap, ShadowMaterial, points/lines/sprites, transparency, blending, side, depth, multi-material and material properties. Use when styling meshes, choosing a material type, making glass/metal/fabric/car paint, fixing transparency or z-fighting, or deciding when material.needsUpdate is required.
---

# Three.js Materials

> **Version:** written for three.js **r186**. See `threejs-migration` for renamed/removed APIs.
> These are the classic materials. With `WebGPURenderer` they are converted to node
> materials automatically, but `ShaderMaterial`/`onBeforeCompile` are not supported there;
> see `threejs-webgpu-tsl`.

## Quick Start

```javascript
import * as THREE from "three";

const material = new THREE.MeshStandardMaterial({
  color: 0x00ff00,
  roughness: 0.5,
  metalness: 0.0,
});
const mesh = new THREE.Mesh(geometry, material);
```

PBR materials look flat and dark without an environment map. Set `scene.environment`
(e.g. `RoomEnvironment` or an HDR, see `threejs-lighting` / `threejs-textures`).

## Material Types Overview

| Material             | Use Case                                        | Lighting            |
| -------------------- | ----------------------------------------------- | ------------------- |
| MeshBasicMaterial    | Unlit, flat colors, UI-like, wireframes         | No                  |
| MeshLambertMaterial  | Matte surfaces, cheap                           | Yes (diffuse)       |
| MeshPhongMaterial    | Shiny plastic, cheap specular                   | Yes (Blinn-Phong)   |
| MeshStandardMaterial | PBR metal/roughness (default choice)            | Yes (PBR)           |
| MeshPhysicalMaterial | Glass, clearcoat, sheen, iridescence, dispersion | Yes (PBR+)         |
| MeshToonMaterial     | Cel shading                                     | Yes (stepped)       |
| MeshMatcapMaterial   | Baked look from a matcap image, no lights       | No (matcap)         |
| MeshNormalMaterial   | Debug normals                                   | No                  |
| MeshDepthMaterial    | Depth visualization / custom depth              | No                  |
| MeshDistanceMaterial | Point-light shadow distance (customDistanceMaterial) | No             |
| ShadowMaterial       | Invisible surface that only shows shadows       | Shadows only        |
| PointsMaterial       | `THREE.Points`                                  | No                  |
| LineBasicMaterial / LineDashedMaterial | `THREE.Line*`                 | No                  |
| SpriteMaterial       | `THREE.Sprite` (always faces camera)            | No                  |
| ShaderMaterial / RawShaderMaterial | Custom GLSL (WebGLRenderer only)  | Custom              |

Cost, cheapest first: Basic < Lambert < Phong < Standard < Physical.

## MeshBasicMaterial

```javascript
new THREE.MeshBasicMaterial({
  color: 0xff0000,
  map: texture,              // sRGB color texture
  alphaMap: alphaTexture,    // green channel = opacity (needs transparent or alphaTest)
  envMap: envTexture,        // fake reflection
  reflectivity: 1,
  combine: THREE.MultiplyOperation, // how envMap mixes: MultiplyOperation, MixOperation, AddOperation
  wireframe: false,
  fog: true,
});
```

## MeshLambertMaterial

```javascript
new THREE.MeshLambertMaterial({
  color: 0x00ff00,
  emissive: 0x000000,
  emissiveIntensity: 1,
  map: texture,
  normalMap: normalTexture, // per-pixel lighting, normal/bump maps work
  envMap: envTexture,
  reflectivity: 0.5,
});
```

## MeshPhongMaterial

```javascript
new THREE.MeshPhongMaterial({
  color: 0x0000ff,
  specular: 0x111111,   // highlight color
  shininess: 30,        // highlight sharpness (default 30)
  emissive: 0x000000,
  flatShading: false,
  map: texture,
  specularMap: specTexture, // grayscale: how strongly each pixel shows specular/env reflection
  normalMap: normalTexture,
  normalScale: new THREE.Vector2(1, 1),
  bumpMap: bumpTexture,
  bumpScale: 1,
  displacementMap: dispTexture, // moves vertices: needs enough geometry segments
  displacementScale: 1,
});
```

## MeshStandardMaterial (PBR)

```javascript
const material = new THREE.MeshStandardMaterial({
  color: 0xffffff,
  roughness: 0.5,   // 0 = mirror, 1 = fully diffuse (default 1)
  metalness: 0.0,   // 0 = dielectric, 1 = metal (default 0). Use 0 or 1 in most cases

  map: colorTexture,          // sRGB
  roughnessMap: roughTexture, // green channel, multiplied by roughness
  metalnessMap: metalTexture, // blue channel, multiplied by metalness (ORM textures work directly)
  normalMap: normalTexture,
  normalScale: new THREE.Vector2(1, 1), // flip y (1, -1) for DirectX-style normal maps
  aoMap: aoTexture,           // red channel. Uses the "uv" set by default (aoTexture.channel = 0)
  aoMapIntensity: 1,
  lightMap: lightTexture,     // baked lighting
  lightMapIntensity: 1,
  displacementMap: dispTexture,
  displacementScale: 0.1,
  displacementBias: 0,
  bumpMap: bumpTexture,
  alphaMap: alphaTexture,

  emissive: 0x000000,         // glow color (needs bloom to actually glow, see threejs-postprocessing)
  emissiveIntensity: 1,
  emissiveMap: emissiveTexture,

  envMap: null,               // null = use scene.environment
  envMapIntensity: 1,
  envMapRotation: new THREE.Euler(),

  flatShading: false,
  wireframe: false,
  fog: true,
});
```

**AO/light maps and UVs:** the old rule "aoMap needs a `uv2` attribute" is gone (r151). Each
texture picks its UV set with `texture.channel` (0 = `uv`, 1 = `uv1`, 2 = `uv2`, 3 = `uv3`).
Only if your AO is baked into a second UV layout do you add it as `uv1` and set
`aoTexture.channel = 1`. GLTFLoader handles this automatically.

## MeshPhysicalMaterial (Advanced PBR)

Everything from MeshStandardMaterial plus:

```javascript
const material = new THREE.MeshPhysicalMaterial({
  // Clearcoat (car paint, lacquer)
  clearcoat: 1.0,
  clearcoatRoughness: 0.1,
  clearcoatMap: ccTexture,
  clearcoatRoughnessMap: ccrTexture,
  clearcoatNormalMap: ccnTexture,
  clearcoatNormalScale: new THREE.Vector2(1, 1),

  // Transmission (glass, water, gems) - physically based transparency, not .opacity
  transmission: 1.0,
  transmissionMap: transTexture,
  thickness: 0.5,             // volume thickness for refraction (needs a closed mesh)
  thicknessMap: thickTexture,
  attenuationColor: new THREE.Color(0xffffff),
  attenuationDistance: Infinity,
  ior: 1.5,                   // index of refraction, 1.0-2.333
  dispersion: 0,              // chromatic dispersion (prisms, diamonds), e.g. 1-5

  // Sheen (fabric, velvet)
  sheen: 1.0,
  sheenRoughness: 0.5,
  sheenColor: new THREE.Color(0xffffff),
  sheenColorMap: sheenTexture,
  sheenRoughnessMap: sheenRoughTexture,

  // Iridescence (soap bubbles, oil slicks)
  iridescence: 1.0,
  iridescenceIOR: 1.3,
  iridescenceThicknessRange: [100, 400], // nanometers
  iridescenceMap: iridTexture,
  iridescenceThicknessMap: iridThickTexture,

  // Anisotropy (brushed metal, hair)
  anisotropy: 1.0,
  anisotropyRotation: 0,
  anisotropyMap: anisoTexture,

  // Specular layer for dielectrics
  specularIntensity: 1,
  specularColor: new THREE.Color(0xffffff),
  specularIntensityMap: specIntTexture,
  specularColorMap: specColorTexture,
});
```

Transmission renders the opaque scene into an extra texture: it costs an extra pass and
transmissive objects do not see each other. Glass needs an environment map to look right.

### Recipes

```javascript
// Clear glass
new THREE.MeshPhysicalMaterial({ color: 0xffffff, metalness: 0, roughness: 0, transmission: 1, thickness: 0.5, ior: 1.5 });

// Frosted glass
new THREE.MeshPhysicalMaterial({ roughness: 0.4, transmission: 1, thickness: 1 });

// Car paint
new THREE.MeshPhysicalMaterial({ color: 0xff0000, metalness: 0.9, roughness: 0.5, clearcoat: 1, clearcoatRoughness: 0.03 });

// Gold
new THREE.MeshStandardMaterial({ color: 0xffd700, metalness: 1, roughness: 0.25 });

// Velvet
new THREE.MeshPhysicalMaterial({ color: 0x550022, roughness: 0.8, sheen: 1, sheenColor: 0xff88aa, sheenRoughness: 0.4 });
```

## MeshToonMaterial

```javascript
const colors = new Uint8Array([60, 160, 255]); // 3 shading bands
const gradientMap = new THREE.DataTexture(colors, colors.length, 1, THREE.RedFormat);
gradientMap.minFilter = THREE.NearestFilter;
gradientMap.magFilter = THREE.NearestFilter;
gradientMap.needsUpdate = true;

new THREE.MeshToonMaterial({ color: 0x00ff00, gradientMap });
```

Outlines: `OutlineEffect` (`three/addons/effects/OutlineEffect.js`) or the inverted-hull trick (BackSide, slightly scaled copy).

## MeshMatcapMaterial

```javascript
const matcap = new THREE.TextureLoader().load("matcap.png");
matcap.colorSpace = THREE.SRGBColorSpace;
new THREE.MeshMatcapMaterial({ matcap, color: 0xffffff, normalMap: null });
// No lights needed; looks like a lit sculpt. Great for previews and stylized looks.
```

## ShadowMaterial

```javascript
// Floor that is invisible except for received shadows (product shots, AR)
const floor = new THREE.Mesh(
  new THREE.PlaneGeometry(10, 10).rotateX(-Math.PI / 2),
  new THREE.ShadowMaterial({ opacity: 0.3, color: 0x000000 }),
);
floor.receiveShadow = true;
```

## Debug Materials

```javascript
new THREE.MeshNormalMaterial({ flatShading: false }); // RGB = normal direction
new THREE.MeshDepthMaterial({ depthPacking: THREE.RGBADepthPacking });
```

## PointsMaterial

```javascript
new THREE.PointsMaterial({
  color: 0xffffff,
  size: 0.1,            // world units if sizeAttenuation, else pixels
  sizeAttenuation: true,
  map: spriteTexture,   // round/soft particles
  alphaMap: alphaTexture,
  transparent: true,
  alphaTest: 0.01,      // or depthWrite: false to avoid square edges hiding particles
  depthWrite: false,
  blending: THREE.AdditiveBlending,
  vertexColors: true,
});
```

## LineBasicMaterial & LineDashedMaterial

```javascript
new THREE.LineBasicMaterial({ color: 0xffffff }); // linewidth is ignored by WebGL (always 1px)

const dashed = new THREE.LineDashedMaterial({ color: 0xffffff, dashSize: 0.5, gapSize: 0.25, scale: 1 });
const line = new THREE.Line(geometry, dashed);
line.computeLineDistances(); // REQUIRED for dashes
```

Thick or world-unit lines: `LineMaterial` + `Line2` (see `threejs-objects`).

## SpriteMaterial

```javascript
const sprite = new THREE.Sprite(new THREE.SpriteMaterial({ map: texture, color: 0xffffff, sizeAttenuation: true, rotation: 0 }));
sprite.scale.set(2, 1, 1);
```

## ShaderMaterial / RawShaderMaterial

Custom GLSL, `WebGLRenderer` only. Full guide in `threejs-shaders`.

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: { uTime: { value: 0 }, uColor: { value: new THREE.Color(0xff0000) } },
  vertexShader: /* glsl */ `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    uniform vec3 uColor;
    uniform float uTime;
    varying vec2 vUv;
    void main() {
      gl_FragColor = vec4(uColor * (0.5 + 0.5 * sin(uTime + vUv.x * 10.0)), 1.0);
    }
  `,
});
material.uniforms.uTime.value = timer.getElapsed(); // each frame
```

- `ShaderMaterial` prepends built-in uniforms/attributes (`projectionMatrix`, `modelViewMatrix`,
  `position`, `uv`, ...) and precision/defines for you.
- `RawShaderMaterial` prepends nothing: declare everything yourself. The renderer still
  fills the built-in matrix uniforms each draw, so you only declare them; do not pass them
  in `uniforms`.

To tweak a built-in material instead of writing everything: `material.onBeforeCompile` (see `threejs-shaders`).

## Common Material Properties

```javascript
material.visible = true;
material.transparent = false;  // needed for opacity < 1 / alpha textures with blending
material.opacity = 1.0;
material.alphaTest = 0;        // discard pixels with alpha below this (no sorting needed)
material.alphaHash = false;    // dithered transparency, no sorting issues (pairs well with TAA)
material.alphaToCoverage = false; // MSAA-smoothed cutouts (needs antialias: true)

material.side = THREE.FrontSide; // FrontSide, BackSide, DoubleSide
material.shadowSide = null;      // side used when rendering shadows

material.depthTest = true;
material.depthWrite = true;      // set false for most transparent particles/overlays
material.depthFunc = THREE.LessEqualDepth;
material.colorWrite = true;

material.blending = THREE.NormalBlending;
// NoBlending, NormalBlending, AdditiveBlending, SubtractiveBlending, MultiplyBlending, CustomBlending
material.premultipliedAlpha = false;

material.polygonOffset = true;   // fix z-fighting for coplanar decals
material.polygonOffsetFactor = -1;
material.polygonOffsetUnits = -1;

material.stencilWrite = false;
material.stencilFunc = THREE.AlwaysStencilFunc;
material.stencilRef = 0;
material.stencilZPass = THREE.ReplaceStencilOp;

material.clippingPlanes = null;  // see threejs-objects (Clipping)
material.dithering = false;      // reduces banding in gradients
material.toneMapped = true;      // false for UI/emissive-only materials that must stay exact
material.vertexColors = false;
material.forceSinglePass = false; // DoubleSide transparent objects render in 2 passes by default
material.userData = {};
```

### Transparency Checklist

1. Prefer `alphaTest` (cutouts: leaves, fences) or `alphaHash` over `transparent: true`.
2. Transparent objects are sorted per object, not per triangle. Overlapping transparent meshes
   can still be wrong: tune `renderOrder`, set `depthWrite: false`, or split meshes.
3. For additive glow/particles: `transparent: true, depthWrite: false, blending: THREE.AdditiveBlending`.

## Multiple Materials

```javascript
// BoxGeometry groups: +X, -X, +Y, -Y, +Z, -Z
const materials = [0xff0000, 0x00ff00, 0x0000ff, 0xffff00, 0xff00ff, 0x00ffff].map(
  (c) => new THREE.MeshBasicMaterial({ color: c }),
);
const cube = new THREE.Mesh(new THREE.BoxGeometry(), materials);
```

Custom groups: `geometry.addGroup(start, count, materialIndex)` (see `threejs-geometry`).

## Environment Maps

```javascript
scene.environment = envMap; // all PBR materials use it unless material.envMap is set
material.envMapIntensity = 1;
```

Loading HDR/EXR/cube maps and PMREM: see `threejs-textures` (Environment Maps). Quick neutral
studio lighting without files: `RoomEnvironment` (see `threejs-lighting`).

## Changing Materials at Runtime

```javascript
material.color.set(0xff0000); // uniforms: no recompile needed
material.roughness = 0.2;
material.map = otherTexture;  // swapping one texture for another: no recompile

// material.needsUpdate = true is required when the shader itself changes:
// - adding/removing a texture slot (null <-> texture) for map, normalMap, envMap, etc.
// - flatShading, vertexColors, fog, side to/from DoubleSide in some cases
// - alphaTest switching between 0 and > 0, transparent, alphaHash, alphaToCoverage
// - defines on ShaderMaterial, or a new onBeforeCompile
material.needsUpdate = true;

const clone = material.clone(); // shares textures; dispose textures separately
```

## Performance Tips

1. **Reuse materials**: identical material objects share one shader program.
2. **Fewer unique material configs** = fewer shader compiles (stutters). Use `renderer.compileAsync`.
3. **Avoid transparency** when alphaTest/alphaHash works.
4. **Cheaper material types** for distant/background objects.
5. **Limit transmission and shadow-casting lights**: each adds passes.

```javascript
const materialCache = new Map();
function getMaterial(hex) {
  if (!materialCache.has(hex)) materialCache.set(hex, new THREE.MeshStandardMaterial({ color: hex }));
  return materialCache.get(hex);
}
material.dispose(); // when no mesh uses it anymore (textures are disposed separately)
```

## See Also

- `threejs-textures` - Texture loading, color spaces, environment maps
- `threejs-shaders` - ShaderMaterial and onBeforeCompile
- `threejs-lighting` - How lights affect materials
- `threejs-webgpu-tsl` - Node materials for WebGPURenderer
