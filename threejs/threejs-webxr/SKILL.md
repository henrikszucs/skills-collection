---
name: threejs-webxr
description: Three.js WebXR for VR and AR - enabling renderer.xr, VRButton/ARButton/XRButton, session features, reference spaces and player rigs, controllers (select/squeeze events, controller models, raycasting from controllers), hand tracking, teleportation, AR hit-testing, plane detection, light estimation, DOM overlay, in-VR UI, testing and performance. Use when building VR or AR experiences, handling headset controllers or hands, placing objects on real surfaces, or adapting a Three.js scene for Quest/Vision Pro/phones.
---

# Three.js WebXR (VR / AR)

> **Version:** written for three.js **r186**. Requires HTTPS (or localhost) and a WebXR browser
> (Meta Quest Browser, Chrome on Android, Vision Pro Safari, PC VR via Chrome/Edge).

## Quick Start (VR)

```javascript
import * as THREE from "three";
import { VRButton } from "three/addons/webxr/VRButton.js";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.xr.enabled = true;
document.body.appendChild(renderer.domElement);
document.body.appendChild(VRButton.createButton(renderer)); // "Enter VR" button

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(0, 1.6, 3); // used outside VR only; in VR the headset drives the camera

// REQUIRED: setAnimationLoop (window.requestAnimationFrame does not run in an XR session)
renderer.setAnimationLoop((time, frame) => {
  renderer.render(scene, camera);
});
```

- Units are **meters**; the default reference space `local-floor` puts y = 0 on the real floor.
- In XR the camera's position/rotation are overwritten every frame. Move the **player** by moving a parent group (rig) or the reference space (teleport), not the camera.
- `WebGPURenderer` also has `renderer.xr`, but VR/AR with WebGLRenderer is the most mature path.

## Buttons and Session Features

```javascript
import { VRButton } from "three/addons/webxr/VRButton.js"; // immersive-vr
import { ARButton } from "three/addons/webxr/ARButton.js"; // immersive-ar
import { XRButton } from "three/addons/webxr/XRButton.js"; // AR if supported, else VR (passthrough headsets)

document.body.appendChild(VRButton.createButton(renderer, {
  optionalFeatures: ["hand-tracking", "layers"],
}));

document.body.appendChild(ARButton.createButton(renderer, {
  requiredFeatures: ["hit-test"],
  optionalFeatures: ["dom-overlay", "plane-detection", "anchors", "light-estimation"],
  domOverlay: { root: document.getElementById("overlay") },
}));

renderer.xr.addEventListener("sessionstart", () => {});
renderer.xr.addEventListener("sessionend", () => {});
renderer.xr.isPresenting; // true while in XR
renderer.xr.setReferenceSpaceType("local-floor"); // before the session: 'local', 'local-floor', 'bounded-floor', 'unbounded'
```

For AR, create the renderer with `alpha: true` and leave `scene.background` null so the camera feed shows through.

## Player Rig (move the user)

```javascript
const rig = new THREE.Group();
rig.add(camera);
scene.add(rig);

// Controllers and hands must be children of the same rig so they move with the player
rig.add(controller1, controller2, grip1, grip2);

// Smooth locomotion from thumbstick input (see Gamepad below)
rig.position.addScaledVector(forwardDirection, speed * dt);
```

## Controllers

```javascript
import { XRControllerModelFactory } from "three/addons/webxr/XRControllerModelFactory.js";

// Target-ray space: where the controller points (use for rays/lasers)
const controller1 = renderer.xr.getController(0);
const controller2 = renderer.xr.getController(1);
// Grip space: where the hand holds it (use for models/held objects)
const grip1 = renderer.xr.getControllerGrip(0);
const grip2 = renderer.xr.getControllerGrip(1);

const modelFactory = new XRControllerModelFactory(); // loads the right model for the device
grip1.add(modelFactory.createControllerModel(grip1));
grip2.add(modelFactory.createControllerModel(grip2));
scene.add(controller1, controller2, grip1, grip2); // or rig.add(...)

for (const controller of [controller1, controller2]) {
  controller.addEventListener("connected", (e) => {
    controller.userData.handedness = e.data.handedness; // 'left' | 'right'
    controller.userData.inputSource = e.data;            // has .gamepad
    controller.add(makeLaser());
  });
  controller.addEventListener("disconnected", () => controller.clear());
  controller.addEventListener("selectstart", onSelectStart); // trigger down
  controller.addEventListener("selectend", onSelectEnd);
  controller.addEventListener("select", onSelect);           // full click
  controller.addEventListener("squeezestart", onGrab);       // grip button
  controller.addEventListener("squeezeend", onRelease);
}

function makeLaser() {
  const geometry = new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(0, 0, 0), new THREE.Vector3(0, 0, -1)]);
  const line = new THREE.Line(geometry, new THREE.LineBasicMaterial({ color: 0xffffff }));
  line.scale.z = 5;
  return line;
}
```

### Raycasting From a Controller

```javascript
const raycaster = new THREE.Raycaster();

function getIntersections(controller, objects) {
  controller.updateMatrixWorld();
  raycaster.setFromXRController(controller); // origin + -Z direction of the target ray
  return raycaster.intersectObjects(objects, false);
}

function onSelectStart(event) {
  const controller = event.target;
  const hit = getIntersections(controller, grabbables)[0];
  if (hit) {
    controller.attach(hit.object);           // pick up, keeping the world transform
    controller.userData.selected = hit.object;
  }
}
function onSelectEnd(event) {
  const controller = event.target;
  const obj = controller.userData.selected;
  if (obj) {
    scene.attach(obj);                       // drop
    controller.userData.selected = undefined;
  }
}
```

### Gamepad (thumbsticks, buttons)

```javascript
renderer.setAnimationLoop((t, frame) => {
  const session = renderer.xr.getSession();
  if (session) {
    for (const source of session.inputSources) {
      const gp = source.gamepad;
      if (!gp) continue;
      const [, , x, y] = gp.axes;            // xr-standard mapping: axes[2], axes[3] = thumbstick
      if (source.handedness === "left") moveRig(x, y);
      if (gp.buttons[4]?.pressed) jump();    // A/X button
    }
  }
  renderer.render(scene, camera);
});
```

## Hand Tracking

```javascript
import { XRHandModelFactory } from "three/addons/webxr/XRHandModelFactory.js";
// Request 'hand-tracking' in the session features

const handFactory = new XRHandModelFactory();
const hand1 = renderer.xr.getHand(0);
hand1.add(handFactory.createHandModel(hand1, "mesh")); // 'spheres' | 'boxes' | 'mesh'
scene.add(hand1);

hand1.addEventListener("pinchstart", () => {});
hand1.addEventListener("pinchend", () => {});

// Joint positions (XRHand joint names)
const indexTip = hand1.joints["index-finger-tip"];
if (indexTip) indexTip.getWorldPosition(tmpVec);
```

Pinch on hands also fires `selectstart`/`selectend` on `getController(i)`, so controller code usually works with hands too.

## Teleportation (official technique)

```javascript
let baseReferenceSpace;
renderer.xr.addEventListener("sessionstart", () => {
  baseReferenceSpace = renderer.xr.getReferenceSpace();
});

function teleportTo(point) { // point: Vector3 on the floor from a controller raycast
  const offsetPosition = { x: -point.x, y: -point.y, z: -point.z, w: 1 };
  const transform = new XRRigidTransform(offsetPosition, new THREE.Quaternion());
  renderer.xr.setReferenceSpace(baseReferenceSpace.getOffsetReferenceSpace(transform));
}
```

## AR: Hit Testing (place objects on real surfaces)

```javascript
import { ARButton } from "three/addons/webxr/ARButton.js";

const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.xr.enabled = true;
document.body.appendChild(ARButton.createButton(renderer, { requiredFeatures: ["hit-test"] }));

const reticle = new THREE.Mesh(
  new THREE.RingGeometry(0.15, 0.2, 32).rotateX(-Math.PI / 2),
  new THREE.MeshBasicMaterial(),
);
reticle.matrixAutoUpdate = false;
reticle.visible = false;
scene.add(reticle);

const controller = renderer.xr.getController(0); // screen tap on phones
controller.addEventListener("select", () => {
  if (!reticle.visible) return;
  const obj = model.clone();
  reticle.matrix.decompose(obj.position, obj.quaternion, obj.scale);
  scene.add(obj);
});
scene.add(controller);

let hitTestSource = null;
let hitTestSourceRequested = false;

renderer.setAnimationLoop((timestamp, frame) => {
  if (frame) {
    const referenceSpace = renderer.xr.getReferenceSpace();
    const session = renderer.xr.getSession();
    if (!hitTestSourceRequested) {
      session.requestReferenceSpace("viewer").then((viewerSpace) => {
        session.requestHitTestSource({ space: viewerSpace }).then((source) => (hitTestSource = source));
      });
      session.addEventListener("end", () => {
        hitTestSourceRequested = false;
        hitTestSource = null;
      });
      hitTestSourceRequested = true;
    }
    if (hitTestSource) {
      const results = frame.getHitTestResults(hitTestSource);
      reticle.visible = results.length > 0;
      if (results.length) reticle.matrix.fromArray(results[0].getPose(referenceSpace).transform.matrix);
    }
  }
  renderer.render(scene, camera);
});
```

## AR Extras

```javascript
// Detected planes visualized as meshes (feature 'plane-detection')
import { XRPlanes } from "three/addons/webxr/XRPlanes.js";
scene.add(new XRPlanes(renderer));

// Real-world lighting estimate (feature 'light-estimation')
import { XREstimatedLight } from "three/addons/webxr/XREstimatedLight.js";
const xrLight = new XREstimatedLight(renderer);
xrLight.addEventListener("estimationstart", () => {
  scene.add(xrLight);
  if (xrLight.environment) scene.environment = xrLight.environment;
});
xrLight.addEventListener("estimationend", () => scene.remove(xrLight));
```

DOM overlay: HTML in `domOverlay.root` stays visible on top of the AR view (phones). Use it for buttons/menus.

## UI in VR

- `HTMLMesh` + `InteractiveGroup` render DOM (including lil-gui) onto a plane and route controller
  rays to it: `group.listenToXRControllerEvents(controller1)` (see `threejs-objects`).
- `Text2D` helper: `import { createText } from "three/addons/webxr/Text2D.js"` for simple text planes.
- Keep text large (at least ~1-2 cm per meter of distance) and UI within ~1-3 m.

## Testing Without a Headset

- **Immersive Web Emulator** browser extension (Meta) emulates Quest controllers in desktop Chrome/Edge.
- Quest: connect via USB and use `chrome://inspect` to debug the headset browser.
- Serve over HTTPS on the LAN (e.g. `vite --host` + a dev certificate) or use `adb reverse` for localhost.

## Performance

1. **Hit the refresh rate** (72-120 Hz) per eye: budget ~100-200 draw calls on standalone headsets.
2. **Foveation**: `renderer.xr.setFoveation(1)` (0-1, higher = more peripheral reduction).
3. **Framebuffer scale**: `renderer.xr.setFramebufferScaleFactor(0.8)` before the session to trade sharpness for speed.
4. **Avoid heavy post-processing** in XR (EffectComposer is not XR-friendly); bake lighting; use KTX2 textures.
5. **No camera-moving animations** the user does not control (motion sickness); prefer teleport or snap turning.
6. Shadows: one directional light with a small, tight shadow frustum at most.

## See Also

- `threejs-fundamentals` - setAnimationLoop and renderer setup
- `threejs-interaction` - Raycasting basics
- `threejs-objects` - HTMLMesh / InteractiveGroup UI
- `threejs-loaders` - Compressed assets for standalone headsets
