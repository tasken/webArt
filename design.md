# Flux — Design Spec

**Date:** 2026-04-05
**Status:** Current

---

## Overview

A personal creative playground for generative, interactive text-mode art rendered as a character grid. The live app is a hybrid CPU+GPU renderer: a CPU-side Navier-Stokes fluid simulation feeds density and velocity data into a WebGL fragment shader, which generates the motion field, samples a font atlas to render glyphs, and applies OKLch perceptual color. Pointer input injects forces into the fluid sim and adds instant visual glow in the shader. Lyric lines emerge as giant background letters — a split-flap cycling animation renders text onto a small off-screen bitmap that the shader maps across the entire grid, warped by procedural noise, so massive words form from the underlying character density field.

---

## Stack

| Layer | Choice | Reason |
|---|---|---|
| Runtime | WebGL + Canvas font atlas + CPU fluid sim | GPU character-grid rendering with real fluid dynamics |
| Color | OKLch (Björn Ottosson, 2020) | Perceptually uniform: equal L steps look equally bright regardless of hue |
| Dev server | Vite | Instant HMR — sketch file changes reflect without page reload |
| Language | Vanilla JS (ES modules) | No framework overhead; browser-native modules |
| Deploy | GitHub Pages via GitHub Actions | Automatic deploy on push to main |

---

## Project Structure

```
webArt/
├── index.html              ← fullscreen canvas, imports main.js
├── src/
│   ├── main.js             ← entry: boot, pointer state, animation loop
│   ├── renderer.js         ← WebGL program, font atlas, fluid/word textures, resize/draw/dispose
│   ├── sketch.js           ← GLSL shader sources, visual config, staticUniforms, OKLch color
│   ├── settings.js         ← all tunable constants (font, fluid, field, pointer, word emergence)
│   ├── simulation.js       ← wraps CPU solver into frame-steppable sim with RGBA packing
│   ├── fluid.js            ← pure Navier-Stokes solver (addSource, diffuse, advect, project)
│   └── words.js            ← split-flap lyric cycler with off-screen bitmap output
├── design.md               ← this file
├── docs/superpowers/plans/ ← implementation plans (planning artifacts, not project docs)
├── .github/
│   ├── workflows/deploy.yml← GitHub Pages deploy workflow
│   └── instructions/       ← Copilot instruction files
├── vite.config.js          ← dev server config (host: true)
└── package.json            ← deps: vite (dev)
```

---

## Architecture

The app is split into five runtime modules:

### `main.js`
Entry point. Loads fonts, creates the renderer and word cycler, manages the animation loop, tracks pointer position and velocity, injects forces into the fluid simulation, and handles cleanup during hot-module replacement.

### `renderer.js`
Compiles the shader program, builds a single-row glyph atlas using real font metrics (`fontBoundingBoxAscent`/`Descent`) with glyphs centered in their cells, manages three GPU textures (font atlas on unit 0, fluid data on unit 1 with NEAREST filtering, word bitmap on unit 2 with LINEAR filtering for smooth upscale), keeps the canvas sized to device pixels, and pushes runtime uniforms into WebGL on each frame. Exposes `recompile(vertSrc, fragSrc, newStaticUniforms)` for shader-only HMR, `uploadFluid()` for CPU→GPU fluid transfer, and `uploadWordTexture()` for canvas-based word bitmap upload. A random `u_seed` uniform is set once per session so each page load looks different.

On creation, `renderer.js` accepts a `staticUniforms` map from the caller and applies each entry as a float uniform. These are re-applied after every `recompile()` call, so visual constants survive shader hot-reloads.

### `sketch.js`
Contains the editable art logic: the vertex shader, the fragment shader, and the character/font configuration. Exports two objects:

- **`config`** — `{ fontSize, fontFamily, chars }` consumed by the renderer to build the glyph atlas
- **`staticUniforms`** — `{ u_fieldTimeScale, u_fieldAmplitude, u_wordAspect, u_densityCharCount }` passed to the renderer at init and on recompile; these constants are uniform values rather than baked template literals, so they survive shader recompilation without string interpolation

The fragment shader generates the animated value field using domain warping with irrational frequency ratios (φ, √2), samples the CPU fluid texture for organic distortion, converts values to glyph lookups via `u_densityCharCount`, and shades with OKLch perceptual color. The giant-letter system samples a word bitmap texture scaled to fill the entire grid with aspect-ratio correction (`u_wordAspect`) and noise warp, blending the text shape into the background density.

### `simulation.js`
Wraps the CPU Navier-Stokes solver into a frame-steppable simulation. Maintains density and velocity fields as Float32Arrays, exposes `injectForce()` for pointer interaction, `step()` for advancing the simulation, and packs results into an RGBA Uint8Array (`pixels`) for GPU texture upload.

`resize(newCols, newRows)` recreates all internal buffers at the new size and re-assigns them onto the state object via `Object.assign`. **Callers must never cache direct references to buffer properties** (e.g. `const d = sim.density`) — always access them through the state object (`sim.density`, `sim.pixels`). Internal functions (`step`, `injectForce`) are safe because they always read through `state`.

### `words.js`
Split-flap word cycler that animates through a curated lyric list. Renders white text on a 1024×128 off-screen canvas with auto-scaling font size (scales down for long lines), real font metric centering, and characters cycling through the alphabet toward their targets — staggered per position like an airport departure board. Arrive/depart two-phase animation: the full line appears before any transition to the next begins. Returns the canvas for GPU texture upload each frame.

The word bitmap is drawn with `ctx.scale(1, wordScaleY)` where `wordScaleY = 0.25` — an intentional vertical squash. Values below 1 compress letters vertically, creating the tall/narrow split-flap aesthetic in the giant background characters. Tune freely in `settings.js` (0.1–1.0).

### `settings.js`
Single source of truth for all tunable constants: grid/font config, density character ramp, word emergence parameters (stagger, frame skip, canvas dimensions, `wordScaleY`), fluid simulation coefficients, procedural field timing (`fieldTimeScale`, `fieldAmplitude`), and pointer interaction forces. Constants needed by the shader are exported and assembled into `staticUniforms` in `sketch.js`.

### Runtime Flow

1. `document.fonts.ready` resolves in `main.js`
2. `createRenderer()` builds the WebGL program, glyph atlas, fluid/word textures, and applies `staticUniforms`
3. `createWordCycler()` prepares the split-flap animation canvas
4. On resize, `renderer.resize()` returns `{ cols, rows }` for simulation sizing
5. Each frame:
   - Pointer forces are injected into the fluid sim via `sim.injectForce()`
   - `sim.step()` advances the Navier-Stokes solver
   - `renderer.uploadFluid()` sends packed RGBA pixels to the GPU
   - `wordCycler.update()` advances the flap animation and renders to its canvas
   - `renderer.uploadWordTexture()` uploads the word canvas to the GPU
   - `renderer.draw()` sends time, pointer state, and uniforms to the shader
   - The fragment shader samples the fluid texture, warps coordinates, blends giant-letter density, picks characters, and applies OKLch color

---

## Shader Behavior

The fragment shader combines stateless procedural animation with live fluid simulation data:

1. **Fluid sampling** reads density, velocity, and speed from the CPU simulation texture
2. **Procedural background** uses domain warping (3 passes) and wave interference (5 layers) with irrational frequency ratios (φ, √2) to produce non-repeating ambient motion; strength and speed controlled by `u_fieldAmplitude` / `u_fieldTimeScale`
3. **UV warping** displaces coordinates by fluid velocity for organic distortion
4. **Pointer influence** adds glow/burst terms near the cursor for instant visual feedback
5. **Giant letters** scale the word bitmap across the full grid with aspect-ratio correction (`u_wordAspect`) and noise warp, blending text density into the background value
6. **Glyph lookup** converts the combined scalar value into a character index using `u_densityCharCount`
7. **OKLch color** maps vorticity, speed, and density to a cold palette (blue→cyan→purple, ~3.4–5.2 rad) that shifts warm (orange/red) on click

The `u_seed` uniform offsets time by a random amount per session, ensuring each page load starts with a different visual state.

---

## Character Rendering

The configured character ramp is:

```text
 .·:;-=+*abcXYZ#@W
```

The renderer measures the maximum glyph width across all characters and uses `fontBoundingBoxAscent` and `fontBoundingBoxDescent` for accurate cell height. Glyphs are drawn centered in their cells using `textAlign: 'center'` and `textBaseline: 'alphabetic'` positioning. The atlas is uploaded as a texture and sampled in the fragment shader using per-cell UVs. Different fonts or ramps change the entire feel of the piece.

---

## Color System

Colors use the OKLch perceptual color space (Björn Ottosson's OKLab, 2020). The pipeline:

1. Vorticity (fVy − fVx), background value, and pointer interaction drive a hue base
2. Cold palette clamps hue to ~3.4–5.2 rad (blue/cyan/purple)
3. Click burst blends toward a warm hue (~0.6 rad, orange/red) for tactile feedback
4. Chroma is driven by background amplitude, fluid speed, and pointer proximity
5. Luminance tracks the combined density/procedural value, capped at 0.95

---

## Interaction

- **Pointer move** — injects directional forces into the fluid sim + adds shader glow
- **Pointer down** — amplifies force injection and triggers warm color burst
- **Pointer leave / idle** — forces decay; fluid drains via per-frame velocity/density decay

---

## Dev Workflow

```bash
npm install
npm run dev        # Vite starts with HMR
# edit src/sketch.js → shader recompiles instantly (no page reload)
npm run build      # production bundle
```

The `import.meta.hot.dispose()` cleanup in `main.js` removes listeners, cancels the animation frame, and tears down GPU resources before the next module instance takes over. Shader-only edits trigger `recompile()` instead of a full page reload. `staticUniforms` from the new module are applied alongside the recompiled shader.

---

## Deployment

GitHub Pages deployment is automated via `.github/workflows/deploy.yml`. On push to `main`, the workflow installs dependencies, builds with `--base=/webArt/`, and deploys the dist folder.

---

## Roadmap

Roughly ordered by impact vs effort.

### Near-term

- **Multi-touch interaction** — track multiple simultaneous touch points so each contact injects forces independently; GPU glow renders for primary touch only
- **Low-res mode** — `?lowres` URL param (and auto-detect for low-end devices) halves the fluid simulation grid; the GPU upscales the texture, which is acceptable given fluid softness
- **Feedback / frame history** — `copyTexImage2D` captures each frame into `u_prevFrame`; shader blends it back with a decay factor for motion trails and ghosting

### Later

- **Screenshot / GIF export** — `S` key saves a PNG via `canvas.toBlob()`; `R` key toggles `.webm` recording via `MediaRecorder`
- **GPU-only simulation** — move the solver to WebGL 2 transform feedback or WebGPU compute shaders to eliminate the CPU→GPU transfer bottleneck and allow larger grids

### Out of scope (current)

- Multiple sketches / gallery
- Per-character size, rotation, opacity
- Saving/sharing sketches
- Parameter UI / dat.gui
