# Procedural Nature Synth — Developer Handoff Notes

Project goal: a single-file (or small bundle) HTML/JS Web Audio app that generates calming “natural soundscapes” **procedurally** (no samples by default), using **modules + macro knobs + presets**, with smooth transitions and a mobile-friendly UI (iOS Safari).

This handoff references the current baseline implementation (Noise AudioWorklet + modules: Rain, Wind, Ocean, Fire, Crickets, Stream) and outlines how to extend it to additional sounds and features discussed.

---

## 1. High-level Architecture

### 1.1 Audio graph
- One `AudioContext`
- Common master chain:
  - `modulesBus` → `dryGain` → `compressor` → `masterLP` → `masterGain` → `destination`
  - `modulesBus` → `wetGain` → `convolver` → `compressor` → ...
- `masterLP` used as a global distance/air absorption “damping” control.

### 1.2 Modules
Each sound type is a **module** that:
- `start(destinationNode)`
- `stop()`
- `set(macros)` where `macros = { intensity, density, brightness, variation, distance, master, reverb }`

Guideline:
- Keep module internals self-contained.
- Use smoothing (`setTargetAtTime`) for all AudioParams to avoid clicks.
- Use *event generation* via timers for discrete sounds (drops, crackles, chirps, drips).
- Use *continuous noise + filters + slow modulation* for beds (wind, ocean body, steady rain hiss).

### 1.3 Shared building blocks
Current:
- `AudioWorkletNode` `noise-gen` supports types:
  - `0=white`, `1=pink approx`, `2=brown-ish`
- Utilities:
  - `lerp`, `expRand`, `setParam`, etc.
Add / improve:
- Stereo: use 2-channel buses or add a “stereo widener” (dual filters with slight decorrelation).
- Global random LFO: a simple “smooth random” generator per module.

---

## 2. Preset System Plan

### 2.1 Presets format
```js
{
  name: "Gentle Rain on Leaves",
  modules: { rain:true, wind:true, ocean:false, ... },
  p: { master:0.33, intensity:0.40, density:0.55, brightness:0.55, variation:0.35, distance:0.28, reverb:0.14 }
}
```

### 2.2 Future preset expansion
- Expand categories beyond current: `Water`, `Wind`, `Night`, `Forest`, `Fire`, `Snow`, `Cave`, `Minimal`, etc.
- “Top 5 per category” is a good initial target. Use a consistent loudness baseline across presets.

### 2.3 User presets (feature)
- Add “Save Preset” button to store current toggles + sliders to `localStorage`.
- Add “Export/Import” (JSON text area) to move presets across devices.

---

## 3. Macro knobs mapping (intended behavior)

### 3.1 Meaning
- **Intensity**: overall energy of module (often gain + sometimes bandwidth)
- **Density**: event rate (drops/crackles/chirps/bubbles) and/or swell rate
- **Brightness**: filter tilt (LP/BP center/Q) and “sparkle”
- **Variation**: how alive/animated (mod depth, gustiness, drift speed)
- **Distance**: attenuate highs + increase reverb + reduce transient sharpness

### 3.2 Guidelines
- Each module should interpret macros slightly differently for natural results.
- Avoid direct linear mapping everywhere; use gentle curves:
  - Example: `effective = Math.pow(x, 1.4)` for density, or `sqrt` for gain.

---

## 4. Feature Roadmap

### 4.1 Immediate improvements (short)
1. **Crossfade improvements**
   - Right now: delayed stop + param smoothing.
   - Better: per-module gain “fade node” so enabling/disabling is click-free and truly crossfaded.
2. **Wake Lock**
   - Optional: `navigator.wakeLock.request('screen')` during playback.
   - Handle `visibilitychange` to re-request.
3. **Sleep timer**
   - 15/30/60/90 min, fade out over last 10–20 seconds.
4. **Responsive UI**
   - One-column layout on small screens.
   - Big play/stop buttons for iPhone.

### 4.2 Medium features
1. **Mixer view**
   - Per-module volume sliders (in addition to macro knobs).
2. **Stereo width**
   - Simple approach: split module output into L/R with tiny filter diffs + delays.
3. **Auto-level**
   - Apply soft-knee compression or a limiter to avoid peaks.
4. **Preset morphing**
   - Interpolate between two presets over N seconds.

### 4.3 Advanced features
1. **Scene builder**
   - Multi-layer scenes (e.g., Rain + Wind + Distant Thunder + Stream).
2. **Sequencing**
   - Slowly shift over time: “sunset → night insects”.
3. **Performance**
   - Move event scheduling off `setInterval` into `AudioContext`-time scheduled envelopes and a single scheduler loop.

---

## 5. New Sound Modules to Add (Procedural Recipes)

Below are recommended models to cover “as many as possible” from earlier categories while staying procedural.

> Implementation note: For each module, prefer:
> - A **bed** (continuous) component and optionally
> - **events** (discrete) component(s)

### 5.1 Water: Drizzle / Mist
- Bed: pink noise with LP (6–10 kHz), low gain
- Micro-drops: higher BP (4–9 kHz) with tiny envelopes at low density

### 5.2 Water: Heavy rain / Storm rain (no thunder yet)
- Bed: stronger white/pink mix, LP lower (3–7 kHz)
- Drops: more frequent + higher peaks but compressed/limited
- Add “roof” variant: comb-ish resonances:
  - Use 2–3 narrow BPs around 600–2k and 3–5k mixed subtly

### 5.3 Water: Drips (cave / gutter / leaf drip)
- Event-only module:
  - Random Poisson times: `expRand(meanInterval)`
  - Each event: resonant BP or sine partial + exponential decay
  - Optional pitch jitter by choosing random resonant frequencies per event
- Optional “splash” tail: short noise burst lowpassed.

### 5.4 Water: Lake lapping / shoreline
- Brown noise → LP (100–400 Hz) for body
- “Lap” envelope: slow quasi-periodic amplitude envelope
- Add higher band for “edge fizz”: pink noise BP (500–1500 Hz) with same envelope

### 5.5 Water: Waterfall (distant vs close)
- Mostly continuous:
  - Pink/brown noise + wide BP (200–2500 Hz)
  - Distant: lower highs, more reverb, less transient events
  - Close: more high band (2–8 kHz) with subtle modulation

### 5.6 Wind: “Trees” vs “Grass” vs “Canyon”
- **Trees rustle** layer: BP noise (600–4k) with bursty envelope driven by gusts
- **Grass**: slightly higher center (1–6k), faster bursts, lower body
- **Canyon**: add subtle resonances (narrow BPs 120–350 Hz) + longer reverb

### 5.7 Forest ambience / Leaves
- Base: pink noise BP 700–5k
- Bursty “rustle events”:
  - Envelope 80–250 ms, Q moderate, random center 1–4k
- Variation increases burst density and drift speed.

### 5.8 Insects: Cicadas / Katydids
- Cicadas: continuous narrow BP (2–6k) with slow amplitude flutter
- Katydids: event chirps like crickets but longer and lower (2–4k) with irregular spacing
- Temperature knob idea (optional): tie chirp rate to a “temp” slider.

### 5.9 Frogs (pond ambience)
- Discrete “croak” events:
  - Low sine/triangle (150–400 Hz) with pitch drop over 150–400 ms
  - Add a soft noise component for throat texture
- Distant: more reverb + reduced highs.

### 5.10 Thunder (distant)
- Infrequent low-frequency events:
  - Brown noise + LP (20–120 Hz) + long envelope (1–6 s)
  - Add mid band rumble (100–500 Hz) slight delay/echo
- Must be subtle to remain calming. Density controls how often.

### 5.11 Snow / winter hush
- Continuous “near silence” bed:
  - Very low-level pink noise, strong LP (1–3 kHz)
  - Occasional faint wind gust events
- Optional “ice ticks”:
  - Very sparse high BP ticks, extremely low volume.

### 5.12 Cave ambience
- Low drone: brown noise LP (50–200 Hz) + slow movement
- Drips module layered (see 5.3)
- Reverb preset: longer IR.

### 5.13 Minimal / “near silence”
- Use only a tiny noise floor:
  - Pink noise, LP 1–2 kHz, extremely low gain
- Add ultra-slow gain mod (avoid dead flatness).

### 5.14 Birds (procedural-only policy)
- Procedural birds are hardest to make convincing without samples.
- Recommend “simple chirps” for ambience:
  - Oscillator (sine/triangle) with pitch glide (e.g., 1200 → 2400 Hz)
  - Envelope 50–250 ms
  - Random inter-chirp time 2–12 s (very sparse)
- Include “Birds (Minimal)” module for occasional chirps, not a full chorus.

---

## 6. Implementation Guidelines (Audio)

### 6.1 Avoid clicks / zipper noise
- Always ramp AudioParams:
  - `setTargetAtTime` or `linearRampToValueAtTime`
- For event gain envelopes:
  - Start at tiny nonzero (`0.0001`) for exponential ramps.

### 6.2 Scheduling (current vs future)
Current approach: multiple `setInterval` timers.
- OK for MVP; simplify later.

Future scheduler improvement:
- Single global scheduler loop that plans events ahead in AudioContext time.
- Keep a queue for each module.

### 6.3 Performance on iPhone
- Minimize node counts.
- Prefer one shared noise source per module rather than many.
- Keep reverb impulse small; offer a toggle for low-power mode (disable convolver).

### 6.4 Loudness normalization
- Keep module defaults conservative.
- Consider adding a final limiter:
  - `DynamicsCompressor` (already present) plus a `GainNode` ceiling.

---

## 7. UI / UX Enhancements

### 7.1 Controls
- Keep current macro sliders; add optional “Advanced” accordion:
  - per-module volume
  - per-module “character” knob (e.g., rain drop size, wind tree/grass mix)
- Add “Scene” chips: toggling multiple modules quickly.

### 7.2 Smooth transitions
- Implement per-module `fadeGain`:
  - Module internal output → `fadeGain` → bus
- On enable:
  - start module at gain=0 then ramp up to 1
- On disable:
  - ramp down then stop+disconnect

### 7.3 Persistence
- Save UI state to `localStorage` every change:
  - last category/preset, macro values, toggles.

---

## 8. Known Issues / Suggested Fixes

1. **Module stop is coarse**
   - Currently uses delayed stop; can still feel abrupt in some cases.
   - Fix: per-module fadeGain (see 7.2).

2. **Reverb is static**
   - Convolver IR rebuilt only on start; can rebuild when distance/reverb changes, but don’t do it too often.
   - Option: 2–3 precomputed IR buffers (small/medium/large) and crossfade wet gain.

3. **No stereo imaging**
   - Add a simple stereo decorrelation stage:
     - Split into L/R with tiny random delay (0–15 ms) and slight filter differences.

---

## 9. Coding Conventions

- Prefer ES6 classes for modules.
- All module nodes created in `start()` and disconnected in `stop()`.
- Keep `set()` lightweight; avoid allocating nodes there.
- Centralize macro readouts and label updates.

---

## 10. Testing Checklist

### 10.1 Functionality
- Play/Stop works on iOS Safari (must be a tap).
- Switching presets while playing produces no loud clicks.
- Enabling/disabling modules is smooth (after fadeGain is added).

### 10.2 Audio quality
- No audible DC offsets or “pumping” from compressor.
- No harsh resonances at default settings.
- Presets have roughly consistent perceived loudness.

### 10.3 Performance
- CPU stable on iPhone (no dropouts).
- Battery acceptable with reverb on/off modes.

---

## 11. Suggested Next PR Breakdown

1. **PR1: Per-module fadeGain + clean toggles**
2. **PR2: Add Drips + Leaves/Rustle modules + presets**
3. **PR3: Add Thunder (distant) + Cave + Snow**
4. **PR4: User presets (save/export) + sleep timer + wake lock**
5. **PR5: Stereo width + mixer advanced panel**

---

## 12. Quick Notes on “How to model” new sounds
Rule of thumb:
- If the sound is “continuous” → start with **noise color + filter + slow modulation**
- If the sound has “events” → add **randomly-timed bursts** shaped by envelopes + resonant filters
- If it’s “distance” → reduce highs, reduce transients, increase reverb, slow variation
