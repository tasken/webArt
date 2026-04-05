# ertdfgcvb.xyz — Technical Analysis
**Subject:** `https://ertdfgcvb.xyz/?mode=screensaver` · `https://play.ertdfgcvb.xyz/`
**Author:** Andreas Gysin ([@andreasgysin](https://twitter.com/andreasgysin))
**Engine:** [play.core](https://github.com/ertdfgcvb/play.core) (open source, Apache 2.0)
**Date of analysis:** 2026-04-05
**Coverage:** Full source analysis of all 45 example programs across 6 categories

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

---

---

# Program Catalogue — All 45 Examples

Complete source analysis of every program available at `play.ertdfgcvb.xyz`, organized by category. Each entry documents the technique, the key code pattern, and what it teaches.

---

## Category 1: Basics (13 programs)

Foundational programs illustrating the API surface. Every concept here is prerequisite for the demos.

---

### Simple output
**The minimum viable program.**

```javascript
export function main() { return '?' }
// Arrow form: export const main = () => '?'
// Golf form:  export let main = o => 0
```

`main()` ignores all parameters and returns the same char every frame. Because the buffer isn't cleared, unchanged cells stay from the previous frame — here that means a static grid of `?`. Demonstrates that `coord`, `context`, `cursor`, `buffer` are all optional arguments.

---

### Coordinates: x, y
**Reading `coord.x` and `coord.y` for animated density ramps.**

```javascript
const density = 'Ñ@#W$9876543210?!abc;:+=-,._ '

export function main(coord, context) {
    const { cols, frame } = context
    const { x, y } = coord
    const sign = y % 2 * 2 - 1          // alternates −1/+1 per row
    const index = (cols + y + x * sign + frame) % density.length
    return density[index]
}
```

`sign` flips the scroll direction on even vs odd rows, creating a chevron-wave effect. `frame` drives the scroll. The pattern is purely arithmetic — no trig, no noise.

---

### Coordinates: index
**Using `coord.index` directly.**

```javascript
const pattern = '| |.|,|:|;|x|K|Ñ|R|a|+|=|-|_'

export function main(coord) {
    return pattern[coord.index % pattern.length]
}
```

`coord.index = y * cols + x`. The pattern repeats modulo its length, creating a diagonal stripe because `index` increments both along rows and down columns. Resizing the window changes the stripe angle because `cols` changes.

---

### Time: milliseconds
**`context.time` for smooth wave animation.**

```javascript
const pattern = 'ABCxyz01═|+:. '

export function main(coord, context) {
    const t = context.time * 0.0001
    const o = Math.sin(coord.y * Math.sin(t) * 0.2 + coord.x * 0.04 + t) * 20
    const i = Math.round(Math.abs(coord.x + coord.y + o)) % pattern.length
    return { char: pattern[i], fontWeight: '100' }
}
```

`context.time` is milliseconds since start — multiply by a small constant to get a slow-moving wave. The `fontWeight: '100'` (light) is set globally here; only overrides are needed per-cell.

---

### Time: frames
**`context.frame` for discrete animation — fake 3D perspective.**

```javascript
export function main(coord, context) {
    const z = Math.floor(coord.y - context.rows / 2)
    if (z == 0) return ' '
    const val = (coord.x - context.cols / 2) / z   // perspective divide
    const code = Math.floor(val + context.cols / 2 + context.frame * 0.3) % 94 + 32
    return String.fromCharCode(code)
}
```

Perspective divide `x / z` projects a 2D plane into fake 3D. `frame * 0.3` scrolls at a fixed rate. `charCode % 94 + 32` cycles through printable ASCII (32–126). Division by zero at `z == 0` is guarded explicitly.

---

### Cursor
**Reading `cursor.x/y` to draw a crosshair.**

```javascript
export function main(coord, context, cursor) {
    const x = Math.floor(cursor.x)
    const y = Math.floor(cursor.y)
    if (coord.x == x && coord.y == y) return '┼'
    if (coord.x == x) return '│'
    if (coord.y == y) return '─'
    return (coord.x + coord.y) % 2 ? '·' : ' '
}
```

`cursor.x` is fractional (pixel / cellWidth). `Math.floor` converts to cell column. The background checkerboard uses the parity of `x + y`. This is the canonical cursor interaction example — `main()` is stateless, the crosshair exists only as a function of coord vs cursor.

---

### How to draw a circle
**Aspect ratio correction for round shapes.**

```javascript
export function main(coord, context, cursor) {
    const a = cursor.pressed ? 1 : context.metrics.aspect   // toggle aspect for debug
    const m = Math.min(context.cols * a, context.rows)
    const st = {
        x : 2.0 * (coord.x - context.cols / 2) / m * a,
        y : 2.0 * (coord.y - context.rows / 2) / m
    }
    return length(st) < 0.7 ? 'X' : '.'
}
```

Without `* a` on `st.x`, circles become ovals because cells are taller than wide. Holding the mouse disables correction — the oval reappears. This is the definitive aspect-ratio demo: click to see why it matters.

---

### How to draw a square
**SDF box function + rotation.**

```javascript
export function box(p, size) {
    const dx = Math.max(Math.abs(p.x) - size.x, 0)
    const dy = Math.max(Math.abs(p.y) - size.y, 0)
    return Math.sqrt(dx * dx + dy * dy)
}

export function main(coord, context) {
    const t = context.time
    const ang = t * 0.0015
    const p = { x: st.x * cos(-ang) - st.y * sin(-ang),
                y: st.x * sin(-ang) + st.y * cos(-ang) }
    const size = map(Math.sin(t * 0.0023), -1, 1, 0.1, 2)
    const d = box(p, { x: size, y: size })
    return d == 0 ? ' ' : ('' + d).charAt(2)   // 3rd digit of distance string
}
```

`('' + d).charAt(2)` is a clever hack: the distance field value converted to a decimal string produces a digit that encodes magnitude. Interior (`d == 0`) is space; exterior shows contour lines.

---

### How to log
**Limiting console output inside the per-cell loop.**

```javascript
if (coord.index == 100 && context.frame % 10 == 0) {
    console.log("dist = " + dist)
}
```

`main()` runs 5,000–8,000 times per frame. Logging every call floods the console instantly. The pattern: gate on a single cell index AND a frame interval. The visual is Chebyshev distance (L∞ norm: `max(|dx|, |dy|)`) mapped to a rolling character ramp.

---

### Name game
**Module-scope state + non-obvious indexing.**

```javascript
export const settings = { fontSize: '3em', fontWeight: 'lighter' }

export function main(coord, context) {
    const a = context.frame * 0.05
    const f = Math.floor((1 - Math.cos(a)) * 10) + 1
    const g = Math.floor(a / (Math.PI * 2)) % 10 + 1
    const i = coord.index % (coord.y * g + 1) % (f % context.cols)
    return 'Ada'[i]  // undefined → space
}
```

`'Ada'[i]` returns `undefined` for out-of-range `i`, which the engine renders as a space. The formula produces quasi-random sparsity that shifts with time. `fontSize: '3em'` enlarges the grid cells, reducing cell count and increasing visual impact.

---

### Performance test
**Demonstrating horizontal vs vertical style-change cost.**

```javascript
const direction = cursor.pressed ? coord.x : coord.y   // toggle direction

export function main(coord, context, cursor) {
    const r = map(cos(direction * 0.06 + 1 - f), -1, 1, 0, 255)
    // ... g, b similarly ...
    return { char, color: `rgb(${r2},${g2},${b2})`, backgroundColor: `rgb(${r1},${g1},${b1})` }
}
```

Vertical gradient (default): each row is one color → few spans → fast. Horizontal gradient (mouse held): every cell is a different color → one span per cell → slow. The FPS counter (from `drawInfo`) shows the cost live. This is the most important performance lesson in the whole catalogue.

---

### Canvas renderer
**Switching to the canvas 2D backend.**

```javascript
export const settings = {
    renderer: 'canvas',
    canvasOffset: { x: 'auto', y: 20 },  // center horizontally, 20px from top
    canvasSize: { width: 400, height: 500 },
    cols: 42, rows: 22,
    backgroundColor: 'pink'
}
```

`canvasOffset.x: 'auto'` centers the fixed-size canvas. The canvas renderer redraws fully every frame — no dirty checking — and is better for dense color changes but skips the row-skip optimization.

---

### Sequence export
**Frame-by-frame PNG export via `exportFrame()`.**

```javascript
import { exportFrame } from '/src/modules/exportframe.js'
export const settings = { renderer: 'canvas', restoreState: false, fps: 2 }

export function pre(context) {
    exportFrame(context, 'export.png', 10, 20)  // export frames 10–20
}
```

`restoreState: false` resets time on each run so export is deterministic. Low `fps: 2` gives browsers time to handle the download triggers. Output files (`export_10.png` … `export_20.png`) can be assembled with FFmpeg: `ffmpeg -framerate 30 -i "export_%d.png" output.mp4`.

---

## Category 2: SDF (Signed Distance Functions, 5 programs)

All SDF programs share a coordinate normalization idiom and the `sdf.js` module. The core insight: compute distance from a shape, then map that scalar to a character.

### The SDF Module

```javascript
// Circle: distance from origin to circle edge
sdCircle(p, radius)  →  length(p) - radius

// Box: Euclidean distance to nearest box edge (0 inside)
sdBox(p, size)       →  max(length(max(abs(p) - size, 0)), 0) + min(max(d.x, d.y), 0)

// Line segment with thickness
sdSegment(p, a, b, thickness)

// Smooth boolean union — merges two shapes with a rounded blend
opSmoothUnion(d1, d2, k)   →  mix(d2, d1, h) - k * h * (1 - h)   where h = clamp(0.5 + 0.5*(d2-d1)/k)
// k controls blend radius: k=0 is sharp union, k=1 is very soft
```

All primitives are ported from Inigo Quilez's [distfunctions](https://www.iquilezles.org/www/articles/distfunctions/distfunctions.htm).

### The Standard Distance-to-Char Pattern

Every SDF program maps distance to a character index with exponential fall-off:

```javascript
const c = 1.0 - Math.exp(-5 * Math.abs(d))   // 0 at shape edge, approaches 1 far away
const index = Math.floor(c * density.length)
return density[index]
```

`exp(-k * |d|)` gives a bright edge that fades exponentially outward. `k` controls sharpness — higher k = harder edge, lower k = softer halo.

---

### Circle
**Animated SDF circle with oscillating radius.**

```javascript
import { sort } from '/src/modules/sort.js'
const density = sort('/\\MXYZabc!?=-. ', 'Simple Console', false)

export function main(coord, context) {
    const radius = Math.cos(context.time * 0.002) * 0.4 + 0.5   // oscillates 0.1–0.9
    const d = sdCircle(st, radius)
    const c = 1.0 - Math.exp(-5 * Math.abs(d))
    return { char: coord.x % 2 ? '│' : density[Math.floor(c * density.length)], ... }
}
```

`sort()` (from `sort.js`) reorders characters by their visual weight in a specific font — so the density ramp is perceptually linear rather than code-point order. Every other column forces a `│` to create a vertical stripe texture inside the circle.

---

### Two Circles
**Smooth union driven by the cursor.**

```javascript
const d1 = sdCircle(st, 0.2)                   // fixed at origin
const d2 = sdCircle(sub(st, pointer), 0.2)      // follows cursor
const d = opSmoothUnion(d1, d2, 0.7)            // k=0.7: wide blend
```

The cursor becomes a second circle that smoothly merges with the fixed one as it approaches. This is the definitive interactive SDF demo: `k=0.7` means circles start blending when they're 1.4 units apart. The comment notes an optimization: `pointer` (cursor in scene space) could be computed once in `pre()` rather than once per cell in `main()`.

---

### Balls
**12 animated SDF balls with smooth union.**

```javascript
let d = Number.MAX_VALUE
const num = 12
for (let i = 0; i < num; i++) {
    const r = map(cos(t * 0.95 * (i+1) / (num+1)), -1, 1, 0.1, 0.3)
    const x = map(cos(t * 0.23 * (i/num * PI + PI)), -1, 1, -1.2, 1.2)
    const y = map(sin(t * 0.37 * (i/num * PI + PI)), -1, 1, -1.2, 1.2)
    const f = transform(st, {x, y}, t)          // translate + rotate
    d = opSmoothUnion(d, sdCircle(f, r), s)     // accumulate minimum
}
let c = 1.0 - Math.exp(-3 * Math.abs(d))
```

Each ball has a unique time multiplier (e.g. `t * 0.95 * (i+1) / (num+1)`) preventing any two balls from synchronizing — they never return to the same configuration. `transform()` applies translation and rotation around the local center. The smooth parameter `s` oscillates: when `s=0` the union is sharp, when `s=0.9` shapes melt together.

---

### Rectangles
**Grid of rotating SDFs with smooth union.**

```javascript
const s = map(Math.sin(t * 0.0005), -1, 1, 0.0, 0.4)
const g = 1.2
for (let by = -g; by <= g; by += g * 0.33) {
    for (let bx = -g; bx <= g; bx += g * 0.33) {
        const r = t * 0.0004 * (bx + g*2) + (by + g*2)   // unique rotation per box
        const d1 = sdBox(transform(st, {x:bx, y:by}, r), {x: g*0.33, y: 0.01})
        d = opSmoothUnion(d, d1, s)
    }
}
```

Each box is a thin horizontal line (`y: 0.01`) that rotates at a rate proportional to its grid position. The result is a grid of spinning needles that melt into each other via smooth union.

---

### Wireframe Cube
**Full 3D perspective projection in `pre()`, SDF segment per edge in `main()`.**

```javascript
// pre(): project 3D vertices to 2D
export function pre(context, cursor) {
    const rot = vec3(t * 0.11, t * 0.13, -t * 0.15)  // tumble on all axes
    for (let i = 0; i < box.vertices.length; i++) {
        let vt = v3.rotX(box.vertices[i], rot.x)
        vt = v3.rotY(vt, rot.y)
        vt = v3.rotZ(vt, rot.z)
        boxProj[i] = v2.mulN(vec2(vt.x, vt.y), d / (vt.z - zOffs))  // perspective divide
    }
}

// main(): SDF segment for each of the 12 edges
let d = 1e10
for (let i = 0; i < box.edges.length; i++) {
    const a = boxProj[box.edges[i][0]]
    const b = boxProj[box.edges[i][1]]
    d = Math.min(d, sdSegment(st, a, b, thickness))
}
const idx = Math.floor(Math.exp(expMul * Math.abs(d)) * density.length)
```

The cursor controls `thickness` (X axis) and `expMul` (Y axis — edge sharpness). `expMul` from the cursor Y: at the top of the screen edges are razor-thin with high contrast; at the bottom they become wide blobs. Background shows a tiling `┼──────` / `│` grid that stops where the cube interior is.

---

## Category 3: Demos (15 programs)

More complex programs combining multiple techniques.

---

### 10 PRINT
**The shortest creative program — one line of output logic.**

```javascript
export const settings = { once: true }   // run exactly once, then stop

export function main() {
    return Math.random() < 0.5 ? '╱' : '╲'
}
```

`settings.once: true` fills the grid exactly once and stops the animation loop. The diagonal slash / backslash pattern produces the iconic maze-like structure of the BASIC one-liner. Suggested alternatives in comments: `╩ ╦`, `▄ ░`.

---

### Mod Xor
**Integer patterns from XOR and modulo.**

```javascript
const pattern = '└┧─┨┕┪┖┫┘┩┙┪━'

export function main(coord, context) {
    const t1 = Math.floor(context.frame / 2)
    const t2 = Math.floor(context.frame / 128)
    const x = coord.x
    const y = coord.y + t1           // scroll down
    const m = t2 * 2 % 30 + 31      // slowly changing modulus
    const i = (x + y ^ x - y) % m & 1   // XOR produces interference pattern
    const c = (t2 + i) % pattern.length
    return pattern[c]
}
```

`x + y ^ x - y` uses bitwise XOR between two linear expressions — this generates interference fringes. `% m & 1` creates binary stripes. `t2` (slow time) cycles through the box-drawing character palette. No floating point at all — a pure integer demo.

---

### Sin Sin (checker + wave)
Two variations on the same `sin(x) * sin(y)` base:

**Checker variation:**
```javascript
const o = sin(x * y * 0.0017 + y * 0.0033 + t) * 40  // coupled xy freq
const i = floor(abs(x + y + o))
const c = (floor(coord.x * 0.09) + floor(coord.y * 0.09)) % 2  // 2x2 checker
return { char: pattern[c][i % pattern[c].length], fontWeight: weights[c] }
```

Two interleaved patterns (`pattern[0]` and `pattern[1]`) with different font weights (`100` vs `700`) are selected by a 2×2 checker. The checker cell at low frequency creates bold/light bands.

**Wave variation:**
```javascript
const o = sin(y * x * sin(t) * 0.003 + y * 0.01 + t) * 20
const i = round(abs(x + y + o)) % pattern.length
return pattern[i]          // box-drawing chars: ┌┘└┐╰╮╭╯
```

The coupling `y * x * sin(t)` creates self-modifying frequency — when `sin(t)` is near zero the pattern flattens; when near ±1 the interference is dense. Box-drawing characters align at their natural boundaries, creating an emergent tiling illusion.

---

### Numbers
**Pixel-font digits via integer bitmaps + CGA palette.**

```javascript
const ints = [488162862, 147460255, 487657759, ...]  // one int per digit 0–9

const bit = (n, k) => n >> k & 1  // extract bit k from integer n

export function main(coord, context) {
    const nx = Math.floor(x % sx)  // position within glyph cell
    const ny = Math.floor(y % sy)
    const char = nx < numX && ny < numY
        ? bit(ints[num], (numX - nx - 1) + (numY - ny - 1) * numX)
        : 0
    return { char: '.▇'[char], color: char ? CGA[color].hex : CGA[5].hex }
}
```

Each digit is encoded as a 5×6 bitmap packed into a 30-bit integer. `n >> k & 1` extracts one pixel. The `scale` variable grows with `sin(frame)`, zooming the digits in and out. The CGA palette (with some entries spliced out) provides the color. This is a complete software font renderer in ~50 lines.

---

### Donut
**a1k0n's famous donut — writes directly to `buffer` in `pre()`.**

```javascript
export function pre(context, cursor, buffer) {
    const z = []   // z-buffer (depth)

    // Clear buffer and z-buffer
    for (let k = 0; k < width * height; k++) {
        buffer[k].char = ' '
        z[k] = 0
    }

    for (let j = 0; j < TAU; j += 0.05) {      // theta: cross-section circle
        for (let i = 0; i < TAU; i += 0.01) {   // phi: revolution
            const D = 1 / (sp * h * sA + st * cA + 5)    // 1/z (perspective)
            const N = 8 * ((st * sA - sp * ct * cA) * cB - ...) // luminance (0–11)
            if (D > z[o]) {
                z[o] = D
                buffer[o].char = '.,-~:;=!*#$@'[N > 0 ? N : 0]
            }
        }
    }
}
export function main() {}  // main() does nothing — buffer already written
```

This is a "brute force" renderer: `pre()` writes the entire frame directly, doing 3D torus rasterization with a software z-buffer. `main()` is empty. Luminance `N` maps to 12 ASCII shading characters. The double-loop step sizes (0.05 and 0.01) must be small enough to avoid gaps at the projected resolution.

---

### Box Fun
**`drawBox()` utility for text overlay boxes.**

```javascript
import { drawBox } from '/src/modules/drawbox.js'

export function post(context, cursor, buffer) {
    for (let j = 0; j < numY; j++) {
        for (let i = 0; i < numX; i++) {
            const ox = floor(sin((i+j) * 0.6 + t*3) * spacingX)  // animated offset
            drawBox(txt, { x, y, width, height, backgroundColor: 'white', borderStyle: 'double', shadowStyle: 'light' }, buffer, cols, rows)
        }
    }
}
```

`drawBox()` writes directly to the buffer — it overwrites whatever `main()` put there. The boxes animate by wobbling their x/y positions with sin/cos. `borderStyle: 'double'` draws a `╔═╗╚╝` border; `shadowStyle: 'light'` adds a gray drop shadow.

---

### Chroma Spiral
**Shadertoy port: iterated UV distortion + color palette.**

```javascript
for (let i = 0; i < 3; i++) {
    const o = i * 3
    const v = vec2(sin(t*3 + o), cos(t*2 + o))
    add(st, v, st)                         // translate
    const ang = -t + length(subN(st, 0.5))
    rot(st, ang, st)                       // rotate by distance-dependent angle
}
mulN(st, 0.6, st)                          // scale down

const colors = ['deeppink', 'black', 'red', 'blue', 'orange', 'yellow']
const color = floor(c * (colors.length - 1))
return { char: density[index], color: colors[color] }
```

Three iterations of translate + angle-dependent rotation create a spiral warp. This is a direct port of a Shadertoy GLSL shader — the vec2 module's `add`, `rot`, `mulN` map directly to GLSL equivalents. Color is indexed from a named CSS color array, not computed.

---

### Spiral
**Shadertoy port: polar coordinates + sine modulation.**

```javascript
const radius = length(st)
const rot = 0.03 * TAU * t
const turn = atan(st.y, st.x) / TAU + rot     // normalized angle [0,1]

const n_sub = 1.5
const turn_sub = n_sub * turn % n_sub           // n-fold symmetry

const k = 0.1 * sin(3.0 * t)
const s = k * sin(50.0 * (pow(radius, 0.1) - 0.4 * t))  // radial waves
const turn_sine = turn_sub + s

const i_turn   = floor(density.length * turn_sine % density.length)
const i_radius = floor(1.5 / pow(radius * 0.5, 0.6) + 5.0 * t)
const idx = (i_turn + i_radius) % density.length
```

Two independent index contributions (angular + radial) are summed modulo density length, creating Moiré-like interference between the two fields. `pow(radius, 0.1)` compresses the radial scale logarithmically. The `sort()` module sorts the density string by visual weight for the active font.

---

### Wobbly
**Tiling SDF circles with time-varying scale + rotation.**

```javascript
st = rot(st, 0.6 * Math.sin(0.62 * t) * length(st) * 2.5)  // rotation scaled by distance
st = rot(st, t * 0.2)                                          // global rotation

const s = map(Math.sin(t), -1, 1, 0.5, 1.8)  // oscillating tile scale
const pt = { x: fract(st.x * s) - 0.5, y: fract(st.y * s) - 0.5 }  // tile

const d = sdCircle(pt, r)           // circle SDF in tile space
const k = smoothstep(width, width + 0.2, Math.sin(10 * d + t))  // animated band
const c = (1 - Math.exp(-3 * Math.abs(d))) * k
return { char: density[index], color: k == 0 ? 'orangered' : 'royalblue' }
```

`fract(st * s) - 0.5` tiles the coordinate space — a grid of repeated circles. Distance-weighted rotation (`length(st) * 2.5`) creates a swirl that increases toward the center. Color switches between two values at the smoothstep threshold — orange inside bands, blue outside.

---

### Dyna
**Paul Haeberli's Dynadraw — physics-based brush with Bresenham line.**

```javascript
class Dyna {
    update(cursor) {
        const force = sub(cursor, this.pos)       // spring force toward cursor
        const acc = divN(force, this.mass)
        this.vel = mulN(add(this.vel, acc), this.damp)  // Euler integration with damping
        this.pre = copy(this.pos)
        this.pos = add(this.pos, this.vel)
    }
}

// pre(): draw line from previous to current position
const points = line(dyna.pos, dyna.pre)  // Bresenham
for (const p of points) {
    // paint radius-circle of values into buffer[idx].value
    buffer[idx].value = Math.max(buffer[idx].value, falloff)
}

// main(): read buffer.value → density char, decay
buffer[i].value *= 0.99   // slow fade
return density[Math.floor(smoothstep(0, 0.9, v) * (density.length-1))]
```

The physics brush is a mass-spring-damper: `force = cursor − pos`, `acc = force / mass`, velocity is damped each step. Between frames, Bresenham's line fills in the trail. `buffer[idx].value` is a custom per-cell float stored as a buffer property — demonstrates that the buffer can carry arbitrary data, not just display fields.

---

### Moiré Explorer
**Two frequency fields multiplied — mode cycling on click.**

```javascript
export const boot = (context) => context.settings.element.style.cursor = 'pointer'
export const pointerDown = () => mode = ++mode % 3   // event handler

export function main(coord, context, cursor) {
    const A = mode % 2 == 0 ? atan2(...centerA) : dist(st, centerA)  // angle or distance
    const B = mode == 0     ? atan2(...centerB) : dist(st, centerB)
    const aMod = map(cos(t*2.12), -1, 1, 6, 60)  // frequency oscillates
    const a = cos(A * aMod)
    const b = cos(B * bMod)
    const i = ((a * b) + 1) / 2   // product → [0, 1]
    return density[floor(i * density.length)]
}
```

`pointerDown` is a custom event handler (not part of the four main callbacks) — the runner's event queue dispatches it. Three modes: `(angle, angle)`, `(dist, angle)`, `(dist, dist)` produce visually distinct Moiré patterns. `boot` is used here only to set the cursor CSS style.

---

### Hotlink
**Runtime code loading via `fetch()` + `new Function()`.**

```javascript
fetch("https://raw.githubusercontent.com/blindman67/SimplexNoiseJS/master/simplexNoise.js")
.then(e => e.text())
.then(e => {
    const openSimplexNoise = new Function("return " + e)()
    noise3D = openSimplexNoise(Date.now()).noise3D
})

function noise3D() { return 0 }   // stub until fetch completes
```

The comment says "Don't do this 🙂" — but it works. `new Function("return " + code)()` evaluates the fetched source as a function and calls it. The stub ensures the program renders (silently) while the network request is in-flight. Demonstrates `fetch()` inside a play.core module.

---

### Golgol (Game of Life — double resolution)
**Half-block characters for 2× vertical resolution.**

```javascript
// Each char cell hosts TWO automata cells vertically
// '█' = both alive, '▀' = upper alive, '▄' = lower alive, ' ' = both dead

// Double-height buffer with ping-pong
data[0] = []; data[1] = [];  // two arrays for prev/curr frames
const prev = data[ context.frame % 2]
const curr = data[(context.frame + 1) % 2]

// Read from prev, write to curr
const neighbors = get(x-1, y-1, ...) + ... + get(x+1, y+1, ...)  // 8-neighbor sum
curr[i] = current ? (neighbors == 2 || neighbors == 3 ? 1 : 0) : (neighbors == 3 ? 1 : 0)

// Render
export function main(coord, context) {
    const upper = curr[coord.x + coord.y * 2 * context.cols]
    const lower = curr[coord.x + coord.y * 2 * context.cols + context.cols]
    if (upper && lower) return '█'
    if (upper)          return '▀'
    if (lower)          return '▄'
    return ' '
}
```

The simulation runs at `cols × (rows × 2)` — double height. Each display cell maps to two simulation cells via `▀`/`▄`/`█`. Ping-pong double-buffer: `frame % 2` selects read vs write array. Click fills a 10×10 region with random state.

---

## Category 4: Camera (3 programs)

All three use the same setup: `Camera.init()` + `Canvas` helper, `can.cover(cam, aspect)` to scale the webcam to cell resolution, then `writeTo(data)` to extract per-pixel color objects.

```javascript
const cam = Camera.init()    // starts getUserMedia
const can = new Canvas()     // off-screen canvas helper

export function pre(context) {
    can.resize(context.cols, context.rows)
    can.cover(cam, context.metrics.aspect).mirrorX().writeTo(data)
    // data[i] = { r, g, b, a, v, hex, css }
}
```

`cover()` scales the camera image to fill `cols × rows` with the correct aspect ratio. `mirrorX()` flips horizontally for a natural selfie view. `writeTo()` extracts pixel data into an array of color objects.

---

### Camera grayscale
**Luminance → density char.**

```javascript
const density = sort(' .x?▂▄▆█', 'Simple Console', false)  // sorted by visual weight

export function main(coord) {
    const color = data[coord.index]
    const index = Math.floor(color.v * (density.length - 1))  // .v = gray value [0,1]
    return density[index]
}
```

`color.v` is `rgb2gray(color)` — the perceptual luminance. `sort()` orders the density string by the actual rendered pixel area of each glyph in the target font, so the ramp is visually linear.

---

### Camera RGB
**Palette quantization → background color + char.**

```javascript
can.cover(cam, a).mirrorX().quantize(pal).writeTo(data)
// quantize() replaces each pixel with the nearest palette color

export function main(coord) {
    const color = data[coord.index]
    const index = Math.floor(color.v * (density.length - 1))
    return { char: density[index], backgroundColor: rgb2hex(color) }
}
```

`quantize(pal)` maps each camera pixel to the nearest of 5 custom colors — black, red, yellow, blue, cyan. The char encodes luminance; `backgroundColor` uses the quantized hue. The effect is a posterized color portrait.

---

### Camera double resolution
**Half-block trick applied to camera input.**

```javascript
can.resize(context.cols, context.rows * 2)   // double height canvas
can.cover(cam, a * 2).quantize(pal).mirrorX().writeTo(data)

export function main(coord) {
    const upper = data[coord.y * context.cols * 2 + coord.x]
    const lower = data[coord.y * context.cols * 2 + coord.x + context.cols]
    return { char: '▄', color: lower.hex, backgroundColor: upper.hex }
}
```

Always returns `▄` (lower half block). The `color` (foreground) is the lower pixel; `backgroundColor` is the upper pixel. Two camera pixels appear in one character cell, doubling vertical resolution. The palette here is CSS3 named colors.

---

## Category 5: Contributed (9 programs)

Programs by community contributors. Each brings a distinct technique or aesthetic.

---

### Color Waves (Eliza)
**Three layered trig fields mapped to named color palette.**

```javascript
const chars = '¯\_(ツ)_/¯.::.ᕦ(ò_óˇ)ᕤ '.split('')
const colors = ['mediumvioletred', 'gold', 'orange', 'chartreuse', 'blueviolet', 'deeppink']

export function main(coord, context) {
    const a = cos(y * cos(t) * 0.2 + x * 0.04 + t)
    const b = sin(x * sin(t) * 0.2 * y * 0.04 + t)
    const c = cos(y * cos(t) * 0.2 + x * 0.04 + t)
    const o = a + b + c * 20
    const i = round(abs(x + y + o)) % chars.length
    return { char: chars[i], color: colors[i % colors.length] }
}
```

`chars` is an emoji/symbol string — split into an array so individual characters (including multi-codepoint) index correctly. Bold (`fontWeight: 700`) makes the small characters visually dense.

---

### Emoji Wave (ilithya)
**Unicode emoji as a density ramp — center-offset scrolling.**

```javascript
const density = '☆ ☺︎ 👀 🌈 🌮🌮 🌈 👀 ☺︎ ☆'
const posCenter = floor((c - density.length) * 0.5)
const wave = sin(y * cos(t)) * 5
const i = floor(x + wave) - posCenter
return density[i]   // undefined outside range → space
```

`posCenter` centers the string horizontally. The sinusoidal `wave` offsets each row, creating the undulation. Out-of-bounds indexing silently returns `undefined`, which the engine renders as a space — used intentionally here to create empty edges. Note: emoji rendering in monospace varies by platform.

---

### EQUAL TEA TALK #65 (nkint)
**Truchet tiles — Frederick Hammersley 1969 inspired.**

```javascript
const _st = mul(st, vec2(5.0, 1.0))    // 5:1 aspect for tile grid
const tileIndex = step(1, mod(_st.x, 2.0))   // alternates 0/1 per 2 columns
const color = tileIndex === 0 ? _st.y : (1 - _st.y)  // gradient flips per tile
const i = floor(map(color, 0, 1, 0, chars.length - 1))
```

Each tile alternates between a dark-to-light and light-to-dark vertical gradient. The separator between tiles (`-` centered, space on either side) is added as a pixel-space check rather than a UV-space calculation, demonstrating that you can mix UV and pixel math in `main()`.

---

### oeö (nkint)
**Two animated regular polygons via SDF — Ernst Jandl 1964 inspired.**

```javascript
function polygon(center, edges, time) {
    const a = (atan2(p.x, p.y) + 2 + time * PI) / (2 * PI)  // normalized angle
    const b = (floor(a * N) + 0.5) / N                        // nearest vertex angle
    const c = length(p) * cos((a - b) * 2 * PI)              // distance to nearest edge
    return smoothstep(0.3, 0.31, c)                           // sharp threshold
}

const triangle = colorT <= 0.1 ? 1 : 0
const quadrato = colorQ <= 0.1 ? 2 : 0
const i = triangle + quadrato    // 0: ' ', 1: 'e', 2: 'o', 3: 'ö'
```

The polygon SDF uses polar coordinates — `a` is the normalized angle, `b` is rounded to the nearest N-th of a full rotation (selecting the nearest polygon vertex direction), then `cos()` of the angle difference gives distance to the edge. The `i` combination creates letter overlaps: where triangle and square overlap, `'ö'` appears.

---

### GOL (Alex Miller)
**Conway's Game of Life — `pre()` copies buffer for temporal access.**

```javascript
export function pre(context, cursor, buffer) {
    prevFrame = [...buffer]   // spread operator: shallow copy of buffer array
}

export function main(coord, context, cursor, buffer) {
    // Read state from prevFrame, write to buffer (via return value)
    const current = get(x, y)  // reads prevFrame[y * width + x].char
    const neighbors = get(x-1,y-1) + ... + get(x+1,y+1)
    return current ? (neighbors==2||neighbors==3 ? '▒' : ' ') : (neighbors==3 ? 'x' : ' ')
}
```

`[...buffer]` is a shallow copy — each element is still the same object reference, but the array itself is new. Since the engine overwrites `buffer[i]` by replacing the object (not mutating it), `prevFrame` correctly holds the previous frame. Cursor click spawns new life in a radius-3 circle.

---

### Sand Game (Alex Miller)
**Falling-sand simulation — gravity via buffer neighbor reads.**

```javascript
export function main(coord, context, cursor, buffer) {
    const me = get(x, y)
    const below = get(x, y + 1)
    const above = get(x, y - 1)

    if (alive(me)) {
        // Fall if below is empty and one diagonal is empty (alternates by frame)
        char = (alive(below) && alive(frame%2==0 ? bottomright : bottomleft)) ? me : ' '
    } else {
        // Receive falling sand from above or diagonals
        if (alive(above)) char = above
        else if (alive(left) && frame%2==0 && alive(topleft)) char = topleft
        // ...
    }
}
```

Each `get()` reads from `prevFrame` (copied in `pre()`). Gravity: sand falls down, then diagonally. `frame % 2` alternates which diagonal is tried — prevents symmetric jamming. Boundary cells are styled walls and ground using `backgroundColor`. Four letters of `'sand'` are randomly assigned as particle chars, giving visual texture.

---

### Pathfinder (Alex Miller)
**Self-propagating box-drawing roads via neighbor adjacency rules.**

```javascript
const roads = '┃━┏┓┗┛┣┫┳┻╋'

export function main(coord) {
    if (last == ' ') {
        if ('┃┫┣╋┏┓┳'.includes(top))    char = choose('┃┃┃...┗┫┣┻╋')   // extend down
        else if ('━┓┛┫┳┻╋'.includes(right)) char = choose('━━━...┏┗┣┳┻╋')  // extend left
        // ...
    }
}
```

Each empty cell checks its four neighbors (from `prevFrame`). If a neighbor has a road endpoint pointing toward it, the cell draws a connecting road. `choose()` picks randomly from a weighted string — straight runs appear more often than corners. Clicking spawns a seed road. The result is a city that organically grows from clicks.

---

### Slime Dish (zspotter)
**Physarum (slime mold) agent simulation with zoom/pan view.**

```javascript
// 1500 agents on a 400×400 grid
// Each agent: sense-rotate-move-deposit per frame (pre())
class Agent {
    react(chem) {
        const forwardChem = this.sense(0, chem)   // sample ahead
        const leftChem    = this.sense(-1, chem)  // sample left
        const rightChem   = this.sense(1, chem)   // sample right
        // Rotate toward highest concentration
        if (forwardChem > left && forwardChem > right) rotate = 0
        else if (left < right) rotate = AGT_ANGLE
        // Move in current direction
        this.pos = add(this.pos, mulN(this.dir, AGT_SPEED))
    }
    deposit(chem) {
        chem[pos.y*HEIGHT + pos.x] = min(1, chem[pos.y*HEIGHT+pos.x] + DEPOSIT)
    }
}

// Diffuse + decay the chemical field (pre())
for each cell: chem[i] = DECAY * blur3x3(chem, row, col)

// Render (main()): nearest-neighbor downsampling + alternating texture rows
const texRow = (coord.x + coord.y) % TEXTURE.length   // alternating rows
const texCol = ceil(val * (TEXTURE[0].length - 1))
return TEXTURE[texRow][texCol]
```

The simulation runs at 400×400, the display is `cols × rows` — a significant scale difference. `main()` performs nearest-neighbor downsampling from simulation to display coordinates. Pressing the mouse zooms to 1:1 (one simulation cell per display cell), focused on cursor position. The `boot()` callback initializes agents, chemical array, and view parameters through the `data` object — the first example in the catalogue that uses the `data` (userData) parameter for persistent simulation state.

---

### Stacked Sin Waves (Raurir)
**Layered sine boundaries with block character fill.**

```javascript
const chars = "█▓▒░ ".split('')

function wave(t, y, seeds, amps) {
    return (
        (sin(t + y * seeds[0]) + 1) * amps[0]
        + (sin(t + y * seeds[1]) + 1) * amps[1]
        + sin(t + y * seeds[2]) * amps[2]
    )
}

export function main(coord, context) {
    const v0 = cols/4 + wave(t, y, [0.15, 0.13, 0.37], [10,8,5]) * 0.9
    const v1 = v0 + wave(t, y, [0.12, 0.14, 0.27], [3,6,5]) * 0.8
    const v2 = v1 + wave(t, y, [0.089, 0.023, 0.217], [2,4,2]) * 0.3
    const v3 = v2 + wave(t, y, [0.167, 0.054, 0.147], [4,6,7]) * 0.4

    const i = x > v3 ? 4 : x > v2 ? 3 : x > v1 ? 2 : x > v0 ? 1 : 0
    return chars[i]
}
```

Four accumulating wave thresholds divide each row into 5 horizontal bands. Each band uses a different block character (`█▓▒░ `), creating a stacked wave landscape with visible density gradations. Seeds and amplitudes chosen empirically for visual balance.

---

## Cross-Cutting Techniques Observed Across All Programs

### The `sort()` Module
Several programs use `sort('/\\MXYZabc!?=-. ', 'Simple Console', false)` to reorder density strings. The module renders each character into a canvas and measures actual pixel coverage, producing a ramp that is perceptually linear for the specific font. Without this, a density ramp in code-point order may not match visual weight.

### The `exp()` Edge Pattern
The formula `1 - exp(-k * abs(d))` appears across all SDF programs and some procedural demos. It maps any signed distance field to [0, 1] with an exponential edge (bright at zero, approaching 1 far away). Adjusting `k` controls edge sharpness. `k=3` is soft; `k=10+` is nearly binary.

### Buffer Pre-Copy for Temporal Access
Any program needing to read the previous frame must copy the buffer in `pre()`: `prevFrame = [...buffer]`. Spreading creates a new array but each element is still a reference — this is correct because the engine *replaces* `buffer[i]` (doesn't mutate its properties). Used in: GOL, Sand, Pathfinder.

### Custom Buffer Properties
`buffer[i]` can carry arbitrary custom fields alongside `{char, color, backgroundColor, fontWeight}`. Examples:
- `buffer[i].value = 0.7` (Dynadraw — ink accumulation float)
- `buffer[i].value` decays by `*= 0.99` each frame

The engine only reads the four standard fields when rendering; extra fields persist across frames as long as they're not clobbered by resize.

### Simulation in `pre()`, Display in `main()`
Every simulation program (GOL, Sand, Slime, Donut, Dynadraw, Golgol) places all state advancement in `pre()` and makes `main()` a pure lookup. This is the idiomatic pattern:

```
pre()  → advance physics/automaton → write to data[]
main() → read data[coord.index] → return char
```

The separation makes main() testable in isolation and keeps per-cell cost minimal.

### Event Handlers as Program Exports
```javascript
export const pointerDown = () => mode = ++mode % 3   // Moiré
export const boot = (ctx) => ctx.settings.element.style.cursor = 'pointer'
```

The runner's event queue dispatches named events to matching program exports. Any of `pointerMove`, `pointerDown`, `pointerUp` can be exported to receive events *after* the frame renders (at step 6 of the loop), keeping physics separate from rendering.

---

## Module Reference Summary

| Module | Key exports | Used in |
|--------|-------------|---------|
| `num.js` | `map`, `mix`, `clamp`, `smoothstep`, `fract` | Nearly all programs |
| `vec2.js` | `vec2`, `add`, `sub`, `length`, `rot`, `dot`, `dist`, `mulN` | SDF, camera, Dyna, Slime |
| `sdf.js` | `sdCircle`, `sdBox`, `sdSegment`, `opSmoothUnion` | All SDF programs |
| `color.js` | `CSS4`, `CGA`, `C64`, `css()`, `rgb2hex()`, `rgb2gray()` | Numbers, camera, Color Waves |
| `camera.js` | `Camera.init()` | All camera programs |
| `canvas.js` | `new Canvas()`, `.cover()`, `.mirrorX()`, `.quantize()`, `.writeTo()` | All camera programs |
| `drawbox.js` | `drawBox()`, `drawInfo()` | Most demos (HUD overlay) |
| `sort.js` | `sort(str, font, ascending)` | Circle, Spiral, camera_gray |
| `exportframe.js` | `exportFrame(context, filename, start, end)` | Sequence export |
| `image.js` | Image loading + sampling | (not covered here) |
