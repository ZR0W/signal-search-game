# Agent handoff

Read this first, then `docs/DESIGN.md`. The design document says what the game is and why; this file says how the code is laid out and how to work on it.

## What this is

A single-file browser game prototype. `index.html` holds markup, CSS, and one inline `<script>`. No build, no dependencies, no tests. Fonts come from Google Fonts with system fallbacks.

It was prototyped in a chat session and published as a hosted page whose content security policy only allows inline scripts and a few CDNs. Keep the single-file, no-fetch shape unless the owner asks to change it, so the same file still works there.

## Run and check

- Open `index.html` in a browser, or serve the folder: `python3 -m http.server`.
- Syntax check the script:
  ```sh
  python3 -c "import re;s=open('index.html').read();open('/tmp/t.js','w').write(re.findall(r'<script>(.*?)</script>',s,re.S)[0])" && node --check /tmp/t.js
  ```
- The generation code (everything above the `/* ---------- game state` comment) has no DOM dependency apart from the `$` helper and the `localStorage` read, so it can be run under Node to measure presets. This is how relief was calibrated.

## Code map (`index.html` script, top to bottom)

| Section | Key names | Notes |
| --- | --- | --- |
| Constants | `STEP`, `ELL`, `NOISE`, `G`, `MAXOBS`, `NG` | Step size, GP length scale and noise, belief grid, reading cap, terrain grid |
| Noise and terrain blocks | `perlin`, `fbm`, `ridged`, `warp`, `TF`, `patch`, `erode`, `normTo` | `TF` holds the optimisation test functions |
| World settings | `OFF`, `mk`, `PRESETS`, `PRESET_NAMES`, `DEF`, `S`, `KEY` | `S` is the live settings object; saved to `localStorage` with a version field `v` |
| World generation | `generate(S, seed)`, `buildWorld(S, seed?)` | `generate` returns `{H, cls, base, min, route, stats, dist, physics, relief, ...}`; `buildWorld` rerolls up to 12 seeds to satisfy fairness rules |
| Game settings | `GDEF`, `GS`, `GM`, `GKEY` | `GS` is what the start screen edits and saves; `GM` is the snapshot taken in `newGame` and is what game code reads |
| Terrain sampling | `h(x, y)`, `grad(x, y)` | Bicubic interpolation of `W.H`; gradient by finite difference |
| Belief | `addObs`, `fitBelief`, `interp` | Hand-written Gaussian process with Cholesky; `belief = {mean, sd}` on a `G×G` grid |
| Node simulation | `cfg`, `mult`, `runAgent`, `STAT` | `runAgent` returns a list of agents (more than one after a split), each `{pts, status, E, ...}` |
| Turn flow | `updatePlan`, `commit`, `applyRun` | Forecast, run and animation, then readings and anchors are applied |
| Drawing | `ramp`, `buildImage`, `render`, `renderReport`, `updateStats`, `updateRules` | Canvas 2D; map canvas is 640², report is 720×380 |
| Play input | pointer handlers on `#map`, `NPRE`, `newGame(world)` | `NPRE` is the node preset table |
| World settings view | `SCHEMA`, `buildForm`, `syncForm`, `schedule`, `regen`, `drawPreview` | The form is generated from `SCHEMA`; add a control there and it is wired automatically by its settings path |
| Start view | `GPRE`, `GCTL`, `syncStart`, `#bStart` handler | Game presets and controls; Start regenerates the world (new seed if enabled) and calls `newGame` |
| Views | `route` | empty hash shows start, `#play` shows the game, `#world` shows world settings |

Global game state: `W` (current world), `PW` (preview world), `obs`, `runs`, `anchors`, `turn`, `reserve`, `path`, `belief`, `lastRun`, `S2` (GP prior variance, set from relief in `newGame`).

Coordinates are in the unit square with y pointing down, matching canvas space.

## How to make common changes

- **New terrain layer:** add defaults to `OFF`, add its key to `LAYERS`, add its contribution inside the per-cell loop in `generate`, and add a section to `SCHEMA`. Then re-measure slope percentiles and set preset relief so the stall share stays sensible.
- **New node movement mode:** add an `<option>` to `#fMove`, a branch in the automatic-mode block of `runAgent`, a label in `names` (`applyRun`) and `autoTxt` (`updatePlan`), and optionally a row in `NPRE`.
- **New sensor or report type:** extend `mult` for its cost, `commit` for what is `known`, and `applyRun` for which readings are added.
- **Changing saved settings shape:** bump the version check (`s.v!==2`) and the saved `v`, or old saved settings will load into the new shape.
- **New game setting:** add a default to `GDEF`, a control in the start view markup, wire it in the start view section (`GCTL` for sliders), read it as `GM.<name>` in game code, and mention it in `updateRules`.
- **Physics defaults:** `DEF.physics`. The forecast, simulation, preview stall zones, route search, and rules text all read from `W.physics` or `S.physics`.

## Known issues and gaps

- Never played through in a real browser by the author of the code. Syntax and world generation were checked under Node only. Expect layout and balance problems.
- Minimal end-of-game state: the forecast panel reports deepest reading against true minimum. No score or summary screen.
- A jsdom smoke test (canvas stubbed) of start → play → commit has been run by hand; there is no test file in the repo.
- The belief estimate ignores slope readings and uses a fixed length scale (see design document, section 6).
- The forecast's uncertainty band is a heuristic and the automatic part of a run is not forecast.
- `fitBelief` is O(n³) in readings (cap 480) and runs on the main thread. Erosion at high drop counts is also slow. Neither uses a worker.
- The stall-free route check is on the 160 grid and 8-connected; the node simulation uses 0.004 steps on the interpolated surface, so they can disagree at the margin.
- Map canvases have fixed pixel sizes and are scaled by CSS.

## Suggested next steps

In rough priority order; confirm with the owner before starting, since goals and opponents are still open design questions.

1. Play it and tune: stall limit, climb cost, reserve, turn count, fog clearing.
2. Add a proper end screen and a score (for example, deepest point found relative to the true minimum).
3. Use slope readings in the belief estimate (gradient observations in the GP), possibly as an upgrade.
4. Add the unbuilt node types: Planner (A* on belief), Annealer, Relay pole, Sonar, Curvature probe.
5. Design objectives and opponents with the owner.
6. If the file grows much further, split into modules and add a small test harness for `generate` and `runAgent`.

## Working conventions

- Keep the design document current when mechanics or numbers change.
- User-facing text: plain, short sentences; sentence case; say exactly what a control does.
- Colours come from the CSS custom properties in `:root`: amber (`--signal`) is the player's signals and chrome, green (`--cyan`, `--green`) is terrain readings, cyan (`--coral`) is the slope trace, red is alerts. The variable names predate the neon restyle. Canvas drawing code uses the same hex values as literals, plus the `ramp` function for terrain; change both together.
- Keep the art direction original: inspired by anime control-room displays, with no logos, names, or graphics from existing works.
- Respect `prefers-reduced-motion` (the run animation is already skipped under it).
