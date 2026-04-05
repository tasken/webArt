# ertdfgcvb.xyz — Technical Analysis
**Subject:** `https://ertdfgcvb.xyz/?mode=screensaver`
**Author:** Andreas Gysin ([@andreasgysin](https://twitter.com/andreasgysin))
**Engine:** [play.core](https://github.com/ertdfgcvb/play.core) (open source, Apache 2.0)
**Date of analysis:** 2026-04-05

---

## Overview

ertdfgcvb.xyz is a generative text-mode art platform. In `?mode=screensaver` the navigation is hidden and the sketches run fullscreen, uninterrupted. Every visual on the site is produced by the same underlying engine — **play.core** — a browser-based character-grid renderer that treats each character cell the way a GPU treats a fragment: as an independent evaluation site for a pure function of position and time.

The system is simultaneously an art engine, a live-coding environment, and a publishing platform. Screensaver mode is simply the engine running with the chrome stripped away.

---

## Core Paradigm: The Cell as Fragment

The defining design decision of play.core is this: **instead of painting geometric primitives, you write a single function that is invoked for every cell.** This mirrors GLSL fragment shaders — `main(coord, context, cursor, buffer)` receives `{x, y, index}` and returns a character (or a styled cell object). The engine calls it for every cell, every frame.

```javascript
// Minimal program — fills the grid with a scrolling wave of dots
export function main(coord, context) {
    const t = context.time * 0.001
    const v = Math.sin(coord.x * 0.3 + coord.y * 0.5 + t)
    return v > 0 ? '.' : ' '
}
```

A full browser window at typical font sizes holds 5,000–8,000 cells. Each frame `main()` is called that many times — sequentially, on the CPU. This constraint shapes every design choice: effects must be cheap per-cell, global state is managed in `pre()`, and the buffer is the communication channel between phases.

---

## Engine Architecture: play.core

### Module Structure

```
src/
├── run.js                  ← program runner (the engine)
├── core/
│   ├── textrenderer.js     ← DOM <span> renderer
│   ├── canvasrenderer.js   ← Canvas 2D renderer
│   ├── fps.js              ← precise FPS tracking
│   └── storage.js          ← localStorage wrapper
└── modules/
    ├── num.js              ← GLSL math: map, mix, smoothstep, clamp…
    ├── color.js            ← CSS1/3/4, C64, CGA palettes + converters
    ├── vec2.js / vec3.js   ← vector math
    ├── sdf.js              ← signed distance functions
    ├── image.js            ← image loading + sampling
    ├── camera.js           ← webcam input
    └── drawbox.js          ← text-box overlay helpers
```

### The Four Callbacks

Programs export up to four functions, called in this order each frame:

| Export | Frequency | Purpose |
|--------|-----------|---------|
| `boot(context, buffer, data)` | Once at startup | Initialize state; access metrics before rendering |
| `pre(context, cursor, buffer, data)` | Once per frame | Prepare global frame data; run simulations; clear/seed buffer |
| `main(coord, context, cursor, buffer, data)` | Per cell | Core logic; return char or `{char, color, backgroundColor, fontWeight}` |
| `post(context, cursor, buffer, data)` | Once per frame | Overlays, HUD, buffer post-processing |

This maps cleanly to a graphics pipeline: `pre` is the vertex/uniform stage, `main` is the fragment stage, `post` is the compositing stage.

### The `context` Object (Immutable Per Frame)

```javascript
context = Object.freeze({
    frame   : 42,          // integer frame counter
    time    : 1400,        // ms since start (persists across live-code cycles)
    cols    : 180,         // character columns
    rows    : 45,          // character rows
    width   : 1440,        // container px width
    height  : 900,         // container px height
    metrics : {
        cellWidth   : 8.0,    // px per character (fractional)
        lineHeight  : 20.0,   // px per row
        aspect      : 0.4,    // cellWidth / lineHeight
        fontFamily  : 'monospace',
        fontSize    : 16
    },
    settings : { ... },    // merged program + runner settings
    runtime  : { fps, cycle }
})
```

### The `cursor` Object

```javascript
cursor = {
    x       : 42.7,   // fractional cell column (pointer pixel / cellWidth)
    y       : 12.1,   // fractional cell row
    pressed : true,
    p : {             // previous frame snapshot
        x, y, pressed
    }
}
```

### The Cell Buffer

The buffer is a flat array of cell objects, length `cols × rows`, laid out in row-major order:

```javascript
buffer[y * cols + x] = {
    char            : 'A',
    color           : 'royalblue',
    backgroundColor : 'black',
    fontWeight      : 700        // 300, 400, or 700
}
```

The buffer is **not automatically cleared** between frames. Programs must manage their own clearing in `pre()`, or use persistence intentionally (e.g. for trails). When `main()` returns a plain string, only `char` is updated; style properties carry over from the previous frame.

---

## The Animation Loop

```javascript
function loop(t) {
    const delta = t - timeSample
    if (delta < interval) {
        // Frame skipped — below fps cap
        requestAnimationFrame(loop)
        return
    }

    timeSample = t - delta % interval  // remove accumulated drift
    state.time = t + timeOffset        // timeOffset restores time after live-code reload
    state.frame++

    // 1. Resize / reinitialize buffer if cols/rows changed
    // 2. call pre()
    // 3. call main() for every cell
    // 4. call post()
    // 5. renderer.render(context, buffer, settings)
    // 6. flush queued pointer events → call program.pointerMove/Down/Up() if defined
    // 7. requestAnimationFrame(loop)
}
```

Key subtlety: **timeSample drift correction** (`t - delta % interval`) prevents the effective fps from drifting below the cap over time due to rAF jitter.

### State Persistence Across Live-Code Reloads

`state.time` and `state.frame` are stored in `localStorage` after every frame (when `settings.restoreState: true`). On the next boot, they are restored with a `cycle++` increment. This means editing code doesn't reset the animation clock — the piece continues from where it was.

---

## Dual Renderer System

### Text Renderer (`textrenderer.js`)

Renders into a `<pre>` element using `<span>` elements for per-cell styling.

**Algorithm:**
1. For each row, compare the new buffer against a `backBuffer` (previous frame's state)
2. If a row has no changes, skip it entirely — **row-level dirty checking**
3. For changed rows, build an HTML string by accumulating runs of same-style characters inside a single `<span>`; open a new span only when style changes
4. Write the accumulated HTML to `element.childNodes[j].innerHTML`

```javascript
// Simplified span-accumulation logic
for (let i = 0; i < cols; i++) {
    const curr = buffer[i + offs]
    if (!isSameCellStyle(curr, prev)) {
        if (tagIsOpen) html += '</span>'
        html += `<span style="color:${curr.color};background:${curr.backgroundColor};">`
        tagIsOpen = true
    }
    html += curr.char
    prev = curr
}
```

**Performance note from the docs:** Frequent *horizontal* style changes are expensive because each change forces a new `<span>`. Vertical changes (entire rows in different colors) are cheap because the dirty-check skips unmodified rows entirely. This guides visual design — horizontal color gradients should be quantized to reduce span count.

### Canvas Renderer (`canvasrenderer.js`)

Renders into a `<canvas>` element using the Canvas 2D API.

**Algorithm:**
1. Scale canvas to `devicePixelRatio` for crisp rendering on HiDPI screens
2. Fill background with a solid color `fillRect`
3. For each cell: draw per-cell `backgroundColor` rectangle (only if different from global bg), then draw the character with `ctx.fillText()`
4. Font is set per-cell via `ctx.font = fontWeight + fontSize + fontFamily`

```javascript
for (let j = 0; j < rows; j++) {
    for (let i = 0; i < cols; i++) {
        const cell = buffer[j * cols + i]
        const x = i * cw
        const y = j * ch
        if (cell.backgroundColor && cell.backgroundColor != bg) {
            ctx.fillStyle = cell.backgroundColor
            ctx.fillRect(Math.round(x), y, Math.ceil(cw), ch)
        }
        ctx.font = (cell.fontWeight || fontWeight) + ff
        ctx.fillStyle = cell.color || fg
        ctx.fillText(cell.char, x, y)
    }
}
```

**No backbuffer / dirty checking** — the canvas is redrawn fully every frame. This trades CPU comparison overhead for GPU compositing, which is a net win for dense color changes.

**HiDPI:** `canvas.width = context.width * devicePixelRatio`, then `ctx.scale(scale, scale)` so drawing coordinates stay in CSS pixels.

---

## Font Metrics

Getting precise character dimensions is critical — cell layout breaks if width is wrong.

```javascript
function calcMetrics(el) {
    const style    = getComputedStyle(el)
    const fontSize = parseFloat(style.getPropertyValue('font-size'))
    const lineHeight = parseFloat(style.getPropertyValue('line-height'))

    let cellWidth
    if (el.nodeName == 'CANVAS') {
        // Canvas path: measureText is authoritative
        const ctx = el.getContext('2d')
        ctx.font = fontSize + 'px ' + fontFamily
        cellWidth = ctx.measureText(''.padEnd(50, 'X')).width / 50
    } else {
        // DOM path: inject a span, measure it, remove
        const span = document.createElement('span')
        el.appendChild(span)
        span.innerHTML = ''.padEnd(50, 'X')
        cellWidth = span.getBoundingClientRect().width / 50
        el.removeChild(span)
    }

    return { aspect: cellWidth / lineHeight, cellWidth, lineHeight, ... }
}
```

Measuring 50 characters and dividing averages out sub-pixel rounding errors. The result is a fractional `cellWidth` which is accumulated carefully in cursor-to-cell coordinate conversion (`pointer.x / metrics.cellWidth`).

**Safari font-loading bug:** `document.fonts.ready` can resolve before fonts are actually available on Safari 13–14. The workaround is to wait 3 animation frames after `fonts.ready` before running `boot()`.

---

## Pointer Handling

Pointer position is tracked in absolute pixels and converted to fractional cell coordinates on each frame:

```javascript
// DOM event (pixels)
pointer.x = e.clientX - rect.left
pointer.y = e.clientY - rect.top

// Cursor (cell coordinates, converted in loop)
cursor.x = Math.min(context.cols - 1, pointer.x / metrics.cellWidth)
cursor.y = Math.min(context.rows - 1, pointer.y / metrics.lineHeight)
```

The `Math.min` clamp is necessary because the canvas may be slightly larger than `cols × cellWidth` (the remainder from floor division). Programs access `cursor.x` and `cursor.y` as fractional cell coordinates (e.g. `14.7`).

Previous frame state is preserved in `cursor.p` for velocity / drag calculations.

Touch events are also handled:
```javascript
element.addEventListener('touchmove', e => {
    pointer.x = e.touches[0].clientX - rect.left
    pointer.y = e.touches[0].clientY - rect.top
})
```

Custom program-level event handlers (`pointerMove`, `pointerDown`, `pointerUp`) are supported via the queued event system — they fire at the end of each frame, after rendering.

---

## Visual Techniques Used in the Demos

### Plasma (oldschool demo)

```javascript
const density = '$?01▄abc+-><:. '

export function main(coord, context) {
    const t1 = context.time * 0.0009
    const t2 = context.time * 0.0003
    const m = Math.min(context.cols, context.rows)
    const a = context.metrics.aspect   // corrects for non-square cells

    // Normalize to centered [-1, 1] space with aspect correction
    let st = {
        x : 2.0 * (coord.x - context.cols / 2) / m * a,
        y : 2.0 * (coord.y - context.rows / 2) / m
    }

    // Three layered wave functions (classic plasma)
    const v1 = sin(dot(coord, vec2(sin(t1), cos(t1))) * 0.08)
    const v2 = cos(length(sub(st, center)) * 4.0)
    const v3 = v1 + v2

    // Map combined value to density ramp character
    const idx = floor(map(v3, -2, 2, 0, 1) * density.length)

    // Quantized RGB (low quant = hard gradient, better perf)
    const quant = 2
    const r = floor(map(sin(v3 * PI   + t1), -1, 1, 0, quant)) * (255 / (quant - 1))
    const g = floor(map(sin(v3 * PI23 + t2), -1, 1, 0, quant)) * (255 / (quant - 1))
    const b = floor(map(sin(v3 * PI43 - t1), -1, 1, 0, quant)) * (255 / (quant - 1))

    return { char: density[idx], color: 'white', backgroundColor: css(r, g, b) }
}
```

**Key techniques:**
- **Aspect correction** via `metrics.aspect` (cellWidth / lineHeight) — without this, circles would appear oval
- **Density character ramp** maps scalar → char index (same technique as Flux)
- **Color quantization** reduces unique CSS strings per frame, cutting DOM span overhead
- **Irrational time multipliers** (`t1 = time * 0.0009`, `t2 = time * 0.0003`) prevent the animation from repeating

### Doom Flame (cellular automaton + value noise)

```javascript
const data = []   // flat float array: one value per cell

export function pre(context, cursor, buffer) {
    // Bottom row: seed with value noise driven by time
    const last = cols * (rows - 1)
    for (let i = 0; i < cols; i++) {
        const val = floor(map(noise(i * 0.05, t), 0, 1, 5, 40))
        data[last + i] = min(val, data[last + i] + 2)
    }

    // Propagate upward with horizontal jitter — the automaton step
    for (let i = 0; i < data.length; i++) {
        const row = floor(i / cols)
        const col = i % cols
        const dest = row * cols + clamp(col + rndi(-1, 1), 0, cols - 1)
        const src  = min(rows - 1, row + 1) * cols + col
        data[dest] = max(0, data[src] - rndi(0, 2))
    }
}

export function main(coord, context) {
    const u = data[coord.index]
    if (u === 0) return  // empty cell = space char
    return { char: flame[clamp(u, 0, flame.length - 1)], fontWeight: u > 20 ? 700 : 100 }
}
```

**Key techniques:**
- `pre()` owns the simulation state — `main()` is a pure read-only lookup
- The automaton runs in-place on `data[]`, bottom-to-top, with random horizontal drift
- `fontWeight` varies per intensity: bold at high values, light at low — creates a visual heat gradient without color
- Value noise uses a permutation table for spatial smoothness; `smoothstep` eases lattice transitions

### Value Noise Implementation

```javascript
function valueNoise() {
    const tableSize = 256
    const r = new Array(tableSize).fill(0).map(() => Math.random())
    const perm = new Array(tableSize * 2)

    // Initialize + Fisher-Yates shuffle
    for (let k = 0; k < tableSize; k++) perm[k] = k
    for (let k = 0; k < tableSize; k++) {
        const i = floor(Math.random() * tableSize)
        ;[perm[k], perm[i]] = [perm[i], perm[k]]
        perm[k + tableSize] = perm[k]
    }

    return function(px, py) {
        const xi = floor(px), yi = floor(py)
        const tx = px - xi,  ty = py - yi
        // Lattice corners via permutation table
        const c00 = r[perm[perm[xi % 256] + yi % 256]]
        const c10 = r[perm[perm[(xi+1) % 256] + yi % 256]]
        const c01 = r[perm[perm[xi % 256] + (yi+1) % 256]]
        const c11 = r[perm[perm[(xi+1) % 256] + (yi+1) % 256]]
        // Bilinear interpolation with smoothstep easing
        const sx = smoothstep(0, 1, tx)
        const sy = smoothstep(0, 1, ty)
        return mix(mix(c00, c10, sx), mix(c01, c11, sx), sy)
    }
}
```

---

## The GLSL-Ported Math Library (`num.js`)

A deliberate decision: port GLSL's most useful math functions verbatim to JS. This means programs feel like writing shaders, and the mental model transfers.

| Function | GLSL equivalent | Purpose |
|----------|----------------|---------|
| `map(v, inA, inB, outA, outB)` | — | Range remapping (GLSL has no built-in) |
| `fract(v)` | `fract` | Fractional part |
| `clamp(v, min, max)` | `clamp` | Clamping |
| `mix(v1, v2, a)` | `mix` | Linear interpolation |
| `smoothstep(e0, e1, t)` | `smoothstep` | Cubic S-curve |
| `smootherstep(e0, e1, t)` | — | Quintic S-curve (Ken Perlin) |
| `sign(n)` | `sign` | −1 / 0 / +1 |

`map()` is the most-used function in the codebase — it translates the output of any wave or noise function into whatever range the program needs, without manual arithmetic.

---

## Color System (`color.js`)

Colors are CSS strings throughout — `backgroundColor: 'rgb(255, 0, 0)'`. The library provides:

- **Palette maps:** `CSS4`, `CSS3`, `CSS1` (named colors as objects with `{r, g, b, hex, css, v}`)
- **Retro palettes:** `C64` and `CGA` as indexed arrays for authentic palette cycling
- **Converters:** `rgb()`, `css()`, `rgb2hex()`, `rgb2gray()`, `int2rgb()`

```javascript
// Build a CSS color string from components
css(r, g, b)        // → 'rgb(255, 0, 0)'
css(r, g, b, 0.5)   // → 'rgba(255, 0, 0, 0.5)'

// Named palette color
CSS4['royalblue']   // → { r: 65, g: 105, b: 225, hex: '#4169e1', css: 'rgb(65,105,225)', v: 0.43 }
```

**Performance guidance from the plasma demo:** Quantize color components before converting to CSS strings. Fewer unique CSS string values = fewer unique span styles = less DOM work in the text renderer. The demo quantizes to 2 levels (binary), producing at most 8 unique background colors per frame.

---

## Settings System

Programs export an optional `settings` object. The runner merges: `defaultSettings → runnerSettings → program.settings` (program wins):

```javascript
export const settings = {
    fps             : 60,
    renderer        : 'canvas',   // or 'text'
    backgroundColor : 'black',
    color           : 'white',
    fontFamily      : 'monospace',
    fontSize        : '16px',
    cols            : 0,          // 0 = auto-fill container
    rows            : 0,
    once            : false,      // run once then stop (for export)
    restoreState    : true,       // persist time/frame across live-code reloads
    allowSelect     : false,
}
```

CSS properties in `settings` are applied directly to the container element's `style`. This sets the global default color and background, so individual cells only need to override when they differ — minimizing span count.

---

## Screensaver Mode

`?mode=screensaver` is a URL parameter parsed by the site wrapper (not the engine). It:

1. Hides the navigation bar and code editor UI
2. Keeps the render target fullscreen
3. Runs the current sketch (or cycles through a playlist of sketches) without user intervention

From a technical standpoint, the screensaver is the play.core engine running at full fidelity with no UI chrome — the engine itself doesn't have a screensaver concept, it's purely a presentation layer decision. The visual effects (plasma, flame, geometric patterns) are the actual programs.

---

## Key Design Patterns

### Stateless vs Stateful Programs

**Stateless** (preferred): `main()` is a pure function of `coord` and `context.time`. No mutable state. These are trivially restartable and composable.

```javascript
// Completely stateless — every output is derivable from coord + time
export function main(coord, context) {
    const t = context.time * 0.001
    return Math.sin(coord.x * 0.2 + t) > 0 ? '▓' : '░'
}
```

**Stateful** (for simulations): State lives in module-scope variables, managed in `pre()`. The buffer is treated as a read/write scratch space.

```javascript
let data = []   // simulation state

export function pre(context) {
    // advance simulation in-place
}

export function main(coord) {
    return density[data[coord.index]]  // pure lookup
}
```

### Aspect Ratio Normalization

Monospaced character cells are not square. A typical ratio is ~0.4–0.5 (width/height). Every program that places shapes in 2D space must compensate:

```javascript
const a = context.metrics.aspect  // cellWidth / lineHeight ≈ 0.45
const st = {
    x : 2.0 * (coord.x - context.cols * 0.5) / Math.min(context.cols, context.rows) * a,
    y : 2.0 * (coord.y - context.rows * 0.5) / Math.min(context.cols, context.rows)
}
// st.x and st.y are now in isometric space: circles look round
```

### Buffer as Communication Channel

The buffer persists between `pre`, `main`, and `post`. This enables:
- `pre()` to seed regions (e.g., borders, static labels) that `main()` leaves unchanged
- `post()` to overlay HUDs on top of whatever `main()` produced
- Trailing effects — if `main()` only writes to a subset of cells, the rest retain their previous frame

---

## Comparison with Flux (webArt)

| Dimension | ertdfgcvb / play.core | Flux (webArt) |
|-----------|----------------------|---------------|
| **Per-cell evaluation** | CPU, JavaScript `main()` per cell | GPU, GLSL fragment shader per fragment |
| **Renderer** | DOM `<span>` or Canvas 2D | WebGL (GLSL, fullscreen quad) |
| **Simulation** | Per-cell JS callbacks; cellular automata in `pre()` | Navier-Stokes CPU solver → GPU texture upload |
| **Color** | CSS strings (`'rgb(r,g,b)'`) | OKLch perceptual space, computed in GLSL |
| **Character mapping** | Index into JS string via `density[i]` | Index into font atlas texture in shader |
| **Font atlas** | Font rendered live by browser (canvas or DOM) | Pre-rendered `<canvas>` → WebGL texture |
| **Scale** | 5k–8k cells; CPU-bound per cell | Millions of fragments; GPU-bound |
| **Flexibility** | High — arbitrary JS per cell, any algorithm | High — arbitrary GLSL; harder to prototype |
| **Live editing** | Full live-code IDE built in | Vite HMR on shader file save |
| **State persistence** | localStorage for time/frame across reloads | `u_seed` per session; no persistence |

The two systems are complementary. play.core's strength is expressive flexibility per cell — you can run a physics simulation, sample an image, or hit an API in `pre()`, and the result flows naturally into per-cell logic. Flux's strength is raw GPU throughput and fluid dynamics at full resolution, which would be prohibitively slow in a JS per-cell loop.

---

## Summary of Implementation Techniques

1. **Cell-as-fragment** — one JS function called per cell, per frame; mirrors GLSL `main()`
2. **Four-phase pipeline** — `boot / pre / main / post` cleanly separates init, simulation, rendering, and compositing
3. **Row-level dirty checking** (text renderer) — skips DOM updates for unchanged rows; critical for performance on slow-change scenes
4. **50-character font measurement** — averages out sub-pixel rounding for reliable cell width
5. **Aspect correction** — `metrics.aspect` scales X coordinates so spatial math produces correct shapes on non-square cells
6. **Color quantization** — limiting unique CSS color strings reduces span count and DOM overhead
7. **State in `pre()`, read in `main()`** — keeps `main()` pure; enables clean simulation/display separation
8. **GLSL-ported math** — `map`, `mix`, `smoothstep` port directly from shader mental model to JS
9. **Permutation-table value noise** — efficient, seeded, continuable noise without an external library
10. **`restoreState`** — persists `time` and `frame` in localStorage so live-code edits don't reset the clock
11. **Screensaver mode** — purely a UI wrapper decision; the engine is unmodified
