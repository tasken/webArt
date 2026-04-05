# Code Quality & Feature Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix two code quality issues and implement a prioritized backlog of features — from lightweight docs fixes through multi-touch, low-res mode, and feedback FBO trails.

**Architecture:** All JS is ES module, no bundler transforms beyond Vite. WebGL 1. No test framework — correctness is verified by running the dev server and inspecting the result visually. Commits are made after each task.

**Tech Stack:** JavaScript ES modules, WebGL 1, Vite HMR, `canvas.toBlob`, `MediaRecorder`

---

## File Map

| File | Changes |
|------|---------|
| `src/simulation.js` | Add JSDoc comment to `resize()` |
| `src/settings.js` | Add comment clarifying `wordScaleY`; add `feedbackDecay`, `lowResScale` |
| `src/sketch.js` | Remove baked template constants → 4 new uniforms; export `staticUniforms`; add `u_prevFrame`; update color blend |
| `src/renderer.js` | Add uniform names; accept + apply `staticUniforms`; add `copyPrevFrame()` method; handle FBO texture on resize |
| `src/main.js` | Pass `staticUniforms` to renderer; refactor pointer → multi-touch Map; add low-res detection; call `copyPrevFrame` |
| `README.md` | Document multi-touch behaviour and `?lowres` param |

---

## Tier 1 — High impact / low effort

---

### Task 1: Document `simulation.js resize()` no-cache constraint

**Files:**
- Modify: `src/simulation.js:41`

The `resize()` function creates new `Float32Array` instances and updates `state` via `Object.assign`. If a caller ever cached a direct reference to `sim.density` before calling resize, that reference becomes stale. Document this so future contributors don't introduce the bug.

- [ ] **Step 1: Add JSDoc comment above `resize()`**

Open `src/simulation.js`. Replace line 41:
```js
  function resize(newCols, newRows) {
```
with:
```js
  /**
   * Resize the simulation grid. All internal Float32Array buffers are
   * recreated at the new size and re-assigned onto `state` via Object.assign.
   *
   * IMPORTANT: Callers must never cache direct references to buffer properties
   * (e.g. `const d = sim.density`). Always access them through the returned
   * state object (e.g. `sim.density`, `sim.pixels`). Internal functions
   * (`step`, `injectForce`) are safe because they always read through `state`.
   */
  function resize(newCols, newRows) {
```

- [ ] **Step 2: Verify dev server still starts**

Run: `npm run dev`
Expected: server starts on port 58707, no console errors.

- [ ] **Step 3: Commit**

```bash
git add src/simulation.js
git commit -m "docs: document resize() no-cache assumption for state buffers"
```

---

### Task 2: Clarify `wordScaleY` intent in settings

**Files:**
- Modify: `src/settings.js:25`
- Modify: `src/words.js:206-208`

`wordScaleY = 0.25` vertically squashes the word bitmap. This is intentional — the split-flap letters are meant to appear compressed/tall in the giant background effect. But the intent is not documented, and `0.25` looks like it might be an accident.

- [ ] **Step 1: Expand the comment in `settings.js`**

In `src/settings.js`, replace line 25:
```js
export const wordScaleY        = 0.25    // vertical scale for lyric aspect ratio
```
with:
```js
export const wordScaleY        = 0.25    // intentional vertical squash on the word bitmap.
                                          // Values < 1 compress letters vertically, creating
                                          // the tall/narrow split-flap aesthetic in the
                                          // giant background characters. Tune freely (0.1–1.0).
```

- [ ] **Step 2: Add an inline comment at the draw site in `words.js`**

In `src/words.js`, replace lines 206-208:
```js
    ctx.save()
    ctx.translate(0, H * 0.5 * (1 - wordScaleY))
    ctx.scale(1, wordScaleY)
```
with:
```js
    ctx.save()
    // Squash text vertically so giant background letters have a compressed,
    // split-flap aesthetic.  wordScaleY < 1 = taller/narrower chars.
    ctx.translate(0, H * 0.5 * (1 - wordScaleY))
    ctx.scale(1, wordScaleY)
```

- [ ] **Step 3: Verify visually**

Run `npm run dev`. Confirm the giant background letters still appear and look like squashed letterforms (not full-height natural text).

- [ ] **Step 4: Commit**

```bash
git add src/settings.js src/words.js
git commit -m "docs: clarify wordScaleY is intentional vertical squash, not a bug"
```

---

### Task 3: Convert baked shader constants to uniforms in `sketch.js`

**Files:**
- Modify: `src/sketch.js`
- Modify: `src/renderer.js`
- Modify: `src/main.js`

Currently `sketch.js` imports `fieldTimeScale`, `fieldAmplitude`, `wordCanvasW`, `wordCanvasH`, `densityChars` solely to bake them into template literal strings inside the shader. This makes `sketch.js` an awkward intermediary. Converting them to uniforms makes them proper runtime values, removes the template smell, and enables live-tuning without shader recompile.

**New uniforms:** `u_fieldTimeScale` (float), `u_fieldAmplitude` (float), `u_wordAspect` (float — precomputed `wordCanvasH / wordCanvasW`), `u_densityCharCount` (float).

- [ ] **Step 1: Update the shader in `sketch.js`**

In `src/sketch.js`, change the import on lines 7-11 from:
```js
import {
  fontSize, fontFamily, chars, densityChars,
  fieldTimeScale, fieldAmplitude,
  wordCanvasW, wordCanvasH,
} from './settings.js'
```
to:
```js
import {
  fontSize, fontFamily, chars, densityChars,
  fieldTimeScale, fieldAmplitude,
  wordCanvasW, wordCanvasH,
} from './settings.js'
```
(Import list stays the same — they move from shader template literals to `staticUniforms`.)

Replace line 13:
```js
export const config = { fontSize, fontFamily, chars }
```
with:
```js
export const config = { fontSize, fontFamily, chars }

export const staticUniforms = {
  u_fieldTimeScale:   fieldTimeScale,
  u_fieldAmplitude:   fieldAmplitude,
  u_wordAspect:       wordCanvasH / wordCanvasW,
  u_densityCharCount: densityChars.length,
}
```

In the fragment shader string, add four uniform declarations after `uniform sampler2D u_wordTex;` (line 44):
```glsl
uniform float     u_fieldTimeScale;  // time → shader time multiplier
uniform float     u_fieldAmplitude;  // background noise strength
uniform float     u_wordAspect;      // wordCanvasH / wordCanvasW
uniform float     u_densityCharCount; // number of chars in the density ramp
```

Replace the baked template on line 144:
```js
  float t     = u_time * ${g(fieldTimeScale)} + u_seed;
```
with:
```glsl
  float t     = u_time * u_fieldTimeScale + u_seed;
```

Replace line 145:
```js
  float bgVal = procValue(uv, t) * ${g(fieldAmplitude)};          // bold ambient backdrop
```
with:
```glsl
  float bgVal = procValue(uv, t) * u_fieldAmplitude;
```

Replace lines 169-170:
```js
  float aspect = (u_gridSize.x / u_gridSize.y)
               * (${g(wordCanvasH)} / ${g(wordCanvasW)});  // grid aspect / tex aspect
```
with:
```glsl
  float aspect = (u_gridSize.x / u_gridSize.y) * u_wordAspect;
```

Replace line 191:
```js
  float charIdx = clamp(floor(d * ${g(densityChars.length)}), 0.0, ${g(densityChars.length - 1)});
```
with:
```glsl
  float charIdx = clamp(floor(d * u_densityCharCount), 0.0, u_densityCharCount - 1.0);
```

- [ ] **Step 2: Update `renderer.js` to handle `staticUniforms`**

In `src/renderer.js`, add the four new names to `UNIFORM_NAMES` (after `'u_wordTex'`):
```js
const UNIFORM_NAMES = [
  'u_time',
  'u_resolution',
  'u_gridSize',
  'u_cellSize',
  'u_atlas',
  'u_charCount',
  'u_pointer',
  'u_pointerDelta',
  'u_pointerActive',
  'u_pointerDown',
  'u_fluid',
  'u_seed',
  'u_wordTex',
  'u_fieldTimeScale',
  'u_fieldAmplitude',
  'u_wordAspect',
  'u_densityCharCount',
]
```

In `createRenderer`, destructure `staticUniforms` from `opts` and store as a mutable variable. Add this right after the existing destructure on line 117:
```js
export function createRenderer(canvas, opts) {
  const { vertexSource, fragmentSource, fontSize, fontFamily, chars } = opts
  let staticUniforms = opts.staticUniforms || {}
```

Add `applyStaticUniforms` helper after `getUniforms` is first called (after line 138):
```js
  function applyStaticUniforms() {
    gl.useProgram(program)
    for (const [name, value] of Object.entries(staticUniforms)) {
      if (u[name] !== undefined) gl.uniform1f(u[name], value)
    }
  }
```

Call `applyStaticUniforms()` right after the `gl.uniform1f(u.u_seed, seed)` call (after line 185):
```js
  gl.useProgram(program)
  gl.uniform1f(u.u_seed, seed)
  applyStaticUniforms()
```

Update `recompile` to accept and apply new static uniforms. Replace the `recompile` method body:
```js
    recompile(newVertexSource, newFragmentSource, newStaticUniforms) {
      const newProgram = link(
        gl,
        compile(gl, gl.VERTEX_SHADER, newVertexSource),
        compile(gl, gl.FRAGMENT_SHADER, newFragmentSource),
      )
      gl.deleteProgram(program)
      program = newProgram
      aPos = getAttrib(gl, program, 'a_position')
      Object.assign(u, getUniforms(gl, program, UNIFORM_NAMES))
      if (newStaticUniforms) staticUniforms = newStaticUniforms
      bindAtlas()
      bindFluid()
      bindWord()
      gl.useProgram(program)
      gl.uniform1f(u.u_seed, seed)
      applyStaticUniforms()
      this.resize()
    },
```

- [ ] **Step 3: Update `main.js` to pass `staticUniforms`**

In `src/main.js`, change line 2:
```js
import { vertexSource, fragmentSource, config } from './sketch.js'
```
to:
```js
import { vertexSource, fragmentSource, config, staticUniforms } from './sketch.js'
```

Change line 46:
```js
  const renderer = createRenderer(canvas, { vertexSource, fragmentSource, ...config })
```
to:
```js
  const renderer = createRenderer(canvas, { vertexSource, fragmentSource, ...config, staticUniforms })
```

Change the HMR accept handler (line 150):
```js
      renderer.recompile(newSketch.vertexSource, newSketch.fragmentSource)
```
to:
```js
      renderer.recompile(newSketch.vertexSource, newSketch.fragmentSource, newSketch.staticUniforms)
```

- [ ] **Step 4: Verify visually**

Run `npm run dev`. The visual output must be identical to before — no change in the art. Open browser devtools console: no errors.

Test HMR: change `fieldAmplitude` in `settings.js` to `1.2`, save `sketch.js` (re-export triggers recompile). The background noise should become bolder without a full page reload. Revert.

- [ ] **Step 5: Commit**

```bash
git add src/sketch.js src/renderer.js src/main.js
git commit -m "refactor: convert baked shader constants to uniforms, export staticUniforms from sketch"
```

---

## Tier 2 — Medium impact / medium effort

---

### Task 4: Multi-touch support

**Files:**
- Modify: `src/main.js`
- Modify: `README.md`

Currently only one pointer is tracked. On tablets/phones, simultaneous touches are ignored. This refactors the pointer state to a `Map<pointerId, pointer>` so all active contacts inject fluid forces. The GPU glow still renders for the primary (first active) pointer only — changing that would require multi-pointer uniforms in the shader, which is a bigger change.

- [ ] **Step 1: Refactor pointer state in `main.js`**

Replace the `pointer` object and `lastMoveAt` scalar with a Map. Find this block (lines 50-58):
```js
  const pointer = {
    x: 0.5,
    y: 0.5,
    dx: 0,
    dy: 0,
    active: 0,
    down: 0,
  }
  let rafId = 0
  let lastMoveAt = 0
```
Replace with:
```js
  const pointers    = new Map()   // pointerId → {x, y, dx, dy, active, down}
  const lastMoveAt  = new Map()   // pointerId → timestamp
  let rafId = 0

  function getOrCreatePointer(id) {
    if (!pointers.has(id)) {
      pointers.set(id, { x: 0.5, y: 0.5, dx: 0, dy: 0, active: 0, down: 0 })
    }
    return pointers.get(id)
  }
```

- [ ] **Step 2: Update `updatePointer` to key by pointerId**

Replace `updatePointer` (lines 61-74):
```js
  function updatePointer(event) {
    const rect = canvas.getBoundingClientRect()
    if (!rect.width || !rect.height) return

    const nextX = Math.min(1, Math.max(0, (event.clientX - rect.left) / rect.width))
    const nextY = Math.min(1, Math.max(0, (event.clientY - rect.top) / rect.height))

    pointer.dx = nextX - pointer.x
    pointer.dy = nextY - pointer.y
    pointer.x = nextX
    pointer.y = nextY
    pointer.active = 1
    lastMoveAt = performance.now()
  }
```
with:
```js
  function updatePointer(event) {
    const rect = canvas.getBoundingClientRect()
    if (!rect.width || !rect.height) return

    const p = getOrCreatePointer(event.pointerId)
    const nextX = Math.min(1, Math.max(0, (event.clientX - rect.left) / rect.width))
    const nextY = Math.min(1, Math.max(0, (event.clientY - rect.top) / rect.height))

    p.dx = nextX - p.x
    p.dy = nextY - p.y
    p.x = nextX
    p.y = nextY
    p.active = 1
    lastMoveAt.set(event.pointerId, performance.now())
  }
```

- [ ] **Step 3: Update the `frame` loop**

Replace lines 76-101 (the `frame` function body):
```js
  function frame(now) {
    if (pointer.active && now - lastMoveAt > pointerIdleMs) {
      pointer.active = 0
    }

    // Inject pointer forces into the fluid sim
    if (sim && pointer.active) {
      const force = pointer.down ? pointerForceDown : pointerForce
      const densityAmt = pointer.down ? pointerDensityDown : pointerDensity
      sim.injectForce(pointer.x, pointer.y, pointer.dx * force, pointer.dy * force, densityAmt, pointerRadius)
    }

    // Step the simulation and upload to GPU
    if (sim) {
      sim.step()
      renderer.uploadFluid(sim.pixels, sim.cols, sim.rows)
    }

    // Update word bitmap each frame (cheap: small canvas + texture upload)
    const wordCanvas = wordCycler.update()
    renderer.uploadWordTexture(wordCanvas)

    pointer.dx *= pointerDeltaDecay
    pointer.dy *= pointerDeltaDecay
    renderer.draw(now, pointer)
    rafId = requestAnimationFrame(frame)
  }
```
with:
```js
  function frame(now) {
    // Expire idle pointers; clean up released ones
    for (const [id, p] of pointers) {
      if (p.active && now - (lastMoveAt.get(id) ?? 0) > pointerIdleMs) {
        p.active = 0
      }
    }

    // Inject forces from every active touch / pointer
    if (sim) {
      for (const p of pointers.values()) {
        if (!p.active) continue
        const force = p.down ? pointerForceDown : pointerForce
        const densityAmt = p.down ? pointerDensityDown : pointerDensity
        sim.injectForce(p.x, p.y, p.dx * force, p.dy * force, densityAmt, pointerRadius)
      }
    }

    // Step the simulation and upload to GPU
    if (sim) {
      sim.step()
      renderer.uploadFluid(sim.pixels, sim.cols, sim.rows)
    }

    // Update word bitmap each frame (cheap: small canvas + texture upload)
    const wordCanvas = wordCycler.update()
    renderer.uploadWordTexture(wordCanvas)

    // Decay delta for all pointers
    for (const p of pointers.values()) {
      p.dx *= pointerDeltaDecay
      p.dy *= pointerDeltaDecay
    }

    // Primary pointer for GPU glow = first active, or a silent default
    let primary = { x: 0.5, y: 0.5, dx: 0, dy: 0, active: 0, down: 0 }
    for (const p of pointers.values()) {
      if (p.active) { primary = p; break }
    }

    renderer.draw(now, primary)
    rafId = requestAnimationFrame(frame)
  }
```

- [ ] **Step 4: Update event handlers to key by pointerId**

Replace `handlePointerMove`, `handlePointerEnter`, `handlePointerLeave`, `handlePointerDown`, `handlePointerUp` (lines 113-134):
```js
  function handlePointerMove(event) {
    updatePointer(event)
  }

  function handlePointerEnter(event) {
    updatePointer(event)
    getOrCreatePointer(event.pointerId).active = 1
  }

  function handlePointerLeave(event) {
    const p = pointers.get(event.pointerId)
    if (p) { p.active = 0; p.down = 0 }
    pointers.delete(event.pointerId)
    lastMoveAt.delete(event.pointerId)
  }

  function handlePointerDown(event) {
    updatePointer(event)
    getOrCreatePointer(event.pointerId).down = 1
  }

  function handlePointerUp(event) {
    const p = pointers.get(event.pointerId)
    if (p) p.down = 0
  }
```

- [ ] **Step 5: Update README — Interaction section**

In `README.md`, replace the Interaction section:
```md
## Interaction

- **Move pointer** — injects forces into the fluid sim + adds shader glow
- **Click and drag** — amplifies forces and shifts palette warm
```
with:
```md
## Interaction

- **Move pointer** — injects forces into the fluid sim + adds shader glow
- **Click and drag** — amplifies forces and shifts palette warm
- **Multi-touch** — each simultaneous touch injects forces independently;
  the GPU glow effect renders for the primary (first active) touch only
```

- [ ] **Step 6: Verify**

Run `npm run dev`. Mouse still works. On a touchscreen (or Chrome DevTools touch emulation), two simultaneous drags should both disturb the fluid.

- [ ] **Step 7: Commit**

```bash
git add src/main.js README.md
git commit -m "feat: multi-touch support — all active pointers inject fluid forces"
```

---

### Task 5: Low-res mode (`?lowres` + auto-detect)

**Files:**
- Modify: `src/settings.js`
- Modify: `src/main.js`
- Modify: `README.md`

The CPU solver runs at full character-grid resolution. On low-end devices this can drop below 30 fps. This task adds a half-resolution simulation mode triggered by `?lowres` in the URL or auto-detected device conditions. The fluid texture is simply upscaled by the GPU — since fluid is soft, this looks fine.

- [ ] **Step 1: Add `lowResScale` to `settings.js`**

At the end of `src/settings.js`, add:
```js
// ── Low-res mode ─────────────────────────────────────────────────────────────
// When active, the fluid simulation runs at 1/lowResScale of the grid
// dimensions. The GPU upscales the texture — acceptable since fluid is soft.
export const lowResScale = 2   // divide grid by this factor when low-res is active
```

- [ ] **Step 2: Add low-res detection and grid scaling in `main.js`**

In `src/main.js`, add the import for `lowResScale` to the existing settings import on line 5:
```js
import { pointerForce, pointerForceDown, pointerDensity, pointerDensityDown, pointerRadius, pointerIdleMs, pointerDeltaDecay, lowResScale } from './settings.js'
```

Directly after the `boot()` function opens and before the `canvas` lookup, add:
```js
  // Low-res mode: ?lowres in URL, or auto-detected underpowered device.
  // Halves the fluid grid to reduce CPU solver load.
  const autoLowRes = (
    (typeof navigator.deviceMemory !== 'undefined' && navigator.deviceMemory < 4) ||
    (typeof navigator.hardwareConcurrency !== 'undefined' && navigator.hardwareConcurrency <= 2)
  )
  const isLowRes = new URLSearchParams(location.search).has('lowres') || autoLowRes
```

Update `handleResize` to pass scaled dimensions to the sim:
```js
  function handleResize() {
    const { cols, rows } = renderer.resize()
    const simCols = isLowRes ? Math.max(1, Math.ceil(cols / lowResScale)) : cols
    const simRows = isLowRes ? Math.max(1, Math.ceil(rows / lowResScale)) : rows
    if (!sim) {
      sim = createSimulation(simCols, simRows)
    } else {
      sim.resize(simCols, simRows)
    }
  }
```

- [ ] **Step 3: Update README**

Add a note to **Getting Started** or a new **Performance** section:
```md
## Performance

On low-end devices the CPU fluid solver may drop below 30 fps. Append `?lowres`
to the URL to halve the simulation grid resolution:

```
http://localhost:58707/?lowres
```

The app also auto-detects devices with fewer than 4 GB RAM or 2 CPU cores and
enables low-res mode automatically.
```

- [ ] **Step 4: Verify**

Open `http://localhost:58707/?lowres`. Art should be visible and frame rate improved. The fluid may look slightly softer/blockier — this is expected and acceptable.

- [ ] **Step 5: Commit**

```bash
git add src/settings.js src/main.js README.md
git commit -m "feat: low-res mode via ?lowres param and auto-detection for underpowered devices"
```

---

### Task 6: Feedback FBO — motion trails via `copyTexImage2D`

**Files:**
- Modify: `src/settings.js`
- Modify: `src/sketch.js`
- Modify: `src/renderer.js`
- Modify: `src/main.js`

Stores the rendered output each frame and blends it into the next, creating motion trails and ghosting. Uses `copyTexImage2D` to capture the drawn canvas into a texture — simpler than ping-pong FBOs, sufficient for an art context.

- [ ] **Step 1: Add `feedbackDecay` to `settings.js`**

```js
// ── Temporal feedback ─────────────────────────────────────────────────────────
export const feedbackDecay = 0.88  // fraction of prev frame blended into next (0 = off, 1 = permanent)
```

- [ ] **Step 2: Add `u_prevFrame` to `sketch.js` shader**

Add `feedbackDecay` to the `settings.js` import in `sketch.js`:
```js
import {
  fontSize, fontFamily, chars, densityChars,
  fieldTimeScale, fieldAmplitude,
  wordCanvasW, wordCanvasH,
  feedbackDecay,
} from './settings.js'
```

Add `feedbackDecay` to `staticUniforms`:
```js
export const staticUniforms = {
  u_fieldTimeScale:   fieldTimeScale,
  u_fieldAmplitude:   fieldAmplitude,
  u_wordAspect:       wordCanvasH / wordCanvasW,
  u_densityCharCount: densityChars.length,
  u_feedbackDecay:    feedbackDecay,
}
```

In the fragment shader, add the new uniform declarations (after `u_densityCharCount`):
```glsl
uniform sampler2D u_prevFrame;    // previous rendered frame for temporal feedback
uniform float     u_feedbackDecay; // fraction of prev frame to carry forward
```

In `main()`, replace the final line:
```glsl
  gl_FragColor = vec4(rgb * alpha, 1.0);
```
with:
```glsl
  vec2  prevUV  = gl_FragCoord.xy / u_resolution;
  vec3  prev    = texture2D(u_prevFrame, prevUV).rgb;
  vec3  trail   = prev * u_feedbackDecay;
  vec3  lit     = rgb * alpha;
  gl_FragColor  = vec4(max(lit, trail), 1.0);
```

- [ ] **Step 3: Add `u_prevFrame` to `renderer.js`**

Add `'u_prevFrame'` to `UNIFORM_NAMES`.

Add a `prevFrameTex` that `copyTexImage2D` will fill each frame. After the `wordTex` setup block (around line 169), add:
```js
  // Previous-frame texture — filled each frame via copyTexImage2D after draw
  let prevFrameTex = gl.createTexture()
  gl.bindTexture(gl.TEXTURE_2D, prevFrameTex)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE)
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE)
  // Allocate at 1×1 initially; resize() will reallocate to canvas size
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, 1, 1, 0, gl.RGBA, gl.UNSIGNED_BYTE, null)

  function bindPrevFrame() {
    gl.useProgram(program)
    gl.activeTexture(gl.TEXTURE3)
    gl.bindTexture(gl.TEXTURE_2D, prevFrameTex)
    gl.uniform1i(u.u_prevFrame, 3)
  }
```

Call `bindPrevFrame()` alongside the other bind calls (after `bindWord()`):
```js
  bindAtlas()
  bindFluid()
  bindWord()
  bindPrevFrame()
```

In `resize()`, after setting the viewport, reallocate the prev-frame texture to match canvas size:
```js
      // Reallocate prevFrameTex to match new canvas dimensions
      gl.activeTexture(gl.TEXTURE3)
      gl.bindTexture(gl.TEXTURE_2D, prevFrameTex)
      gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, canvas.width, canvas.height, 0, gl.RGBA, gl.UNSIGNED_BYTE, null)
```

Add a `copyPrevFrame()` method to the returned object (alongside `draw`, `uploadFluid`, etc.):
```js
    copyPrevFrame() {
      gl.activeTexture(gl.TEXTURE3)
      gl.bindTexture(gl.TEXTURE_2D, prevFrameTex)
      gl.copyTexImage2D(gl.TEXTURE_2D, 0, gl.RGBA, 0, 0, canvas.width, canvas.height, 0)
    },
```

In `dispose()`, add:
```js
      gl.deleteTexture(prevFrameTex)
```

In `recompile()`, add `bindPrevFrame()` call alongside the others:
```js
      bindAtlas()
      bindFluid()
      bindWord()
      bindPrevFrame()
```

- [ ] **Step 4: Call `copyPrevFrame` in `main.js` after each draw**

In `src/main.js`, after `renderer.draw(now, primary)`:
```js
    renderer.draw(now, primary)
    renderer.copyPrevFrame()
    rafId = requestAnimationFrame(frame)
```

- [ ] **Step 5: Add `u_feedbackDecay` to `UNIFORM_NAMES` in `renderer.js`**

```js
  'u_feedbackDecay',
```

- [ ] **Step 6: Verify visually**

Run `npm run dev`. Characters should leave faint trails when the fluid moves through them. Set `feedbackDecay = 0` in `settings.js` to verify trails disappear (effectively disables feedback). Revert to `0.88`.

- [ ] **Step 7: Commit**

```bash
git add src/settings.js src/sketch.js src/renderer.js src/main.js
git commit -m "feat: temporal feedback via copyTexImage2D — motion trails and ghosting"
```

---

## Tier 3 — Lower priority / higher complexity

---

### Task 7: Screenshot and GIF/video export

**Files:**
- Modify: `src/main.js`
- Modify: `README.md`

Pressing **S** saves a PNG. Pressing **R** toggles recording a `.webm` clip via `MediaRecorder`.

- [ ] **Step 1: Add screenshot handler in `main.js`**

Inside `boot()`, after the `handlePointerUp` definition, add:
```js
  function handleKeydown(event) {
    if (event.key === 's' || event.key === 'S') {
      canvas.toBlob((blob) => {
        if (!blob) return
        const url = URL.createObjectURL(blob)
        const a = document.createElement('a')
        a.href = url
        a.download = `flux-${Date.now()}.png`
        a.click()
        URL.revokeObjectURL(url)
      }, 'image/png')
    }

    if (event.key === 'r' || event.key === 'R') {
      toggleRecording()
    }
  }
```

- [ ] **Step 2: Add recording state and `toggleRecording` in `main.js`**

Add these variables alongside `rafId`:
```js
  let mediaRecorder = null
  let recordedChunks = []
```

Add the `toggleRecording` function:
```js
  function toggleRecording() {
    if (mediaRecorder && mediaRecorder.state === 'recording') {
      mediaRecorder.stop()
      return
    }

    const stream = canvas.captureStream(60)
    mediaRecorder = new MediaRecorder(stream, { mimeType: 'video/webm' })
    recordedChunks = []

    mediaRecorder.ondataavailable = (e) => {
      if (e.data.size > 0) recordedChunks.push(e.data)
    }

    mediaRecorder.onstop = () => {
      const blob = new Blob(recordedChunks, { type: 'video/webm' })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = `flux-${Date.now()}.webm`
      a.click()
      URL.revokeObjectURL(url)
      recordedChunks = []
    }

    mediaRecorder.start()
  }
```

- [ ] **Step 3: Register and clean up the keydown listener**

Add to the event listener registration block:
```js
  window.addEventListener('keydown', handleKeydown)
```

Add to the `import.meta.hot.dispose` cleanup block:
```js
      window.removeEventListener('keydown', handleKeydown)
      if (mediaRecorder && mediaRecorder.state === 'recording') mediaRecorder.stop()
```

- [ ] **Step 4: Update README Interaction section**

```md
## Interaction

- **Move pointer** — injects forces into the fluid sim + adds shader glow
- **Click and drag** — amplifies forces and shifts palette warm
- **Multi-touch** — each simultaneous touch injects forces independently;
  the GPU glow effect renders for the primary (first active) touch only
- **S** — save a PNG screenshot
- **R** — start / stop recording a `.webm` clip
```

- [ ] **Step 5: Verify**

Run `npm run dev`. Press **S** — a PNG should download. Press **R**, drag around for a few seconds, press **R** again — a `.webm` should download and be playable.

- [ ] **Step 6: Commit**

```bash
git add src/main.js README.md
git commit -m "feat: screenshot (S) and webm recording (R) via canvas.toBlob / MediaRecorder"
```

---

### Task 8: GPU-only solver — WebGPU compute shaders (future milestone)

**Files:**
- Create: `src/simulation-gpu.js`
- Modify: `src/main.js` (feature-detect and swap in gpu solver)

> **Note:** WebGPU is available in Chrome 113+ / Edge 113+. Firefox support is behind a flag. This task adds a WebGPU compute solver that runs in parallel with the CPU solver as a drop-in replacement, gated by feature detection. The CPU solver remains the fallback.

The WebGPU solver uses a ping-pong texture pair and compute shaders for advect, diffuse, and project. It eliminates the CPU→GPU transfer bottleneck (no `copyTexImage2D` each frame) and allows much larger grid sizes.

- [ ] **Step 1: Feature-detect WebGPU support in `main.js`**

In `boot()`, before creating the renderer:
```js
  const hasWebGPU = typeof navigator.gpu !== 'undefined'
  // Future: if (hasWebGPU) { sim = await createGpuSimulation(cols, rows) }
  // For now, always fall through to CPU sim below.
```

- [ ] **Step 2: Create `src/simulation-gpu.js` with the public API contract**

```js
// GPU fluid simulation using WebGPU compute shaders.
// Drop-in replacement for simulation.js — exposes the same state interface:
//   { cols, rows, pixels, resize(c, r), injectForce(nx, ny, fx, fy, d, r), step() }
//
// Differences from CPU solver:
//   - `pixels` is a GPUBuffer, not a Uint8Array — use uploadFluidFromGPU() on the renderer
//   - `step()` is async (submits to GPU queue)

export async function createGpuSimulation(cols, rows) {
  if (!navigator.gpu) throw new Error('WebGPU not available')
  const adapter = await navigator.gpu.requestAdapter()
  if (!adapter) throw new Error('No WebGPU adapter found')
  const device = await adapter.requestDevice()

  // TODO: implement advect/diffuse/project as WGSL compute shaders
  // See roadmap in docs/superpowers/plans/webgpu-solver.md (to be written)

  throw new Error('GPU solver not yet implemented — use CPU solver')
}
```

- [ ] **Step 3: Commit the stub**

```bash
git add src/simulation-gpu.js src/main.js
git commit -m "feat(stub): WebGPU solver scaffold with feature detection — CPU solver remains default"
```

---

## Self-review

**Spec coverage:**
- [x] Document resize() constraint → Task 1
- [x] Verify wordScaleY intent → Task 2
- [x] Convert baked shader constants to uniforms → Task 3
- [x] Multi-touch support → Task 4
- [x] Note multi-touch in README → Task 4 Step 5
- [x] Low-res ?lowres + auto-detect → Task 5
- [x] Feedback FBO → Task 6
- [x] Screenshot/GIF export → Task 7
- [x] GPU-only solver (stub) → Task 8

**Placeholder scan:** No TBD, TODO, or vague steps outside Task 8's intentional stub.

**Type consistency:** All uniform names used in shader (Task 3/6) match the names added to `UNIFORM_NAMES` in `renderer.js`. `staticUniforms` keys in `sketch.js` match the GLSL `uniform` declarations. `copyPrevFrame` method added in Task 6 is called in `main.js` Task 6 Step 4. `handleKeydown` registered and cleaned up in Task 7.
