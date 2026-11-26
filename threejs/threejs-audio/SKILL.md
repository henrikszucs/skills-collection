---
name: threejs-audio
description: Three.js audio - AudioListener, global Audio and 3D PositionalAudio, AudioLoader, browser autoplay rules, distance models and directional cones, streaming from media elements, AudioAnalyser for music visualizers, filters and cleanup. Use when adding sound effects, background music, spatial/3D sound attached to objects, or audio-reactive visuals.
---

# Three.js Audio

> **Version:** written for three.js **r186**. Audio wraps the Web Audio API and works with both
> `WebGLRenderer` and `WebGPURenderer`.

## Quick Start

```javascript
import * as THREE from "three";

// 1. One listener, usually on the camera (it is the "ears")
const listener = new THREE.AudioListener();
camera.add(listener);

// 2. Load a buffer
const buffer = await new THREE.AudioLoader().loadAsync("sounds/music.mp3");

// 3. Non-positional (same volume everywhere): music, UI sounds
const music = new THREE.Audio(listener);
music.setBuffer(buffer);
music.setLoop(true);
music.setVolume(0.5);

// 4. Browsers block audio until a user gesture: start from a click/tap
startButton.addEventListener("click", () => {
  listener.context.resume(); // unlocks the shared AudioContext
  music.play();
});
```

## Autoplay Rules

- Audio can only start after a user gesture (click, tap, key). Show a "Start"/"Enable sound" button.
- `THREE.AudioContext.getContext()` returns the shared context; `context.state` is `"suspended"` until resumed.
- Create audio objects early, call `play()` inside the gesture handler (or after `context.resume()`).

## PositionalAudio (3D sound)

Volume and panning depend on the position relative to the listener.

```javascript
const engine = new THREE.PositionalAudio(listener);
engine.setBuffer(await new THREE.AudioLoader().loadAsync("sounds/engine.ogg"));
engine.setLoop(true);
engine.setRefDistance(2);      // full volume within ~2 units
engine.setRolloffFactor(1);    // how fast it fades
engine.setDistanceModel("inverse"); // 'linear' | 'inverse' (default) | 'exponential'
engine.setMaxDistance(50);     // used by 'linear'
car.add(engine);               // follows the object
engine.play();

// Directional sound (speaker facing +Z of its parent)
engine.setDirectionalCone(90, 180, 0.2); // innerAngle, outerAngle (degrees), outerGain

import { PositionalAudioHelper } from "three/addons/helpers/PositionalAudioHelper.js";
engine.add(new PositionalAudioHelper(engine, 1));
```

Units are scene units (meters by convention). The listener's position/orientation is updated
from the camera's world matrix every frame automatically.

## Playback Control

```javascript
sound.play(delaySeconds = 0);
sound.pause();
sound.stop();
sound.isPlaying;
sound.setVolume(0.8); sound.getVolume();
sound.setPlaybackRate(1.5);  // pitch + speed
sound.setDetune(100);        // cents
sound.setLoop(true);
sound.setLoopStart(2); sound.setLoopEnd(10);
sound.offset = 3;            // start position (seconds) for the next play()
sound.duration = 5;          // play only 5 seconds

listener.setMasterVolume(0.5); // global volume (mute button: 0)
```

### One-Shot Sound Effects (can overlap)

A single `Audio` restarts when you call `play()` while playing. For overlapping hits, keep a small pool:

```javascript
class SoundPool {
  constructor(listener, buffer, size = 4, positional = false) {
    this.sounds = Array.from({ length: size }, () => {
      const s = positional ? new THREE.PositionalAudio(listener) : new THREE.Audio(listener);
      s.setBuffer(buffer);
      return s;
    });
    this.index = 0;
  }
  play(volume = 1) {
    const s = this.sounds[this.index];
    this.index = (this.index + 1) % this.sounds.length;
    if (s.isPlaying) s.stop();
    s.setVolume(volume);
    s.play();
    return s;
  }
}
```

## Streaming Long Files (media element)

```javascript
const audioElement = document.createElement("audio");
audioElement.src = "sounds/long-track.mp3";
audioElement.loop = true;
audioElement.crossOrigin = "anonymous";

const stream = new THREE.PositionalAudio(listener);
stream.setMediaElementSource(audioElement); // no full decode in memory
speaker.add(stream);

button.addEventListener("click", () => {
  listener.context.resume();
  audioElement.play(); // control playback through the element
});
```

Microphone: `sound.setMediaStreamSource(await navigator.mediaDevices.getUserMedia({ audio: true }))`.
Video soundtrack: `setMediaElementSource(videoElement)` (see VideoTexture in `threejs-textures`).

## Audio-Reactive Visuals (AudioAnalyser)

```javascript
const analyser = new THREE.AudioAnalyser(music, 64); // fftSize (power of two); 32 frequency bins

renderer.setAnimationLoop(() => {
  const avg = analyser.getAverageFrequency();      // 0-255
  const bins = analyser.getFrequencyData();        // Uint8Array(fftSize / 2), 0-255 each
  mesh.scale.setScalar(1 + avg / 255);
  bars.forEach((bar, i) => (bar.scale.y = 0.1 + bins[i] / 64));
  renderer.render(scene, camera);
});
```

Pass the data to a shader as a texture:

```javascript
const fftTexture = new THREE.DataTexture(analyser.data, analyser.data.length, 1, THREE.RedFormat);
// every frame:
analyser.getFrequencyData();
fftTexture.needsUpdate = true;
// GLSL: float level = texture2D(tAudioData, vec2(vUv.x, 0.0)).r;
```

## Filters and Effects

```javascript
const ctx = listener.context;
const lowpass = ctx.createBiquadFilter();
lowpass.type = "lowpass";
lowpass.frequency.value = 800;
sound.setFilter(lowpass);     // or setFilters([f1, f2]) to chain
listener.setFilter(reverb);   // on everything (e.g. a ConvolverNode)

// Underwater effect: tween lowpass.frequency when the camera goes below the surface
```

## Cleanup

```javascript
sound.stop();
sound.disconnect();      // detach from the audio graph
sound.removeFromParent();
// Buffers are freed by garbage collection once nothing references them.
```

## Tips

1. **Formats**: `.mp3` and `.ogg`/`.opus` (Safari supports opus in recent versions; ship mp3 as a fallback); `.wav` for short sounds.
2. **Decode once**: reuse `AudioBuffer`s across many `Audio` objects.
3. **Large tracks**: stream through a media element instead of `AudioLoader`.
4. **Mobile**: resume the context on the first touch; iOS silent mode can mute Web Audio.
5. **Mixing**: keep master volume below 1 when many sounds overlap to avoid clipping.

## See Also

- `threejs-fundamentals` - Camera setup (listener attaches to it)
- `threejs-loaders` - LoadingManager progress including audio files
- `threejs-shaders` - Using frequency data in shaders
