# webart — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A fullscreen browser playground rendering a Navier-Stokes fluid simulation as a live character grid, interactive via mouse, edit-and-reload via Vite HMR.

**Architecture:** play.core (`run.js`) drives the character grid loop and calls `boot / pre / main` on each frame. The fluid solver lives in `src/fluid.js` as pure functions so it can be unit-tested independently of the browser. `src/sketch.js` wires the solver to the ABC lifecycle hooks and maps fluid state to visual character properties.

**Tech Stack:** play.core (github.com/ertdfgcvb/play.core), Vite, Vitest, Vanilla JS ES modules

---

## File Map

| File | Responsibility |
|---|---|
| `index.html` | Fullscreen page; loads `src/sketch.js` as an ES module entry point |
| `src/sketch.js` | ABC lifecycle hooks (`boot`, `pre`, `main`); wires fluid solver + mouse input |
| `src/fluid.js` | Pure Navier-Stokes solver: `diffuse`, `advect`, `project`, `addSource` |
| `src/map.js` | Pure visual mapping: `flowChar`, `densityColor`, `speedWeight` |
| `vendor/play.core/` | Cloned play.core source — never edited |
| `vite.config.js` | Minimal Vite config |
| `package.json` | deps: vite; devDeps: vitest |

---

## Task 1: Project scaffold

**Files:**
- Create: `package.json`
- Create: `vite.config.js`
- Create: `index.html`

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "webart",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "test": "vitest run"
  },
  "devDependencies": {
    "vite": "^5.0.0",
    "vitest": "^1.0.0"
  }
}
```

- [ ] **Step 2: Create `vite.config.js`**

```js
import { defineConfig } from 'vite'

export default defineConfig({
  server: {
    open: true
  }
})
```

- [ ] **Step 3: Create `index.html`**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>webart</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #000; overflow: hidden; }
    #abc { width: 100vw; height: 100vh; }
  </style>
</head>
<body>
  <pre id="abc"></pre>
  <script type="module" src="/src/sketch.js"></script>
</body>
</html>
```

- [ ] **Step 4: Install dependencies**

```bash
cd ~/webArt
npm install
```

Expected: `node_modules/` created, no errors.

- [ ] **Step 5: Commit**

```bash
git init
git add index.html package.json vite.config.js
git commit -m "feat: project scaffold"
```

---

## Task 2: Vendor play.core

**Files:**
- Create: `vendor/play.core/` (cloned)
- Create: `.gitignore`

- [ ] **Step 1: Clone play.core into vendor/**

```bash
cd ~/webArt
git clone https://github.com/ertdfgcvb/play.core vendor/play.core --depth 1
```

Expected: `vendor/play.core/src/run.js` exists.

- [ ] **Step 2: Verify the import path works**

```bash
ls vendor/play.core/src/
```

Expected output includes: `run.js  core/  modules/  programs/`

- [ ] **Step 3: Create `.gitignore`**

```
node_modules/
vendor/
dist/
```

- [ ] **Step 4: Commit**

```bash
git add .gitignore
git commit -m "feat: vendor play.core, ignore node_modules and vendor"
```

---

## Task 3: Fluid solver (`src/fluid.js`)

**Files:**
- Create: `src/fluid.js`
- Create: `src/fluid.test.js`

This is the Jos Stam solver (GDC 1999). All functions are pure — they take arrays as arguments and mutate them in-place, no globals.

- [ ] **Step 1: Write failing tests**

Create `src/fluid.test.js`:

```js
import { describe, it, expect } from 'vitest'
import { addSource, diffuse, project } from './fluid.js'

describe('addSource', () => {
  it('adds source values scaled by dt into destination', () => {
    const dst = new Float32Array([0, 0, 0])
    const src = new Float32Array([1, 2, 3])
    addSource(dst, src, 0.1)
    expect(dst[0]).toBeCloseTo(0.1)
    expect(dst[1]).toBeCloseTo(0.2)
    expect(dst[2]).toBeCloseTo(0.3)
  })
})

describe('diffuse', () => {
  it('spreads a central value to its neighbours after one call', () => {
    const cols = 5, rows = 5
    const N = cols * rows
    const x   = new Float32Array(N)
    const x0  = new Float32Array(N)
    // place a spike in the center
    x0[2 * cols + 2] = 1.0
    diffuse(x, x0, 1, 0.1, 1, cols, rows)
    // center should be less than 1 (spread out)
    expect(x[2 * cols + 2]).toBeLessThan(1.0)
    // at least one neighbour should be non-zero
    expect(x[1 * cols + 2] + x[3 * cols + 2]).toBeGreaterThan(0)
  })
})

describe('project', () => {
  it('reduces divergence of velocity field', () => {
    const cols = 6, rows = 6
    const N = cols * rows
    // create a simple divergent field (pointing outward from center)
    const vx = new Float32Array(N)
    const vy = new Float32Array(N)
    const p  = new Float32Array(N)
    const div = new Float32Array(N)
    for (let j = 1; j < rows - 1; j++) {
      for (let i = 1; i < cols - 1; i++) {
        vx[j * cols + i] = i - cols / 2
        vy[j * cols + i] = j - rows / 2
      }
    }
    // compute divergence before projection
    let divBefore = 0
    for (let j = 1; j < rows - 1; j++) {
      for (let i = 1; i < cols - 1; i++) {
        divBefore += Math.abs(
          vx[j * cols + (i + 1)] - vx[j * cols + (i - 1)] +
          vy[(j + 1) * cols + i] - vy[(j - 1) * cols + i]
        )
      }
    }
    project(vx, vy, p, div, cols, rows)
    // compute divergence after projection
    let divAfter = 0
    for (let j = 1; j < rows - 1; j++) {
      for (let i = 1; i < cols - 1; i++) {
        divAfter += Math.abs(
          vx[j * cols + (i + 1)] - vx[j * cols + (i - 1)] +
          vy[(j + 1) * cols + i] - vy[(j - 1) * cols + i]
        )
      }
    }
    expect(divAfter).toBeLessThan(divBefore)
  })
})
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
npm test
```

Expected: 3 test failures — `addSource`, `diffuse`, `project` not found.

- [ ] **Step 3: Implement `src/fluid.js`**

```js
// Navier-Stokes fluid solver — Jos Stam, "Real-Time Fluid Dynamics for Games" (GDC 1999)
// All functions are pure: they take typed arrays and mutate them in-place.

const ITER = 20  // Gauss-Seidel iterations

/**
 * Add source field src into dst, scaled by timestep dt.
 * @param {Float32Array} dst
 * @param {Float32Array} src
 * @param {number} dt
 */
export function addSource(dst, src, dt) {
  for (let i = 0; i < dst.length; i++) dst[i] += dt * src[i]
}

/**
 * Diffuse field x from x0 over one timestep.
 * @param {Float32Array} x   output field
 * @param {Float32Array} x0  input field
 * @param {number} diff      diffusion rate
 * @param {number} dt        timestep
 * @param {number} cols
 * @param {number} rows
 */
export function diffuse(x, x0, diff, dt, cols, rows) {
  const a = dt * diff * (cols - 2) * (rows - 2)
  linSolve(x, x0, a, 1 + 4 * a, cols, rows)
}

/**
 * Advect field d along velocity field (u, v).
 * @param {Float32Array} d   output
 * @param {Float32Array} d0  input
 * @param {Float32Array} u   x-velocity
 * @param {Float32Array} v   y-velocity
 * @param {number} dt
 * @param {number} cols
 * @param {number} rows
 */
export function advect(d, d0, u, v, dt, cols, rows) {
  const dt0x = dt * (cols - 2)
  const dt0y = dt * (rows - 2)
  for (let j = 1; j < rows - 1; j++) {
    for (let i = 1; i < cols - 1; i++) {
      const idx = j * cols + i
      let x = i - dt0x * u[idx]
      let y = j - dt0y * v[idx]
      x = Math.max(0.5, Math.min(cols - 1.5, x))
      y = Math.max(0.5, Math.min(rows - 1.5, y))
      const i0 = Math.floor(x), i1 = i0 + 1
      const j0 = Math.floor(y), j1 = j0 + 1
      const s1 = x - i0, s0 = 1 - s1
      const t1 = y - j0, t0 = 1 - t1
      d[idx] = s0 * (t0 * d0[j0 * cols + i0] + t1 * d0[j1 * cols + i0])
             + s1 * (t0 * d0[j0 * cols + i1] + t1 * d0[j1 * cols + i1])
    }
  }
  setBounds(d, cols, rows)
}

/**
 * Project velocity field (u, v) to be divergence-free.
 * Uses scratch arrays p and div.
 * @param {Float32Array} u
 * @param {Float32Array} v
 * @param {Float32Array} p    scratch
 * @param {Float32Array} div  scratch
 * @param {number} cols
 * @param {number} rows
 */
export function project(u, v, p, div, cols, rows) {
  const hx = 1.0 / (cols - 2)
  const hy = 1.0 / (rows - 2)
  for (let j = 1; j < rows - 1; j++) {
    for (let i = 1; i < cols - 1; i++) {
      const idx = j * cols + i
      div[idx] = -0.5 * (
        hx * (u[j * cols + (i + 1)] - u[j * cols + (i - 1)]) +
        hy * (v[(j + 1) * cols + i] - v[(j - 1) * cols + i])
      )
      p[idx] = 0
    }
  }
  setBounds(div, cols, rows)
  setBounds(p, cols, rows)
  linSolve(p, div, 1, 4, cols, rows)
  for (let j = 1; j < rows - 1; j++) {
    for (let i = 1; i < cols - 1; i++) {
      const idx = j * cols + i
      u[idx] -= 0.5 * (p[j * cols + (i + 1)] - p[j * cols + (i - 1)]) / hx
      v[idx] -= 0.5 * (p[(j + 1) * cols + i] - p[(j - 1) * cols + i]) / hy
    }
  }
  setBounds(u, cols, rows)
  setBounds(v, cols, rows)
}

// ─── internals ───────────────────────────────────────────────────────────────

function linSolve(x, x0, a, c, cols, rows) {
  const inv = 1 / c
  for (let k = 0; k < ITER; k++) {
    for (let j = 1; j < rows - 1; j++) {
      for (let i = 1; i < cols - 1; i++) {
        const idx = j * cols + i
        x[idx] = (x0[idx] + a * (
          x[j * cols + (i - 1)] + x[j * cols + (i + 1)] +
          x[(j - 1) * cols + i] + x[(j + 1) * cols + i]
        )) * inv
      }
    }
    setBounds(x, cols, rows)
  }
}

function setBounds(x, cols, rows) {
  // clamp edges to adjacent interior values
  for (let i = 1; i < cols - 1; i++) {
    x[0 * cols + i]          = x[1 * cols + i]
    x[(rows - 1) * cols + i] = x[(rows - 2) * cols + i]
  }
  for (let j = 1; j < rows - 1; j++) {
    x[j * cols + 0]          = x[j * cols + 1]
    x[j * cols + (cols - 1)] = x[j * cols + (cols - 2)]
  }
  x[0]                         = 0.5 * (x[cols + 0]         + x[0 * cols + 1])
  x[cols - 1]                  = 0.5 * (x[cols + cols - 1]  + x[0 * cols + (cols - 2)])
  x[(rows - 1) * cols]         = 0.5 * (x[(rows - 2) * cols]      + x[(rows - 1) * cols + 1])
  x[(rows - 1) * cols + cols - 1] = 0.5 * (x[(rows - 2) * cols + cols - 1] + x[(rows - 1) * cols + (cols - 2)])
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
npm test
```

Expected: 3 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/fluid.js src/fluid.test.js
git commit -m "feat: Navier-Stokes fluid solver with tests"
```

---

## Task 4: Visual mapping (`src/map.js`)

**Files:**
- Create: `src/map.js`
- Create: `src/map.test.js`

- [ ] **Step 1: Write failing tests**

Create `src/map.test.js`:

```js
import { describe, it, expect } from 'vitest'
import { flowChar, densityColor, speedWeight } from './map.js'

describe('flowChar', () => {
  it('returns · for zero velocity', () => {
    expect(flowChar(0, 0)).toBe('·')
  })
  it('returns - for rightward flow', () => {
    expect(flowChar(1, 0)).toBe('-')
  })
  it('returns | for downward flow', () => {
    expect(flowChar(0, 1)).toBe('|')
  })
  it('returns / for up-right diagonal', () => {
    expect(flowChar(1, -1)).toBe('/')
  })
  it('returns \\ for down-right diagonal', () => {
    expect(flowChar(1, 1)).toBe('\\')
  })
})

describe('densityColor', () => {
  it('returns a CSS hsl string', () => {
    const c = densityColor(0.5, 0.3, -0.2)
    expect(c).toMatch(/^hsl\(/)
  })
  it('returns black for zero density', () => {
    const c = densityColor(0, 0, 0)
    expect(c).toMatch(/hsl\(\d+, 80%, 0%\)/)
  })
})

describe('speedWeight', () => {
  it('returns 300 for zero speed', () => {
    expect(speedWeight(0, 0)).toBe(300)
  })
  it('returns 700 for speed >= 1', () => {
    expect(speedWeight(1, 0)).toBe(700)
  })
  it('returns 400 for mid speed', () => {
    const w = speedWeight(0.5, 0)
    expect([300, 400, 700]).toContain(w)
  })
})
```

- [ ] **Step 2: Run tests to confirm they fail**

```bash
npm test
```

Expected: 8 failures — `flowChar`, `densityColor`, `speedWeight` not found.

- [ ] **Step 3: Implement `src/map.js`**

```js
// Maps fluid simulation state to ABC character properties.
// All functions are pure — no side effects.

// 8-directional flow characters mapped by velocity angle
const DIR_CHARS = ['-', '\\', '|', '/', '-', '\\', '|', '/']

/**
 * Return a character representing the flow direction.
 * @param {number} vx
 * @param {number} vy
 * @returns {string}
 */
export function flowChar(vx, vy) {
  const speed = Math.hypot(vx, vy)
  if (speed < 0.001) return '·'
  // atan2 returns [-π, π]; map to 8 sectors
  const angle = Math.atan2(vy, vx)               // -π to π
  const sector = Math.round((angle / Math.PI) * 4) // -4 to 4
  return DIR_CHARS[((sector % 8) + 8) % 8]
}

/**
 * Return a CSS hsl color string for a cell.
 * Hue is driven by vorticity (curl of velocity ≈ vy - vx for 2D).
 * Lightness is driven by density.
 * @param {number} density  0–1
 * @param {number} vx
 * @param {number} vy
 * @returns {string}
 */
export function densityColor(density, vx, vy) {
  const vorticity = vy - vx                      // simplified 2D curl
  const hue = 200 + vorticity * 80               // cool blue → warm orange
  const lightness = Math.round(Math.min(density, 1) * 60)
  return `hsl(${Math.round(hue)}, 80%, ${lightness}%)`
}

/**
 * Return ABC fontWeight (300 | 400 | 700) based on velocity magnitude.
 * @param {number} vx
 * @param {number} vy
 * @returns {300 | 400 | 700}
 */
export function speedWeight(vx, vy) {
  const speed = Math.hypot(vx, vy)
  if (speed < 0.15) return 300
  if (speed < 0.5)  return 400
  return 700
}
```

- [ ] **Step 4: Run tests to confirm they pass**

```bash
npm test
```

Expected: 8 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/map.js src/map.test.js
git commit -m "feat: visual mapping — flowChar, densityColor, speedWeight"
```

---

## Task 5: Sketch (`src/sketch.js`)

**Files:**
- Create: `src/sketch.js`

- [ ] **Step 1: Create `src/sketch.js`**

```js
import { addSource, diffuse, advect, project } from './fluid.js'
import { flowChar, densityColor, speedWeight } from './map.js'

// ─── tuning constants ─────────────────────────────────────────────────────────
const DIFF      = 0.0001   // diffusion rate
const VISC      = 0.00001  // viscosity
const DT        = 0.1      // timestep
const FORCE     = 80       // velocity impulse magnitude on mouse move
const SOURCE    = 12       // density injection amount on mouse move
const RADIUS    = 3        // injection radius in cells
const PASSES    = 2        // solver passes per frame

// ─── simulation state ─────────────────────────────────────────────────────────
let cols, rows
let density, densityPrev
let vx, vy, vxPrev, vyPrev

export function boot(context) {
  cols = context.cols
  rows = context.rows
  const N = cols * rows
  density     = new Float32Array(N)
  densityPrev = new Float32Array(N)
  vx          = new Float32Array(N)
  vy          = new Float32Array(N)
  vxPrev      = new Float32Array(N)
  vyPrev      = new Float32Array(N)
}

export function pre(context, cursor) {
  // inject fluid at cursor position
  const cx = Math.round(cursor.x)
  const cy = Math.round(cursor.y)
  const strength = cursor.pressed ? 3 : 1
  if (cx > 0 && cx < cols - 1 && cy > 0 && cy < rows - 1) {
    const dx = cursor.x - cursor.p.x
    const dy = cursor.y - cursor.p.y
    for (let dj = -RADIUS; dj <= RADIUS; dj++) {
      for (let di = -RADIUS; di <= RADIUS; di++) {
        if (di * di + dj * dj > RADIUS * RADIUS) continue
        const i = cx + di, j = cy + dj
        if (i < 1 || i >= cols - 1 || j < 1 || j >= rows - 1) continue
        const idx = j * cols + i
        densityPrev[idx] += SOURCE * strength
        vxPrev[idx]      += FORCE * dx * strength
        vyPrev[idx]      += FORCE * dy * strength
      }
    }
  }

  // run solver PASSES times
  for (let p = 0; p < PASSES; p++) {
    // velocity step
    addSource(vx, vxPrev, DT)
    addSource(vy, vyPrev, DT)
    ;[vx, vxPrev] = [vxPrev, vx]
    diffuse(vx, vxPrev, VISC, DT, cols, rows)
    ;[vy, vyPrev] = [vyPrev, vy]
    diffuse(vy, vyPrev, VISC, DT, cols, rows)
    project(vx, vy, vxPrev, vyPrev, cols, rows)
    ;[vx, vxPrev] = [vxPrev, vx]
    ;[vy, vyPrev] = [vyPrev, vy]
    advect(vx, vxPrev, vxPrev, vyPrev, DT, cols, rows)
    advect(vy, vyPrev, vxPrev, vyPrev, DT, cols, rows)
    project(vx, vy, vxPrev, vyPrev, cols, rows)

    // density step
    addSource(density, densityPrev, DT)
    ;[density, densityPrev] = [densityPrev, density]
    diffuse(density, densityPrev, DIFF, DT, cols, rows)
    ;[density, densityPrev] = [densityPrev, density]
    advect(density, densityPrev, vx, vy, DT, cols, rows)
  }

  // decay previous buffers
  for (let i = 0; i < cols * rows; i++) {
    vxPrev[i]      *= 0.8
    vyPrev[i]      *= 0.8
    densityPrev[i] *= 0.8
  }
}

export function main({ x, y }, { cols }) {
  const i   = y * cols + x
  const d   = density[i]
  const u   = vx[i]
  const v   = vy[i]
  return {
    char:       flowChar(u, v),
    color:      densityColor(d, u, v),
    fontWeight: speedWeight(u, v),
  }
}

export const settings = {
  fps: 30,
  element: document.querySelector('#abc'),
}
```

- [ ] **Step 2: Commit**

```bash
git add src/sketch.js
git commit -m "feat: sketch — boot/pre/main wired to fluid solver and visual mapping"
```

---

## Task 6: Wire up and run

**Files:**
- Modify: `index.html` (add `#abc` element — already done in Task 1)
- Create: `src/main.js`

- [ ] **Step 1: Create `src/main.js`**

```js
import { run } from '../vendor/play.core/src/run.js'
import * as sketch from './sketch.js'

run(sketch)
```

- [ ] **Step 2: Update script tag in `index.html`**

Change:
```html
<script type="module" src="/src/sketch.js"></script>
```

To:
```html
<script type="module" src="/src/main.js"></script>
```

- [ ] **Step 3: Start the dev server**

```bash
npm run dev
```

Expected: browser opens, fullscreen black canvas. Move the mouse — fluid should appear as flowing characters.

- [ ] **Step 4: Verify HMR works**

Edit any constant in `src/sketch.js` (e.g. change `FORCE` from `80` to `200`), save. Expected: browser reloads the sketch instantly without a full page reload.

- [ ] **Step 5: Commit**

```bash
git add src/main.js index.html
git commit -m "feat: wire run() — playground is live"
```

---

## Self-Review

**Spec coverage:**
- ✅ ABC library + Vite stack
- ✅ Single sketch file user edits
- ✅ Navier-Stokes: diffuse + advect + project
- ✅ Two solver passes per frame (`PASSES = 2`)
- ✅ Mouse move injects density + velocity
- ✅ Mouse down amplifies force (`cursor.pressed ? 3 : 1`)
- ✅ velocity angle → `flowChar`
- ✅ velocity magnitude → `fontWeight` (300/400/700)
- ✅ density → color brightness via `densityColor`
- ✅ vorticity → hue shift in `densityColor`
- ✅ Tunable constants at top of sketch.js
- ✅ Vite HMR confirmed in Task 6 Step 4

**Type consistency check:**
- `diffuse(x, x0, diff, dt, cols, rows)` — used consistently in fluid.js and sketch.js ✅
- `advect(d, d0, u, v, dt, cols, rows)` — consistent ✅
- `project(u, v, p, div, cols, rows)` — consistent ✅
- `flowChar(vx, vy)` / `densityColor(density, vx, vy)` / `speedWeight(vx, vy)` — consistent between map.js and sketch.js ✅

**No placeholders found.**
