# webart — Design Spec

**Date:** 2026-04-01  
**Status:** Approved

---

## Overview

A personal creative playground for generative, interactive fluid art rendered as a character grid. The user edits a single sketch file locally; the browser reloads instantly via Vite HMR. Inspired by [play.ertdfgcvb.xyz](https://play.ertdfgcvb.xyz), which is built on the ABC library.

---

## Stack

| Layer | Choice | Reason |
|---|---|---|
| Runtime | [ABC library](https://play.ertdfgcvb.xyz) | Same lib as ertdfgcvb; provides char grid loop, lifecycle hooks, DOM renderer |
| Dev server | Vite | Instant HMR — sketch file changes reflect without page reload |
| Language | Vanilla JS (ES modules) | No framework overhead; ABC is module-native |

---

## Project Structure

```
~/webArt/
├── index.html        ← fullscreen ABC output canvas, imports sketch
├── src/
│   └── sketch.js     ← the only file the user edits
├── docs/
│   └── design.md     ← this file
├── vite.config.js    ← minimal config, no plugins needed
└── package.json      ← deps: abc-js, vite
```

---

## Architecture

ABC provides the character grid loop. The sketch exports four lifecycle hooks:

### `boot({ cols, rows })`
Runs once before the first frame. Allocates the fluid simulation grids:
- `vel` — `Float32Array(cols * rows * 2)` for vx, vy per cell
- `density` — `Float32Array(cols * rows)` for fluid density per cell
- `velPrev` / `densityPrev` — scratch buffers for the solver

### `pre(context, cursor)`
Runs once per frame before `main()`. Responsible for:
1. Injecting density and velocity at the cursor position (interactive)
2. Running the Navier-Stokes solver **twice** (two full passes of diffuse → advect → project)

Two solver passes per frame improve stability and produce richer vortex behaviour without a meaningful performance cost on a ~5000-cell char grid.

### `main(coord, context, cursor)`
Runs once per cell per frame. Reads from the fluid state and returns a styled character:

```js
export function main({ x, y }, { cols }) {
  const i   = y * cols + x
  const d   = density[i]
  const vx  = vel[i * 2]
  const vy  = vel[i * 2 + 1]
  return {
    char:       flowChar(vx, vy),
    color:      densityColor(d, vx, vy),
    fontWeight: speedWeight(vx, vy),
  }
}
```

### `post()` (optional)
Reserved for future overlays (e.g. debug velocity arrows, cursor indicator).

---

## Fluid Simulation: Navier-Stokes (Jos Stam, 1999)

Based on *Real-Time Fluid Dynamics for Games* (Stam, GDC 1999). Each solver pass runs three steps on both the density and velocity fields:

1. **Diffuse** — spread values to neighbours via Gauss-Seidel relaxation (20 iterations)
2. **Advect** — move values along the velocity field using bilinear interpolation
3. **Project** — enforce divergence-free flow (prevents energy blow-up); also uses Gauss-Seidel

Boundary conditions: wrap or clamp at grid edges (configurable).

---

## Character Visual Mapping

| Fluid signal | Visual property | Detail |
|---|---|---|
| Velocity angle | Character | 8-direction set: `· - \| / \ ↗ ↘ ↙ ↖` |
| Velocity magnitude | `fontWeight` | `300` (slow) → `700` (fast) |
| Density | Color brightness | `hsl(h, 80%, density * 60%)` |
| Vorticity (curl of vel) | Hue | Clockwise → warm (orange/red), counter → cool (blue/cyan) |

---

## Interaction

- **Mouse move** — injects a density burst + velocity impulse at cursor position each frame
- **Mouse down** — increases injection strength (stronger force)
- Force magnitude and radius are tunable constants at the top of `sketch.js`

---

## Dev Workflow

```bash
cd ~/webArt
npm install
npm run dev        # Vite starts, opens browser
# edit src/sketch.js → browser reflects changes instantly (no page reload)
```

ABC reruns `boot()` and restarts the animation loop when the module hot-reloads.

---

## Out of Scope (v1)

- Multiple sketches / gallery
- Per-character size, rotation, opacity (requires Canvas 2D renderer — deferred)
- Saving/sharing sketches
- Mobile touch support
