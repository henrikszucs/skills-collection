---
name: threejs-lighting
description: Three.js lighting - light types and physical intensity units, shadows (PCF/VSM, bias, frustum), contact/fake shadows, image-based lighting with RoomEnvironment/HDR, helpers and lighting setups. Use when adding lights, scenes look too dark or too bright, configuring or debugging shadows, setting up IBL/environment lighting, or optimizing lighting performance.
---

# Three.js Lighting

> three.js **r187**. See `threejs-migration` for renamed APIs.

## Quick Start

```javascript
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";

// Image-based lighting: the biggest quality win for PBR materials
const pmrem = new THREE.PMREMGenerator(renderer);
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;

const sun = new THREE.DirectionalLight(0xffffff, 3); // key light for direction + shadows
sun.position.set(5, 10, 5);
scene.add(sun);
renderer.toneMapping = THREE.ACESFilmicToneMapping; // NeutralToneMapping for product colors, AgX for bright lights
```

## Intensity Units (read first)

Lighting is **physical** (legacy mode removed in r165); 1 unit = 1 meter.

| Light | Unit | Typical | Shadows |
| --- | --- | --- | --- |
| `AmbientLight` | unitless | 0.1-1 | no |
| `HemisphereLight(sky, ground, i)` | unitless | 0.5-3 | no |
| `DirectionalLight` | lux-like | 1-5 (sun ~3) | yes |
| `PointLight` | candela | 10-1000+ | yes (6 renders, expensive) |
| `SpotLight` | candela | 10-1000+ | yes |
| `RectAreaLight` | nits | 1-20 | no |

Point/spot lights fall off with inverse square (`decay = 2`). `PointLight(0xffffff, 1)` 5 m away is
almost black: raise **intensity**, not decay. Old tutorials using `1` assume the removed legacy mode.

WebGPU-only: `IESSpotLight`, `ProjectorLight` (`threejs-webgpu-tsl`).

## Lights Gotchas

- Directional and spot lights shine from `position` toward `target` (default origin). If you move
  `light.target`, add it to the scene so its matrix updates.
- `SpotLight(color, intensity, distance, angle, penumbra, decay)`; `spot.map = texture` projects a
  gobo (needs `castShadow = true`).
- `RectAreaLight` emits along local -Z (`lookAt` it), affects only Standard/Physical, no shadows.
  WebGL needs `RectAreaLightUniformsLib.init()` once
  (`three/addons/lights/RectAreaLightUniformsLib.js`); WebGPU setup is in `threejs-webgpu-tsl`.
- Changing the **number** of lights or shadow casters recompiles every material. Toggle
  `intensity` instead of adding/removing lights.

## Shadows

```javascript
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap; // default; VSMShadowMap = very soft. PCFSoftShadowMap is gone (r186)

sun.castShadow = true;
sun.shadow.mapSize.set(2048, 2048);
const cam = sun.shadow.camera;            // orthographic: keep it TIGHT around the visible area
cam.left = -10; cam.right = 10; cam.top = 10; cam.bottom = -10; cam.near = 0.5; cam.far = 50;
cam.updateProjectionMatrix();
sun.shadow.bias = -0.0001;                // acne (stripes)
sun.shadow.normalBias = 0.02;             // curved surfaces; too high = peter-panning
sun.shadow.radius = 4;                    // softness
sun.shadow.intensity = 0.8;               // lighten shadows without dimming the light

mesh.castShadow = true;
floor.receiveShadow = true;
scene.add(new THREE.CameraHelper(sun.shadow.camera)); // debug the frustum

renderer.shadowMap.autoUpdate = false;    // static scenes: render once...
renderer.shadowMap.needsUpdate = true;    // ...and again whenever something moves
```

**No shadows?** Check in order: `shadowMap.enabled`, `light.castShadow`, `mesh.castShadow`,
`floor.receiveShadow`, object inside the shadow frustum, lit material (Basic receives none).
**Blocky** = frustum too big or `mapSize` too small. Large outdoor scenes: `CSM` (`threejs-addons`).

### Fake / Contact Shadows

three.js has no `ContactShadows` class (that is drei). Options, cheapest last:
1. `ShadowMaterial` floor + a real light shadow.
2. Planar projected shadow: `ShadowMesh` (`three/addons/objects/ShadowMesh.js`).
3. Soft contact shadow: render `MeshDepthMaterial` top-down into a render target and blur it
   (official example `webgl_shadow_contact`).
4. Blob: a transparent plane with a radial-gradient texture, `depthWrite: false`.

## Environment Lighting (IBL)

No files: `RoomEnvironment` (Quick Start). From an HDR: load it with `HDRLoader`, set
`mapping = THREE.EquirectangularReflectionMapping` and assign it to `scene.environment` (the
renderer pre-filters it) and optionally `scene.background`. Tune with `scene.environmentIntensity`,
`environmentRotation.y` and `backgroundBlurriness`. Code, EXR, cube maps, explicit PMREM and
grounded skyboxes: `threejs-textures`.

## Setups

```javascript
// Product / studio: RoomEnvironment IBL (Quick Start)
renderer.toneMapping = THREE.NeutralToneMapping;
// + one DirectionalLight(0xffffff, 2) with castShadow + ShadowMaterial floor

// Outdoor
const sunLight = new THREE.DirectionalLight(0xfff4e5, 3);
scene.add(sunLight, new THREE.HemisphereLight(0x87ceeb, 0x6b5a3a, 1)); // + Sky addon

// Indoor night
const lamp = new THREE.PointLight(0xffc58f, 30, 0, 2); // warm bulb, candela
scene.add(lamp, new THREE.AmbientLight(0x202030, 0.3));
renderer.toneMapping = THREE.AgXToneMapping;
```

**Selective lighting:** a light affects everything if it shares a layer with the **camera**; there
is no per-object light list in WebGLRenderer. Render a second scene with its own lights on top
(`autoClear = false`, `clearDepth()`), or use unlit materials. WebGPU: `material.lightsNode = lights([...])`.

Helpers: `DirectionalLightHelper`, `PointLightHelper`, `SpotLightHelper` (call `update()` after
changes), `HemisphereLightHelper`, `RectAreaLightHelper` (addon).

## Performance

1. Few dynamic lights; use an environment map for fill.
2. Bake static lighting into `lightMap`s.
3. Shadow maps 1024-2048; a tight frustum matters more than resolution.
4. Avoid point-light shadows; only important objects cast.
5. Static shadows: `shadowMap.autoUpdate = false`.

See also: `threejs-materials`, `threejs-textures`, `threejs-postprocessing`, `threejs-addons`.
