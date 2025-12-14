# General-Purpose Procedural Synth Platform (HTML/JS) — Plan

Goal: Build a **general-purpose, modular Web Audio synthesizer** that uses **familiar, standard synth techniques** (oscillators, noise sources, filters, envelopes, LFOs, modulators, effects) as a reusable backend platform for designing **procedural natural soundscapes** (rain, wind, ocean, stream, fire, insects, frogs, thunder, snow, cave, forest, minimal ambience, etc.).  
The platform should be easy to tweak manually and also easy for AI agents to adjust via **structured patch JSON** and well-defined APIs.

This plan assumes:
- **Web Audio API** (AudioWorklet for noise + optional DSP)
- **Single-page HTML/JS** (no framework required, but can be added later)
- Procedural-first: no samples required, but allow optional samples later.
- **Deployment**: Cloudflare Pages for static hosting with global CDN
- **Theme**: Consistent dark theme matching other apps in the project

---

## 0) Current State & Migration Path

### 0.1 Existing Implementation
The current `nature-sounds.html` provides:
- ✅ Working procedural nature sounds (Rain, Wind, Ocean, Fire, Stream, Crickets)
- ✅ Class-based module architecture (each sound is a self-contained class)
- ✅ Macro controls (Intensity, Density, Brightness, Variation, Distance)
- ✅ Preset system with categories
- ✅ AudioWorklet noise generation with ScriptProcessorNode fallback
- ✅ Dark theme UI matching project style
- ✅ Wake Lock API integration
- ✅ CORS-safe AudioWorklet loading (data URL approach)

### 0.2 Migration Strategy
**Phase 0.5: Refactor Current Modules to Patch Format**
1. Create patch JSON representations of existing modules
2. Build `AudioEngine` that can load both:
   - Legacy class-based modules (backward compatibility)
   - New patch-based modules
3. Gradually convert modules one-by-one to patch format
4. Maintain existing UI during transition

**Mapping Current Classes → Patches:**
- `Rain` class → Rain patch JSON (noise + filters + event clock)
- `Wind` class → Wind patch JSON (noise + LFO modulation)
- `Ocean` class → Ocean patch JSON (dual noise sources + swell envelope)
- etc.

**Backward Compatibility:**
- Keep existing class constructors as "patch compilers"
- `class Rain { ... }` → `Rain.toPatch()` returns patch JSON
- Allows gradual migration without breaking existing code

---

## 1) Product Requirements

### 1.1 Must-have capabilities
1. **Modular signal graph**
   - Build synth patches from nodes: sources → modifiers → effects → mixers → output.
2. **Standard synth controls**
   - Gain, pan, filter type/cutoff/Q, ADSR envelopes, LFOs, noise types, distortion/saturation, compressor/limiter, reverb/delay.
3. **Event system**
   - Schedule and generate random “events” (raindrops, crackles, drips, chirps, croaks, thunder claps) with envelopes and randomized parameters.
4. **Procedural noise toolkit**
   - White / pink / brown noise
   - Band-limited noise
   - Burst/noise impulses
5. **Preset/patch system**
   - Load/save/export/import patches as JSON.
6. **“Macro” mapping layer**
   - High-level knobs (Intensity, Density, Brightness, Variation, Distance, StereoWidth) mapped to many underlying parameters.
7. **Smooth parameter automation**
   - All parameter changes should ramp smoothly to avoid clicks.
8. **Mobile Safari compatibility**
   - Audio starts only after user gesture.
   - Worklets supported; provide fallback if needed.

### 1.2 Nice-to-have
- Visual patch editor (node graph UI)
- Oscilloscope/spectrum analyzer for tuning
- AI “agent hooks” (structured parameter edits, constraint checks, auto-leveling)
- Performance modes (low CPU / high quality)
- Stereo decorrelation/widening options
- Built-in IR reverb variations

---

## 2) System Architecture Overview

### 2.1 Layers
1. **DSP Layer (Audio Engine)**
   - Web Audio nodes + custom AudioWorklets.
2. **Patch Layer**
   - JSON patch format describes nodes + connections + parameter values + modulators.
3. **Control Layer**
   - UI controls (knobs/sliders) + macro controls.
4. **Agent Layer**
   - Text-to-patch helpers: validates, diffs, constraints, “try variants”.

### 2.2 Key principle
**Everything is a Patch**:
- A “rain sound” is a patch preset.
- A “natural soundscape” is a patch that mixes multiple subpatches.
- “Macro knobs” apply a consistent mapping to different patch families.

---

## 3) Core Building Blocks (Modules)

You’ll implement a small set of reusable node modules; all natural sounds above can be built by combining them.

### 3.1 Sources
- **OscillatorSource**
  - sine/triangle/saw/square + detune
- **NoiseSource** *(AudioWorklet)*
  - white/pink/brown + optional band-limited generation
- **ImpulseSource**
  - sparse clicks/impulses (for events; often derived from NoiseSource gated by envelope)
- **GranularNoise (optional later)**
  - tiny grains of filtered noise (rustle realism)

### 3.2 Modifiers
- **Gain**
- **Filter**
  - biquad types: lowpass/highpass/bandpass/notch/peaking/shelf
- **WaveShaper / Saturation**
  - gentle warmth
- **RingMod / AM**
  - for insect textures and flutter
- **StereoPanner**
- **Delay** *(simple feedback delay)*
- **ConvolverReverb**
  - small/medium/large IR buffers (procedurally generated)
- **Dynamics**
  - compressor + limiter

### 3.3 Modulators
- **ADSR Envelope**
  - Triggered per event; target can be gain, filter cutoff, oscillator frequency, etc.
- **OneShot Envelope**
  - fast transient shapes (raindrops/crackles)
- **LFO**
  - sine/triangle/random sample&hold
- **SmoothRandom**
  - continuous wandering values (wind drift, ocean swell randomness)

### 3.4 Event Generators
- **PoissonEventClock**
  - event times from exponential distribution
- **PatternClock**
  - optional rhythmic / semi-regular patterns (cricket chirp clusters)
- **EventRouter**
  - on each event, randomize parameter set, trigger envelopes, optional probability routing

---

## 4) Patch JSON Format (Agent-Friendly)

Design a patch schema that is:
- Easy for humans and AI to edit
- Validatable
- Supports subpatch composition

### 4.1 Minimal schema
```json
{
  "meta": {"name": "Gentle Rain on Leaves", "version": 1},
  "macros": {"Intensity": 0.4, "Density": 0.6, "Brightness": 0.55, "Variation": 0.35, "Distance": 0.25, "StereoWidth": 0.3},
  "nodes": [
    {"id":"noise1","type":"NoiseSource","params":{"noiseType":"pink","amp":0.18}},
    {"id":"lp1","type":"Filter","params":{"type":"lowpass","frequency":8500,"Q":0.7}},
    {"id":"g1","type":"Gain","params":{"gain":0.14}}
  ],
  "connections": [
    {"from":"noise1","to":"lp1"},
    {"from":"lp1","to":"g1"},
    {"from":"g1","to":"OUT"}
  ],
  "events": [
    {
      "id":"drops",
      "clock":{"type":"Poisson","rate":14},
      "actions":[
        {"type":"TriggerEnvelope","target":"dropGain.gain","shape":"oneShot","params":{"attack":0.004,"decay":0.04,"peak":0.12}},
        {"type":"Randomize","target":"dropBP.frequency","range":[2500,7500]}
      ]
    }
  ],
  "macroMappings": [
    {"macro":"Brightness","targets":[{"id":"lp1.frequency","map":"exp","min":4500,"max":12000}]},
    {"macro":"Density","targets":[{"event":"drops.clock.rate","map":"pow","min":2,"max":26,"exp":1.4}]}
  ]
}
```

### 4.2 Implementation notes
- Each node gets a stable `id`.
- Params are numeric or enums; all numbers in native units (Hz, seconds).
- Separate `events` section handles triggers cleanly.
- `macroMappings` provides the AI-friendly mapping contract.

### 4.2.1 Macro Mapping Examples

**Complex Macro: "Distance" affects multiple parameters**
```javascript
// Distance macro (0 = close, 1 = far)
{
  macro: "Distance",
  targets: [
    // Reduce high frequencies (air absorption)
    { id: "masterLP.frequency", map: "linear", min: 16000, max: 2600 },
    // Reduce overall gain
    { id: "rain.baseGain.gain", map: "linear", multiply: -0.35 },
    { id: "wind.gain.gain", map: "linear", multiply: -0.25 },
    // Increase reverb mix
    { id: "wetGain.gain", map: "linear", min: 0.12, max: 0.35 },
    // Slow down variation (less movement when far)
    { id: "wind.variation", map: "linear", multiply: -0.3 },
    // Reduce transient sharpness
    { id: "rain.dropGain.attack", map: "linear", min: 0.004, max: 0.012 }
  ]
}
```

**Density Macro: Event Rate with Power Curve**
```javascript
{
  macro: "Density",
  targets: [
    // Event rate uses power curve for natural feel
    { event: "drops.clock.rate", map: "pow", min: 2, max: 26, exp: 1.4 },
    { event: "crackles.clock.rate", map: "pow", min: 1, max: 30, exp: 1.3 },
    // Also affects swell rate for ocean
    { id: "ocean.swellRate", map: "pow", min: 0.06, max: 0.16, exp: 1.2 }
  ]
}
```

**Brightness Macro: Filter Tilt**
```javascript
{
  macro: "Brightness",
  targets: [
    // Exponential mapping for frequency (more natural)
    { id: "rain.baseLP.frequency", map: "exp", min: 4500, max: 12000 },
    { id: "fire.crackleBP.frequency", map: "exp", min: 1800, max: 5200 },
    // Q factor also changes (brighter = sharper)
    { id: "stream.bp.Q", map: "linear", min: 0.6, max: 1.2 }
  ]
}
```

### 4.3 Subpatches / composition
Support:
- A patch may contain `subpatches` with independent graphs and mix levels.
- A "soundscape patch" is a mixer patch with multiple subpatch instances.

**Example: Rain + Wind Soundscape**
```json
{
  "meta": {"name": "Rainy Wind", "version": 1},
  "macros": {"Intensity": 0.45, "Density": 0.5, "Brightness": 0.5, "Variation": 0.5, "Distance": 0.3, "StereoWidth": 0.4},
  "subpatches": [
    {"id": "rain", "patch": "presets/rain-gentle.json", "mix": 0.6},
    {"id": "wind", "patch": "presets/wind-breeze.json", "mix": 0.4}
  ],
  "mixer": {
    "type": "Mixer",
    "inputs": ["rain", "wind"],
    "output": "OUT"
  },
  "macroMappings": [
    {"macro": "Intensity", "targets": [
      {"subpatch": "rain", "macro": "Intensity"},
      {"subpatch": "wind", "macro": "Intensity"}
    ]}
  ]
}
```

### 4.4 Complete Patch Example: Full Rain Implementation
```json
{
  "meta": {"name": "Gentle Rain on Leaves", "version": 1, "author": "System"},
  "macros": {"Intensity": 0.40, "Density": 0.55, "Brightness": 0.55, "Variation": 0.35, "Distance": 0.28, "StereoWidth": 0.3},
  "nodes": [
    {"id": "noise1", "type": "NoiseSource", "params": {"noiseType": "pink", "amp": 0.18}},
    {"id": "baseLP", "type": "Filter", "params": {"type": "lowpass", "frequency": 8500, "Q": 0.7}},
    {"id": "baseGain", "type": "Gain", "params": {"gain": 0.15}},
    {"id": "dropBP", "type": "Filter", "params": {"type": "bandpass", "frequency": 4500, "Q": 2.2}},
    {"id": "dropGain", "type": "Gain", "params": {"gain": 0.0}},
    {"id": "panner", "type": "StereoPanner", "params": {"pan": 0.0}},
    {"id": "outGain", "type": "Gain", "params": {"gain": 1.0}}
  ],
  "connections": [
    {"from": "noise1", "to": "baseLP"},
    {"from": "baseLP", "to": "baseGain"},
    {"from": "baseGain", "to": "panner"},
    {"from": "noise1", "to": "dropBP"},
    {"from": "dropBP", "to": "dropGain"},
    {"from": "dropGain", "to": "panner"},
    {"from": "panner", "to": "outGain"},
    {"from": "outGain", "to": "OUT"}
  ],
  "events": [
    {
      "id": "drops",
      "clock": {"type": "Poisson", "rate": 14, "jitter": 0.5},
      "actions": [
        {"type": "TriggerEnvelope", "target": "dropGain.gain", "shape": "oneShot", "params": {"attack": 0.004, "decay": 0.04, "peak": 0.12, "randomize": [0.6, 0.8]}},
        {"type": "Randomize", "target": "dropBP.frequency", "range": [2500, 7500], "smooth": true}
      ]
    }
  ],
  "modulators": [
    {"id": "windDrift", "type": "SmoothRandom", "target": "panner.pan", "rate": 0.1, "range": [-0.3, 0.3]}
  ],
  "macroMappings": [
    {"macro": "Intensity", "targets": [
      {"id": "baseGain.gain", "map": "linear", "min": 0.05, "max": 0.30},
      {"id": "drops.actions[0].params.peak", "map": "linear", "min": 0.02, "max": 0.30}
    ]},
    {"macro": "Density", "targets": [
      {"event": "drops.clock.rate", "map": "pow", "min": 2, "max": 26, "exp": 1.4}
    ]},
    {"macro": "Brightness", "targets": [
      {"id": "baseLP.frequency", "map": "exp", "min": 4500, "max": 12000},
      {"id": "dropBP.frequency", "map": "exp", "min": 2500, "max": 7500}
    ]},
    {"macro": "Distance", "targets": [
      {"id": "baseGain.gain", "map": "linear", "multiply": -0.35},
      {"id": "baseLP.frequency", "map": "linear", "multiply": -0.45}
    ]}
  ]
}
```

### 4.5 Patch Validation Schema
```javascript
const PatchSchema = {
  meta: { name: String, version: Number, author: String? },
  macros: { [String]: Number }, // 0-1 range
  nodes: [{
    id: String, // unique, alphanumeric + underscore
    type: Enum(["NoiseSource", "OscillatorSource", "Filter", "Gain", ...]),
    params: Object // type-specific
  }],
  connections: [{
    from: String, // node id or "OUT"
    to: String    // node id or "OUT"
  }],
  events: [{
    id: String,
    clock: { type: Enum(["Poisson", "Pattern"]), ... },
    actions: [...]
  }],
  macroMappings: [...]
};

// Validation rules:
// - All node IDs must be unique
// - All connections must reference existing node IDs
// - All event targets must reference existing nodes/params
// - Macro values must be 0-1
// - Circular connections detected and rejected
```

---

## 5) Engine Implementation Plan

### 5.1 Phase 1 — Core Audio Engine (MVP)
1. Create `AudioEngine` class:
   - `init()` creates `AudioContext`, master chain, meters
   - `loadPatch(patchJson)`
   - `start()` / `stop()`
   - `setMacro(name, value)`
2. Implement AudioWorklet `NoiseSource` (white/pink/brown).
3. Implement node factory:
   - maps patch node types → WebAudio node creation
4. Implement connection resolver:
   - connect nodes by id
5. Implement parameter set + smoothing:
   - `setParam(nodeId, param, value, rampTime)`

Deliverable: can load a patch and hear output.

### 5.2 Phase 2 — Modulators + Events
1. Implement ADSR and OneShot envelopes:
   - uses `AudioParam` automation
   - `setTargetAtTime` / `exponentialRampToValueAtTime` for smooth transitions
2. Implement LFO:
   - oscillator + gain to modulate AudioParam
   - Support sine/triangle/random waveforms
3. Implement SmoothRandom modulator:
   - scheduler that updates values slowly with smoothing
   - Uses `setTargetAtTime` with long time constants (0.1-2.0s)
4. Implement PoissonEventClock:
   - **Critical**: Use `AudioContext.currentTime` for scheduling, not `setTimeout`
   - Lookahead scheduler: schedule events 0.1-0.25s ahead
   - Event queue with exponential distribution: `-Math.log(1-Math.random()) * meanInterval`
   - Trigger actions at scheduled times using `AudioParam` automation

**Scheduler Implementation Pattern:**
```javascript
class EventScheduler {
  constructor(ctx) {
    this.ctx = ctx;
    this.queue = [];
    this.lookahead = 0.1; // seconds
    this.scheduleInterval = 25; // ms
    this.timer = null;
  }
  
  schedule(event) {
    const now = this.ctx.currentTime;
    const nextTime = now + this.lookahead;
    // Calculate next event time using Poisson distribution
    const delay = -Math.log(1-Math.random()) * event.meanInterval;
    this.queue.push({ time: nextTime + delay, event });
    this.queue.sort((a, b) => a.time - b.time);
  }
  
  tick() {
    const now = this.ctx.currentTime;
    const scheduleUntil = now + this.lookahead;
    
    while (this.queue.length && this.queue[0].time < scheduleUntil) {
      const { time, event } = this.queue.shift();
      // Execute event actions using AudioContext time
      event.execute(time);
      // Schedule next occurrence
      this.schedule(event);
    }
  }
  
  start() {
    this.timer = setInterval(() => this.tick(), this.scheduleInterval);
  }
}
```

Deliverable: event-driven "drops/crackles/chirps" are possible in patches with precise timing.

### 5.3 Phase 3 — Patch Authoring UI (standard synth controls)
Add a "rack" UI:
- List of modules (nodes) with familiar controls
- Add/remove module, reorder (visual), connect via a simple routing UI
- Macro panel
- Preset browser (load/save/export)

**UI Component Details:**

**Player Mode (Simple):**
- Large Play/Stop button
- Preset dropdown (Category → Preset)
- Macro knobs (6 sliders: Intensity, Density, Brightness, Variation, Distance, StereoWidth)
- Module toggles (checkboxes: Rain, Wind, Ocean, etc.)
- Master volume slider
- Status indicator
- Dark theme matching other apps

**Designer Mode (Advanced):**
- **Module Rack**: 
  - Collapsible panels for each node
  - Parameter controls (sliders, dropdowns, number inputs)
  - Visual connection indicators
  - Drag-to-reorder modules
- **Connection Editor**:
  - Dropdown lists: "From" → "To"
  - Visual connection graph (optional, Phase 3.5)
  - Connection validation (prevent cycles, invalid targets)
- **Event Editor**:
  - List of events with edit buttons
  - Clock type selector (Poisson/Pattern)
  - Action list editor
  - Test trigger button
- **Macro Mapping Editor**:
  - Table view: Macro → Target → Mapping
  - Add/remove mappings
  - Min/max value inputs
  - Mapping type selector (linear/exp/pow)
- **Patch JSON View**:
  - Read-only JSON display
  - Copy to clipboard
  - Validate button
- **Preset Manager**:
  - Load from library
  - Save to library (with name)
  - Export JSON file
  - Import JSON file
  - Delete preset

UI options (choose later):
- Minimal: forms + dropdowns + connection list
- Advanced: node graph UI using Canvas/SVG (Phase 3.5)

### 5.4 Phase 4 — Natural Sound Library
Create curated patch presets for categories (top 5 each):
- Water: rain variants, drizzle, stream, ocean, waterfall, drips, lake lapping
- Wind: breeze/pines/grass/canyon, storm wind
- Night: crickets/katydids/cicadas blend, frogs
- Fire: close/distant campfire
- Thunder: distant rumble (calm)
- Snow: hush + ice ticks
- Cave: low drone + drips
- Minimal: near-silence tone + air noise

Each preset is expressed as patch JSON.

### 5.5 Phase 5 — Agent Integration Hooks
Expose a stable JS API for agents:
- `engine.getPatch()`
- `engine.applyPatchDiff(diff)`
- `engine.validatePatch(patch)`
- `engine.renderPatchSummary(patch)` (human readable)
- `engine.autoLevel()` (optional)

Patch diffs format:
- Add node / remove node
- Change param
- Change macro mapping
- Change event rates / probabilities

---

## 6) Modeling Recipes (How to Build the Natural Sounds)

These are patch patterns; agents can mix-and-match.

### 6.1 Rain (hiss + drops)
- Bed: `Noise(pink)` → LP (4–12 kHz) → Gain
- Drops: same noise → BP (2–8 kHz) → Gain (normally 0) → OneShot envelopes triggered by Poisson clock
- “On leaves”: longer decay + slightly lower BP center and add rustle layer

### 6.2 Wind (brown noise + gust envelope)
- `Noise(brown)` → LP (200–2000 Hz) → Gain
- Gusts: SmoothRandom modulator to Gain (slow) + occasional ramps

### 6.3 Ocean (body + surf swells)
- Body: brown noise LP (80–300 Hz) with slow swell envelope
- Surf: pink noise BP (400–2000 Hz) modulated by the same swell LFO
- Variation: randomize swell period and BP center slowly

### 6.4 Stream (moving bandpass + bubbles)
- Bed: pink noise BP (300–1200 Hz) with SmoothRandom drift in BP center
- Bubbles: Poisson events gate a resonant BP burst + OneShot envelope

### 6.5 Fire (hiss + crackles)
- Bed: pink noise HP (80 Hz) low gain
- Crackles: Poisson bursts through BP (2–6 kHz) with very fast OneShot envelope
- Pops: rare longer events with lower BP

### 6.6 Crickets / insects
- Crickets: white noise BP (3–6 kHz) gated by chirp envelope
- Cicadas: continuous bandpassed noise with AM flutter (ring mod / LFO)
- Katydids: longer chirps, lower center, irregular clusters

### 6.7 Frogs
- Event oscillator: sine/triangle ~150–400 Hz with pitch drop
- Add short noise component for texture
- Sparse, slow; distance adds reverb and damping

### 6.8 Thunder (distant)
- Brown noise LP (20–120 Hz) + long envelope (1–6 s)
- Optional mid-rumble band 100–500 Hz, delayed slightly
- Keep levels low and smooth

### 6.9 Snow / winter hush
- Very low-level pink noise LP (1–3 kHz)
- Occasional faint wind gust modulation
- Optional “ice ticks”: sparse high BP ticks, extremely low

### 6.10 Cave ambience
- Low drone from brown noise LP (50–200 Hz) with slow modulation
- Drips module layered
- Longer reverb IR

### 6.11 Minimal ambience
- Tiny noise floor + ultra-slow gain modulation
- Optional low “air tone” via filtered noise

---

## 7) Effects & Spatialization

### 7.1 Reverb
- Convolver IR generation:
  - Create 2–3 IR buffers: small/medium/large
  - Crossfade wet gain rather than regenerating too often
- Macro Distance increases reverb mix and decreases master LP cutoff.

### 7.2 Stereo
- Minimal stereo widening:
  - Split signal → left/right filters with slight differences + tiny delay offsets (0–15 ms)
  - Keep it subtle; avoid phasey artifacts

---

## 8) UX & Workflow for Sound Design

### 8.1 Two modes
1. **Player Mode**: presets + macro knobs
2. **Designer Mode**: module rack + routing + scopes + patch JSON view

### 8.2 Patch editing workflow
- Build a patch in Designer Mode
- Export patch JSON
- Add to preset library
- Use macro mapping to ensure consistent control feel

### 8.3 AI-assisted workflow
- Provide agents with:
  - current patch JSON
  - target description (“more airy, fewer transients, deeper distance”)
  - constraints (peak loudness, CPU budget)
- Agent proposes patch diff
- Engine applies diff and validates
- Human listens and accepts or iterates

---

## 9) Implementation Checklist

### Phase 1 (1–2 sessions)
- [ ] `AudioEngine` skeleton + master chain + start/stop
- [ ] Noise worklet (white/pink/brown)
- [ ] Node factory (Gain/Filter/Panner/Delay/Convolver/Compressor)
- [ ] Patch load/connect
- [ ] Basic preset load

### Phase 2
- [ ] ADSR + OneShot envelopes
- [ ] LFO + SmoothRandom modulators
- [ ] Event clock (Poisson) + action execution

### Phase 3
- [ ] Rack UI + parameter panels
- [ ] Preset library UI + localStorage
- [ ] JSON export/import

### Phase 4
- [ ] Full natural sound preset library (top 5 per category)
- [ ] Loudness normalization pass

### Phase 5
- [ ] Patch diff + validate API
- [ ] Agent-friendly summaries + constraints

---

## 10) Suggested File Layout (if not single-file)

- `index.html` (UI)
- `engine/AudioEngine.js`
- `engine/worklets/noise-gen.js`
- `engine/patch/PatchSchema.js` (validation)
- `engine/patch/PatchLoader.js`
- `engine/modulators/Envelope.js`
- `engine/modulators/LFO.js`
- `engine/modulators/SmoothRandom.js`
- `engine/events/EventClock.js`
- `presets/*.json`

If single-file is required initially, keep the code in sections and later split.

---

## 11) Notes on Constraints and Quality

### 11.1 Audio Quality
- **Clicks**: always ramp changes; avoid instantaneous parameter jumps.
  - Use `setTargetAtTime` with time constant 0.02-0.1s for most params
  - Use `exponentialRampToValueAtTime` for gain changes
  - Start exponential ramps from 0.0001, not 0
- **DC Offset**: ensure all noise sources are properly centered
- **Aliasing**: use appropriate sample rates; filter high frequencies before downsampling

### 11.2 Browser Compatibility
- **iOS Safari**: require user gesture to start; handle `ctx.resume()`.
- **AudioWorklet Support**: 
  - Chrome/Edge 66+, Firefox 76+, Safari 14.1+
  - **Fallback**: ScriptProcessorNode (deprecated but widely supported)
  - **CORS Fix**: Use data URL (base64) instead of blob URL for AudioWorklet code
- **Wake Lock API**: 
  - Safari 16.4+ (iOS 16.4+)
  - Gracefully degrades on unsupported browsers
  - Re-request on `visibilitychange` event

### 11.3 Performance
- **CPU**: keep node counts modest; prefer shared noise sources per module.
- **Memory**: 
  - Clean up disconnected nodes immediately
  - Limit reverb IR buffer sizes (1-2 seconds max)
  - Dispose of unused patches and modulators
- **Battery**: 
  - Offer "low power mode" that disables reverb/convolver
  - Reduce event rates on mobile devices
  - Use Wake Lock sparingly (only during active playback)

### 11.4 Safety
- **Master Limiter**: include master limiter; default to conservative levels.
- **Parameter Validation**: clamp all user inputs to safe ranges
- **Error Handling**: graceful degradation when nodes fail to create
- **Resource Limits**: 
  - Max nodes per patch: 50
  - Max events per second: 100
  - Max patch file size: 100KB

### 11.5 Memory Management
**Cleanup Pattern:**
```javascript
class AudioEngine {
  dispose() {
    // Stop all schedulers
    this.schedulers.forEach(s => s.stop());
    
    // Disconnect and dispose all nodes
    this.nodes.forEach(node => {
      try { node.disconnect(); } catch(e) {}
      if (node.onaudioprocess) node = null; // ScriptProcessor cleanup
    });
    
    // Clear references
    this.nodes = [];
    this.schedulers = [];
    this.patch = null;
  }
  
  loadPatch(patch) {
    // Dispose old patch before loading new
    this.dispose();
    // ... load new patch
  }
}
```

### 11.6 Patch Versioning
**Version Migration Strategy:**
```javascript
const PATCH_VERSIONS = {
  1: (patch) => patch, // no migration needed
  2: (patch) => {
    // Migrate from v1 to v2
    if (!patch.meta.version) patch.meta.version = 2;
    // Add new required fields with defaults
    if (!patch.macroMappings) patch.macroMappings = [];
    return patch;
  }
};

function migratePatch(patch) {
  const currentVersion = patch.meta.version || 1;
  const targetVersion = LATEST_VERSION;
  
  for (let v = currentVersion + 1; v <= targetVersion; v++) {
    if (PATCH_VERSIONS[v]) {
      patch = PATCH_VERSIONS[v](patch);
      patch.meta.version = v;
    }
  }
  return patch;
}
```

---

## 12) Testing Strategy

### 12.1 Unit Tests
- **Patch Validation**: Test schema validation with valid/invalid patches
- **Node Factory**: Test creation of all node types
- **Connection Resolver**: Test connection logic, detect cycles
- **Macro Mapping**: Test macro-to-parameter mappings
- **Event Scheduling**: Test Poisson distribution, timing accuracy

### 12.2 Integration Tests
- **Patch Loading**: Load complete patches and verify audio graph
- **Parameter Smoothing**: Verify no clicks on parameter changes
- **Event Execution**: Verify events trigger at correct times
- **Subpatch Composition**: Test multi-layer patches

### 12.3 Audio Quality Tests
- **No DC Offset**: Measure output DC component (should be < 0.001)
- **No Clipping**: Verify output stays within [-1, 1] range
- **Frequency Response**: Test filters match expected curves
- **Loudness Normalization**: Verify presets have consistent perceived volume

### 12.4 Performance Tests
- **CPU Usage**: Measure on iPhone (target: < 30% CPU)
- **Memory Usage**: Track node count and memory leaks
- **Battery Impact**: Measure power consumption over 1 hour
- **Startup Time**: Target < 500ms from click to audio

### 12.5 Browser Compatibility Tests
- Test on: Chrome, Firefox, Safari (desktop + iOS), Edge
- Verify AudioWorklet fallback works
- Test Wake Lock on supported devices
- Verify CORS-safe loading

---

## 13) Risks and Mitigations

### 13.1 Technical Risks

**Risk: AudioWorklet not supported**
- **Mitigation**: ScriptProcessorNode fallback (already implemented)
- **Impact**: Slight performance hit, but functional

**Risk: Performance on mobile devices**
- **Mitigation**: 
  - Low-power mode option
  - Limit node counts per patch
  - Optimize scheduler frequency
- **Impact**: May need to reduce complexity on older devices

**Risk: Complex macro mappings become confusing**
- **Mitigation**: 
  - Provide preset mappings as templates
  - Visual mapping editor in Designer Mode
  - Clear documentation with examples
- **Impact**: Learning curve for advanced users

**Risk: Patch format becomes too complex**
- **Mitigation**: 
  - Keep core schema simple
  - Use composition (subpatches) for complexity
  - Version migration path
- **Impact**: Need careful schema design

### 13.2 UX Risks

**Risk: Designer Mode too complex for casual users**
- **Mitigation**: 
  - Keep Player Mode simple (presets + macros)
  - Designer Mode is optional/advanced
  - Provide good defaults
- **Impact**: Two-tier user experience (acceptable)

**Risk: Too many options overwhelm users**
- **Mitigation**: 
  - Progressive disclosure (advanced panels)
  - Good presets cover 90% of use cases
  - Macro knobs hide complexity
- **Impact**: Need careful UI design

---

## 14) Success Metrics

### 14.1 Technical Metrics
- ✅ Patch loads in < 500ms
- ✅ Audio starts in < 200ms after user gesture
- ✅ CPU usage < 30% on iPhone 12+
- ✅ No audio dropouts during 1-hour playback
- ✅ Memory usage stable (no leaks over 1 hour)

### 14.2 Quality Metrics
- ✅ All presets have consistent perceived loudness (±3dB)
- ✅ No audible clicks or pops
- ✅ Natural sounds are recognizable and pleasant
- ✅ Smooth parameter transitions (no zipper noise)

### 14.3 User Experience Metrics
- ✅ Can create a new patch in < 5 minutes (Designer Mode)
- ✅ Can load and play a preset in < 2 clicks (Player Mode)
- ✅ Works reliably on iOS Safari
- ✅ Battery lasts 4+ hours of continuous playback

---

## 15) Quick Start Guide

### 15.1 For Developers

**1. Set up AudioEngine:**
```javascript
const engine = new AudioEngine();
await engine.init();
```

**2. Load a patch:**
```javascript
const patch = await fetch('presets/rain.json').then(r => r.json());
engine.loadPatch(patch);
engine.start();
```

**3. Adjust macros:**
```javascript
engine.setMacro('Intensity', 0.7);
engine.setMacro('Density', 0.5);
```

**4. Create a simple patch:**
```javascript
const myPatch = {
  meta: { name: "My Sound", version: 1 },
  macros: { Intensity: 0.5, Density: 0.5, Brightness: 0.5, Variation: 0.5, Distance: 0.25, StereoWidth: 0.3 },
  nodes: [
    { id: "noise1", type: "NoiseSource", params: { noiseType: "pink", amp: 0.2 } },
    { id: "lp1", type: "Filter", params: { type: "lowpass", frequency: 5000, Q: 0.7 } },
    { id: "gain1", type: "Gain", params: { gain: 0.3 } }
  ],
  connections: [
    { from: "noise1", to: "lp1" },
    { from: "lp1", to: "gain1" },
    { from: "gain1", to: "OUT" }
  ],
  macroMappings: [
    { macro: "Intensity", targets: [{ id: "gain1.gain", map: "linear", min: 0.1, max: 0.5 }] }
  ]
};
engine.loadPatch(myPatch);
```

### 15.2 For AI Agents

**1. Get current patch:**
```javascript
const currentPatch = engine.getPatch();
```

**2. Propose changes:**
```javascript
const diff = {
  type: "paramChange",
  target: "lp1.frequency",
  value: 8000
};
engine.applyPatchDiff(diff);
```

**3. Validate patch:**
```javascript
const errors = engine.validatePatch(patch);
if (errors.length === 0) {
  engine.loadPatch(patch);
}
```

**4. Get human-readable summary:**
```javascript
const summary = engine.renderPatchSummary(patch);
// Returns: "Rain patch with pink noise, lowpass filter at 8500Hz, 
//           gain 0.15, 14 drops/second via Poisson clock"
```

---

## 16) Deployment & Infrastructure

### 16.1 Cloudflare Pages Deployment
- **Static hosting**: All files served as static assets
- **No build process**: Direct HTML/JS deployment
- **Global CDN**: Low latency worldwide
- **Automatic HTTPS**: Included with Cloudflare
- **Git integration**: Auto-deploy from repository

### 16.2 File Organization
```
noise-generator/
├── index.html              # Noise Helper app
├── wind-chime.html         # Wind Chimes Synth
├── nature-sounds.html      # Current nature sounds (legacy)
├── synth-platform.html     # New general-purpose platform
├── engine/
│   ├── AudioEngine.js
│   ├── worklets/
│   │   └── noise-gen.js
│   ├── patch/
│   │   ├── PatchSchema.js
│   │   └── PatchLoader.js
│   ├── modulators/
│   │   ├── Envelope.js
│   │   ├── LFO.js
│   │   └── SmoothRandom.js
│   └── events/
│       └── EventClock.js
├── presets/
│   ├── water/
│   ├── wind/
│   ├── night/
│   └── ...
└── README.md
```

### 16.3 Theme Consistency
- All apps use the same dark theme CSS variables
- Consistent button styles, sliders, cards
- Shared design language across all apps
- Mobile-responsive layouts

### 16.4 LocalStorage Strategy
- **Patch Storage**: Store user-created patches (limit: 50 patches, 100KB each)
- **Settings Storage**: UI state, last preset, macro values
- **Cleanup**: Remove old/unused patches after 90 days
- **Export/Import**: JSON export for backup and sharing

---

## 17) Deliverables

1. Working synth platform MVP with patch loading.
2. Designer UI for building patches with standard controls.
3. Preset library covering all natural sound families.
4. Agent API for iterative procedural sound design.
5. Complete documentation and migration guide.
6. Test suite covering audio quality and performance.

---

End of plan.
