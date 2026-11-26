---
name: threejs-animation
description: Three.js animation - render loop timing with Timer, AnimationMixer/AnimationClip/AnimationAction, keyframe tracks, playing GLTF animations, crossfading and blending, additive animation, skeletal animation and bones, morph targets, cloning animated characters, retargeting and procedural motion. Use when animating objects, playing or blending GLTF/FBX animations, controlling bones or shape keys, or creating procedural/spring motion.
---

# Three.js Animation

> **Version:** written for three.js **r186**. `THREE.Clock` is deprecated (r183): use
> `THREE.Timer`. See `threejs-migration` for other renamed APIs.

## Quick Start

```javascript
import * as THREE from "three";

const timer = new THREE.Timer();
timer.connect(document); // no giant delta after switching tabs

renderer.setAnimationLoop((timestamp) => {
  timer.update(timestamp);         // once per frame, before reading
  const delta = timer.getDelta();  // seconds since last frame
  const elapsed = timer.getElapsed();

  mesh.rotation.y += delta;                    // frame-rate independent
  mesh.position.y = Math.sin(elapsed) * 0.5;

  renderer.render(scene, camera);
});
```

Always scale motion by `delta`; never assume 60 fps (many displays run at 120+ Hz).

## Animation System Overview

1. **AnimationClip**: named set of keyframe tracks (from GLTF/FBX or created in code).
2. **AnimationMixer**: plays clips on one root object and its descendants. Call `mixer.update(delta)` every frame.
3. **AnimationAction**: playback state of one clip on one mixer (play, weight, loop, fade).

## AnimationClip and KeyframeTracks

```javascript
const times = [0, 1, 2]; // seconds
const track = new THREE.NumberKeyframeTrack(".position[y]", times, [0, 1, 0]);
const clip = new THREE.AnimationClip("bounce", -1, [track]); // -1 = compute duration from tracks
```

Track names are property paths resolved from the mixer root:

| Path                          | Meaning                                 |
| ----------------------------- | --------------------------------------- |
| `.position`                   | root object's position                  |
| `.position[y]`                | one component                           |
| `.material.opacity`           | material property                       |
| `Head.quaternion`             | child object named "Head"               |
| `.morphTargetInfluences[smile]` | morph target by name (or index)       |

```javascript
// Number (one float per key)
new THREE.NumberKeyframeTrack(".material.opacity", [0, 1], [1, 0]);

// Vector (3 floats per key for position/scale)
new THREE.VectorKeyframeTrack(".position", [0, 1, 2], [0, 0, 0, 1, 2, 0, 0, 0, 0]);

// Quaternion (4 floats per key, slerped). Use for rotation, not Euler
const q1 = new THREE.Quaternion();
const q2 = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0, 1, 0), Math.PI);
new THREE.QuaternionKeyframeTrack(".quaternion", [0, 1], [...q1.toArray(), ...q2.toArray()]);

// Color (3 floats per key, linear RGB)
new THREE.ColorKeyframeTrack(".material.color", [0, 1, 2], [1, 0, 0, 0, 1, 0, 0, 0, 1]);

// Boolean / String (discrete, no interpolation)
new THREE.BooleanKeyframeTrack(".visible", [0, 0.5, 1], [true, false, true]);
new THREE.StringKeyframeTrack(".name", [0, 1], ["idle", "active"]);
// Morph target influences are numbers: use NumberKeyframeTrack, not StringKeyframeTrack
```

### Interpolation

```javascript
track.setInterpolation(THREE.InterpolateLinear);   // default
track.setInterpolation(THREE.InterpolateSmooth);   // cubic
track.setInterpolation(THREE.InterpolateDiscrete); // step
```

## AnimationMixer

```javascript
const mixer = new THREE.AnimationMixer(model);
const action = mixer.clipAction(clip); // cached: same clip + root returns the same action
action.play();

renderer.setAnimationLoop((t) => {
  timer.update(t);
  mixer.update(timer.getDelta()); // REQUIRED every frame
  renderer.render(scene, camera);
});

mixer.addEventListener("finished", (e) => console.log("finished", e.action.getClip().name)); // LoopOnce only
mixer.addEventListener("loop", (e) => console.log("looped", e.action.getClip().name));

mixer.stopAllAction();
mixer.timeScale = 0.5;      // global slow motion
mixer.setTime(1.2);         // jump all actions
mixer.uncacheRoot(model);   // free memory when removing a character
```

## AnimationAction

```javascript
const action = mixer.clipAction(clip);

action.play();
action.stop();             // stops and resets
action.reset();            // back to start, full weight, not paused
action.halt(0.5);          // decelerate to stop over 0.5 s
action.paused = true;

action.time = 0.5;         // seconds into the clip
action.timeScale = -1;     // reverse
action.setEffectiveTimeScale(2);

action.setLoop(THREE.LoopOnce, 1);       // LoopOnce, LoopRepeat (default), LoopPingPong
action.setLoop(THREE.LoopRepeat, 3);     // repeat 3 times
action.clampWhenFinished = true;         // hold the last frame after LoopOnce

action.setEffectiveWeight(1);            // 0-1 contribution
action.getEffectiveWeight();
action.isRunning();
```

### Fade and Crossfade

```javascript
action.reset().fadeIn(0.5).play();
action.fadeOut(0.5);

// Crossfade: the incoming action must be playing
function crossFade(from, to, duration = 0.3) {
  to.reset().setEffectiveWeight(1).play();
  from.crossFadeTo(to, duration, true); // warp = true syncs time scales (walk <-> run)
}
```

## Loading GLTF Animations

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const gltf = await new GLTFLoader().loadAsync("character.glb");
const model = gltf.scene;
scene.add(model);

const mixer = new THREE.AnimationMixer(model);
console.log(gltf.animations.map((c) => c.name));

const actions = {};
for (const clip of gltf.animations) actions[clip.name] = mixer.clipAction(clip);

const walk = THREE.AnimationClip.findByName(gltf.animations, "Walk");
if (walk) mixer.clipAction(walk).play();

renderer.setAnimationLoop((t) => {
  timer.update(t);
  mixer.update(timer.getDelta());
  renderer.render(scene, camera);
});
```

FBX: `FBXLoader` returns an object with `object.animations`. Mixamo exports often need `object.scale.setScalar(0.01)`.

## Several Copies of an Animated Character

`model.clone()` breaks skinned meshes (clones share the original skeleton). Use `SkeletonUtils.clone`:

```javascript
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";

const soldiers = [];
for (let i = 0; i < 3; i++) {
  const copy = SkeletonUtils.clone(gltf.scene);
  copy.position.x = i * 2;
  scene.add(copy);
  const m = new THREE.AnimationMixer(copy);
  m.clipAction(gltf.animations[0]).play(); // clips are shared, mixers are not
  soldiers.push(m);
}
// in the loop: for (const m of soldiers) m.update(delta);
```

`AnimationObjectGroup` lets one action drive many objects with identical structure.

## Skeletal Animation

```javascript
let skinnedMesh;
model.traverse((o) => { if (o.isSkinnedMesh) skinnedMesh = o; });
const skeleton = skinnedMesh.skeleton;

const head = skeleton.getBoneByName("Head"); // or model.getObjectByName("Head")
scene.add(new THREE.SkeletonHelper(model));
```

### Procedural Bone Control on Top of Clips

The mixer overwrites bone transforms in `mixer.update()`. Apply your changes **after** it:

```javascript
renderer.setAnimationLoop((t) => {
  timer.update(t);
  mixer.update(timer.getDelta());                  // 1. clips
  head.rotation.y += Math.sin(timer.getElapsed()) * 0.3; // 2. additive procedural tweak
  renderer.render(scene, camera);
});
```

Look-at for a bone: compute the target in the bone's parent space, or use the `CCDIKSolver` addon for IK chains.

### Attach Objects to Bones

```javascript
const hand = model.getObjectByName("RightHand");
hand.add(sword);                // follows the hand
sword.position.set(0, 0.1, 0);  // offset in bone space (bone scale applies)
```

### Retargeting

```javascript
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";
// Play a clip made for sourceModel on targetModel (similar rigs, maybe different bone names).
// `names` maps EVERY target bone name to its source bone name (unmapped bones are skipped),
// or pass getBoneName: (targetBone) => sourceBoneName instead.
const names = {};
// example mapping: target bones are "mixamorigHips" etc., source bones are "Hips" etc.
for (const bone of targetSkinnedMesh.skeleton.bones) names[bone.name] = bone.name.replace("mixamorig", "");
const retargeted = SkeletonUtils.retargetClip(targetSkinnedMesh, sourceSkinnedMesh, clip, {
  names,
  hip: "Hips", // SOURCE hip bone name (default "hip")
});
targetMixer.clipAction(retargeted).play();
```

## Morph Targets (Shape Keys)

```javascript
mesh.morphTargetDictionary; // { smile: 0, blink: 1, ... }
mesh.morphTargetInfluences; // [0, 0, ...] weights 0-1

mesh.morphTargetInfluences[mesh.morphTargetDictionary.smile] = 1;

// Keyframed: NumberKeyframeTrack with the name or index
const track = new THREE.NumberKeyframeTrack(".morphTargetInfluences[smile]", [0, 0.5, 1], [0, 1, 0]);
mixer.clipAction(new THREE.AnimationClip("smile", -1, [track])).play();
```

In GLTF files the morph targets live on child meshes; find them with `traverse` and `o.morphTargetInfluences`.

## Blending by Weight (locomotion)

```javascript
const idle = mixer.clipAction(idleClip);
const walk = mixer.clipAction(walkClip);
const run = mixer.clipAction(runClip);
for (const a of [idle, walk, run]) a.play(); // all running, weights decide the mix

function setLocomotion(speed) { // 0..10
  const w = THREE.MathUtils.clamp(speed / 5, 0, 1);
  const r = THREE.MathUtils.clamp((speed - 5) / 5, 0, 1);
  idle.setEffectiveWeight(1 - w);
  walk.setEffectiveWeight(w - r);
  run.setEffectiveWeight(r);
}
```

## Additive Animation

Convert the clip **before** creating its action:

```javascript
THREE.AnimationUtils.makeClipAdditive(breathClip); // relative to its first frame
const base = mixer.clipAction(walkClip);
const additive = mixer.clipAction(breathClip);
additive.blendMode = THREE.AdditiveAnimationBlendMode; // matches the converted clip
base.play();
additive.setEffectiveWeight(0.5).play();
```

## Animation Utilities

```javascript
THREE.AnimationClip.findByName(clips, "Walk");
THREE.AnimationUtils.subclip(clip, "jump", 30, 60, 30); // frames 30-60 at 30 fps
THREE.AnimationUtils.makeClipAdditive(clip, 0, referenceClip);

clip.clone();
clip.duration;
clip.optimize();      // remove redundant keyframes
clip.resetDuration(); // recompute duration from the tracks (after editing them)
clip.trim();          // cut keyframes beyond duration
```

## Procedural Animation Patterns

### Frame-Rate Independent Smoothing

```javascript
// Scalars
value = THREE.MathUtils.damp(value, target, 8, delta); // lambda ~ responsiveness

// Vectors: same exponential formula
const k = 1 - Math.exp(-8 * delta);
mesh.position.lerp(targetPosition, k);
mesh.quaternion.slerp(targetQuaternion, k);
```

### Spring

```javascript
class Spring {
  constructor(stiffness = 120, damping = 12) {
    Object.assign(this, { stiffness, damping, position: 0, velocity: 0, target: 0 });
  }
  update(dt) {
    const force = -this.stiffness * (this.position - this.target) - this.damping * this.velocity;
    this.velocity += force * dt;
    this.position += this.velocity * dt;
    return this.position;
  }
}
```

### Oscillation and Paths

```javascript
const t = timer.getElapsed();
mesh.position.y = Math.sin(t * 2) * 0.5;          // bob
mesh.position.set(Math.cos(t) * 2, 0, Math.sin(t) * 2); // circle

// Follow a curve at constant speed
const curve = new THREE.CatmullRomCurve3(points, true);
const u = (t * 0.05) % 1;
mesh.position.copy(curve.getPointAt(u));
mesh.lookAt(curve.getPointAt((u + 0.01) % 1));
```

### Tweens

Tween.js ships with three: `import TWEEN from "three/addons/libs/tween.module.js";` (call
`TWEEN.update()` in the loop). Or use GSAP, which can tween any `position`/`rotation`/`material` property.

## Performance Tips

1. **Share clips**; give each character its own mixer.
2. **`clip.optimize()`** after loading or generating clips.
3. **Skip mixer updates** for characters that are far away or not visible (do not call `mixer.update`, or update at a lower rate with an accumulated delta).
4. **Remove unused actions**: `mixer.uncacheAction(clip)` / `mixer.uncacheRoot(root)`.
5. **Many identical characters**: GPU skinning is per mesh; for crowds consider baked vertex-animation textures.

## See Also

- `threejs-loaders` - Loading animated GLTF/FBX
- `threejs-fundamentals` - Timer and render loop
- `threejs-shaders` - Vertex animation in shaders
- `threejs-addons` - CCDIKSolver, SkeletonUtils
