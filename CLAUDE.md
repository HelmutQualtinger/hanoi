# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained static web page (`index.html`) implementing a playable Towers of Hanoi game in the browser using Three.js. There is no build system, package manager, bundler, linter, or test suite — all HTML, CSS, and JavaScript live in this one file. `README.md` is empty and `.python-version` is a stray leftover with no bearing on this project (there is no Python code here).

## Running it

The page must be served over HTTP, not opened as a `file://` URL, because it uses ES module imports (`<script type="importmap">` + `<script type="module">`) which browsers block under `file://`.

```
python3 -m http.server 8787
```

Then open `http://localhost:8787/index.html`. Avoid port 5000 (macOS AirPlay uses it).

There is no build/lint/test tooling — verify changes by loading the page in a browser and checking the devtools console for errors.

## Architecture

Everything is in `index.html`, organized top to bottom as: `<style>` (HUD/UI CSS) → HUD markup (title, stats, controls, win overlay) → a single `<script type="module">` containing all Three.js scene setup and game logic. Three.js is loaded from a CDN via an import map (`three` + `three/addons/`), not vendored locally.

The module script is organized in commented sections, in this order:

- **Renderer/scene/camera** — WebGLRenderer, ACES tone mapping, PMREM-generated environment (via `RoomEnvironment`) for PBR reflections, and `OrbitControls` (damped, distance/polar-angle clamped, auto-rotates when idle).
- **Procedural textures** — `makeStoneTexture()` and `makeGlowSprite()` generate canvas-based textures at runtime (moss/stone surfaces, firefly glow sprites). No external image/texture assets are used anywhere.
- **Lighting** — ambient/hemisphere fill, a shadow-casting directional "moon" light, and two flickering torch point lights (animated in the render loop).
- **Environment geometry** — ground plane, three stacked cylinder "dais" tiers, scattered underbrush, and four concentric rings of instanced trees (`InstancedMesh` for trunks/foliage) surrounding the scene at 360°, plus a `THREE.Points` firefly system.
- **Game state** — `pegs` is the source of truth: an array of 3 arrays, each holding disk sizes bottom-to-top (largest number = biggest disk). `diskMeshes` maps disk size → its `THREE.Mesh`. Disk visuals are fully rebuilt by `buildDisks()`/`layoutPegInstant()` whenever the peg state changes structurally (new game, disk count change).
- **Interaction** — raycasting against three invisible full-height `hitCylinders` (one per peg) rather than the visible peg/disk meshes, so clicking anywhere along a peg's column selects/targets it. Click detection is `pointerdown`+`pointerup` with a distance/time threshold (not a native `click` event), specifically to distinguish a tap from an `OrbitControls` drag.
- **Move animation** — a single hand-rolled tween (`anim` object + `stepAnim()`, no animation library) moves one disk at a time along an eased arc; a `busy` flag blocks new input until it completes, and `autoSolving` additionally blocks input during auto-solve playback.
- **Auto-solve** — `planSolve()` is a general recursive Hanoi solver that computes the optimal move list from *whatever the current peg state is* (not just a fresh stack), so "Solve" works mid-game. `solvePuzzle()` then replays that plan through the same move/animate path as manual play, pacing itself with `setTimeout` gated on `MOVE_DURATION`.
- **Background music** — an original ambient piece synthesized live via the Web Audio API (drone oscillator + randomly-plucked Vietnamese pentatonic-scale notes + a feedback delay), not an embedded audio file. No copyrighted audio is used or fetched. Toggled on/off by the user (browsers block audio autoplay without a user gesture anyway).
- **Render loop** — a single `tick()` drives `requestAnimationFrame`, updating the move animation, torch flicker, firefly drift, the HUD timer, and `OrbitControls`.

### Key invariants when modifying game logic

- `pegs[i]` must always stay validly ordered (descending size from index 0); all move validation happens in `pegClick()` before mutating `pegs`.
- Visual disk position is derived from `pegs` (`layoutPegInstant`) — never move a disk mesh without also updating `pegs`, or the two will desync.
- Peg x-positions (`PEG_X`), disk radius (`diskRadius()`), and the hit-cylinder radius are geometrically related (hit cylinders must stay wider than the largest disk for a given `diskCount`, currently supports 3–7 disks). Changing max disk count or spacing requires re-checking these together.
- Any full-screen/overlay UI element (see `#win-panel`, `#toast`) needs `pointer-events: none` by default and `auto` only while actively shown — the `.panel` class sets `pointer-events: auto` unconditionally, which will silently swallow clicks on the 3D scene underneath if a panel is left in the DOM but visually hidden only via `opacity`/`transform`. This exact bug was hit once with `#win-panel`; keep it in mind when adding new overlays.

### Testing changes manually

There is no automated test suite. When verifying interaction changes in an automated browser (e.g. Claude in Chrome), be aware that a backgrounded/hidden tab (`document.hidden === true`) heavily throttles `requestAnimationFrame`, which will make the move animation appear to hang and `busy`/`autoSolving` appear stuck — this is a browser throttling artifact, not a game bug. Prefer driving interactions via `element.click()` / dispatched `PointerEvent`s at coordinates read from `getBoundingClientRect()` (or via `camera.project()` for 3D peg positions) over guessed pixel coordinates, since the screenshot pixel size and the page's actual CSS pixel viewport can differ in scale.
