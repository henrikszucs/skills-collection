---
name: threejs-math
description: Three.js math API - Vector2/3/4, Matrix3/4, Quaternion, Euler, Color, MathUtils, Box3, Sphere, Ray, Plane, Frustum, Triangle, Line3, Spherical/Cylindrical coordinates, curves, plus math addons (OBB, Octree/Capsule collision, ConvexHull, noise, MeshSurfaceSampler). Use for transforms and rotations, bounding boxes and fitting, intersection and distance tests, frustum/visibility checks, orbit/spherical coordinates, interpolation, smoothing, random numbers, or simple collision detection.
---

# Three.js Math

> **Version:** written for three.js **r186**. See `threejs-migration` for renamed APIs.

## Conventions (read first)

- **Methods mutate `this` and return it** (chainable): `v.add(a).normalize()`. Use `.clone()` to keep the original.
- **Getters take a `target`**: `box.getCenter(new THREE.Vector3())`, `obj.getWorldPosition(target)`.
- **Reuse scratch objects** in the render loop instead of `new` every frame:

```javascript
const _v = new THREE.Vector3(); // module-level scratch
function update() {
  mesh.getWorldPosition(_v);
}
```

- Angles are **radians**; right-handed, Y-up; matrices are column-major internally (`elements`),
  but `Matrix4.set()` takes **row-major** arguments.

## Vector3 (and Vector2/Vector4)

```javascript
const v = new THREE.Vector3(1, 2, 3);
v.set(x, y, z); v.setScalar(0); v.copy(other); v.clone();
v.add(a); v.sub(a); v.multiply(a); v.divide(a);
v.addScaledVector(dir, distance); // v += dir * distance
v.multiplyScalar(2); v.divideScalar(2); v.negate();
v.normalize(); v.setLength(5); v.clampLength(0, 10);
v.min(a); v.max(a); v.clamp(minV, maxV); v.floor(); v.round();

v.length(); v.lengthSq();      // lengthSq is cheaper for comparisons
v.distanceTo(a); v.distanceToSquared(a); v.manhattanDistanceTo(a);
v.dot(a); v.angleTo(a);        // radians
v.cross(a); v.crossVectors(a, b);
v.equals(a);

v.lerp(target, t); v.lerpVectors(a, b, t);
v.projectOnVector(axis); v.projectOnPlane(planeNormal); v.reflect(normal);

v.applyMatrix3(m3); v.applyMatrix4(m4); v.applyQuaternion(q); v.applyEuler(e);
v.applyAxisAngle(axis, angle);
v.transformDirection(m4);      // rotate only + normalize
v.project(camera);             // world -> NDC (x, y in -1..1; z in -1..1 if within near/far)
v.unproject(camera);           // NDC -> world

v.setFromMatrixPosition(obj.matrixWorld);
v.setFromSphericalCoords(radius, phi, theta);
v.randomDirection();           // random unit vector
v.fromArray(arr, offset); v.toArray(arr, offset);
v.fromBufferAttribute(geometry.attributes.position, i);
```

## Quaternion and Euler

Prefer quaternions for interpolation and composition; use Euler for intuitive input.

```javascript
const q = new THREE.Quaternion();
q.setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI / 2);
q.setFromEuler(new THREE.Euler(0, Math.PI / 2, 0, "YXZ"));
q.setFromRotationMatrix(m4);
q.setFromUnitVectors(vFrom, vTo); // rotation taking one direction to another (both normalized)

q.multiply(q2);      // this = this * q2  (apply q2 first, then this)
q.premultiply(q2);   // this = q2 * this
q.invert(); q.normalize();
q.slerp(target, t); q.slerpQuaternions(a, b, t);
q.rotateTowards(target, maxStepRadians); // constant angular speed
q.angleTo(q2);

const e = new THREE.Euler(x, y, z, "XYZ"); // order: 'XYZ' (default), 'YXZ' (FPS cameras), ...
e.setFromQuaternion(q, "YXZ");
object.rotation.reorder("YXZ");
```

Rotate an object to face a direction: `object.quaternion.setFromUnitVectors(new THREE.Vector3(0, 0, 1), dir.normalize())`
or `object.lookAt(worldPoint)`.

## Matrix4 / Matrix3

```javascript
const m = new THREE.Matrix4();
m.identity(); m.copy(o); m.clone();
m.makeTranslation(x, y, z);   // or makeTranslation(vector3)
m.makeRotationX(a); m.makeRotationY(a); m.makeRotationZ(a);
m.makeRotationAxis(axis, angle); m.makeRotationFromQuaternion(q); m.makeRotationFromEuler(e);
m.makeScale(x, y, z);
m.lookAt(eye, target, up);    // rotation only
m.compose(position, quaternion, scale);
m.decompose(position, quaternion, scale);

m.multiply(b);     // this = this * b
m.premultiply(b);  // this = b * this
m.multiplyMatrices(a, b);
m.invert(); m.transpose(); m.determinant();
m.setPosition(x, y, z);
m.extractRotation(other);

// Matrix3
const n = new THREE.Matrix3().getNormalMatrix(modelViewMatrix);
const r = new THREE.Matrix3().makeRotation(angle); // 2D (rotate()/scale()/translate() deprecated r185)
```

World transform of an object: `object.matrixWorld` (call `object.updateMatrixWorld()` first if you changed transforms this frame outside the render).

## Color

```javascript
const c = new THREE.Color(0xff8800);  // also "orange", "#f80", "rgb(255,136,0)", "hsl(30,100%,50%)"
c.set(0x00ff00);
c.setRGB(1, 0.5, 0);                  // LINEAR values by default
c.setRGB(1, 0.5, 0, THREE.SRGBColorSpace); // sRGB values (e.g. from a design tool)
c.setHSL(h, s, l);
c.getHex(); c.getHexString(); c.getStyle();
c.getHSL(target);
c.lerp(other, t); c.lerpHSL(other, t);
c.offsetHSL(0.1, 0, 0);
c.multiplyScalar(2);                  // HDR colors > 1 are fine (bloom)
```

## MathUtils

```javascript
const { MathUtils } = THREE;
MathUtils.clamp(x, min, max);
MathUtils.lerp(a, b, t);
MathUtils.inverseLerp(a, b, value);           // where value sits between a and b (0..1)
MathUtils.mapLinear(x, inMin, inMax, outMin, outMax);
MathUtils.damp(current, target, lambda, dt);  // frame-rate independent smoothing
MathUtils.smoothstep(x, min, max); MathUtils.smootherstep(x, min, max);
MathUtils.pingpong(x, length);
MathUtils.euclideanModulo(n, m);               // always positive modulo
MathUtils.degToRad(deg); MathUtils.radToDeg(rad);
MathUtils.randFloat(min, max); MathUtils.randInt(min, max); MathUtils.randFloatSpread(range);
MathUtils.seededRandom(seed);                  // deterministic 0..1
MathUtils.generateUUID();
MathUtils.isPowerOfTwo(n); MathUtils.ceilPowerOfTwo(n);
```

## Box3 (axis-aligned bounding box)

```javascript
const box = new THREE.Box3().setFromObject(object);     // world-space AABB of an object tree
const precise = new THREE.Box3().setFromObject(object, true); // exact (vertex-level) for rotated meshes
box.setFromPoints(points); box.setFromBufferAttribute(positionAttribute);

box.getCenter(target); box.getSize(target);
box.min; box.max;
box.isEmpty();
box.expandByPoint(p); box.expandByScalar(0.1); box.expandByObject(obj);
box.union(otherBox); box.intersect(otherBox);
box.containsPoint(p); box.containsBox(b);
box.intersectsBox(b); box.intersectsSphere(s); box.intersectsPlane(p);
box.distanceToPoint(p); box.clampPoint(p, target);
box.applyMatrix4(m);    // transform (result is still axis-aligned)

geometry.computeBoundingBox(); geometry.boundingBox; // local-space box of a geometry
```

Fit object into a unit box or frame the camera: see `threejs-loaders` (Prepare a Loaded Model).

## Sphere

```javascript
const sphere = new THREE.Sphere(center, radius);
box.getBoundingSphere(sphere);
sphere.setFromPoints(points);
sphere.containsPoint(p); sphere.intersectsSphere(s); sphere.intersectsBox(b);
sphere.distanceToPoint(p);
sphere.applyMatrix4(m);
```

## Ray

```javascript
const ray = new THREE.Ray(origin, direction.normalize()); // raycaster.ray is one
ray.at(t, target);                       // origin + direction * t
ray.intersectPlane(plane, target);       // Vector3 or null
ray.intersectBox(box, target);
ray.intersectSphere(sphere, target);
ray.intersectTriangle(a, b, c, backfaceCulling, target);
ray.intersectsBox(box);                  // boolean versions
ray.distanceToPoint(p);
ray.closestPointToPoint(p, target);
```

## Plane

```javascript
// normal . point + constant = 0   (constant = -distance from origin along the normal)
const ground = new THREE.Plane(new THREE.Vector3(0, 1, 0), 0);
const plane = new THREE.Plane().setFromNormalAndCoplanarPoint(normal, point);
plane.setFromCoplanarPoints(a, b, c);
plane.distanceToPoint(p);     // signed: positive on the normal side
plane.projectPoint(p, target);
plane.intersectLine(line3, target);
plane.applyMatrix4(m);
plane.negate();
```

## Frustum (visibility tests)

```javascript
const frustum = new THREE.Frustum();
const projScreen = new THREE.Matrix4();

camera.updateMatrixWorld();
projScreen.multiplyMatrices(camera.projectionMatrix, camera.matrixWorldInverse);
frustum.setFromProjectionMatrix(projScreen);

frustum.intersectsObject(mesh);  // uses geometry bounding sphere + matrixWorld
frustum.intersectsBox(box);
frustum.intersectsSphere(sphere);
frustum.containsPoint(point);
```

## Triangle and Line3

```javascript
const tri = new THREE.Triangle(a, b, c);
tri.getNormal(target); tri.getArea(); tri.getMidpoint(target);
tri.containsPoint(p);
tri.closestPointToPoint(p, target);
THREE.Triangle.getBarycoord(p, a, b, c, target);

const seg = new THREE.Line3(start, end);
seg.closestPointToPoint(p, true, target); // clamp to segment
seg.distance(); seg.getCenter(target); seg.at(t, target);
```

## Spherical and Cylindrical Coordinates

```javascript
// phi = polar angle from +Y (0..PI), theta = azimuth around Y from +Z
const s = new THREE.Spherical().setFromVector3(camera.position.clone().sub(target));
s.theta += 0.01;          // orbit horizontally
s.phi = THREE.MathUtils.clamp(s.phi, 0.1, Math.PI / 2);
s.makeSafe();
camera.position.setFromSpherical(s).add(target);
camera.lookAt(target);

// Latitude/longitude on a globe of radius R
const phi = THREE.MathUtils.degToRad(90 - lat);
const theta = THREE.MathUtils.degToRad(lon + 180);
const pos = new THREE.Vector3().setFromSphericalCoords(R, phi, theta);

const cyl = new THREE.Cylindrical(radius, theta, y);
const p = new THREE.Vector3().setFromCylindrical(cyl);
```

## Curves

```javascript
const curve = new THREE.CatmullRomCurve3(points, true /* closed */, "centripetal");
curve.getPoint(t);        // t: 0..1 by parameter (uneven speed)
curve.getPointAt(u);      // u: 0..1 by arc length (even speed)
curve.getTangentAt(u);
curve.getPoints(100);     // for line geometry
curve.getSpacedPoints(100);
curve.getLength();
curve.computeFrenetFrames(segments, closed); // for orienting objects along the path

// Others: CubicBezierCurve3, QuadraticBezierCurve3, LineCurve3, EllipseCurve, ArcCurve,
// SplineCurve (2D), CurvePath (chain), Path/Shape (2D, see threejs-geometry)
```

## Math Addons (`three/addons/math/...`)

```javascript
import { OBB } from "three/addons/math/OBB.js";           // oriented bounding box
const obb = new OBB().fromBox3(geometry.boundingBox);
obb.applyMatrix4(mesh.matrixWorld);
obb.intersectsOBB(otherObb);

import { Octree } from "three/addons/math/Octree.js";     // static world collision
import { Capsule } from "three/addons/math/Capsule.js";
const worldOctree = new Octree().fromGraphNode(levelScene);
const player = new Capsule(new THREE.Vector3(0, 0.35, 0), new THREE.Vector3(0, 1, 0), 0.35);
const result = worldOctree.capsuleIntersect(player); // { normal, depth } or false
if (result) player.translate(result.normal.multiplyScalar(result.depth));
// Full FPS example: games_fps

import { ConvexHull } from "three/addons/math/ConvexHull.js";
import { SimplexNoise } from "three/addons/math/SimplexNoise.js";
const noise = new SimplexNoise();
noise.noise(x, y); noise.noise3d(x, y, z);
import { ImprovedNoise } from "three/addons/math/ImprovedNoise.js"; // Perlin: new ImprovedNoise().noise(x, y, z)

import { MeshSurfaceSampler } from "three/addons/math/MeshSurfaceSampler.js";
const sampler = new MeshSurfaceSampler(mesh).build();  // random points on a surface
const pos = new THREE.Vector3(), nrm = new THREE.Vector3();
sampler.sample(pos, nrm);                              // e.g. scatter grass/instances

import { Lut } from "three/addons/math/Lut.js";       // scalar -> color maps for data viz
const lut = new Lut("rainbow", 512);
lut.setMin(0); lut.setMax(100);
const color = lut.getColor(42);
```

Physics engines: the addons include wrappers `RapierPhysics`, `JoltPhysics`, `AmmoPhysics`
(`three/addons/physics/...`). For full physics use Rapier/cannon-es/Jolt directly.

## See Also

- `threejs-fundamentals` - Object3D transforms
- `threejs-interaction` - Raycasting, screen/world conversion
- `threejs-geometry` - Curves to geometry, bounding volumes
- `threejs-animation` - Interpolation and damping in motion
