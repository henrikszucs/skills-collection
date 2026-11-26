---
name: threejs-geometry
description: Three.js geometry - built-in shapes, BufferGeometry and BufferAttribute, custom meshes, shapes/extrusion/text, points and lines, InstancedMesh, morph targets, geometry utilities. Use when creating 3D shapes, editing vertices, building custom meshes, extruding 2D shapes, drawing points/lines, or rendering many copies with instancing.
---

# Three.js Geometry

> **Version:** written for three.js **r186**. See `threejs-migration` for renamed/removed APIs.

## Quick Start

```javascript
import * as THREE from "three";

const box = new THREE.BoxGeometry(1, 1, 1);
const sphere = new THREE.SphereGeometry(0.5, 32, 16);
const plane = new THREE.PlaneGeometry(10, 10); // lies in XY; rotate for a floor:
plane.rotateX(-Math.PI / 2);

const mesh = new THREE.Mesh(box, new THREE.MeshStandardMaterial({ color: 0x00ff00 }));
scene.add(mesh);
```

## Built-in Geometries

All geometries are `BufferGeometry` subclasses; constructor args are stored in `geometry.parameters`.

### Basic Shapes

```javascript
// Box - width, height, depth, widthSegments, heightSegments, depthSegments
new THREE.BoxGeometry(1, 1, 1, 1, 1, 1);

// Sphere - radius, widthSegments, heightSegments, phiStart, phiLength, thetaStart, thetaLength
new THREE.SphereGeometry(1, 32, 16);
new THREE.SphereGeometry(1, 32, 16, 0, Math.PI * 2, 0, Math.PI / 2); // top hemisphere (thetaLength)

// Plane - width, height, widthSegments, heightSegments
new THREE.PlaneGeometry(10, 10, 1, 1);

// Circle - radius, segments, thetaStart, thetaLength
new THREE.CircleGeometry(1, 32);
new THREE.CircleGeometry(1, 32, 0, Math.PI); // semicircle

// Cylinder - radiusTop, radiusBottom, height, radialSegments, heightSegments, openEnded, thetaStart, thetaLength
new THREE.CylinderGeometry(1, 1, 2, 32, 1, false);
new THREE.CylinderGeometry(1, 1, 2, 6); // hexagonal prism

// Cone - radius, height, radialSegments, heightSegments, openEnded, thetaStart, thetaLength
new THREE.ConeGeometry(1, 2, 32);

// Torus - radius, tube, radialSegments, tubularSegments, arc
new THREE.TorusGeometry(1, 0.4, 16, 100);

// TorusKnot - radius, tube, tubularSegments, radialSegments, p, q
new THREE.TorusKnotGeometry(1, 0.4, 100, 16, 2, 3);

// Ring - innerRadius, outerRadius, thetaSegments, phiSegments, thetaStart, thetaLength
new THREE.RingGeometry(0.5, 1, 32, 1);

// Capsule - radius, height (of the middle section), capSegments, radialSegments, heightSegments
new THREE.CapsuleGeometry(0.5, 1, 4, 8, 1);
```

### Polyhedra

```javascript
// radius, detail (0 = base shape; higher subdivides toward a sphere)
new THREE.TetrahedronGeometry(1, 0);
new THREE.OctahedronGeometry(1, 0);
new THREE.DodecahedronGeometry(1, 0);
new THREE.IcosahedronGeometry(1, 0); // detail 3+ is a good low-poly-free "geosphere"

// Polyhedron - vertices (flat array), indices (flat array), radius, detail
const vertices = [1, 1, 1, -1, -1, 1, -1, 1, -1, 1, -1, -1];
const indices = [2, 1, 0, 0, 3, 2, 1, 3, 0, 2, 3, 1];
new THREE.PolyhedronGeometry(vertices, indices, 1, 0);
```

### Shapes, Extrusion, Lathe, Tube

```javascript
// 2D shape (XY plane). Holes go in shape.holes.
const shape = new THREE.Shape();
shape.moveTo(0, 0);
shape.lineTo(2, 0);
shape.quadraticCurveTo(2, 1, 1, 1.5);
shape.bezierCurveTo(0.5, 1.5, 0, 1, 0, 0);

const hole = new THREE.Path();
hole.absarc(1, 0.6, 0.25, 0, Math.PI * 2, false);
shape.holes.push(hole);

// Flat filled shape
new THREE.ShapeGeometry(shape, 12); // curveSegments

// Extrude - shape(s), options
new THREE.ExtrudeGeometry(shape, {
  depth: 0.5,          // extrusion length along +Z
  steps: 1,
  curveSegments: 12,
  bevelEnabled: true,
  bevelThickness: 0.1,
  bevelSize: 0.05,
  bevelOffset: 0,
  bevelSegments: 3,
  // extrudePath: curve3, // extrude along a 3D curve instead of Z
});

// Lathe - points (Vector2, x = radius, y = height), segments, phiStart, phiLength
const profile = [new THREE.Vector2(0, 0), new THREE.Vector2(0.5, 0), new THREE.Vector2(0.4, 1), new THREE.Vector2(0, 1.2)];
new THREE.LatheGeometry(profile, 32);

// Tube - path (Curve), tubularSegments, radius, radialSegments, closed
const curve = new THREE.CatmullRomCurve3([
  new THREE.Vector3(-1, 0, 0),
  new THREE.Vector3(0, 1, 0),
  new THREE.Vector3(1, 0, 0),
]);
new THREE.TubeGeometry(curve, 64, 0.2, 8, false);
```

Curves (`CatmullRomCurve3`, `CubicBezierCurve3`, `QuadraticBezierCurve3`, `LineCurve3`,
`EllipseCurve`, `CurvePath`) share `getPoint(t)`, `getPoints(n)`, `getSpacedPoints(n)`,
`getTangent(t)`, `getLength()`, and `getPointAt(u)` (arc-length parameterized). See `threejs-math`.

SVG paths: `SVGLoader` (`three/addons/loaders/SVGLoader.js`) returns `{ paths }` (ShapePaths);
`path.toShapes()` gives shapes for `ShapeGeometry`/`ExtrudeGeometry` (`SVGLoader.createShapes()` is deprecated since r185).
SVG geometry is Y-down: flip with `mesh.scale.y = -1`.

### Text Geometry

```javascript
import { FontLoader } from "three/addons/loaders/FontLoader.js";
import { TextGeometry } from "three/addons/geometries/TextGeometry.js";

const font = await new FontLoader().loadAsync("fonts/helvetiker_regular.typeface.json");
// fonts ship with three: node_modules/three/examples/fonts/ (copy to your public folder)
// TTF directly: TTFLoader from three/addons/loaders/TTFLoader.js, then new Font(json)

const geometry = new TextGeometry("Hello", {
  font,
  size: 1,
  depth: 0.2,          // extrusion; 'height' was renamed to 'depth' (r163)
  curveSegments: 12,
  bevelEnabled: true,
  bevelThickness: 0.03,
  bevelSize: 0.02,
  bevelSegments: 5,
});
geometry.center(); // computes the bounding box internally
scene.add(new THREE.Mesh(geometry, material));
```

For lots of dynamic text, prefer HTML labels (`CSS2DRenderer`, see `threejs-objects`) or an SDF text library: TextGeometry is heavy.

## BufferGeometry

Data lives in typed arrays (`BufferAttribute`) that are uploaded to the GPU.

### Custom Geometry (indexed quad)

```javascript
const geometry = new THREE.BufferGeometry();

const positions = new Float32Array([
  -1, -1, 0, // 0
   1, -1, 0, // 1
   1,  1, 0, // 2
  -1,  1, 0, // 3
]);
geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));

geometry.setIndex([0, 1, 2, 0, 2, 3]); // counter-clockwise = front face

geometry.setAttribute("normal", new THREE.Float32BufferAttribute([0, 0, 1, 0, 0, 1, 0, 0, 1, 0, 0, 1], 3));
geometry.setAttribute("uv", new THREE.Float32BufferAttribute([0, 0, 1, 0, 1, 1, 0, 1], 2));
geometry.setAttribute("color", new THREE.Float32BufferAttribute([1, 0, 0, 0, 1, 0, 0, 0, 1, 1, 1, 0], 3));
// vertex colors need material.vertexColors = true; values are linear

// Or let three compute normals
geometry.computeVertexNormals();
```

`setFromPoints` builds a position attribute from Vector2/Vector3 arrays:

```javascript
const g = new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(0, 0, 0), new THREE.Vector3(1, 1, 0)]);
```

### Attribute Types

```javascript
new THREE.BufferAttribute(typedArray, itemSize, normalized = false);
new THREE.Float32BufferAttribute(arrayOrCount, itemSize); // typed helpers for every type
new THREE.Uint16BufferAttribute(...);                      // indices up to 65535 vertices
new THREE.Uint32BufferAttribute(...);                      // bigger meshes (setIndex picks automatically for arrays)
new THREE.Uint8BufferAttribute(colors, 3, true);           // normalized: 0-255 read as 0-1

// itemSize: position 3, normal 3, uv 2, color 3 or 4, tangent 4, skinIndex/skinWeight 4
// Extra UV sets are named uv1, uv2, uv3 (selected per texture via texture.channel)
```

### Modifying Vertices

```javascript
const pos = geometry.attributes.position;
pos.setUsage(THREE.DynamicDrawUsage); // hint for frequently updated buffers (before first render)

for (let i = 0; i < pos.count; i++) {
  const y = pos.getY(i);
  pos.setY(i, y + Math.sin(i) * 0.1);
}
pos.needsUpdate = true;          // upload again
geometry.computeVertexNormals(); // lighting follows the new shape
geometry.computeBoundingSphere(); // keep frustum culling and raycasting correct

// Partial update of a large buffer
pos.clearUpdateRanges();
pos.addUpdateRange(startIndex * 3, count * 3);
pos.needsUpdate = true;

// Draw only part of the geometry
geometry.setDrawRange(0, 300); // start, count (vertices or indices)
```

You cannot grow a BufferAttribute in place. Allocate the maximum size up front and use `setDrawRange`, or create a new attribute (and `geometry.dispose()` the old buffers).

### Groups (multi-material)

```javascript
geometry.clearGroups();
geometry.addGroup(0, 6, 0); // start (index), count, materialIndex
geometry.addGroup(6, 6, 1);
const mesh = new THREE.Mesh(geometry, [materialA, materialB]);
```

### Interleaved Buffers (Advanced)

```javascript
const interleaved = new THREE.InterleavedBuffer(
  new Float32Array([
    // x, y, z, u, v  per vertex
    -1, -1, 0, 0, 0,
     1, -1, 0, 1, 0,
     1,  1, 0, 1, 1,
    -1,  1, 0, 0, 1,
  ]),
  5, // stride in floats
);
geometry.setAttribute("position", new THREE.InterleavedBufferAttribute(interleaved, 3, 0));
geometry.setAttribute("uv", new THREE.InterleavedBufferAttribute(interleaved, 2, 3));
```

### Transforms and Conversions

```javascript
geometry.translate(0, 1, 0);   // bakes into vertices (unlike mesh.position)
geometry.rotateX(Math.PI / 2);
geometry.scale(2, 2, 2);
geometry.center();             // move the bounding-box center to the origin
geometry.applyMatrix4(matrix);
geometry.lookAt(vector);

const nonIndexed = geometry.toNonIndexed(); // e.g. for flat per-face data
geometry.computeTangents();    // needs index, position, normal, uv (for normal maps without tangents)

geometry.computeBoundingBox();
const size = geometry.boundingBox.getSize(new THREE.Vector3());

geometry.dispose(); // free GPU buffers when no longer used
```

## EdgesGeometry and WireframeGeometry

```javascript
// Only "hard" edges where the face angle exceeds thresholdAngle (degrees, default 1)
const edges = new THREE.EdgesGeometry(boxGeometry, 15);
scene.add(new THREE.LineSegments(edges, new THREE.LineBasicMaterial({ color: 0xffffff })));

// Every triangle edge
const wire = new THREE.WireframeGeometry(boxGeometry);
scene.add(new THREE.LineSegments(wire, new THREE.LineBasicMaterial({ color: 0xffffff })));
```

WebGL lines are always 1px wide. For thick lines use `Line2`/`LineSegments2` (see `threejs-objects`).

## Points

```javascript
const count = 1000;
const positions = new Float32Array(count * 3);
for (let i = 0; i < count * 3; i++) positions[i] = (Math.random() - 0.5) * 10;

const geometry = new THREE.BufferGeometry();
geometry.setAttribute("position", new THREE.BufferAttribute(positions, 3));

const points = new THREE.Points(
  geometry,
  new THREE.PointsMaterial({ size: 0.1, sizeAttenuation: true, color: 0xffffff }),
);
scene.add(points);
```

## Lines

```javascript
const geometry = new THREE.BufferGeometry().setFromPoints([
  new THREE.Vector3(-1, 0, 0),
  new THREE.Vector3(0, 1, 0),
  new THREE.Vector3(1, 0, 0),
]);
const material = new THREE.LineBasicMaterial({ color: 0xff0000 });

new THREE.Line(geometry, material);         // connected strip
new THREE.LineLoop(geometry, material);     // closed strip
new THREE.LineSegments(geometry, material); // independent pairs: (0,1), (2,3), ...

// Smooth curve as a line
const curvePoints = curve.getPoints(100);
new THREE.Line(new THREE.BufferGeometry().setFromPoints(curvePoints), material);
```

## InstancedMesh

One draw call for many copies of the same geometry and material.

```javascript
const count = 1000;
const mesh = new THREE.InstancedMesh(
  new THREE.BoxGeometry(1, 1, 1),
  new THREE.MeshStandardMaterial(),
  count,
);
mesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage); // only if you update often

const dummy = new THREE.Object3D();
const color = new THREE.Color();
for (let i = 0; i < count; i++) {
  dummy.position.randomDirection().multiplyScalar(Math.random() * 20);
  dummy.rotation.set(Math.random() * Math.PI, Math.random() * Math.PI, 0);
  dummy.scale.setScalar(0.5 + Math.random());
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
  mesh.setColorAt(i, color.setHSL(Math.random(), 0.7, 0.5)); // creates instanceColor on first call
}
mesh.instanceMatrix.needsUpdate = true;
mesh.instanceColor.needsUpdate = true;
mesh.computeBoundingSphere(); // REQUIRED for correct frustum culling/raycasting after moving instances
scene.add(mesh);

mesh.count = 500; // render only the first N instances (must be <= the constructor count)
```

```javascript
// Update one instance
const m = new THREE.Matrix4();
mesh.getMatrixAt(index, m);
m.setPosition(x, y, z);
mesh.setMatrixAt(index, m);
mesh.instanceMatrix.needsUpdate = true;

// Raycasting returns instanceId
const hit = raycaster.intersectObject(mesh)[0];
if (hit) console.log(hit.instanceId);
```

Different geometries in one draw call: `BatchedMesh` (see `threejs-objects`).

### Custom Per-Instance Attributes

```javascript
const offsets = new Float32Array(count);
for (let i = 0; i < count; i++) offsets[i] = Math.random();
geometry.setAttribute("aOffset", new THREE.InstancedBufferAttribute(offsets, 1));
// GLSL: attribute float aOffset;  (use with ShaderMaterial or onBeforeCompile, see threejs-shaders)
```

`InstancedBufferGeometry` + `instanceCount` is the lower-level alternative when you do not want `instanceMatrix` at all.

## Morph Targets

```javascript
const geometry = new THREE.BoxGeometry(1, 1, 1, 4, 4, 4);
const target = geometry.attributes.position.array.slice();
for (let i = 0; i < target.length; i += 3) {
  target[i] *= 2;       // stretch X
  target[i + 1] *= 0.5; // squash Y
}
geometry.morphAttributes.position = [new THREE.Float32BufferAttribute(target, 3)];
// geometry.morphAttributes.normal = [...] for correct lighting
// geometry.morphTargetsRelative = true if the targets are deltas

const mesh = new THREE.Mesh(geometry, material); // reads morph targets on construction
mesh.morphTargetInfluences[0] = 0.5;
// If you add morph attributes after creating the mesh: mesh.updateMorphTargets();
```

Animating morph targets and GLTF shape keys: see `threejs-animation`.

## BufferGeometryUtils

```javascript
import * as BufferGeometryUtils from "three/addons/utils/BufferGeometryUtils.js";

// Merge (all inputs need the same attributes and index-ness)
const merged = BufferGeometryUtils.mergeGeometries([geo1, geo2, geo3]);
const withGroups = BufferGeometryUtils.mergeGeometries([geo1, geo2], true); // one group per input

// Weld duplicate vertices (e.g. before computeVertexNormals for smooth shading)
const welded = BufferGeometryUtils.mergeVertices(geometry, 1e-4);

// Smooth normals while keeping hard edges
const creased = BufferGeometryUtils.toCreasedNormals(geometry, Math.PI / 6);

// Interleave
const [pos, nrm] = BufferGeometryUtils.interleaveAttributes([geometry.attributes.position, geometry.attributes.normal]);

// MikkTSpace tangents (matches Blender), needs the mikktspace lib
import * as MikkTSpace from "three/addons/libs/mikktspace.module.js";
await MikkTSpace.ready;
BufferGeometryUtils.computeMikkTSpaceTangents(geometry, MikkTSpace);
// Simple tangents without the lib: geometry.computeTangents()
```

Addon geometries (import from `three/addons/geometries/...`): `RoundedBoxGeometry`,
`ConvexGeometry`, `DecalGeometry`, `ParametricGeometry`, `TeapotGeometry`, `BoxLineGeometry`.
See `threejs-addons`.

## Common Patterns

### Scale to Fit a Unit Box

```javascript
geometry.computeBoundingBox();
const size = geometry.boundingBox.getSize(new THREE.Vector3());
geometry.scale(...Array(3).fill(1 / Math.max(size.x, size.y, size.z)));
geometry.center();
```

### Terrain From a Plane

```javascript
const terrain = new THREE.PlaneGeometry(100, 100, 128, 128);
terrain.rotateX(-Math.PI / 2);
const p = terrain.attributes.position;
for (let i = 0; i < p.count; i++) {
  p.setY(i, Math.sin(p.getX(i) * 0.1) * Math.cos(p.getZ(i) * 0.1) * 5);
}
terrain.computeVertexNormals();
```

## Performance Tips

1. **Indexed geometry** reuses vertices (built-ins are indexed).
2. **Merge static meshes** with `mergeGeometries` to cut draw calls.
3. **InstancedMesh / BatchedMesh** for many objects.
4. **Moderate segment counts**: `SphereGeometry(1, 32, 16)` is plenty for most uses.
5. **Share geometries** between meshes; `dispose()` when done.
6. **Compressed meshes**: Draco/Meshopt via GLTFLoader (see `threejs-loaders`).

## See Also

- `threejs-fundamentals` - Scene setup and Object3D
- `threejs-materials` - Materials for meshes, points and lines
- `threejs-objects` - BatchedMesh, Sprite, LOD, fat lines
- `threejs-shaders` - Vertex manipulation in shaders
- `threejs-math` - Curves, vectors, bounding volumes
