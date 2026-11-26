---
name: threejs-animation
description: Three.js animation - render loop timing with Timer, AnimationMixer/AnimationClip/AnimationAction, keyframe tracks, playing GLTF animations, crossfading and blending, additive animation, skeletal animation and bones, morph targets, cloning animated characters, retargeting and procedural motion. Use when animating objects, playing or blending GLTF/FBX animations, controlling bones or shape keys, or creating procedural/spring motion.
---

# Three.js Animation

> three.js **r187**. Use `THREE.Timer`, not the deprecated `Clock` (r183).

## Timing

Per frame: `timer.update(t)`, then `const dt = timer.getDelta()` (seconds; `THREE.Timer` loop setup
in `threejs-fundamentals`). Always scale motion by `dt`: displays run at 60-144 Hz.

## Playing Clips (GLTF/FBX)

```javascript
const mixer = new THREE.AnimationMixer(gltf.scene);
const actions = Object.fromEntries(gltf.animations.map((c) => [c.name, mixer.clipAction(c)]));
actions.Walk.play();

// loop: mixer.update(dt)  <- REQUIRED every frame
mixer.addEventListener("finished", (e) => {}); // LoopOnce actions only
```

Action control:

```javascript
action.setLoop(THREE.LoopOnce, 1);  // LoopRepeat (default), LoopPingPong
action.clampWhenFinished = true;    // hold the last frame
action.timeScale = -1;              // reverse; mixer.timeScale for global slow motion
action.setEffectiveWeight(0.5);
action.reset().fadeIn(0.3).play();
action.fadeOut(0.3);

function crossFade(from, to, duration = 0.3) {
  to.reset().setEffectiveWeight(1).play(); // incoming action must be playing
  from.crossFadeTo(to, duration, true);    // warp syncs time scales (walk <-> run)
}
```

**Locomotion blend:** play idle/walk/run together and set their weights from speed.
**Additive:** `THREE.AnimationUtils.makeClipAdditive(clip)` **before** creating the action, then
`action.blendMode = THREE.AdditiveAnimationBlendMode`.

## Keyframe Clips in Code

```javascript
const clip = new THREE.AnimationClip("bounce", -1, [ // -1 = duration from tracks
  new THREE.VectorKeyframeTrack(".position", [0, 1, 2], [0,0,0, 0,2,0, 0,0,0]),
  new THREE.QuaternionKeyframeTrack(".quaternion", [0, 1], [...q1.toArray(), ...q2.toArray()]), // not Euler
  new THREE.NumberKeyframeTrack(".material.opacity", [0, 2], [1, 0]),
  new THREE.ColorKeyframeTrack(".material.color", [0, 2], [1,0,0, 0,0,1]),
  new THREE.BooleanKeyframeTrack(".visible", [0, 1], [true, false]),
]);
```

Track paths resolve from the mixer root: `.position[y]`, `Head.quaternion` (child named Head),
`.morphTargetInfluences[smile]`. Interpolation: `track.setInterpolation(THREE.InterpolateSmooth | InterpolateDiscrete)`.

Utilities: `AnimationClip.findByName(clips, "Walk")`, `AnimationUtils.subclip(clip, name, startFrame, endFrame, fps)`,
`clip.optimize()`, `clip.resetDuration()`, `clip.trim()`.

## Characters

- **Several copies:** `SkeletonUtils.clone(gltf.scene)` (plain `clone()` shares the skeleton).
  Clips are shareable; give each copy its own mixer. `AnimationObjectGroup` lets one action drive
  identical objects.
- **Procedural bones:** `mixer.update()` overwrites bones, so apply your tweaks **after** it.
- **Attach props:** `model.getObjectByName("RightHand").add(sword)` (offset is in bone space).
- **IK:** `CCDIKSolver` (`threejs-addons`), updated after the mixer.
- **Retarget:**

```javascript
import * as SkeletonUtils from "three/addons/utils/SkeletonUtils.js";
const names = {}; // EVERY target bone -> source bone name (unmapped bones are skipped)
for (const b of target.skeleton.bones) names[b.name] = b.name.replace("mixamorig", "");
const clip2 = SkeletonUtils.retargetClip(target, source, clip, { names, hip: "Hips" }); // hip = SOURCE name
```

- **Free memory:** `mixer.uncacheRoot(model)` when removing a character.

## Morph Targets

```javascript
mesh.morphTargetInfluences[mesh.morphTargetDictionary.smile] = 1;
// keyframed: NumberKeyframeTrack(".morphTargetInfluences[smile]", [0, 0.5, 1], [0, 1, 0])
```

In glTF, morph targets live on child meshes: find them with `traverse`.

## Procedural Motion

```javascript
value = THREE.MathUtils.damp(value, target, 8, dt);   // frame-rate independent smoothing
const k = 1 - Math.exp(-8 * dt);
mesh.position.lerp(targetPos, k); mesh.quaternion.slerp(targetQuat, k);

const u = (elapsed * 0.05) % 1;                       // follow a curve at constant speed
mesh.position.copy(curve.getPointAt(u));
mesh.lookAt(curve.getPointAt((u + 0.01) % 1));
```

Springs: integrate `velocity += (-k * (x - target) - c * velocity) * dt`. Tweens:
`three/addons/libs/tween.module.js` (`TWEEN.update()` in the loop) or GSAP.

## Performance

1. Share clips; one mixer per character.
2. Skip or throttle `mixer.update` for off-screen or distant characters.
3. `clip.optimize()`; `uncacheAction` / `uncacheRoot` when done.
4. Big crowds: baked vertex-animation textures instead of per-mesh skinning.

See also: `threejs-loaders`, `threejs-addons` (SkeletonUtils, CCDIKSolver), `threejs-shaders`.
