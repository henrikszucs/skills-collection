---
name: threejs-audio
description: Three.js audio - AudioListener, global Audio and 3D PositionalAudio, AudioLoader, browser autoplay rules, distance models and directional cones, streaming from media elements, AudioAnalyser for music visualizers, filters and cleanup. Use when adding sound effects, background music, spatial/3D sound attached to objects, or audio-reactive visuals.
---

# Three.js Audio

> three.js **r187**. Wraps the Web Audio API; works with WebGL and WebGPU renderers.

## Quick Start

```javascript
const listener = new THREE.AudioListener();
camera.add(listener); // the "ears"; follows the camera automatically

const buffer = await new THREE.AudioLoader().loadAsync("music.mp3");
const music = new THREE.Audio(listener); // non-positional: music, UI
music.setBuffer(buffer).setLoop(true).setVolume(0.5);

startButton.addEventListener("click", () => { // browsers block audio until a user gesture
  listener.context.resume();
  music.play();
});
```

Show a "Start" / "Enable sound" button. The shared context (`THREE.AudioContext.getContext()`)
stays `"suspended"` until resumed from a gesture. iOS silent mode can mute Web Audio.

## 3D Sound (PositionalAudio)

```javascript
const engine = new THREE.PositionalAudio(listener);
engine.setBuffer(buffer);
engine.setLoop(true);
engine.setRefDistance(2);             // full volume within ~2 m
engine.setRolloffFactor(1);
engine.setDistanceModel("inverse");   // 'linear' (uses setMaxDistance) | 'inverse' | 'exponential'
engine.setDirectionalCone(90, 180, 0.2); // inner, outer (degrees), outer gain; faces parent +Z
car.add(engine);                      // follows the object
engine.play();
// debug: engine.add(new PositionalAudioHelper(engine, 1)) from three/addons/helpers/
```

## Playback

`play(delay)`, `pause()`, `stop()`, `isPlaying`, `setVolume`, `setPlaybackRate` (pitch + speed),
`setDetune(cents)`, `setLoopStart/End`, `offset` (start position for the next play),
`duration`. Global volume/mute: `listener.setMasterVolume(0)`.

**Overlapping one-shots:** one `Audio` restarts on `play()`. Keep a small round-robin pool of
`Audio`/`PositionalAudio` objects that share the same buffer.

## Streaming, Mic, Video

```javascript
const el = Object.assign(document.createElement("audio"), { src: "long.mp3", loop: true, crossOrigin: "anonymous" });
const stream = new THREE.PositionalAudio(listener);
stream.setMediaElementSource(el);   // no full decode in memory; control playback via el.play()
// microphone: setMediaStreamSource(await navigator.mediaDevices.getUserMedia({ audio: true }))
// video soundtrack: setMediaElementSource(videoElement)
```

## Audio-Reactive Visuals

```javascript
const analyser = new THREE.AudioAnalyser(music, 64); // fftSize; 32 bins
// loop:
const avg = analyser.getAverageFrequency(); // 0-255
const bins = analyser.getFrequencyData();   // Uint8Array(fftSize / 2)

// To a shader:
const fft = new THREE.DataTexture(analyser.data, analyser.data.length, 1, THREE.RedFormat);
// each frame: analyser.getFrequencyData(); fft.needsUpdate = true;
```

## Filters

```javascript
const lowpass = listener.context.createBiquadFilter();
lowpass.type = "lowpass"; lowpass.frequency.value = 800;
sound.setFilter(lowpass);   // setFilters([...]) to chain
listener.setFilter(reverb); // everything (e.g. a ConvolverNode)
```

## Cleanup and Tips

- `sound.stop(); sound.disconnect(); sound.removeFromParent();` Buffers are garbage-collected.
- Formats: `.mp3` everywhere, `.ogg`/`.opus` smaller (ship an mp3 fallback), `.wav` for short sounds.
- Decode once and reuse `AudioBuffer`s. Stream long tracks. Keep master volume below 1 when many sounds overlap.

See also: `threejs-fundamentals`, `threejs-loaders`, `threejs-shaders`.
