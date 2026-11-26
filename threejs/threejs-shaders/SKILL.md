---
name: threejs-shaders
description: Three.js GLSL shaders for WebGLRenderer - ShaderMaterial vs RawShaderMaterial, built-in uniforms/attributes, uniforms and varyings, GLSL 3, color space/tone mapping in custom shaders, instancing in shaders, extending built-in materials with onBeforeCompile and shader chunks, common effects (fresnel, dissolve, noise, displacement) and debugging. Use when writing custom GLSL, modifying vertices, writing fragment shaders, or patching MeshStandardMaterial. For WebGPURenderer use threejs-webgpu-tsl instead.
---

# Three.js Shaders (GLSL)

> three.js **r187**. GLSL works with `WebGLRenderer` only. `WebGPURenderer` does not support
> `ShaderMaterial`, `RawShaderMaterial` or `onBeforeCompile`: use TSL (`threejs-webgpu-tsl`).

## Quick Start

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: { uTime: { value: 0 }, uColor: { value: new THREE.Color(0xff0000) } },
  vertexShader: /* glsl */ `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }`,
  fragmentShader: /* glsl */ `
    uniform vec3 uColor; uniform float uTime; varying vec2 vUv;
    void main() {
      gl_FragColor = vec4(uColor * (0.5 + 0.5 * sin(uTime + vUv.x * 6.2831)), 1.0);
      #include <tonemapping_fragment>
      #include <colorspace_fragment>
    }`,
});
// loop: material.uniforms.uTime.value = timer.getElapsed();  (mutate .value: no recompile)
```

## Rules That Bite

- **ShaderMaterial** prepends precision, defines and built-ins: `modelMatrix`, `modelViewMatrix`,
  `projectionMatrix`, `viewMatrix`, `normalMatrix`, `cameraPosition`, `isOrthographic`, and the
  attributes `position`, `normal`, `uv` (+ `instanceMatrix`, `uv1-3`, `tangent`, `color` when used).
  **RawShaderMaterial** prepends nothing: declare everything (the renderer still fills the
  built-in matrices; don't put them in `uniforms`).
- **Color:** uniforms and textures arrive **linear**. End the fragment shader with
  `#include <tonemapping_fragment>` and `#include <colorspace_fragment>`, or custom materials look
  darker than built-ins. (In an EffectComposer, `OutputPass` does this instead.)
- **GLSL version:** WebGL 2 only (r163+). The default syntax (`attribute`, `varying`,
  `texture2D`, `gl_FragColor`) still works. With `glslVersion: THREE.GLSL3` write `in`/`out`,
  `texture()`, and declare `out vec4 fragColor;` (`layout(location = 1) out ...` for MRT).
- **Uniform types:** number -> `float` (or `int`/`bool`), `Vector2-4` -> `vec2-4`,
  `Color` -> `vec3`, `Matrix3/4` -> `mat3/4`, `Texture` -> `sampler2D`, `CubeTexture` -> `samplerCube`,
  arrays -> `float u[3]`. Share uniforms between materials by sharing the `{ value }` object.
- **`defines`** are compile-time; changing them needs `material.needsUpdate = true`.
- **Extensions:** only `clipCullDistance` and `multiDraw` remain; derivatives, `gl_FragDepth`,
  MRT and `textureLod` are always available.
- **Displaced vertices** leave the bounding sphere: `mesh.frustumCulled = false` or enlarge it.
  Normals are not recomputed; derive them in the shader if lighting matters.
- **Shadows** of dissolved/displaced meshes need a matching `customDepthMaterial` (and
  `customDistanceMaterial` for point lights).
- **Instancing:** `instanceMatrix` is declared but you apply it:
  `modelMatrix * instanceMatrix * vec4(position, 1.0)`. Custom data:
  `InstancedBufferAttribute` + `attribute float aOffset;`.

## Snippets

```glsl
// Fresnel rim (fragment). Add #include <common> for transformNormalByInverseViewMatrix, saturate, PI, pow2.
vec3 viewDir = normalize(cameraPosition - vWorldPosition);
vec3 n = transformNormalByInverseViewMatrix(vNormal, viewMatrix); // vNormal in view space; inverseTransformDirection is deprecated (r185)
float fresnel = pow(1.0 - saturate(dot(viewDir, n)), 3.0);

// Dissolve
float noise = texture2D(uNoise, vUv).r;
if (noise < uProgress) discard;
vec3 col = mix(vec3(1.0, 0.5, 0.0) * 4.0, baseColor, smoothstep(uProgress, uProgress + 0.05, noise));

// Cheap hash / value noise
float hash(vec2 p) { return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453); }

// Linear depth from a depth texture (#include <packing>)
float viewZ = perspectiveDepthToViewZ(texture2D(tDepth, vUv).x, cameraNear, cameraFar);
float linearDepth = viewZToOrthographicDepth(viewZ, cameraNear, cameraFar); // 0..1
```

## Extending Built-in Materials (onBeforeCompile)

Keeps PBR lighting, shadows and fog, and injects code:

```javascript
const uniforms = { uTime: { value: 0 } };
const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
material.onBeforeCompile = (shader) => {
  shader.uniforms.uTime = uniforms.uTime; // same object, so updates flow through
  shader.vertexShader = shader.vertexShader
    .replace("#include <common>", "#include <common>\nuniform float uTime;")
    .replace("#include <begin_vertex>", "#include <begin_vertex>\ntransformed.y += sin(position.x * 10.0 + uTime) * 0.1;");
};
material.customProgramCacheKey = () => "wobble-v1"; // unique per distinct onBeforeCompile variant
```

`.replace()` silently does nothing on a wrong chunk name: `console.log(shader.vertexShader)` once.

Injection points (r187):
- Vertex: `common` (declarations), `uv_vertex`, `beginnormal_vertex` (`objectNormal`),
  `begin_vertex` (`vec3 transformed`: displace here), `project_vertex` (`mvPosition`, `gl_Position`), `worldpos_vertex`.
- Fragment: `common`, `map_fragment` / `color_fragment` (`diffuseColor`), `roughnessmap_fragment`
  (`roughnessFactor`), `metalnessmap_fragment`, `normal_fragment_maps` (`normal`),
  `emissivemap_fragment` (`totalEmissiveRadiance`), `lights_fragment_end` (`reflectedLight`),
  `opaque_fragment` (was `output_fragment` before r154), `tonemapping_fragment`,
  `colorspace_fragment`, `fog_fragment`.

Sources: `node_modules/three/src/renderers/shaders/ShaderChunk/` and `ShaderLib/`. `#include <name>`
also works inside your own ShaderMaterial. Lights in a raw ShaderMaterial need
`UniformsUtils.merge([UniformsLib.lights, ...])` + `lights: true`; onBeforeCompile is usually easier.

## Tooling and Debugging

- Vite: `import frag from "./frag.glsl?raw"` (or `vite-plugin-glsl` for includes).
- Compile errors print with line numbers (`renderer.debug.checkShaderErrors`, default true;
  `renderer.debug.onShaderError` for custom reporting).
- Visual debugging: output the value as color, e.g. `gl_FragColor = vec4(normalize(vNormal) * 0.5 + 0.5, 1.0);`.

## Performance

Move work to the vertex shader and pass varyings; precompute in JS; prefer `mix`/`step` over
divergent branches; use lookup textures for expensive functions; watch overdraw from large
transparent quads; avoid many `defines` permutations (each compiles separately).

See also: `threejs-materials`, `threejs-postprocessing`, `threejs-webgpu-tsl`.
