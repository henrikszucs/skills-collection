---
name: threejs-math
description: Three.js math API - Vector2/3/4, Matrix3/4, Quaternion, Euler, Color, MathUtils, Box3, Sphere, Ray, Plane, Frustum, Triangle, Line3, Spherical/Cylindrical coordinates, curves, plus math addons (OBB, Octree/Capsule collision, ConvexHull, noise, MeshSurfaceSampler). Use for transforms and rotations, bounding boxes and fitting, intersection and distance tests, frustum/visibility checks, orbit/spherical coordinates, interpolation, smoothing, random numbers, or simple collision detection.
---

# Three.js Math

> three.js **r187**. See `threejs-migration` for renamed APIs.

## Conventions (where bugs come from)

- **Methods mutate `this` and return it** (chainable). `.clone()` first to keep the original.
- **Getters take a `target`**: `box.getCenter(new THREE.Vector3())`.
- **Reuse scratch objects** in the render loop: `const _v = new THREE.Vector3()` at module level.
- Radians; right-handed, Y-up. Matrices are column-major in `elements`, but `Matrix4.set()` takes
  **row-major** arguments.
- `q.multiply(b)` = this × b (apply b first); `premultiply(b)` = b × this. Same for matrices.

## Non-Obvious API

```javascript
v.addScaledVector(dir, dist);            // v += dir * dist
v.project(camera); v.unproject(camera);  // world <-> NDC (-1..1)
v.transformDirection(m4);                // rotate only + normalize
v.setFromMatrixPosition(obj.matrixWorld);
v.fromBufferAttribute(geometry.attributes.position, i);
v.randomDirection();

q.setFromUnitVectors(from, to);          // rotation between two normalized directions
q.rotateTowards(target, maxStep);        // constant angular speed
e.setFromQuaternion(q, "YXZ");           // YXZ order for FPS cameras

m.compose(pos, quat, scale); m.decompose(pos, quat, scale);
m.makeRotationFromQuaternion(q); m.lookAt(eye, target, up);
new THREE.Matrix3().getNormalMatrix(modelViewMatrix);
new THREE.Matrix3().makeRotation(angle); // 2D; rotate()/scale()/translate() deprecated (r185)

c.setRGB(1, 0.5, 0);                     // LINEAR by default
c.setRGB(1, 0.5, 0, THREE.SRGBColorSpace);
c.setStyle("hsl(-30, 100%, 50%)");       // r187 accepts negative hues and decimal percentages
```

## MathUtils

`clamp`, `lerp`, `inverseLerp`, `mapLinear`, `damp(current, target, lambda, dt)` (frame-rate
independent), `smoothstep`, `smootherstep`, `pingpong`, `euclideanModulo`, `degToRad`, `radToDeg`,
`randFloat`, `randInt`, `randFloatSpread`, `seededRandom`, `generateUUID`, `ceilPowerOfTwo`.

## Bounding Volumes and Tests

```javascript
const box = new THREE.Box3().setFromObject(obj);       // world AABB of the whole tree
new THREE.Box3().setFromObject(obj, true);             // precise: per-vertex, slower (ignored for Instanced/BatchedMesh)
box.getCenter(t); box.getSize(t); box.expandByScalar(0.1);
box.intersectsBox(b); box.containsPoint(p); box.distanceToPoint(p);
box.getBoundingSphere(new THREE.Sphere());

const ray = raycaster.ray;                              // or new THREE.Ray(origin, dir.normalize())
ray.intersectPlane(plane, t); ray.intersectBox(box, t); ray.intersectSphere(s, t);
ray.intersectTriangle(a, b, c, backfaceCulling, t); ray.closestPointToPoint(p, t);

const plane = new THREE.Plane(normal, constant);        // normal . p + constant = 0
plane.setFromNormalAndCoplanarPoint(n, p);
plane.distanceToPoint(p);                               // signed, positive on the normal side

const frustum = new THREE.Frustum().setFromProjectionMatrix(
  new THREE.Matrix4().multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse),
);
frustum.intersectsObject(mesh); frustum.intersectsBox(box); frustum.containsPoint(p);

THREE.Triangle.getBarycoord(p, a, b, c, target);
new THREE.Line3(start, end).closestPointToPoint(p, true, target); // clamp to segment
```

## Spherical Coordinates

```javascript
// phi = polar angle from +Y, theta = azimuth around Y from +Z
const s = new THREE.Spherical().setFromVector3(camera.position.clone().sub(target));
s.theta += 0.01; s.phi = THREE.MathUtils.clamp(s.phi, 0.1, Math.PI / 2); s.makeSafe();
camera.position.setFromSpherical(s).add(target);

// lat/lon on a globe of radius R
const p = new THREE.Vector3().setFromSphericalCoords(R, THREE.MathUtils.degToRad(90 - lat), THREE.MathUtils.degToRad(lon + 180));
```

## Curves

`CatmullRomCurve3(points, closed, "centripetal")`, `CubicBezierCurve3`, `QuadraticBezierCurve3`,
`LineCurve3`, `EllipseCurve`, `CurvePath`. `getPoint(t)` is by parameter (uneven speed);
**`getPointAt(u)` is by arc length** (even speed). Also `getTangentAt`, `getSpacedPoints`,
`getLength`, `computeFrenetFrames`.

## Math Addons (`three/addons/math/`)

```javascript
import { Octree } from "three/addons/math/Octree.js";   // static-world collision (games_fps example)
import { Capsule } from "three/addons/math/Capsule.js";
const world = new Octree().fromGraphNode(level);
const player = new Capsule(new THREE.Vector3(0, 0.35, 0), new THREE.Vector3(0, 1, 0), 0.35);
const hit = world.capsuleIntersect(player);             // { normal, depth } or false
if (hit) player.translate(hit.normal.multiplyScalar(hit.depth));

import { MeshSurfaceSampler } from "three/addons/math/MeshSurfaceSampler.js";
const sampler = new MeshSurfaceSampler(mesh).build();   // random surface points (scatter grass)
sampler.sample(position, normal);
```

Also `OBB` (`fromBox3`, `applyMatrix4`, `intersectsOBB`), `ConvexHull`, `SimplexNoise`
(`noise(x, y)`, `noise3d`), `ImprovedNoise` (Perlin), `Lut` (scalar -> color for data viz).

See also: `threejs-interaction` (screen/world), `threejs-geometry`, `threejs-animation`.
