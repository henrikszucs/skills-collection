---
name: threejs-lighting
description: Three.js lighting - light types and physical intensity units, shadows (PCF/VSM, bias, frustum), contact/fake shadows, image-based lighting with RoomEnvironment/HDR, light probes, helpers and lighting setups. Use when adding lights, scenes look too dark or too bright, configuring or debugging shadows, setting up IBL/environment lighting, or optimizing lighting performance.
---

# Three.js Lighting

> **Version:** written for three.js **r186**. See `threejs-migration` for renamed/removed APIs.

## Quick Start

```javascript
import * as THREE from "three";
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";

// Image-based lighting: the single biggest quality win for PBR materials
const pmrem = new THREE.PMREMGenerator(renderer);
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;

// A key light for direction and shadows
const sun = new THREE.DirectionalLight(0xffffff, 3);
sun.position.set(5, 10, 5);
scene.add(sun);

renderer.toneMapping = THREE.ACESFilmicToneMapping; // or NeutralToneMapping / AgXToneMapping
```

## Intensity Units (read this first)

Lighting is **physically based** (the legacy mode was removed in r165). Assume 1 unit = 1 meter.

| Light            | Unit            | Typical values                     |
| ---------------- | --------------- | ---------------------------------- |
| AmbientLight     | unitless        | 0.1 - 1                            |
| HemisphereLight  | unitless        | 0.5 - 3                            |
| DirectionalLight | lux-like        | 1 - 5 (sun ~3)                     |
| PointLight       | candela         | 10 - 1000+ depending on distance   |
| SpotLight        | candela         | 10 - 1000+                         |
| RectAreaLight    | nits            | 1 - 20                             |

Point and spot lights fall off with the inverse square of distance (`decay = 2`, default).
`PointLight(0xffffff, 1)` 5 m away is nearly black: raise intensity, not decay. Old tutorials
using intensity 1 for point lights were written for the removed legacy mode.

## Light Types Overview

| Light            | Description            | Shadows | Cost     |
| ---------------- | ---------------------- | ------- | -------- |
| AmbientLight     | Uniform everywhere     | No      | Very Low |
| HemisphereLight  | Sky/ground gradient    | No      | Very Low |
| DirectionalLight | Parallel rays (sun)    | Yes     | Low      |
| PointLight       | Omnidirectional (bulb) | Yes (6 renders) | Medium |
| SpotLight        | Cone (flashlight)      | Yes     | Medium   |
| RectAreaLight    | Rectangle (window, softbox) | No | High     |
| LightProbe       | Spherical-harmonics ambient | No | Very Low |

WebGPU-only: `IESSpotLight` (photometric profiles), `ProjectorLight` (projects a texture). See `threejs-webgpu-tsl`.

## AmbientLight

```javascript
const ambient = new THREE.AmbientLight(0xffffff, 0.4); // color, intensity
scene.add(ambient);
```

Flat, no direction: prefer `HemisphereLight` or an environment map for nicer fill.

## HemisphereLight

```javascript
// skyColor, groundColor, intensity
const hemi = new THREE.HemisphereLight(0x87ceeb, 0x8b5a2b, 1.5);
hemi.position.set(0, 50, 0); // only the direction matters
scene.add(hemi);
```

## DirectionalLight

```javascript
const dirLight = new THREE.DirectionalLight(0xffffff, 3);
dirLight.position.set(5, 10, 5);
scene.add(dirLight);

// It shines from position toward target (default target at 0,0,0).
// To move the target, add it to the scene so its matrix updates:
dirLight.target.position.set(2, 0, 0);
scene.add(dirLight.target);
```

### DirectionalLight Shadows

```javascript
renderer.shadowMap.enabled = true;

dirLight.castShadow = true;
dirLight.shadow.mapSize.set(2048, 2048);

// Orthographic shadow camera: make it as TIGHT as possible around the visible area
const cam = dirLight.shadow.camera;
cam.left = -10; cam.right = 10; cam.top = 10; cam.bottom = -10;
cam.near = 0.5; cam.far = 50;
cam.updateProjectionMatrix(); // after changing the frustum

dirLight.shadow.bias = -0.0001;   // fixes shadow acne (stripes)
dirLight.shadow.normalBias = 0.02; // better for curved surfaces; too high = peter-panning
dirLight.shadow.radius = 4;        // softness (PCFShadowMap and VSMShadowMap)
dirLight.shadow.intensity = 1;     // 0-1, lighten shadows without changing the light

scene.add(new THREE.CameraHelper(dirLight.shadow.camera)); // debug the frustum
```

Large outdoor scenes: cascaded shadow maps (`CSM` from `three/addons/csm/CSM.js`, see `threejs-addons`).

## PointLight

```javascript
// color, intensity (candela), distance (0 = no cutoff), decay (2 = physical)
const bulb = new THREE.PointLight(0xffaa66, 50, 0, 2);
bulb.position.set(0, 3, 0);
scene.add(bulb);

bulb.castShadow = true;            // renders 6 shadow maps: expensive
bulb.shadow.mapSize.set(1024, 1024);
bulb.shadow.camera.near = 0.1;
bulb.shadow.camera.far = 20;
bulb.shadow.bias = -0.001;
```

## SpotLight

```javascript
// color, intensity, distance, angle (max PI/2), penumbra (0-1), decay
const spot = new THREE.SpotLight(0xffffff, 100, 0, Math.PI / 6, 0.3, 2);
spot.position.set(0, 8, 0);
spot.target.position.set(0, 0, 0);
scene.add(spot, spot.target);

spot.castShadow = true;
spot.shadow.mapSize.set(1024, 1024);
spot.shadow.camera.near = 0.5;
spot.shadow.camera.far = 30;
spot.shadow.focus = 1; // shrinks the shadow frustum (0-1)

// Project a texture (gobo/cookie). Requires castShadow = true.
spot.map = goboTexture;
```

## RectAreaLight

```javascript
import { RectAreaLightUniformsLib } from "three/addons/lights/RectAreaLightUniformsLib.js";
import { RectAreaLightHelper } from "three/addons/helpers/RectAreaLightHelper.js";

RectAreaLightUniformsLib.init(); // once, before rendering (WebGLRenderer only)

// color, intensity, width, height
const rectLight = new THREE.RectAreaLight(0xffffff, 5, 4, 2);
rectLight.position.set(0, 5, 0);
rectLight.lookAt(0, 0, 0);       // emits along its local -Z
scene.add(rectLight);
rectLight.add(new RectAreaLightHelper(rectLight));
```

Only affects `MeshStandardMaterial` / `MeshPhysicalMaterial`; no shadows. WebGPU setup differs (see `threejs-webgpu-tsl`).

## Shadow Setup Checklist

```javascript
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFShadowMap; // default
// THREE.VSMShadowMap  - very soft, can light-bleed; uses shadow.radius and shadow.blurSamples
// THREE.BasicShadowMap - unfiltered, fastest
// THREE.PCFSoftShadowMap is deprecated (r186): the renderer falls back to PCF with a warning. Use PCF + radius

light.castShadow = true;
mesh.castShadow = true;    // objects that cast
floor.receiveShadow = true; // objects that receive

// Static scene: render the shadow map once
renderer.shadowMap.autoUpdate = false;
renderer.shadowMap.needsUpdate = true; // set again whenever something moves
```

**No shadows showing?** Check, in order: renderer `shadowMap.enabled`, `light.castShadow`,
`mesh.castShadow`, `floor.receiveShadow`, the object is inside the shadow camera frustum
(use `CameraHelper`), and the material is lit (MeshBasicMaterial does not receive shadows).

**Artifacts:** stripes = acne (raise `normalBias`/negative `bias`); shadow detached from the
object = too much bias; blocky = frustum too large or `mapSize` too small.

### Fake / Contact Shadows

three.js has no `ContactShadows` class (that is from the React library drei). Options:

```javascript
// 1. Shadow-only floor with a real light shadow (simplest)
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(20, 20).rotateX(-Math.PI / 2),
  new THREE.ShadowMaterial({ opacity: 0.25 }),
);
ground.receiveShadow = true;
scene.add(ground);

// 2. Blob shadow: a transparent plane with a radial-gradient texture under each object (cheapest)
const blob = new THREE.Mesh(
  new THREE.PlaneGeometry(1, 1).rotateX(-Math.PI / 2),
  new THREE.MeshBasicMaterial({ map: radialGradientTexture, transparent: true, depthWrite: false }),
);
blob.position.y = 0.001;

// 3. Planar projected shadow: ShadowMesh from three/addons/objects/ShadowMesh.js
// 4. Soft contact shadow: render a MeshDepthMaterial top-down into a render target and blur it
//    (official example: webgl_shadow_contact)
```

## Light Helpers

```javascript
import { RectAreaLightHelper } from "three/addons/helpers/RectAreaLightHelper.js";

scene.add(new THREE.DirectionalLightHelper(dirLight, 1));
scene.add(new THREE.PointLightHelper(pointLight, 0.5));
const spotHelper = new THREE.SpotLightHelper(spotLight);
scene.add(spotHelper);
scene.add(new THREE.HemisphereLightHelper(hemiLight, 1));
rectLight.add(new RectAreaLightHelper(rectLight));
scene.add(new THREE.CameraHelper(light.shadow.camera));

spotHelper.update(); // after moving/changing the light
```

## Environment Lighting (IBL)

An environment map lights PBR materials from all directions and gives reflections.
`scene.environment` accepts an equirectangular texture directly: the renderer pre-filters it (PMREM) for you.

```javascript
import { HDRLoader } from "three/addons/loaders/HDRLoader.js"; // RGBELoader is deprecated (r180)

const hdr = await new HDRLoader().loadAsync("environment.hdr");
hdr.mapping = THREE.EquirectangularReflectionMapping;
scene.environment = hdr;
scene.background = hdr;            // optional
scene.backgroundBlurriness = 0.2;  // 0-1
scene.environmentIntensity = 1;
scene.environmentRotation.y = Math.PI / 2;
```

No asset needed:

```javascript
import { RoomEnvironment } from "three/addons/environments/RoomEnvironment.js";
const pmrem = new THREE.PMREMGenerator(renderer);
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture; // sigma = blur
```

Cube maps, EXR, explicit PMREM, ground-projected skyboxes: see `threejs-textures` (Environment Maps).

## Light Probes

Capture ambient lighting as spherical harmonics (cheap diffuse IBL).

```javascript
import { LightProbeGenerator } from "three/addons/lights/LightProbeGenerator.js";

const lightProbe = new THREE.LightProbe();
scene.add(lightProbe);

// From a cube texture (synchronous)
lightProbe.copy(LightProbeGenerator.fromCubeTexture(cubeTexture));

// From a CubeCamera render (async since r16x: await it)
const cubeRenderTarget = new THREE.WebGLCubeRenderTarget(256);
const cubeCamera = new THREE.CubeCamera(0.1, 100, cubeRenderTarget);
cubeCamera.update(renderer, scene);
lightProbe.copy(await LightProbeGenerator.fromCubeRenderTarget(renderer, cubeRenderTarget));
```

## Common Lighting Setups

### Product / Studio

```javascript
scene.environment = pmrem.fromScene(new RoomEnvironment(), 0.04).texture;
renderer.toneMapping = THREE.NeutralToneMapping; // keeps brand colors accurate
const key = new THREE.DirectionalLight(0xffffff, 2);
key.position.set(3, 5, 2);
key.castShadow = true;
scene.add(key);
// + ShadowMaterial floor (above)
```

### Three-Point Lighting

```javascript
const keyLight = new THREE.DirectionalLight(0xffffff, 3);
keyLight.position.set(5, 5, 5);
const fillLight = new THREE.DirectionalLight(0xffffff, 1);
fillLight.position.set(-5, 3, 5);
const backLight = new THREE.DirectionalLight(0xffffff, 2);
backLight.position.set(0, 5, -5);
scene.add(keyLight, fillLight, backLight, new THREE.HemisphereLight(0xffffff, 0x444444, 0.5));
```

### Outdoor Daylight

```javascript
const sun = new THREE.DirectionalLight(0xfff4e5, 3);
sun.position.set(50, 100, 50);
sun.castShadow = true;
scene.add(sun, new THREE.HemisphereLight(0x87ceeb, 0x6b5a3a, 1));
// Physical sky: Sky addon (see threejs-addons)
```

### Indoor Night (bulbs)

```javascript
const lamp = new THREE.PointLight(0xffc58f, 30, 0, 2); // warm bulb, candela
lamp.position.set(0, 2.5, 0);
scene.add(lamp, new THREE.AmbientLight(0x202030, 0.3));
renderer.toneMapping = THREE.AgXToneMapping;
```

## Light Animation

```javascript
renderer.setAnimationLoop((t) => {
  timer.update(t);
  const time = timer.getElapsed();
  light.position.set(Math.cos(time) * 5, 3, Math.sin(time) * 5);
  light.intensity = 50 + Math.sin(time * 2) * 20;
  light.color.setHSL((time * 0.1) % 1, 1, 0.5);
  renderer.render(scene, camera);
});
```

## Selective Lighting (Layers)

A light is included when `light.layers` shares a layer with the **camera**; it then lights
every object. three.js has no per-object light inclusion in WebGLRenderer. Options:

```javascript
// Hide/show lights per render pass (e.g. a separate overlay scene rendered on top)
renderer.autoClear = false;
renderer.render(mainScene, camera);
renderer.clearDepth();
renderer.render(overlayScene, camera); // overlayScene has its own lights

// Or exclude objects from light influence by using an unlit material (MeshBasicMaterial)
```

## Performance Tips

1. **Few dynamic lights**: each light adds shader work for every lit pixel. Use an environment map for fill.
2. **Bake** static lighting into `lightMap`s (Blender) for complex static scenes.
3. **Shadow maps**: 1024-2048 is usually enough; tight frustums matter more than resolution.
4. **Avoid point-light shadows** (6 renders each).
5. **Static shadows**: `renderer.shadowMap.autoUpdate = false`.
6. **Only important objects cast** shadows; small props often do not need to.
7. Changing the **number** of lights or shadow-casting lights recompiles shaders. Toggle `intensity` instead of adding/removing lights.

## See Also

- `threejs-materials` - How materials respond to light
- `threejs-textures` - HDR/cube environment maps, PMREM
- `threejs-postprocessing` - Bloom, SSAO/GTAO
- `threejs-addons` - Sky, CSM
