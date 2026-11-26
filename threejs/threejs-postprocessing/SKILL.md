---
name: threejs-postprocessing
description: Three.js post-processing - EffectComposer with RenderPass and OutputPass, bloom and selective bloom, anti-aliasing (FXAA/SMAA/TAA/MSAA), SSAO/GTAO, depth of field, outline, film/glitch/pixel/halftone/LUT effects, custom ShaderPass, resize handling, plus the WebGPU RenderPipeline/TSL equivalent. Use when adding screen effects, glow, color grading, blur, ambient occlusion, outlines, or writing custom full-screen shaders.
---

# Three.js Post-Processing

> **Version:** written for three.js **r186**. `EffectComposer` works with `WebGLRenderer` only.
> For `WebGPURenderer` use `RenderPipeline` + TSL (bottom of this file and `threejs-webgpu-tsl`).

## Quick Start

```javascript
import * as THREE from "three";
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

const bloomPass = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  1.0,  // strength
  0.4,  // radius
  0.85, // threshold (luminance above which pixels glow)
);
composer.addPass(bloomPass);

composer.addPass(new OutputPass()); // ALWAYS last: tone mapping + sRGB conversion

renderer.setAnimationLoop(() => {
  composer.render(); // instead of renderer.render()
});

window.addEventListener("resize", () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  composer.setSize(window.innerWidth, window.innerHeight); // resizes every pass
});
```

## How the Composer Works (and the OutputPass rule)

- Passes render into internal **linear, half-float** render targets. `renderer.toneMapping` and
  `renderer.outputColorSpace` are **not** applied while rendering into them.
- `OutputPass` applies tone mapping (from `renderer.toneMapping`/`toneMappingExposure`) and the
  sRGB conversion. Without it the image looks dark/washed out and tone mapping does nothing.
- Put `OutputPass` after HDR effects (bloom, DOF) and **before** FXAA/SMAA, which expect sRGB input.
- Do not use `GammaCorrectionShader` for this anymore; it is an outdated workaround that ignores tone mapping.
- The last enabled pass renders to the screen automatically.
- `composer.setPixelRatio(renderer.getPixelRatio())` happens automatically on creation; call it again if you change the pixel ratio later.

```javascript
// MSAA inside the composer (canvas antialias does not apply to render targets)
const rt = new THREE.WebGLRenderTarget(window.innerWidth, window.innerHeight, {
  type: THREE.HalfFloatType,
  samples: 4,
});
const composer = new EffectComposer(renderer, rt);
```

## Common Effects

### Bloom

```javascript
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";

const bloomPass = new UnrealBloomPass(new THREE.Vector2(w, h), 1.5, 0.4, 0.85);
bloomPass.strength = 2.0;
bloomPass.radius = 0.5;
bloomPass.threshold = 0.0; // 0 = everything blooms a bit
```

Make objects glow: give them bright emissive (`emissive: 0xffffff, emissiveIntensity: 3`) or
`MeshBasicMaterial` colors above 1 (`color: new THREE.Color(4, 1, 0)`, with `toneMapped: false`).

### Selective Bloom (official two-composer technique)

```javascript
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";

const BLOOM_LAYER = 1;
const bloomLayer = new THREE.Layers();
bloomLayer.set(BLOOM_LAYER);
glowingMesh.layers.enable(BLOOM_LAYER); // mark objects that should glow

const renderScene = new RenderPass(scene, camera);
const bloomPass = new UnrealBloomPass(new THREE.Vector2(w, h), 1.5, 0.4, 0);

// 1) Bloom-only composer: renders offscreen
const bloomComposer = new EffectComposer(renderer);
bloomComposer.renderToScreen = false;
bloomComposer.addPass(renderScene);
bloomComposer.addPass(bloomPass);

// 2) Final composer: full scene + bloom texture on top
const mixPass = new ShaderPass(
  new THREE.ShaderMaterial({
    uniforms: {
      baseTexture: { value: null },
      bloomTexture: { value: bloomComposer.renderTarget2.texture },
    },
    vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
    fragmentShader: `uniform sampler2D baseTexture; uniform sampler2D bloomTexture; varying vec2 vUv;
      void main() { gl_FragColor = texture2D(baseTexture, vUv) + texture2D(bloomTexture, vUv); }`,
  }),
  "baseTexture",
);
mixPass.needsSwap = true;

const finalComposer = new EffectComposer(renderer);
finalComposer.addPass(renderScene);
finalComposer.addPass(mixPass);
finalComposer.addPass(new OutputPass());

// Darken non-glowing objects while rendering the bloom composer
const darkMaterial = new THREE.MeshBasicMaterial({ color: 0x000000 });
const savedMaterials = {};
function darkenNonBloomed(obj) {
  if (obj.isMesh && !bloomLayer.test(obj.layers)) {
    savedMaterials[obj.uuid] = obj.material;
    obj.material = darkMaterial;
  }
}
function restoreMaterial(obj) {
  if (savedMaterials[obj.uuid]) {
    obj.material = savedMaterials[obj.uuid];
    delete savedMaterials[obj.uuid];
  }
}

function render() {
  scene.traverse(darkenNonBloomed);
  bloomComposer.render();
  scene.traverse(restoreMaterial);
  finalComposer.render();
}
// on resize: bloomComposer.setSize(w, h); finalComposer.setSize(w, h);
```

Set `scene.background` to null (or swap it to black) during the bloom render if it should not glow.
The WebGPU version is far simpler (MRT, see the bottom of this file).

### Anti-Aliasing

```javascript
// FXAA - cheapest. Place after OutputPass. Resolution is updated by composer.setSize().
import { FXAAPass } from "three/addons/postprocessing/FXAAPass.js";
composer.addPass(new OutputPass());
composer.addPass(new FXAAPass());

// SMAA - better edges, slightly more expensive. No constructor arguments (r186).
import { SMAAPass } from "three/addons/postprocessing/SMAAPass.js";
composer.addPass(new SMAAPass());

// TAA - accumulates jittered frames, best for static camera shots
import { TAARenderPass } from "three/addons/postprocessing/TAARenderPass.js";
const taa = new TAARenderPass(scene, camera);
taa.sampleLevel = 2;
composer.addPass(taa); // replaces RenderPass

// SSAA - supersampling (expensive, screenshots)
import { SSAARenderPass } from "three/addons/postprocessing/SSAARenderPass.js";
```

Or MSAA via `samples: 4` on the composer render target (see above).

### Ambient Occlusion

```javascript
// GTAO - recommended (better quality than SSAO)
import { GTAOPass } from "three/addons/postprocessing/GTAOPass.js";
const gtao = new GTAOPass(scene, camera, w, h);
gtao.updateGtaoMaterial({ radius: 0.25, distanceExponent: 1, thickness: 1, scale: 1, samples: 16 });
gtao.blendIntensity = 1;
gtao.output = GTAOPass.OUTPUT.Default; // Off, Default, Diffuse, Depth, Normal, AO, Denoise
composer.addPass(gtao); // after RenderPass

// SSAO - older
import { SSAOPass } from "three/addons/postprocessing/SSAOPass.js";
const ssao = new SSAOPass(scene, camera, w, h);
ssao.kernelRadius = 16;
ssao.minDistance = 0.005;
ssao.maxDistance = 0.1;
ssao.output = SSAOPass.OUTPUT.Default; // Default, SSAO, Blur, Depth, Normal
```

### Depth of Field

```javascript
import { BokehPass } from "three/addons/postprocessing/BokehPass.js";
const bokeh = new BokehPass(scene, camera, { focus: 10, aperture: 0.025, maxblur: 0.01 });
composer.addPass(bokeh);
bokeh.uniforms.focus.value = camera.position.distanceTo(target.position);
```

### Outline

```javascript
import { OutlinePass } from "three/addons/postprocessing/OutlinePass.js";
const outline = new OutlinePass(new THREE.Vector2(w, h), scene, camera);
outline.edgeStrength = 3;
outline.edgeGlow = 0;
outline.edgeThickness = 1;
outline.pulsePeriod = 0;
outline.visibleEdgeColor.set(0xffffff);
outline.hiddenEdgeColor.set(0x190a05);
outline.selectedObjects = [selectedMesh]; // update on selection
composer.addPass(outline);
```

### Film Grain

```javascript
import { FilmPass } from "three/addons/postprocessing/FilmPass.js";
composer.addPass(new FilmPass(0.35, false)); // (intensity, grayscale). Scanline args were removed.
```

### Glitch

```javascript
import { GlitchPass } from "three/addons/postprocessing/GlitchPass.js";
const glitch = new GlitchPass();
glitch.goWild = false; // false = occasional glitches, true = constant
composer.addPass(glitch);
```

### Pixelation

```javascript
import { RenderPixelatedPass } from "three/addons/postprocessing/RenderPixelatedPass.js";
composer.addPass(new RenderPixelatedPass(6, scene, camera)); // replaces RenderPass; 6 = pixel size
```

### Halftone

```javascript
import { HalftonePass } from "three/addons/postprocessing/HalftonePass.js";
composer.addPass(new HalftonePass({
  shape: 1,          // 1 dot, 2 ellipse, 3 line, 4 square
  radius: 4,
  rotateR: Math.PI / 12,
  rotateB: (Math.PI / 12) * 2,
  rotateG: (Math.PI / 12) * 3,
  scatter: 0,
  blending: 1,
  blendingMode: 1,
  greyscale: false,
}));
```

### Color Grading

```javascript
// 3D LUT (.cube from DaVinci/Photoshop)
import { LUTPass } from "three/addons/postprocessing/LUTPass.js";
import { LUTCubeLoader } from "three/addons/loaders/LUTCubeLoader.js";
const lut = await new LUTCubeLoader().loadAsync("grade.cube");
composer.addPass(new OutputPass());
composer.addPass(new LUTPass({ lut: lut.texture3D, intensity: 1 })); // after OutputPass (LUTs expect sRGB)

// Simple shader-based adjustments
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";
import { BrightnessContrastShader } from "three/addons/shaders/BrightnessContrastShader.js";
import { HueSaturationShader } from "three/addons/shaders/HueSaturationShader.js";
import { VignetteShader } from "three/addons/shaders/VignetteShader.js";

const vignette = new ShaderPass(VignetteShader);
vignette.uniforms.offset.value = 1.0;
vignette.uniforms.darkness.value = 1.2;
composer.addPass(vignette);
```

Other passes: `AfterimagePass` (motion trails), `SSRPass` (screen-space reflections), `SAOPass`,
`DotScreenPass`, `BloomPass`, `TexturePass`, `ClearPass`, `MaskPass`/`ClearMaskPass` (stencil masking),
`RenderTransitionPass` (crossfade between scenes), `SavePass`. Shaders in `three/addons/shaders/`:
`RGBShiftShader`, `SepiaShader`, `SobelOperatorShader`, `ColorCorrectionShader`, `ExposureShader`, ...

## Custom ShaderPass

```javascript
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";

const WaveShader = {
  uniforms: {
    tDiffuse: { value: null }, // filled with the previous pass's output
    uTime: { value: 0 },
    uAmount: { value: 0.01 },
  },
  vertexShader: /* glsl */ `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    uniform sampler2D tDiffuse;
    uniform float uTime;
    uniform float uAmount;
    varying vec2 vUv;
    void main() {
      vec2 uv = vUv;
      uv.x += sin(uv.y * 10.0 + uTime) * uAmount;
      gl_FragColor = texture2D(tDiffuse, uv);
    }
  `,
};

const wavePass = new ShaderPass(WaveShader); // clones the uniforms
composer.addPass(wavePass);
wavePass.uniforms.uTime.value = timer.getElapsed(); // update the PASS uniforms, not WaveShader's
```

Input before `OutputPass` is linear HDR; after it, sRGB. Put color-space-sensitive effects on the right side.

### Chromatic Aberration

```glsl
uniform sampler2D tDiffuse;
uniform float uAmount; // e.g. 0.005
varying vec2 vUv;
void main() {
  vec2 dir = vUv - 0.5;
  float d = length(dir);
  float r = texture2D(tDiffuse, vUv - dir * uAmount * d).r;
  float g = texture2D(tDiffuse, vUv).g;
  float b = texture2D(tDiffuse, vUv + dir * uAmount * d).b;
  gl_FragColor = vec4(r, g, b, 1.0);
}
```

### Depth in a Custom Pass

```javascript
const depthTarget = new THREE.WebGLRenderTarget(w, h, { type: THREE.HalfFloatType });
depthTarget.depthTexture = new THREE.DepthTexture(w, h);
const composer = new EffectComposer(renderer, depthTarget);
// pass depthTarget.depthTexture as a uniform; linearize with <packing> (see threejs-shaders)
```

## Typical Stack

```javascript
const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera)); // 1. scene (HDR, linear)
composer.addPass(new GTAOPass(scene, camera, w, h)); // 2. AO
composer.addPass(new UnrealBloomPass(new THREE.Vector2(w, h), 0.5, 0.4, 0.85)); // 3. bloom
composer.addPass(new OutputPass()); // 4. tone map + sRGB
composer.addPass(new FXAAPass());   // 5. AA on the final image
```

## Toggling and Performance

```javascript
bloomPass.enabled = false; // skip a pass without removing it

// Half-resolution bloom is much cheaper and looks similar
const cheapBloom = new UnrealBloomPass(new THREE.Vector2(w / 2, h / 2), 1, 0.4, 0.85);
```

1. Each pass is at least one full-screen draw: keep the stack short.
2. Prefer FXAA/SMAA over SSAA; MSAA (`samples`) costs memory bandwidth.
3. Lower `renderer.setPixelRatio` on mobile; post-processing scales with pixel count.
4. Disable expensive passes (GTAO, SSR, DOF) on low-end devices.

## Rendering Without a Composer (render to texture)

```javascript
const rt = new THREE.WebGLRenderTarget(512, 512);
renderer.setRenderTarget(rt);
renderer.render(scene, camera);
renderer.setRenderTarget(null);
screenMaterial.map = rt.texture;
```

`FullScreenQuad` (`three/addons/postprocessing/Pass.js`) draws a material over the whole target for your own pipelines.

## WebGPU: RenderPipeline + TSL

With `WebGPURenderer`, effects are TSL nodes. `PostProcessing` was renamed `RenderPipeline` in r183.

```javascript
import * as THREE from "three/webgpu";
import { pass, mrt, output, emissive } from "three/tsl";
import { bloom } from "three/addons/tsl/display/BloomNode.js";
import { fxaa } from "three/addons/tsl/display/FXAANode.js";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init();

const scenePass = pass(scene, camera);
const scenePassColor = scenePass.getTextureNode("output");
const bloomPass = bloom(scenePassColor, 1.0, 0.4, 0.85); // node, strength, radius, threshold

const renderPipeline = new THREE.RenderPipeline(renderer);
renderPipeline.outputNode = scenePassColor.add(bloomPass);

renderer.setAnimationLoop(() => renderPipeline.render());

bloomPass.strength.value = 2; // parameters are uniform nodes
```

Selective bloom with MRT (per-material bloom intensity):

```javascript
const scenePass = pass(scene, camera);
scenePass.setMRT(mrt({ output, emissive })); // write emissive into its own texture
const outputNode = scenePass.getTextureNode("output");
const emissiveNode = scenePass.getTextureNode("emissive");
renderPipeline.outputNode = outputNode.add(bloom(emissiveNode, 2.5, 0.5));
```

Anti-aliasing after tone mapping (FXAA/SMAA expect sRGB):

```javascript
renderPipeline.outputColorTransform = false;              // we place the color transform ourselves
const graded = scenePassColor.add(bloomPass).renderOutput(); // tone mapping + sRGB here
renderPipeline.outputNode = fxaa(graded);
```

TSL effect nodes live in `three/addons/tsl/display/`: `BloomNode`, `FXAANode`, `SMAANode`,
`TRAANode`, `GTAONode`, `SSAONode`, `SSRNode`, `SSGINode`, `DepthOfFieldNode`, `OutlineNode`,
`FilmNode`, `ChromaticAberrationNode`, `Lut3DNode`, `PixelationPassNode`, `GaussianBlurNode`,
`AfterImageNode`, `GodraysNode`, `LensflareNode`, `MotionBlur`, `TransitionNode`, ... See `threejs-webgpu-tsl`.

## See Also

- `threejs-shaders` - GLSL for custom passes
- `threejs-webgpu-tsl` - TSL and RenderPipeline in depth
- `threejs-textures` - Render targets
- `threejs-fundamentals` - Renderer setup, tone mapping
