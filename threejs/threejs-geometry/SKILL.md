---
name: threejs-geometry
description: Three.js geometry - built-in shapes, BufferGeometry and BufferAttribute, custom meshes, shapes/extrusion/text, points and lines, InstancedMesh, morph targets, geometry utilities. Use when creating 3D shapes, editing vertices, building custom meshes, extruding 2D shapes, drawing points/lines, or rendering many copies with instancing.
---

# Three.js Geometry

> three.js **r187**. See `threejs-migration` for renamed APIs.

## Built-ins

All are `BufferGeometry` subclasses (args in `geometry.parameters`):
`BoxGeometry`, `SphereGeometry(r, wSeg, hSeg)`, `PlaneGeometry` (lies in XY: `rotateX(-Math.PI/2)`
for a floor), `CircleGeometry`, `CylinderGeometry(rTop, rBottom, h, radial)`, `ConeGeometry`,
`TorusGeometry`, `TorusKnotGeometry`, `RingGeometry`, `CapsuleGeometry(radius, height, capSeg, radialSeg, heightSeg)`,
`Tetra/Octa/Dodeca/IcosahedronGeometry(radius, detail)`, `PolyhedronGeometry`,
`LatheGeometry(points: Vector2[])`, `TubeGeometry(curve, tubularSeg, radius, radialSeg, closed)`,
`ShapeGeometry`, `ExtrudeGeometry`, `EdgesGeometry`, `WireframeGeometry`.

`SphereGeometry(1, 32, 16)` is plenty for most uses.

## Shapes and Extrusion

```javascript
const shape = new THREE.Shape().moveTo(0, 0).lineTo(2, 0).quadraticCurveTo(2, 1, 1, 1.5).lineTo(0, 0);
const hole = new THREE.Path().absarc(1, 0.6, 0.25, 0, Math.PI * 2, false);
shape.holes.push(hole);

new THREE.ShapeGeometry(shape, 12);
new THREE.ExtrudeGeometry(shape, {
  depth: 0.5, bevelEnabled: true, bevelThickness: 0.1, bevelSize: 0.05, bevelSegments: 3,
  // extrudePath: curve3  -> extrude along a 3D curve
});
```

SVG: `SVGLoader` returns `{ paths }`; `path.toShapes()` (r185; `SVGLoader.createShapes()` is
deprecated). SVG is Y-down: `mesh.scale.y = -1`.

**Text:**

```javascript
import { FontLoader } from "three/addons/loaders/FontLoader.js";
import { TextGeometry } from "three/addons/geometries/TextGeometry.js";
const font = await new FontLoader().loadAsync("fonts/helvetiker_regular.typeface.json"); // from three/examples/fonts/
const g = new TextGeometry("Hello", { font, size: 1, depth: 0.2, bevelEnabled: false }); // 'depth', not 'height' (r163)
g.center();
```

TextGeometry is heavy; for lots of dynamic text use `CSS2DRenderer` labels or SDF text.

## BufferGeometry

```javascript
const g = new THREE.BufferGeometry();
g.setAttribute("position", new THREE.Float32BufferAttribute([-1,-1,0, 1,-1,0, 1,1,0, -1,1,0], 3));
g.setAttribute("uv", new THREE.Float32BufferAttribute([0,0, 1,0, 1,1, 0,1], 2));
g.setIndex([0, 1, 2, 0, 2, 3]); // counter-clockwise = front face
g.computeVertexNormals();

new THREE.BufferGeometry().setFromPoints(vector3Array); // quick position-only geometry
```

- itemSize: position/normal 3, uv 2, color 3-4 (linear, needs `vertexColors: true`), tangent 4,
  skinIndex/skinWeight 4. Extra UV sets are `uv1`, `uv2`, `uv3`.
- `Uint8BufferAttribute(data, 3, true)`: normalized (0-255 read as 0-1).

### Editing Vertices

```javascript
const pos = g.attributes.position;
pos.setUsage(THREE.DynamicDrawUsage);   // before first render, for frequent updates
for (let i = 0; i < pos.count; i++) pos.setY(i, pos.getY(i) + Math.sin(i) * 0.1);
pos.needsUpdate = true;
g.computeVertexNormals();
g.computeBoundingSphere();              // keeps culling and raycasting correct

pos.clearUpdateRanges(); pos.addUpdateRange(start * 3, count * 3); // partial upload
g.setDrawRange(0, 300);                 // draw only part
```

Attributes cannot grow. Allocate the max up front and use `setDrawRange`, or replace the
attribute and `dispose()` the old geometry.

Also: `addGroup(start, count, materialIndex)` for multi-material, `InterleavedBuffer` +
`InterleavedBufferAttribute` for packed data, `translate/rotateX/scale/center/applyMatrix4`
(bake into vertices), `toNonIndexed()`, `computeTangents()`, `dispose()`.

## Points and Lines

```javascript
new THREE.Points(geometry, new THREE.PointsMaterial({ size: 0.1, sizeAttenuation: true }));
new THREE.Line(g, mat);         // strip
new THREE.LineLoop(g, mat);     // closed strip
new THREE.LineSegments(g, mat); // pairs (0,1), (2,3), ...
new THREE.LineSegments(new THREE.EdgesGeometry(box, 15), mat); // hard edges only (angle threshold in degrees)
```

WebGL lines are always 1px: thick lines with `Line2` (`threejs-objects`).

## InstancedMesh

One draw call for many copies of one geometry + material.

```javascript
const mesh = new THREE.InstancedMesh(geometry, material, count);
const dummy = new THREE.Object3D();
const color = new THREE.Color();
for (let i = 0; i < count; i++) {
  dummy.position.randomDirection().multiplyScalar(Math.random() * 20);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
  mesh.setColorAt(i, color.setHSL(Math.random(), 0.7, 0.5)); // creates instanceColor
}
mesh.instanceMatrix.needsUpdate = true;
mesh.instanceColor.needsUpdate = true;
mesh.computeBoundingSphere(); // REQUIRED after moving instances (culling, raycasting)
mesh.count = 500;             // draw only the first N
// raycast hits report hit.instanceId
```

Per-instance data: `geometry.setAttribute("aOffset", new THREE.InstancedBufferAttribute(arr, 1))`
(read it in a shader). Different geometries in one call: `BatchedMesh` (`threejs-objects`).

## Morph Targets

```javascript
geometry.morphAttributes.position = [new THREE.Float32BufferAttribute(targetPositions, 3)];
// morphAttributes.normal for correct lighting; morphTargetsRelative = true for deltas
const mesh = new THREE.Mesh(geometry, material); // or mesh.updateMorphTargets() if added later
mesh.morphTargetInfluences[0] = 0.5;
```

Animating them: `threejs-animation`.

## BufferGeometryUtils

```javascript
import * as BGU from "three/addons/utils/BufferGeometryUtils.js";
BGU.mergeGeometries([a, b], useGroups);   // same attributes and index-ness required
BGU.mergeVertices(g, 1e-4);               // weld, e.g. before computeVertexNormals
BGU.toCreasedNormals(g, Math.PI / 6);     // smooth but keep hard edges
BGU.toTrianglesDrawMode(g, mode);         // r186: modifies in place (clone first if needed)
// MikkTSpace tangents: await MikkTSpace.ready from three/addons/libs/mikktspace.module.js, then computeMikkTSpaceTangents
```

Addon geometries: `RoundedBoxGeometry`, `ConvexGeometry`, `DecalGeometry`, `ParametricGeometry`,
`TeapotGeometry` (`threejs-addons`).

## Performance

1. Indexed geometry; share geometries between meshes; `dispose()` when done.
2. Merge static meshes; instance repeated ones.
3. Moderate segment counts.
4. Compressed meshes (Draco/Meshopt) via GLTFLoader (`threejs-loaders`).

See also: `threejs-objects`, `threejs-materials`, `threejs-math` (curves), `threejs-shaders`.
