---
name: threejs-materials
description: Three.js materials - MeshStandardMaterial/MeshPhysicalMaterial PBR, basic/lambert/phong/toon/matcap, ShadowMaterial, points/lines/sprites, transparency, blending, side, depth, multi-material and material properties. Use when styling meshes, choosing a material type, making glass/metal/fabric/car paint, fixing transparency or z-fighting, or deciding when material.needsUpdate is required.
---

# Three.js Materials

> three.js **r187**. Classic materials work with `WebGPURenderer` too (auto-converted to node
> materials), but `ShaderMaterial`/`onBeforeCompile` do not; see `threejs-webgpu-tsl`.

## Choosing a Material

| Material | Use for | Lit |
| --- | --- | --- |
| `MeshBasicMaterial` | unlit colors, UI, wireframes | no |
| `MeshLambertMaterial` / `MeshPhongMaterial` | cheap matte / cheap shiny | yes |
| `MeshStandardMaterial` | PBR metal/roughness: **default choice** | yes |
| `MeshPhysicalMaterial` | glass, clearcoat, sheen, iridescence, dispersion | yes (PBR+) |
| `MeshToonMaterial` | cel shading (`gradientMap`) | stepped |
| `MeshMatcapMaterial` | baked look from a matcap image, no lights | no |
| `MeshNormalMaterial` / `MeshDepthMaterial` | debugging | no |
| `ShadowMaterial` | invisible surface that only shows received shadows | shadows |
| `PointsMaterial` / `LineBasicMaterial` / `LineDashedMaterial` / `SpriteMaterial` | Points / Line / Sprite | no |
| `ShaderMaterial` / `RawShaderMaterial` | custom GLSL (`threejs-shaders`) | custom |

Cost: Basic < Lambert < Phong < Standard < Physical. **PBR looks flat and dark without an
environment map**: set `scene.environment` (`threejs-lighting`).

## PBR (Standard)

```javascript
const mat = new THREE.MeshStandardMaterial({
  color: 0xffffff,
  roughness: 0.5,          // default 1
  metalness: 0,            // 0 or 1 in most cases
  map: colorTex,           // sRGB
  normalMap: normalTex,    // normalScale (1, -1) for DirectX-style maps
  roughnessMap: ormTex,    // G channel
  metalnessMap: ormTex,    // B channel
  aoMap: ormTex,           // R channel; uses "uv" unless aoTex.channel = 1 (uv1)
  emissive: 0x000000,      // glows only with bloom (threejs-postprocessing)
  emissiveMap: emissiveTex,
  envMapIntensity: 1,      // envMap null = scene.environment
});
```

`aoMap`/`lightMap` no longer need a `uv2` attribute (r151). Pick the UV set per texture with
`texture.channel` (0 = `uv`, 1 = `uv1`, ...).

## Physical Extras

```javascript
new THREE.MeshPhysicalMaterial({
  clearcoat: 1, clearcoatRoughness: 0.05,                   // car paint, lacquer
  transmission: 1, thickness: 0.5, ior: 1.5,                // glass (not .opacity); needs a closed mesh
  attenuationColor: 0xffffff, attenuationDistance: Infinity,
  dispersion: 0,                                            // prisms, gems (e.g. 1-5)
  sheen: 1, sheenRoughness: 0.5, sheenColor: 0xffffff,      // fabric, velvet
  iridescence: 1, iridescenceIOR: 1.3,                      // soap bubbles
  iridescenceThicknessMinimum: 100, iridescenceThicknessMaximum: 400, // nm (r187; replaces iridescenceThicknessRange)
  anisotropy: 1, anisotropyRotation: 0,                     // brushed metal
  specularIntensity: 1, specularColor: 0xffffff,
  diffuseRoughness: 0,                                      // r187: > 0 uses the EON rough-diffuse model (clay, fabric)
});
```

Transmission renders the opaque scene into an extra texture (one more pass); transmissive
objects do not see each other. Glass needs an environment map.

```javascript
// Recipes
new THREE.MeshPhysicalMaterial({ roughness: 0, transmission: 1, thickness: 0.5 });          // clear glass
new THREE.MeshPhysicalMaterial({ roughness: 0.4, transmission: 1, thickness: 1 });          // frosted glass
new THREE.MeshPhysicalMaterial({ color: 0xff0000, metalness: 0.9, roughness: 0.5, clearcoat: 1, clearcoatRoughness: 0.03 }); // car paint
new THREE.MeshStandardMaterial({ color: 0xffd700, metalness: 1, roughness: 0.25 });         // gold
```

## Other Types in Brief

- **Toon:** `gradientMap` = a tiny `DataTexture` (e.g. 3 `RedFormat` texels) with `NearestFilter`
  on min and mag. Outlines: `OutlineEffect` addon or an inverted hull (BackSide, scaled copy).
- **Matcap:** `new MeshMatcapMaterial({ matcap })`, matcap texture in sRGB; no lights needed.
- **ShadowMaterial:** `new ShadowMaterial({ opacity: 0.3 })` on a floor with `receiveShadow = true`.
- **Points:** `size` is world units with `sizeAttenuation`, else pixels. Soft particles:
  `map`, `transparent: true`, `depthWrite: false`, `blending: AdditiveBlending`.
- **Lines:** `linewidth` is ignored (always 1px). Dashes need `line.computeLineDistances()`.
  Thick lines: `Line2` + `LineMaterial` (`threejs-objects`).

## Common Properties

```javascript
material.side = THREE.DoubleSide;     // FrontSide (default), BackSide
material.transparent = true;          // needed for opacity < 1 with blending
material.opacity = 0.5;
material.alphaTest = 0.5;             // cutouts without sorting (leaves, fences)
material.alphaHash = true;            // dithered transparency, no sorting problems
material.alphaToCoverage = true;      // MSAA-smoothed cutouts (antialias: true)
material.depthWrite = false;          // most transparent particles/overlays
material.blending = THREE.AdditiveBlending;
material.polygonOffset = true;        // coplanar decals / z-fighting
material.polygonOffsetFactor = -1;
material.toneMapped = false;          // UI or HDR colors that must stay exact
material.dithering = true;            // reduce gradient banding
material.clippingPlanes = [plane];    // needs renderer.localClippingEnabled
```

**Transparency:** prefer `alphaTest` or `alphaHash`. Transparent objects sort per object, not per
triangle: tune `renderOrder`, `depthWrite: false`, or split meshes.

## Multi-Material

`new THREE.Mesh(geometry, [matA, matB])` uses one material per geometry group (`BoxGeometry` has 6
groups: +X, -X, +Y, -Y, +Z, -Z; custom groups via `geometry.addGroup`).

## When `needsUpdate` Is Required

Uniform-like changes need nothing: `color.set()`, `roughness`, swapping one texture for another.

Set `material.needsUpdate = true` when the **shader** changes:
- a texture slot goes from `null` to a texture or back (`map`, `normalMap`, `envMap`, ...)
- `flatShading`, `vertexColors`, `fog`, `transparent`, `alphaTest` 0 <-> > 0, `alphaHash`
- `defines` on a ShaderMaterial, or a new `onBeforeCompile`

## Performance

1. Reuse material instances; identical configs share a shader program.
2. Fewer distinct configurations = fewer compiles. Use `renderer.compileAsync`.
3. Avoid `transparent` when `alphaTest`/`alphaHash` works.
4. Limit transmission and shadow-casting lights (each adds passes).
5. `material.dispose()` when unused; textures are disposed separately.

See also: `threejs-textures`, `threejs-lighting`, `threejs-shaders`, `threejs-webgpu-tsl`.
