# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev      # Vite dev server with HMR on the local network
npm run build    # Production bundle (also used as the lint/check step)
npm run preview  # Serve production build
```

There are no automated tests. `npm run build` is the only correctness gate — a clean build means no syntax errors in shaders or JS.

## Architecture

Flux is a hybrid CPU+GPU generative art piece. One animation frame:

1. `sim.injectForce()` → pointer → CPU fluid solver (`fluid.js`)
2. `sim.step()` → advances Navier-Stokes, packs RGBA into `sim.pixels`
3. `renderer.uploadFluid(sim.pixels, cols, rows)` → GPU texture unit 1 (NEAREST)
4. `wordCycler.update()` → renders split-flap text onto a 1024×256 off-screen canvas
5. `renderer.uploadWordTexture(canvas)` → GPU texture unit 2 (LINEAR, small→fullscreen)
6. `renderer.draw(time, pointer)` → single fullscreen quad draw call

**`src/sketch.js`** — the editable art layer. Contains the GLSL fragment shader, vertex shader, character/font config (`config`), and `staticUniforms` (constants that survive shader hot-reload). Edit this file to change the visual; Vite HMR calls `renderer.recompile()` without a page refresh.

**`src/settings.js`** — single source of truth for all tunable numbers: grid font, density character ramp, fluid coefficients, pointer forces, word animation parameters. Constants that the shader needs are assembled into `staticUniforms` in `sketch.js`.

**`src/renderer.js`** — WebGL wrapper. Builds the glyph atlas (one-row RGBA texture, centered glyphs, real `fontBoundingBoxAscent/Descent` metrics), manages 5 texture units (atlas=0, fluid=1, wordTex=2, wordDepartTex=3, overlay=4), and exposes `uploadFluid`, `uploadWordTexture`, `uploadDepartWordTexture`, `resize`, `draw`, `recompile`, `dispose`.

**`src/words.js`** — split-flap lyric cycler. Two-phase animation: `arrive` (characters cycle toward `target[]` with center-out stagger) then `depart` (characters cycle toward `next[]`). Renders white glyphs onto a small off-screen canvas. The word bitmap drives the `wordBoost` density term in the shader — giant letters emerge from the character field without any separate render pass. The depart canvas (`departCtx`) is always cleared; old lines are erased by the background fluid naturally.

**`src/simulation.js`** — wraps `fluid.js`. Exposes `injectForce`, `step`, `resize`, and `pixels`. **Never cache direct references to buffer properties** — always read through the state object (e.g., `sim.pixels` not `const p = sim.pixels`) because `resize()` replaces them.

## Key Shader Uniforms

| Uniform | Source | Notes |
|---|---|---|
| `u_wordTex` | `words.js` canvas | sampled via `.r` channel |
| `u_wordDepartTex` | `words.js` departCanvas | sampled via `.a` channel — browsers un-premultiply before `texImage2D`, so `.r` stays 1.0 at glyph pixels when `globalAlpha < 1` |
| `u_fluid` | `simulation.js` pixels | R=density, G=vx packed [0,1], B=vy packed [0,1], A=speed |
| `u_seed` | `renderer.js` | random offset per session, applied once |
| `u_densityCharCount` | `staticUniforms` | length of `gridDensityChars` |

## Build-Time Globals

`vite.config.js` injects `__COMMIT_HASH__`, `__COMMIT_BRANCH__`, and `__BUILD_TIME__` (ART timezone) via `define`. These appear in the overlay shown on pointer enter.

## Deployment

`.github/workflows/deploy.yml` — on push to `main`, builds with `--base=/webArt/` and deploys to GitHub Pages.
