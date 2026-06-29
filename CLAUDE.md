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
- **Staggered Physics (K=2):** only 50% of particles updated per frame (alternating halves). Invisible on smooth Curl-Noise fields.
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

---

## Settings System (`S` object)

All runtime state lives in the global `S` object. `DEFAULTS` holds the factory values and is used by the Reset button.

```javascript
const DEFAULTS = {
  // Particles
  count: 154500,     // slider 500–200000
  life: 410,         // lifetime in frames
  alpha: 1.5,        // brightness (HSLA alpha)
  lw: 2.4,           // line width (gl.lineWidth — clamped to 1 on Windows/ANGLE)

  // Flow
  speed: 0.25,       // flow field force multiplier
  turb: 4.0,         // turbulence = spatial noise scale
  tscale: 18.5,      // time evolution speed of flow field
  damp: 0.90,        // velocity damping per frame

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
  textGravOn: true,
  textContent: 'Joshiboy95',
  textSize: 45,          // font size in logical px (×DPR when rendered)
  textWeight: 150,       // variable font weight 100–900
  textGravStr: 0,        // current strength (animated)
  textGravAnim: true,    // special animation with pause cycle
  textGravMin: 0, textGravMax: 2000, textGravDur: 5, textGravPause: 36,
  // Cycle: [pause at min] → [min→max→min] → repeat

  // Colour animation
  colorAnimOn: true,
  colorAnimHold: 16.5,   // seconds per colour scheme
  colorAnimTrans: 19.8,  // seconds for smooth transition
  colorAnimList: [ ... ],

  // Auto-click
  autoClick: true,
  autoInterval: 10.5,    // seconds between clicks
  autoDuration: 2.3,     // seconds per click
  autoGravity: 32041,    // strength (slider raw² scale: raw≈179 → 32041)
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
| `updateField(t)` | Computes curl-noise flow field (64×45); every 3 frames |
| `tickAnimations(now, doUI)` | Cosine pendulum for all `ANIM_CFG` params; supports `pauseKey` |
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

## Known Limitations / TODOs

1. **`gl.lineWidth > 1` unreliable on Windows/ANGLE** — particles always render 1px wide regardless of `S.lw`. Fix: quad-based rendering (6 vertices per particle as a rectangle).
2. **Colour animation overwrites `S.scheme`** — switching between steps updates the global colour scheme.
3. **`S.colorAnimList` items without `customStops`** always use the current `S.customStops` when `scheme='custom'`.
4. **Text gravity DPI:** font size is multiplied by DPR, so `textSize=45` at DPR=3 renders at 135px physical pixels.
5. **Staggered physics** can show minimal jitter at very high speeds (`speed > 3`).

---

## Git Workflow

This repository uses a simple single-file workflow.

### Branch naming
- Feature / AI session branches: `claude/<short-description>-<id>` (e.g. `claude/claude-md-docs-g632l4`)
- Work directly on the designated branch; never push directly to `main` without review.

### Committing
```bash
# Stage only the main HTML file (avoid accidentally staging binaries or .env)
git add particle-flow-v5.html CLAUDE.md

git commit -m "$(cat <<'EOF'
Brief description of what changed and why

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

### Pushing and PRs
```bash
# First push from a new branch
git push -u origin <branch-name>

# After pushing, always create a draft PR via GitHub MCP tools
# so changes are reviewable before merging to main
```

If a push fails due to a network error, retry with exponential back-off (2s, 4s, 8s, 16s), up to 4 retries.

### Typical change cycle
1. Edit `particle-flow-v5.html`
2. Open the file directly in a browser to test (no server needed)
3. Verify the golden path: animation runs, panel opens/closes, settings persist after export → import
4. `git add particle-flow-v5.html && git commit -m "…"`
5. `git push -u origin <branch>`
6. Create or update a draft PR

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
