# CLAUDE.md — Particle-Website

## Project Overview

Interactive WebGL particle flow simulation. Single self-contained HTML file (`particle-flow-v5.html`) — no build system, no dependencies, no server needed. Open directly in any browser.

**Owner:** Joshiboy95  
**Target platform:** Windows 10, i7-7700K, GTX 1070 (8 GB VRAM); Samsung S22 (mobile)  
**UI language:** German. Code language: English.

---

## Repository Structure

```
particle-flow-v5.html    — Everything in one file (HTML + CSS + JS inline)
CLAUDE.md                — This file
```

The entire application lives in `particle-flow-v5.html`. There is no build step, no npm, no bundler. Editing the file and opening it in a browser is the full dev loop.

---

## Architecture

### Rendering Pipeline

- **WebGL (primary):** `gl.LINES` with a single `gl.drawArrays()` call per frame  
- **Canvas 2D (fallback):** activated automatically if WebGL is unavailable  
- **Trail effect:** `preserveDrawingBuffer: true` + full-screen quad at trail-alpha blended over old frame  
- **Additive blending:** `gl.blendFunc(gl.SRC_ALPHA, gl.ONE)`  
- **devicePixelRatio:** always respected — `canvas.width = innerWidth * dpr`

### Particle Physics

- **Typed Arrays** for all particle data (`Float32Array`, `Uint8Array`)
- **MAX = 200,000** particles pre-allocated at startup
- **Staggered Physics (adaptive `physK`, 2–4):** only `1/physK` of particles updated per frame, alternating segments (`segLen = Math.ceil(COUNT/physK)`, segment index = `time % physK`). Starts at `physK=2` (the original 50%-per-frame split). Invisible on smooth Curl-Noise fields at K=2; K=3/4 trade a bit more of this same invisible jitter for headroom under load (see Performance Governor below and Known Limitations).
- **Force cutoff radius (`FORCE_EPS`):** every point-force (mouse click, auto-click, fireworks, comet pool) computes, once per frame, the squared distance beyond which its contribution (`mag = strength/(d²+softening)`) drops below `FORCE_EPS = 0.02 px/frame²` — imperceptible regardless of accumulated frames — and skips the `sqrt()`+force computation for particles beyond it. At default force strengths this radius covers most/all of a typical viewport (≈900–1900px), so it only meaningfully engages either for genuinely distant particles or while a force's intensity curve (e.g. comet gauss ramp) is naturally weak — i.e. exactly where the force's true reach was already short. Cursor passive repel keeps its existing fixed `_crRad2` (180px) via `Math.min(_crRad2, cutoff)` since that radius is already tighter — unchanged behaviour there.
- **Adaptive performance governor:** `_tickPerfGovernor(now)` adjusts `physK` (2↔4) against the smoothed `_fps` EMA, gated by a 1.5s hysteresis window (`_physKLastChange`) so it doesn't thrash: escalates (more staggering, less physics work) when `_fps < 54`, relaxes back down when `_fps > 58.5`. This is what makes "constant 60fps" an actual runtime guarantee across very different hardware (desktop GPU vs. phone) rather than a hand-tuned fixed value — the perf-display readout (`· Physik 1/${physK}`) shows the current factor live.
- **Adaptive particle-count calibration:** on every fresh load (or after Reset), `S.count` always starts at `COUNT_CALIB_START` (30,000), independent of `DEFAULTS.count`. `_tickCountCalibrator(now, instFps)` (called every frame from `animate()`) measures the raw per-frame fps in rolling `COUNT_CALIB_WINDOW`-long (2s) windows; if the window's average is ≥55 and its 10th-percentile low is ≥50, it grows `S.count` by `COUNT_CALIB_STEP` (10,000, capped at `COUNT_CALIB_MAX` = 200,000) and keeps probing; otherwise it rolls `S.count` back down by one step (floored at `COUNT_CALIB_START`) and disables itself (`_countCalib.active`) for the rest of the session — this is what finds the actual highest sustainable particle count per device instead of a single fixed default. Permanently disabled (`_countCalibStop()`) the moment the user sets a particle count themselves, by any means: dragging/editing the count slider (`disableAnimOnTouch`), clicking a Schnellstart preset, Import, or Surprise Me — from that point on, the user's chosen value simply stays put. Re-armed (back to 30,000, `active:true`) only by the Reset button, since Reset restores factory/default behaviour and the calibrator *is* the default. Live status shown in the perf-display readout (`· Kalibriere Partikelanzahl…`) while active.
- **VBO layout:** Interleaved `[x, y, r, g, b, a] × 2` vertices per particle = 12 floats/particle = 24 bytes/vertex

### Flow Field

- **Curl Noise** (`∂P/∂y, −∂P/∂x` of Perlin Noise) → divergence-free field, produces natural vortices
- **64×45 grid** (COLS×ROWS), recomputed every 3 frames
- **Bilinear interpolation** when sampling

### Colour System

- **16 hue buckets × 2 brightness levels = 32 palette entries** (`PALETTE[]` as HSLA strings + `PALETTE_RGBA[]` as `Float32Array` for WebGL)
- `buildPalette()` fills both arrays; called on any colour change
- `blendPaletteHSL()` interpolates between two HSL palettes for smooth colour-animation transitions
- `hslToRgbf(h,s,l)` converts HSL→RGB for WebGL

### Text Gravity Field

- Offscreen Canvas (`TC=256 × TR=180` grid, 4× the flow-field resolution)
- Text rendered with Figtree Variable Font (100–900 weight), scaled by DPR
- Direction field: per-cell vector pointing toward the nearest text pixel (search radius SR=10 cells)
- Rebuilt only when settings change (`_textDirty` flag), not every frame
- Sampled in the particle loop the same way as the flow field (bilinear, O(1) per particle)
- **Viewport-relative sizing:** the user-facing slider sets `S.textSizePct` (0–100, % of the current viewport width `W`), not the rendered font size directly. `S.textSize` (logical px, what `buildTextField()` actually reads) is a *derived* value, recomputed by `_syncTextSize()` — which calls `_computeViewportTextSize(text, weight, targetWidthPhysical)` to find the logical font size that makes the rendered text exactly `W * (textSizePct/100)` physical px wide — on every event that can change either the target percentage or the rendered width: slider input, text-content input, text-weight input, `resize()`, and `applyAllUI()`. Text stays centred via the existing `textAlign='center'` in `buildTextField()`. This makes the text width track the viewport responsively across window resizes, orientation changes, and desktop/mobile presets, instead of needing a fixed px value re-tuned per device.

---

## Settings System (`S` object)

All runtime state lives in the global `S` object. `DEFAULTS` holds the factory values and is used by the Reset button.

```javascript
const DEFAULTS = {
  // Particles
  count: 100000,     // slider 500–200000
  life: 600,         // lifetime in frames
  alpha: 1.5,        // brightness (HSLA alpha)
  lw: 1,             // line width (gl.lineWidth — clamped to 1 on Windows/ANGLE)

  // Flow
  speed: 0.294,      // flow field force multiplier
  turb: 3.79,        // turbulence = spatial noise scale
  tscale: 13.2,      // time evolution speed of flow field
  damp: 0.92,        // velocity damping per frame

  // Trail
  trail: 0.06,       // fade-overlay alpha (lower = longer trail)
  additive: true,    // additive blending (glow effect)

  // Mouse
  gravity: 32000,    // gravity strength (inverse square law)

  // Colour
  scheme: 'neon',    // blue|dual|indigo|ember|rainbow|aurora|neon|sunset|ocean|custom
  colorDrive: 'velocity', // seed|velocity|posX|posY|age|mixed
  customStops: [{h:17},{h:244},{h:168},{h:246},{h:251},{h:38}],
  customSat: 80,

  // Time animations (all follow the same pattern)
  turbAnim:  true,  turbMin:  1.8,  turbMax:  6.5,  turbDur:  15,
  speedAnim: true,  speedMin: 0.05, speedMax: 0.60, speedDur: 26,
  alphaAnim: false, alphaMin: 0.05, alphaMax: 1.5,  alphaDur: 6,
  trailAnim: false, trailMin: 0.01, trailMax: 0.15, trailDur: 12,
  tscaleAnim:false, tscaleMin:0.0,  tscaleMax:6.0,  tscaleDur:15,

  // Text gravity field
  textGravOn: false,
  textContent: 'Joshiboy95',
  textSizePct: 60,       // target text width as % of viewport width (0–100, centred); derives S.textSize
  textWeight: 150,       // variable font weight 100–900
  textGravStr: -10000,   // current strength (animated), rests at textGravMin
  textGravAnim: true,    // special animation with pause cycle
  textGravMin: -10000, textGravMax: 0, textGravDur: 5, textGravPause: 6,
  // Cycle: [pause at min] → [min→max→min] → repeat

  // Colour animation
  colorAnimOn: true,
  colorAnimHold: 19.5,   // seconds per colour scheme
  colorAnimTrans: 5.7,   // seconds for smooth transition
  colorAnimList: [ ... ],

  // Auto-click
  autoClick: true,
  autoInterval: 10.5,    // seconds between clicks
  autoDuration: 2.3,     // seconds per click
  autoGravity: 37249,    // strength (slider raw² scale)

  // Fireworks
  fwksOn: true, fwksMinClicks: 3, fwksMaxClicks: 7,
  fwksMinStrRaw: 130, fwksMaxStrRaw: 250,  // actual = raw²
  fwksClickDur: 0.4, fwksMinPause: 11, fwksMaxPause: 23,
  fwksBurstOn: true,

  // Komet (Schweifeffekt)
  cometOn: true, cometIntensity: 'linUp',
  cometStrRaw: 265, cometDuration: 1, cometInterval: 8,

  // Kometenschauer
  cometShowerOn: false,
  cometShowerMinCount: 3, cometShowerMaxCount: 6,
  cometShowerMinGap: 600, cometShowerMaxGap: 1800,
  cometShowerMinPause: 20, cometShowerMaxPause: 40,
};
```

---

## Animation System (`ANIM_CFG`)

Generic system driving 6 parameters with cosine pendulum animation:

```javascript
const ANIM_CFG = [
  { key:'turb',        onKey:'turbAnim',   minKey:'turbMin',   maxKey:'turbMax',   durKey:'turbDur',
    sid:'s-turb',   vid:'v-turb',   fmt:'f1' },
  { key:'speed',       onKey:'speedAnim',  ... },
  { key:'alpha',       onKey:'alphaAnim',  ..., onChange:()=>buildPalette() },
  { key:'trail',       onKey:'trailAnim',  ..., onChange:()=>rebuildTrail() },
  { key:'tscale',      onKey:'tscaleAnim', ... },
  { key:'textGravStr', onKey:'textGravAnim', ..., pauseKey:'textGravPause' },
  //                                              ^^^^ pause-cycle variant
];
```

**`tickAnimations(now, doUI)`** — cosine pendulum: `min + (max−min) × (1 − cos(phase×2π)) / 2`  
**Pause mode** (textGravStr only): `[pauseDur seconds at min] → [full anim cycle] → repeat`

---

## Key Functions Quick Reference

| Function | Description |
|---|---|
| `buildPalette()` | Fills `PALETTE[]` + `PALETTE_RGBA[]` from `S.scheme`/`alpha`; no-op during colour transition |
| `buildTextField()` | Builds text gravity field (256×180); only called when `_textDirty=true` |
| `_syncTextSize()` | Recomputes `S.textSize` from `S.textSizePct` × current viewport width `W`; called on slider/content/weight input, `resize()`, and `applyAllUI()` |
| `updateField(t)` | Computes curl-noise flow field (64×45); every 3 frames |
| `tickAnimations(now, doUI)` | Cosine pendulum for all `ANIM_CFG` params; supports `pauseKey` |
| `_tickPerfGovernor(now)` | Adjusts `physK` (2–4) against the `_fps` EMA with 1.5s hysteresis; called every frame from `animate()` before the staggering math |
| `_tickCountCalibrator(now, instFps)` | Ramps `S.count` up from `COUNT_CALIB_START` in 2s windows while fps stays healthy, rolls back + disables itself otherwise; called every frame from `animate()` |
| `_countCalibStop()` | Permanently disables the count calibrator; called from every place the user can set `S.count` themselves |
| `_forceCutoffR2(strength, softening)` | Returns the squared distance beyond which a point-force's contribution drops below `FORCE_EPS`; recomputed per active force per frame |
| `tickColorAnim(now, doUI)` | Colour animation: hold→transition→hold with HSL interpolation |
| `applyAllUI()` | Syncs ALL UI elements from `S`; called on import/reset/startup |
| `rebuildTrail()` | Caches trail string + updates WebGL quad VBO |
| `clearCanvas()` | WebGL `gl.clear` OR Canvas2D `fillRect` |
| `resize()` | Sets W/H with DPR, updates WebGL viewport + `u_res` uniform |
| `initWebGL()` | Creates shaders, program, VBO, QuadVBO; falls back to `ctx` |
| `spawnParticle(i, randomAge)` | Re-initialises particle i (pX/pY/pVX/pVY/pMaxA/pAge) |
| `schemeColor(t, scheme, alpha, bright)` | Returns an HSLA string for palette position `t` |
| `hslToRgbf(h,s,l)` | HSL→RGB float3 for WebGL palette |
| `blendPaletteHSL(a, b, t)` | Writes blended HSLA into `PALETTE[]` + `PALETTE_RGBA[]` |
| `fmtFn(name)` | Returns a formatter (`f1/f2/f3/fi`) for UI display |

---

## WebGL Details

```
Vertex layout:  [x, y, r, g, b, a]  — stride = 24 bytes
VBO size:       MAX×2×6×4 = 9.6 MB at MAX=200k
Draw call:      gl.drawArrays(gl.LINES, 0, vboCount×2)
Blend modes:    - Trail fade:        SRC_ALPHA, ONE_MINUS_SRC_ALPHA
                - Additive particles: SRC_ALPHA, ONE
                - Normal particles:   SRC_ALPHA, ONE_MINUS_SRC_ALPHA
```

**Performance baseline (i7-7700K / GTX 1070):**
- ~27 fps at 80k particles with Canvas2D (pre-WebGL)
- WebGL: 1 draw call instead of 200k Canvas2D bridge calls
- Staggered physics K=2: ~47% less physics work per frame
- Effective capacity: ~200k particles at 60 fps
- Adaptive `physK` (2–4) + force-cutoff radius push this further under heavy simultaneous-force load (e.g. Kometenschauer + Feuerwerk + Auto-Klick at once) while keeping the common case at the original K=2 quality level — see Architecture → Particle Physics

---

## Import / Export

Full `S` state serialised as JSON (date in filename).  
`customStops` and `colorAnimList` are deep-copied on export/import/reset.  
Import calls `applyAllUI()` to re-sync all sliders and toggles.

---

## UI Architecture

### Panel
- Right slide-in panel (300px), opened via the gear icon (top-right)
- Click outside to close
- Import ↑ / Export ↓ buttons in panel header
- HUD buttons (`.hud-btns`: fullscreen/music/settings) slide off-screen left + fade out while the panel is open (`.hud-btns.panel-open`), since the panel covers nearly the full width on mobile and the panel's own `#btn-close` already handles closing

### Basic / Advanced settings split
- Opening the panel shows **Basis-Einstellungen** first: only 5 controls (Farbschema, Partikelanzahl, Helligkeit, Geschwindigkeit, Spurlänge) — the highest-impact, least-overwhelming parameters for casual visitors
- Everything else (Farbantrieb/custom colour editor, colour-animation, Lebensdauer/Linienbreite, Wirbelstärke/Zeitentwicklung/Dämpfung/FBM, additive glow, Maus-Interaktion, Auto-Klick, Feuerwerk-Modus, Text-Gravitationsfeld) lives inside `#advanced-settings`, collapsed (`display:none`) by default, revealed via the `#btn-advanced-toggle` "⚙ Erweiterte Einstellungen" button
- This is purely a DOM/visibility reorganisation — `bindSlider`/`applyAllUI`/animation ticking all operate by element ID, not DOM position, so moving a control between Basic and Advanced never requires touching the binding logic
- New settings should default into Advanced unless they're a primary visual-impact knob a first-time visitor should see immediately

### Slider binding
```javascript
bindSlider(sliderId, valId, key, formatFn, onChangeFn)
// makeValueEditable() makes all number values directly editable on click
// Values outside slider range are possible: slider clamps, S[key] still takes the typed value
```

### Design tokens
- Font: Figtree Variable Font (100–900, Google Fonts)
- Background: `#07070f`
- Accent: `#3b82f6` (blue)
- Border radius: 7–10px
- No glassmorphism, no illustrations

### UI language conventions
- All labels, buttons, and error messages: **German**
- Number formatter: `fi` = `toLocaleString('de-DE')` (dot as thousands separator), `f1/f2/f3` = `toFixed`

---

## Development Patterns

### Adding a new time animation
1. Add entry to `ANIM_CFG` array
2. HTML: add `anim-sub` block with `t-anim-KEY`, `ain-KEY`, `s-amin/amax/adur-KEY`
3. `DEFAULTS`: add `KEYAnim:false, KEYMin:x, KEYMax:y, KEYDur:z`
4. Covered automatically by the generic `tickAnimations()` and binding loop

### Adding a new colour scheme
1. Add `case 'myscheme':` in `schemeColor(t, scheme, alpha, bright)`
2. HTML: add `.sc-btn` with `data-scheme="myscheme"` and a `.sc-sw` gradient preview strip

### Adding a new settings parameter
1. Add to `DEFAULTS` and to `S` initialisation
2. HTML: add slider or toggle in the panel
3. Call `bindSlider()` in the UI section
4. Add to `applyAllUI()` so it syncs on import/reset
5. Export: automatic via `{...S}`; import: `Object.assign(S, data)` handles it

---

## Added Features (v5 → current)

### HUD Buttons (top-right, outside panel)
- **Fullscreen** (`btn-fullscreen`) — toggles `document.fullscreenElement`; `.active` class while in fullscreen
- **Music** (`btn-music`) — starts/stops atmospheric ambient audio; `.active` while playing

### Schnellstart (top of panel)
- **Desktop preset** (`btn-preset-pc`) — sets 180k particles, optimised flow/trail for desktop
- **Mobile preset** (`btn-preset-phone`) — sets 25k particles for smooth mobile performance
- **Surprise Me** (`btn-surprise`) — randomises all parameters within aesthetic ranges; enables random animations and schemes

### Colour Schemes
- **Galaxy** — replaces Dual; deep purple → electric blue → ice-white (great with velocity drive)
- **Plasma** — replaces Indigo; hot magenta → purple → vivid cyan (100% saturation)
- All existing schemes improved: deeper darks, wider brightness contrast range

### Physics — Cursor Passive Repulsion
- `S.cursorRepel` (bool) + `S.cursorRepelStr` (default 1200)
- Applied in staggered physics loop when `!hasForce && input.onCanvas`
- Radius: 180 logical px; strength falls off with `1/(d² + 300)`
- Togglable in Mouse section of panel

### Fireworks Mode (panel section)
State machine: `'burst'` → `'pause'` → `'burst'` …
- `S.fwksOn`, `S.fwksMinClicks/fwksMaxClicks` — random clicks per burst
- `S.fwksMinStrRaw/fwksMaxStrRaw` — strength slider uses `raw²` scale (same as auto-click)
- `S.fwksMinInterval/fwksMaxInterval` — ms between clicks within a burst
- `S.fwksClickDur` — seconds each individual click stays active
- `S.fwksMinPause/fwksMaxPause` — pause seconds between bursts
- Object: `fwks = {state, clicksDone, totalClicks, nextClickTime, nextBurstTime, cx, cy, cEnd, cStr}`
- `tickFireworks(now, doUI)` — called from `animate()`; fires a music note on each click if music is on

### Comet Mode / Schweifeffekt (panel section)
A virtual click-point (or several, concurrently) sweeps along a slightly curved path across the canvas, attracting particles like the other click-based forces. Built on a small pool so multiple comet-tails can be in flight at once (needed for Kometenschauer, where spawn gaps can be shorter than a single comet's travel duration).
- `S.cometOn` — master toggle (default **on**, lives in Advanced)
- `S.cometIntensity` — `'gauss'` (rise→peak→fall) | `'linUp'` | `'linDown'`; min is always 0, the configured strength is the max. Used by the simple single-comet mode; each shower-spawned comet locks in this mode at spawn time.
- `S.cometStrRaw` — strength slider uses `raw²` scale (same pattern as `autoGravity`)
- `S.cometDuration` — seconds to traverse the path (single-comet mode); since path length scales with the canvas diagonal, crossing speed automatically scales with screen size
- `S.cometInterval` — pause seconds between comets (single-comet mode)
- `COMET_POOL_MAX = 8`, `cometPool = []` — shared pool of in-flight comet instances, each `{x0,y0,x1,y1,ctrlX,ctrlY,str,intensity,startTime,endTime,cx,cy,frameStr}`; oldest entry evicted via `.shift()` if full
- `_cometSpawn(now, durationSec, strRaw, intensityMode)` — rejection-samples two random points inside a padded canvas rect (up to 10 tries) until their distance is ≥ 50% of the canvas diagonal, derives a quadratic-Bézier control point via a small random perpendicular offset (≤ 12% of segment length), and pushes a new pool entry
- `cometIntensity(mode, t)` — normalized 0→1→0 (gauss) or linear ramp; `mode` is passed explicitly (not read from `S` live) so each pool entry's curve is locked in at spawn time
- `tickComet(now, doUI)` — called from `animate()` right after `tickFireworks`. Each frame: prunes expired pool entries and updates each remaining entry's `cx`/`cy` (Bézier position) and `frameStr` (`str × cometIntensity(...)`) in place (no per-frame object allocation); then, if `S.cometOn`, drives whichever trigger mode is active
- Physics loop consumes the pool directly: `const cometForces = S.cometOn ? cometPool : EMPTY_ARR` + a small `for` loop applying each entry's inverse-square attraction — same formula as the other click-like forces, just looped over a (typically 0–8 element) array

**Single-comet mode** (default, `S.cometShowerOn = false`):
- State object: `cometSingle = {state, nextTime}`, cycling `'pause'` → `'active'` → `'pause'` …
- One comet at a time, spawned via `_cometSpawn(now, S.cometDuration, S.cometStrRaw, S.cometIntensity)`, then waits `S.cometInterval` seconds

**Kometenschauer (comet shower) mode** (`S.cometShowerOn = true`, sub-toggle inside the Komet section):
Same burst/pause principle as Fireworks, but spawning comet-tails instead of clicks.
- `S.cometShowerOn` — toggle; swaps the panel's single-comet "Pause zw. Kometen" slider for the shower controls (`comet-single-ctrl` vs `comet-shower-ctrl`)
- `S.cometShowerMinCount/cometShowerMaxCount` — random number of comets per shower
- `S.cometShowerMinGap/cometShowerMaxGap` — random ms between comets within a shower
- `S.cometShowerMinPause/cometShowerMaxPause` — random seconds pause between showers
- State object: `cometShower = {state, spawned, totalCount, nextSpawnTime, nextShowerTime}`, cycling `'pause'` → `'burst'` → `'pause'` …; each spawned comet uses the current `S.cometDuration`/`S.cometStrRaw`/`S.cometIntensity` at spawn time

### Atmospheric Music
- `_initAudio()` — lazy-init on first toggle (required for browser autoplay policy)
- 3 oscillators: 55 Hz (A1 sine) + 82.4 Hz (E2 perfect fifth) + 110.3 Hz (A2 triangle, slight detune)
- Delay-feedback reverb via two `GainNode`s chained to `DelayNode`
- `_filterNode` (lowpass, Q=1.4) tracks particle energy: `_energyAvg → 180–2400 Hz cutoff`
- `playMusicNote()` — picks a random `C2 pentatonic` frequency, 4-second exponential fade; also called on fireworks clicks
- `_scheduleNote()` — schedules next ambient note every 4–13 seconds
- Energy sampled every 10 frames over 200 random particles; 94/6 exponential smoothing

---

## Known Limitations / TODOs

1. **`gl.lineWidth > 1` unreliable on Windows/ANGLE** — particles always render 1px wide regardless of `S.lw`. Fix: quad-based rendering (6 vertices per particle as a rectangle).
2. **Colour animation overwrites `S.scheme`** — switching between steps updates the global colour scheme.
3. **`S.colorAnimList` items without `customStops`** always use the current `S.customStops` when `scheme='custom'`.
4. **Text gravity DPI:** font size is multiplied by DPR, so `textSize=45` at DPR=3 renders at 135px physical pixels.
5. **Staggered physics** can show minimal jitter at very high speeds (`speed > 3`); the adaptive performance governor can push `physK` up to 4 under sustained load, which proportionally increases this same jitter (still bounded — particles never freeze for more than 3 extra frames).

---

## Git Workflow

This repository uses a simple single-file workflow. Push directly to `main` — no PRs, no feature branches.

### Committing
```bash
# Stage only the relevant files (avoid accidentally staging binaries or .env)
git add particle-flow-v5.html CLAUDE.md

git commit -m "Brief description of what changed and why"
```

### Pushing
```bash
git push origin main
```

If a push fails due to a network error, retry with exponential back-off (2s, 4s, 8s, 16s), up to 4 retries.

### Typical change cycle
1. Edit `particle-flow-v5.html`
2. Open the file directly in a browser to test (no server needed)
3. Verify the golden path: animation runs, panel opens/closes, settings persist after export → import
4. `git add particle-flow-v5.html && git commit -m "…"`
5. `git push origin main`

### What NOT to commit
- `.env` files or any file containing API keys / tokens
- Build artefacts or generated files (there are none by design)
- OS files (`.DS_Store`, `Thumbs.db`)

---

## Testing / Verification Checklist

Since there is no automated test suite, verify changes manually in a browser:

- [ ] Animation starts and runs at target FPS (≥ 50 fps at default 154,500 particles on desktop)
- [ ] Gear icon opens / closes panel
- [ ] All sliders move and update the visualisation in real time
- [ ] Import a previously exported JSON → all settings restore correctly
- [ ] Export produces a valid JSON file with current settings
- [ ] Mouse left-click → particles attract; right-click → particles repel
- [ ] Text gravity field renders the correct text
- [ ] Colour scheme buttons switch the palette immediately
- [ ] Colour animation cycles through schemes with smooth transitions
- [ ] Canvas resizes correctly on window resize (no stretched/blurry render)
- [ ] Works on mobile (touch attract/repel, no layout overflow)

---

## File Structure (Inside `particle-flow-v5.html`)

```
particle-flow-v5.html
  ├── <link>          Figtree font (Google Fonts)
  ├── <style>         CSS (panel, sliders, buttons, animation subsections)
  ├── HTML body       Canvas + panel markup (static)
  └── <script>
      ├── DEFAULTS/S       Settings object + deep-copy init
      ├── ANIM_CFG         Animation configuration array
      ├── Perlin Noise     Permutation table + gradient function
      ├── Flow Field       Curl noise, 64×45 grid, bilinear sampling
      ├── Text Field       256×180 grid, offscreen canvas, direction vectors
      ├── Colour System    PALETTE, PALETTE_RGBA, HSL→RGB, scheme functions
      ├── Colour Animation PAL_A/PAL_B hold→transition state machine
      ├── Particle Arrays  Float32Arrays, staggered physics loop
      ├── WebGL Init       Shaders, VBO, QuadVBO; Canvas2D fallback
      ├── Input            Mouse + touch with DPR scaling
      ├── Auto-Click       Timing state, random position
      ├── animate()        Main loop: animations → trail → physics → draw
      └── UI               bindSlider, makeValueEditable, panel logic, applyAllUI
```
