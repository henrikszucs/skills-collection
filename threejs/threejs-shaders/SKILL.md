---
name: threejs-shaders
description: Three.js GLSL shaders for WebGLRenderer - ShaderMaterial vs RawShaderMaterial, built-in uniforms/attributes, uniforms and varyings, GLSL 3, color space/tone mapping in custom shaders, instancing in shaders, extending built-in materials with onBeforeCompile and shader chunks, common effects (fresnel, dissolve, noise, displacement) and debugging. Use when writing custom GLSL, modifying vertices, writing fragment shaders, or patching MeshStandardMaterial. For WebGPURenderer use threejs-webgpu-tsl instead.
---

# Three.js Shaders (GLSL)

> **Version:** written for three.js **r186**. GLSL shaders work with `WebGLRenderer` only.
> `WebGPURenderer` does not support `ShaderMaterial`, `RawShaderMaterial` or `onBeforeCompile`:
> use node materials + TSL (`threejs-webgpu-tsl`).

## Quick Start

```javascript
import * as THREE from "three";

const material = new THREE.ShaderMaterial({
  uniforms: {
    uTime: { value: 0 },
    uColor: { value: new THREE.Color(0xff0000) },
  },
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
      gl_FragColor = vec4(uColor * (0.5 + 0.5 * sin(uTime + vUv.x * 6.2831)), 1.0);
      #include <colorspace_fragment>
    }
  `,
});

renderer.setAnimationLoop((t) => {
  timer.update(t);
  material.uniforms.uTime.value = timer.getElapsed();
  renderer.render(scene, camera);
});
```

## ShaderMaterial vs RawShaderMaterial

**ShaderMaterial** prepends a prefix for you: precision, `#define`s, and these declarations:

```glsl
// vertex shader
uniform mat4 modelMatrix;        // object -> world
uniform mat4 modelViewMatrix;    // object -> camera
uniform mat4 projectionMatrix;   // camera -> clip
uniform mat4 viewMatrix;         // world -> camera
uniform mat3 normalMatrix;       // object normals -> camera space
uniform vec3 cameraPosition;     // world space
uniform bool isOrthographic;
attribute vec3 position;
attribute vec3 normal;
attribute vec2 uv;
// + instanceMatrix / instanceColor (InstancedMesh), uv1-3, tangent, color when used

// fragment shader
uniform mat4 viewMatrix;
uniform vec3 cameraPosition;
uniform bool isOrthographic;
```

**RawShaderMaterial** prepends nothing. Declare precision, attributes and uniforms yourself.
The renderer still uploads the built-in matrices (`modelViewMatrix`, `projectionMatrix`, ...)
every draw, so declare them; do not put them in `uniforms`.

```javascript
const raw = new THREE.RawShaderMaterial({
  glslVersion: THREE.GLSL3,
  vertexShader: /* glsl */ `
    precision highp float;
    in vec3 position;
    uniform mat4 modelViewMatrix;
    uniform mat4 projectionMatrix;
    void main() {
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: /* glsl */ `
    precision highp float;
    out vec4 fragColor;
    void main() { fragColor = vec4(1.0, 0.0, 0.0, 1.0); }
  `,
});
```

## GLSL Version

WebGL 2 is always used (WebGL 1 support was removed in r163).

- **Default (GLSL1-style syntax):** write `attribute`, `varying`, `texture2D`, `gl_FragColor`.
  three.js maps them to GLSL ES 3.0 with `#define`s, so they keep working.
- **`glslVersion: THREE.GLSL3`:** write modern syntax: `in`/`out`, `texture()`, and declare your
  own fragment output (`out vec4 fragColor;`, or `layout(location = 1) out vec4 gNormal;` for MRT).

```javascript
new THREE.ShaderMaterial({
  glslVersion: THREE.GLSL3,
  vertexShader: `out vec2 vUv; void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }`,
  fragmentShader: `in vec2 vUv; uniform sampler2D map; out vec4 fragColor; void main() { fragColor = texture(map, vUv); }`,
  uniforms: { map: { value: texture } },
});
```

## Color Space and Tone Mapping (common surprise)

Uniform colors and textures arrive in **linear** space. The canvas is sRGB. A custom shader
outputs linear values unless you convert them. Add these at the end of `main()` in the fragment shader:

```glsl
gl_FragColor = vec4(color, alpha);
#include <tonemapping_fragment>  // applies renderer.toneMapping when material.toneMapped is true
#include <colorspace_fragment>   // linear -> renderer.outputColorSpace (sRGB)
```

Without them, custom materials look darker/different from built-in ones next to them. (Inside an
`EffectComposer` pipeline, `OutputPass` does the conversion instead; see `threejs-postprocessing`.)

## Uniforms

```javascript
const material = new THREE.ShaderMaterial({
  uniforms: {
    uFloat: { value: 1.5 },
    uInt: { value: 1 },                         // declare as int in GLSL
    uBool: { value: true },
    uVec2: { value: new THREE.Vector2(1, 2) },
    uVec3: { value: new THREE.Vector3(1, 2, 3) },
    uVec4: { value: new THREE.Vector4(1, 2, 3, 4) },
    uColor: { value: new THREE.Color(0xff0000) }, // vec3, linear
    uMat3: { value: new THREE.Matrix3() },
    uMat4: { value: new THREE.Matrix4() },
    uTexture: { value: texture },               // sampler2D
    uCube: { value: cubeTexture },              // samplerCube
    uFloats: { value: [1.0, 2.0, 3.0] },        // float uFloats[3]
    uVecs: { value: [new THREE.Vector3(), new THREE.Vector3()] }, // vec3 uVecs[2]
  },
  defines: { NUM_LIGHTS: 4 }, // compile-time constants; changing them needs material.needsUpdate = true
});
```

```glsl
uniform float uFloat;
uniform int uInt;
uniform bool uBool;
uniform vec2 uVec2;
uniform vec3 uVec3;
uniform vec4 uVec4;
uniform vec3 uColor;
uniform mat3 uMat3;
uniform mat4 uMat4;
uniform sampler2D uTexture;
uniform samplerCube uCube;
uniform float uFloats[3];
uniform vec3 uVecs[2];
```

```javascript
// Update by mutating .value (no recompile)
material.uniforms.uTime.value = elapsed;
material.uniforms.uVec3.value.set(x, y, z);
material.uniforms.uColor.value.setHSL(h, 1, 0.5);
material.uniforms.uMat4.value.copy(mesh.matrixWorld);

// Share one uniform object between materials
const shared = { uTime: { value: 0 } };
const matA = new THREE.ShaderMaterial({ uniforms: { ...shared, uColor: { value: new THREE.Color("red") } } });
```

Using lights/fog in a ShaderMaterial: merge `THREE.UniformsLib.lights` / `THREE.UniformsLib.fog`
with `THREE.UniformsUtils.merge([...])`, set `lights: true` / `fog: true`, and include the matching
chunks. Usually `onBeforeCompile` on a built-in material is much easier (below).

## Varyings

```glsl
// vertex
varying vec2 vUv;
varying vec3 vNormal;        // view space
varying vec3 vWorldPosition;
void main() {
  vUv = uv;
  vNormal = normalize(normalMatrix * normal);
  vec4 worldPos = modelMatrix * vec4(position, 1.0);
  vWorldPosition = worldPos.xyz;
  gl_Position = projectionMatrix * viewMatrix * worldPos;
}
```

## Common Shader Patterns

### Vertex Displacement (waves)

```glsl
uniform float uTime;
uniform float uAmplitude;
varying float vElevation;
void main() {
  vec3 pos = position;
  float e = sin(pos.x * 4.0 + uTime) * cos(pos.y * 4.0 + uTime) * uAmplitude;
  pos.z += e;
  vElevation = e;
  gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
}
```

Displaced vertices are outside the original bounding sphere: set `mesh.frustumCulled = false`
or enlarge `geometry.boundingSphere` so the mesh is not culled. Normals are not updated
automatically; recompute them in the shader (finite differences) if lighting matters.

### Fresnel / Rim

```glsl
// fragment (with vNormal, vWorldPosition varyings from above; vNormal in view space)
varying vec3 vNormal;
varying vec3 vWorldPosition;
void main() {
  vec3 viewDir = normalize(cameraPosition - vWorldPosition);
  vec3 worldNormal = transformNormalByInverseViewMatrix(vNormal, viewMatrix); // from <common> (inverseTransformDirection is deprecated since r185)
  float fresnel = pow(1.0 - clamp(dot(viewDir, worldNormal), 0.0, 1.0), 3.0);
  gl_FragColor = vec4(mix(vec3(0.0, 0.0, 0.3), vec3(0.5, 0.8, 1.0), fresnel), 1.0);
  #include <colorspace_fragment>
}
```

Add `#include <common>` at the top of the fragment shader to get helpers like
`transformNormalByInverseViewMatrix`, `transformDirection`, `saturate`, `PI`, `pow2`. Or pass a world-space normal
varying (`normalize(mat3(modelMatrix) * normal)`, fine for uniform scale).

### Noise

```glsl
float hash(vec2 p) { return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453); }

float valueNoise(vec2 p) {
  vec2 i = floor(p), f = fract(p);
  float a = hash(i), b = hash(i + vec2(1, 0)), c = hash(i + vec2(0, 1)), d = hash(i + vec2(1, 1));
  vec2 u = f * f * (3.0 - 2.0 * f);
  return mix(a, b, u.x) + (c - a) * u.y * (1.0 - u.x) + (d - b) * u.x * u.y;
}
```

For high-quality noise, sample a noise texture or paste a simplex-noise GLSL function.
`SimplexNoise` / `ImprovedNoise` addons give the same noise on the CPU.

### Gradients

```glsl
vec3 c = mix(colorA, colorB, vUv.y);                                   // linear
vec3 r = mix(centerColor, edgeColor, distance(vUv, vec2(0.5)) * 2.0);  // radial
vec3 s = mix(colorA, colorB, smoothstep(0.3, 0.7, vUv.y));             // eased
```

### Dissolve

```glsl
uniform float uProgress;
uniform sampler2D uNoise;
varying vec2 vUv;
void main() {
  float n = texture2D(uNoise, vUv).r;
  if (n < uProgress) discard;
  float edge = smoothstep(uProgress, uProgress + 0.05, n);
  gl_FragColor = vec4(mix(vec3(1.0, 0.5, 0.0) * 4.0, vec3(0.6), edge), 1.0);
  #include <colorspace_fragment>
}
```

For shadows that match a dissolve/displacement, assign a `customDepthMaterial`
(and `customDistanceMaterial` for point lights) with the same discard/displacement.

## Instancing in Custom Shaders

With `InstancedMesh`, `instanceMatrix` (and `instanceColor`) are declared for you but you must apply them:

```glsl
void main() {
  vec4 worldPos = modelMatrix * instanceMatrix * vec4(position, 1.0);
  gl_Position = projectionMatrix * viewMatrix * worldPos;
}
```

Custom per-instance data: `geometry.setAttribute('aOffset', new THREE.InstancedBufferAttribute(data, 1))` and `attribute float aOffset;` in the shader.

## Extending Built-in Materials (onBeforeCompile)

Keep all the PBR lighting, shadows and fog of `MeshStandardMaterial` and inject your own code:

```javascript
const uniforms = { uTime: { value: 0 } }; // keep a reference you own

const material = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
material.onBeforeCompile = (shader) => {
  shader.uniforms.uTime = uniforms.uTime; // same object: updates flow through

  shader.vertexShader = shader.vertexShader
    .replace("#include <common>", "#include <common>\nuniform float uTime;")
    .replace(
      "#include <begin_vertex>",
      `#include <begin_vertex>
       transformed.y += sin(position.x * 10.0 + uTime) * 0.1;`,
    );

  // Uniforms must be declared in EVERY stage that uses them
  shader.fragmentShader = shader.fragmentShader
    .replace("#include <common>", "#include <common>\nuniform float uTime;")
    .replace(
      "#include <color_fragment>",
      `#include <color_fragment>
       diffuseColor.rgb *= 0.8 + 0.2 * sin(uTime);`,
    );
};
// If several materials use different onBeforeCompile code, give each a unique cache key:
material.customProgramCacheKey = () => "wobble-v1";

// in the loop
uniforms.uTime.value = timer.getElapsed();
```

`.replace()` silently does nothing if the chunk name is wrong. Verify with
`console.log(shader.fragmentShader)` once, or check that the result string changed.

### Injection Points (r186 chunk names)

```glsl
// Vertex (MeshStandardMaterial order)
#include <common>             // top-level declarations: add uniforms/functions after this
#include <uv_vertex>
#include <beginnormal_vertex> // vec3 objectNormal = normal;
#include <defaultnormal_vertex>
#include <begin_vertex>       // vec3 transformed = position;  <- displace here
#include <project_vertex>     // mvPosition and gl_Position are computed
#include <worldpos_vertex>

// Fragment
#include <common>
#include <map_fragment>          // diffuseColor *= texture
#include <color_fragment>        // vertex colors applied to diffuseColor
#include <roughnessmap_fragment> // float roughnessFactor
#include <metalnessmap_fragment> // float metalnessFactor
#include <normal_fragment_maps>  // vec3 normal (view space)
#include <emissivemap_fragment>  // totalEmissiveRadiance
#include <lights_fragment_end>   // lighting accumulated in reflectedLight
#include <opaque_fragment>       // gl_FragColor = vec4(outgoingLight, diffuseColor.a)  (was output_fragment before r154)
#include <tonemapping_fragment>
#include <colorspace_fragment>
#include <fog_fragment>
#include <premultiplied_alpha_fragment>
#include <dithering_fragment>
```

Full sources: `node_modules/three/src/renderers/shaders/ShaderChunk/` and `ShaderLib/`.

## Using Shader Chunks in ShaderMaterial

```javascript
const fragmentShader = /* glsl */ `
  #include <common>
  #include <packing>
  uniform sampler2D tDepth;
  uniform float cameraNear;
  uniform float cameraFar;
  varying vec2 vUv;
  void main() {
    float depth = texture2D(tDepth, vUv).x;
    float viewZ = perspectiveDepthToViewZ(depth, cameraNear, cameraFar);
    float linear = viewZToOrthographicDepth(viewZ, cameraNear, cameraFar); // 0..1
    gl_FragColor = vec4(vec3(1.0 - linear), 1.0);
  }
`;
```

`#include <name>` works inside ShaderMaterial shaders; three resolves it from `THREE.ShaderChunk`.

## External Shader Files

```javascript
// Vite: append ?raw
import vertexShader from "./shaders/vertex.glsl?raw";
import fragmentShader from "./shaders/fragment.glsl?raw";
// (or vite-plugin-glsl for #include support)
const material = new THREE.ShaderMaterial({ vertexShader, fragmentShader });
```

## Material Options Relevant to Shaders

```javascript
new THREE.ShaderMaterial({
  uniforms, vertexShader, fragmentShader,
  transparent: true,          // needed for alpha < 1 with blending
  depthWrite: false,
  side: THREE.DoubleSide,
  blending: THREE.AdditiveBlending,
  wireframe: false,
  glslVersion: THREE.GLSL3,   // or null for the default syntax
  defines: { USE_FOO: "" },
  extensions: {
    clipCullDistance: false,  // gl_ClipDistance in vertex shaders
    multiDraw: false,         // gl_DrawID (BatchedMesh/multi-draw)
  },
  // derivatives / fragDepth / drawBuffers / shaderTextureLOD no longer exist:
  // dFdx/fwidth, gl_FragDepth, MRT and textureLod are always available in WebGL 2
});
```

## GLSL Built-in Function Reference

```glsl
abs sign floor ceil fract mod min max clamp mix step smoothstep
sin cos tan asin acos atan(y, x) radians degrees
pow exp log exp2 log2 sqrt inversesqrt
length distance dot cross normalize reflect refract faceforward
dFdx dFdy fwidth                        // screen-space derivatives (fragment only)
texture2D(s, uv) textureCube(s, dir)    // default syntax (GLSL3: texture())
textureLod(s, uv, lod) textureSize(s, lod) texelFetch(s, ivec2, lod) // GLSL ES 3.0
lessThan greaterThan equal any all not
```

## Debugging Shaders

```javascript
// Compile errors are printed to the console with the offending line (checkShaderErrors is true by default)
renderer.debug.checkShaderErrors = true;
renderer.debug.onShaderError = (gl, program, vs, fs) => { /* custom reporting */ };

// Print the final source of a built-in material
material.onBeforeCompile = (shader) => console.log(shader.fragmentShader);
```

Visual debugging: output the value as a color.

```glsl
gl_FragColor = vec4(vUv, 0.0, 1.0);               // UVs
gl_FragColor = vec4(normalize(vNormal) * 0.5 + 0.5, 1.0); // normals
gl_FragColor = vec4(vec3(fract(vWorldPosition)), 1.0);    // positions
```

## Performance Tips

1. **Do work in the vertex shader** and pass results as varyings where possible.
2. **Branching** on uniforms is cheap; divergent per-pixel branches are not. `mix`/`step` help.
3. **Precompute** in JS (matrices, constants) and pass uniforms.
4. **Lookup textures** for expensive functions (noise, gradients).
5. **Overdraw**: large transparent/additive quads are costly on mobile.
6. **Each unique shader** compiles once; avoid many slightly different `defines` combinations.

## See Also

- `threejs-materials` - Built-in materials
- `threejs-postprocessing` - Full-screen shader passes
- `threejs-textures` - Texture sampling, render targets
- `threejs-webgpu-tsl` - The node-based alternative for WebGPURenderer
