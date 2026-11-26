---
name: threejs-webxr
description: Three.js WebXR for VR and AR - enabling renderer.xr, VRButton/ARButton/XRButton, session features, reference spaces and player rigs, controllers (select/squeeze events, controller models, raycasting from controllers), hand tracking, teleportation, AR hit-testing, plane detection, light estimation, DOM overlay, in-VR UI, testing and performance. Use when building VR or AR experiences, handling headset controllers or hands, placing objects on real surfaces, or adapting a Three.js scene for Quest/Vision Pro/phones.
---

# Three.js WebXR (VR / AR)

> three.js **r187**. Needs HTTPS (or localhost) and a WebXR browser (Quest Browser, Chrome on
> Android, Vision Pro Safari, PC VR in Chrome/Edge). WebGLRenderer is the most mature XR path.

## Quick Start

```javascript
import { VRButton } from "three/addons/webxr/VRButton.js";

renderer.xr.enabled = true;
document.body.appendChild(VRButton.createButton(renderer, { optionalFeatures: ["hand-tracking"] }));
camera.position.set(0, 1.6, 3); // only used outside XR

renderer.setAnimationLoop((time, frame) => renderer.render(scene, camera)); // REQUIRED: rAF does not run in XR
```

- Units are meters; the default `local-floor` space puts y = 0 on the real floor.
- The headset overwrites the camera every frame. Move the **player** through a parent rig or
  the reference space, never the camera.
- `ARButton` (immersive-ar), `XRButton` (AR if supported, else VR). For AR: `alpha: true` on
  the renderer and `scene.background = null`.
- Events: `renderer.xr.addEventListener("sessionstart" | "sessionend")`; `renderer.xr.isPresenting`.
- Since r187 the XR camera's matrices come from the first sub-camera (the left eye).

## Rig and Controllers

```javascript
import { XRControllerModelFactory } from "three/addons/webxr/XRControllerModelFactory.js";

const rig = new THREE.Group();
rig.add(camera);
scene.add(rig);

const factory = new XRControllerModelFactory();
for (const i of [0, 1]) {
  const controller = renderer.xr.getController(i);   // target-ray space: for pointing/rays
  const grip = renderer.xr.getControllerGrip(i);     // grip space: for held models
  grip.add(factory.createControllerModel(grip));
  rig.add(controller, grip);                         // children of the rig so they move with the player

  controller.addEventListener("connected", (e) => { controller.userData.source = e.data; }); // handedness, gamepad
  controller.addEventListener("selectstart", onSelectStart); // trigger
  controller.addEventListener("selectend", onSelectEnd);
  controller.addEventListener("squeezestart", onGrab);       // grip button
}

const raycaster = new THREE.Raycaster();
function onSelectStart(e) {
  raycaster.setFromXRController(e.target);
  const hit = raycaster.intersectObjects(grabbables, false)[0];
  if (hit) { e.target.attach(hit.object); e.target.userData.held = hit.object; } // keep world transform
}
function onSelectEnd(e) {
  const obj = e.target.userData.held;
  if (obj) { scene.attach(obj); e.target.userData.held = null; }
}
```

**Thumbsticks:** in the loop read `renderer.xr.getSession()?.inputSources`; with the
`xr-standard` mapping `gamepad.axes[2], axes[3]` are the stick and `buttons[4]` is A/X.

## Hands

```javascript
import { XRHandModelFactory } from "three/addons/webxr/XRHandModelFactory.js";
const hand = renderer.xr.getHand(0);
hand.add(new XRHandModelFactory().createHandModel(hand, "mesh")); // 'spheres' | 'boxes' | 'mesh'
rig.add(hand);
hand.addEventListener("pinchstart", () => {});
hand.joints["index-finger-tip"]?.getWorldPosition(tip);
```

A pinch also fires `selectstart`/`selectend` on `getController(i)`, so controller code mostly works with hands.

## Teleport

```javascript
let baseSpace;
renderer.xr.addEventListener("sessionstart", () => (baseSpace = renderer.xr.getReferenceSpace()));
function teleportTo(p) {
  const t = new XRRigidTransform({ x: -p.x, y: -p.y, z: -p.z, w: 1 }, new THREE.Quaternion());
  renderer.xr.setReferenceSpace(baseSpace.getOffsetReferenceSpace(t));
}
```

## AR Hit Testing

```javascript
document.body.appendChild(ARButton.createButton(renderer, { requiredFeatures: ["hit-test"] }));
reticle.matrixAutoUpdate = false;

let source = null, requested = false;
renderer.setAnimationLoop((t, frame) => {
  if (frame) {
    const session = renderer.xr.getSession();
    if (!requested) {
      requested = true;
      session.requestReferenceSpace("viewer").then((space) =>
        session.requestHitTestSource({ space }).then((s) => (source = s)));
      session.addEventListener("end", () => { requested = false; source = null; });
    }
    const results = source ? frame.getHitTestResults(source) : [];
    reticle.visible = results.length > 0;
    if (reticle.visible) reticle.matrix.fromArray(results[0].getPose(renderer.xr.getReferenceSpace()).transform.matrix);
  }
  renderer.render(scene, camera);
});
// place on tap: controller "select" -> reticle.matrix.decompose(obj.position, obj.quaternion, obj.scale)
```

AR extras: `XRPlanes` (feature `plane-detection`), `XREstimatedLight` (feature
`light-estimation`; add it on `estimationstart`, and use its `environment` if present),
DOM overlay (`optionalFeatures: ["dom-overlay"], domOverlay: { root }`) for phone UI.

## UI in VR

`HTMLMesh` + `InteractiveGroup.listenToXRControllerEvents(controller)` (`threejs-objects`), or
`createText` from `three/addons/webxr/Text2D.js`. Keep UI 1-3 m away and text large.

## Testing

Immersive Web Emulator extension (desktop Chrome/Edge); Quest over USB with `chrome://inspect`;
HTTPS on the LAN (`vite --host` + certificate) or `adb reverse` for localhost.

## Performance

1. Hit the refresh rate (72-120 Hz) per eye: ~100-200 draw calls on standalone headsets.
2. `renderer.xr.setFoveation(1)`; `setFramebufferScaleFactor(0.8)` before the session.
3. No EffectComposer in XR; bake lighting; KTX2 textures; at most one tight directional shadow.
4. Avoid camera motion the user doesn't control; prefer teleport and snap turning.

See also: `threejs-interaction`, `threejs-objects`, `threejs-loaders`.
