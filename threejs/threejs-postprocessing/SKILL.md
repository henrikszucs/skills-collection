---
name: threejs-postprocessing
description: Three.js post-processing - EffectComposer with RenderPass and OutputPass, bloom and selective bloom, anti-aliasing (FXAA/SMAA/TAA/MSAA), SSAO/GTAO, depth of field, outline, film/glitch/pixel/halftone/LUT effects, custom ShaderPass, resize handling, plus the WebGPU RenderPipeline/TSL equivalent. Use when adding screen effects, glow, color grading, blur, ambient occlusion, outlines, or writing custom full-screen shaders.
---

# Three.js Post-Processing

> three.js **r187**. `EffectComposer` is WebGL-only. With `WebGPURenderer` use `RenderPipeline` + TSL (bottom).

## Quick Start

```javascript
import { EffectComposer } from "three/addons/postprocessing/EffectComposer.js";
import { RenderPass } from "three/addons/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/addons/postprocessing/UnrealBloomPass.js";
import { OutputPass } from "three/addons/postprocessing/OutputPass.js";

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new UnrealBloomPass(new THREE.Vector2(innerWidth, innerHeight), 1.0, 0.4, 0.85)); // strength, radius, threshold
composer.addPass(new OutputPass()); // tone mapping + sRGB

renderer.setAnimationLoop(() => composer.render()); // instead of renderer.render
// resize: renderer.setSize(w, h); composer.setSize(w, h);
```

## The OutputPass Rule

- Passes render into **linear half-float** targets; `renderer.toneMapping` and
  `outputColorSpace` are not applied there.
- `OutputPass` applies both. Without it the image is dark or washed out and tone mapping does
  nothing. Don't use `GammaCorrectionShader` any more.
- Order: scene -> HDR effects (AO, bloom, DOF) -> **OutputPass** -> FXAA/SMAA/LUT (these expect sRGB).
- Canvas `antialias` doesn't apply inside the composer. For MSAA pass a target:
  `new EffectComposer(renderer, new THREE.WebGLRenderTarget(w, h, { type: THREE.HalfFloatType, samples: 4 }))`.
- `pass.enabled = false` skips a pass. Call `composer.setPixelRatio()` if you change the ratio later.

## Effects (`three/addons/postprocessing/`)

| Effect | Pass | Notes |
| --- | --- | --- |
| Bloom | `UnrealBloomPass(res, strength, radius, threshold)` | glow needs emissive or colors > 1 (`toneMapped: false`); half resolution is much cheaper |
| AA | `FXAAPass()`, `SMAAPass()` (no args) | after OutputPass |
| AA (static shots) | `TAARenderPass(scene, camera)`, `SSAARenderPass` | replace RenderPass |
| AO | `GTAOPass(scene, camera, w, h)` (recommended), `SSAOPass` | right after RenderPass; `gtao.updateGtaoMaterial({ radius, thickness, samples })` |
| Depth of field | `BokehPass(scene, camera, { focus, aperture, maxblur })` | update `uniforms.focus.value` |
| Outline | `OutlinePass(res, scene, camera)` | set `selectedObjects`, `edgeStrength`, `visibleEdgeColor` |
| Film grain | `FilmPass(intensity, grayscale)` | old scanline args are gone |
| Pixelation | `RenderPixelatedPass(pixelSize, scene, camera)` | replaces RenderPass |
| LUT grading | `LUTPass({ lut: (await new LUTCubeLoader().loadAsync(url)).texture3D })` | after OutputPass |
| Other | `GlitchPass`, `HalftonePass(params)`, `AfterimagePass`, `SSRPass`, `RenderTransitionPass`, `MaskPass` | |

Shaders for `ShaderPass` live in `three/addons/shaders/`: `VignetteShader`,
`BrightnessContrastShader`, `HueSaturationShader`, `RGBShiftShader`, `SobelOperatorShader`, ...

## Selective Bloom (WebGL)

The official technique uses two composers. Put glowing objects on a layer, render a bloom-only
composer (`renderToScreen = false`) while all other meshes temporarily use a black material
(and `scene.background` is null), then add `bloomComposer.renderTarget2.texture` in a mix
`ShaderPass` of the final composer before `OutputPass`. See the `webgl_postprocessing_unreal_bloom_selective`
example. On WebGPU this is one line with MRT (below).

## Custom ShaderPass

```javascript
import { ShaderPass } from "three/addons/postprocessing/ShaderPass.js";
const pass = new ShaderPass({
  uniforms: { tDiffuse: { value: null }, uTime: { value: 0 } }, // tDiffuse = previous pass output
  vertexShader: `varying vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
  fragmentShader: `uniform sampler2D tDiffuse; uniform float uTime; varying vec2 vUv;
    void main() { vec2 uv = vUv; uv.x += sin(uv.y * 10.0 + uTime) * 0.01; gl_FragColor = texture2D(tDiffuse, uv); }`,
});
composer.addPass(pass);
pass.uniforms.uTime.value = elapsed; // the PASS's (cloned) uniforms, not the definition's
```

Input before `OutputPass` is linear HDR; after it, sRGB. Need depth? Give the composer a render
target with a `depthTexture` and pass it as a uniform (linearize with `<packing>`, see `threejs-shaders`).
`FullScreenQuad` (from `Pass.js`) draws a material over a target for your own pipelines.

## WebGPU: RenderPipeline + TSL

`PostProcessing` was renamed `RenderPipeline` in r183.

```javascript
import * as THREE from "three/webgpu";
import { pass, mrt, output, emissive } from "three/tsl";
import { bloom } from "three/addons/tsl/display/BloomNode.js";
import { fxaa } from "three/addons/tsl/display/FXAANode.js";

const scenePass = pass(scene, camera);
const color = scenePass.getTextureNode("output");      // also "depth"
const glow = bloom(color, 1.0, 0.4, 0.85);
const pipeline = new THREE.RenderPipeline(renderer);
pipeline.outputNode = color.add(glow);
renderer.setAnimationLoop(() => pipeline.render());
glow.strength.value = 2;                                // parameters are uniform nodes

// Selective bloom via MRT: only emissive glows
scenePass.setMRT(mrt({ output, emissive }));
pipeline.outputNode = color.add(bloom(scenePass.getTextureNode("emissive"), 2.5, 0.5));

// AA after tone mapping
pipeline.outputColorTransform = false;
pipeline.outputNode = fxaa(color.add(glow).renderOutput());
```

TSL effects in `three/addons/tsl/display/`: `BloomNode`, `FXAANode`, `SMAANode`, `TRAANode`,
`GTAONode`, `SSAONode`, `SSRNode`, `SSGINode`, `DepthOfFieldNode`, `OutlineNode`, `FilmNode`,
`ChromaticAberrationNode`, `Lut3DNode`, `PixelationPassNode` (`pixelSize` is a plain number
since r187), `GaussianBlurNode`, `AfterImageNode`, `GodraysNode`, `LensflareNode`, `MotionBlur`,
`TransitionNode`, lens dirt.

## Performance

Each pass is at least one full-screen draw: keep the stack short. Prefer FXAA/SMAA over SSAA,
drop AO/SSR/DOF on low-end devices, and lower the pixel ratio on mobile.

See also: `threejs-shaders`, `threejs-webgpu-tsl`, `threejs-textures` (render targets).
