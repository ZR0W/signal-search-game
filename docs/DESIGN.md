# Signal Search: design document

Last updated 2026-11-10. This records the concept, what the prototype implements, the decisions made so far, and what is still open. Numbers quoted here match `index.html` at the time of writing; the code is the source of truth.

## 1. Concept

An imperfect-information pathfinding game. The player probes a hidden landscape by sending signals along paths, and builds an understanding of it from the data that comes back.

Inspirations:

- **Gradient descent.** A node on a surface knows only the local slope. Peaks, valleys, local minima, and saddles are the hazards.
- **Movie hacking.** You route signals, decide how strong they are, what path they take, and what they do. You read reports; you do not steer anything directly.
- **Data as interface.** The main feedback is telemetry charts ("spectrogram-like" strip charts of a run). The map fills in only as information is gathered.

Core loop: **draw a path → choose a node loadout and commit energy → the run resolves on the true terrain → read the report → the believed map updates → plan the next turn.**

### Origin notes (from the designer's notebook, Nov 7)

- Working titles: "code game", "keyboard hacker game", "cellular automata game".
- Sketches of a topographic contour map with a path routed around hills toward a star (the objective).
- Rule sketch: `if can_move(): move else move(down)`.
- "Map discovery", "single first", "draw path", "max length (energy)", "commit (10 turns)", "search for prizes".
- Display ideas: per-path charts of effort/slope against step, with the area under the curve equal to energy spent (`∫f(x) = max energy`).
- "Poles as help" (relay-type structures), "Objs? moving target that responds".
- Path options: A*, custom gradient, drawn; greedy; "cliff diving"; "map solvability".
- "Composition of a travel signal": PATH algorithm and GATHER function share max strength / resource cost.

## 2. The two layers

1. **The real world** is deterministic. A height function `h(x, y)` exists on the unit square. A run's cost and outcome are fully determined by the terrain under it.
2. **The player's belief** is an estimate rebuilt from readings. The player never sees `h` directly (except with the debug "Reveal truth" toggle).

The tension of the game is the gap between the forecast (computed on belief) and the result (computed on truth).

## 3. World

### Representation

Terrain is generated onto a 160×160 height grid (`NG`), then sampled with bicubic (Catmull-Rom) interpolation, so movement and slopes are continuous. There is no gameplay grid: paths are free curves. The gradient is a central finite difference of the interpolated surface.

Earlier builds used a closed-form sum of Gaussians. The grid replaced it so that every generator, including erosion, plugs into the same physics.

### Generator layers

Layers are summed, optionally eroded, then normalised to ±`relief`.

| Layer | What it adds | Controls |
| --- | --- | --- |
| Layered noise | Rolling ground at several scales (fractal Perlin noise). Roughness sets how many small pits exist | weight, base frequency, octaves, roughness |
| Ridges | Sharp crests with passes (ridged noise), bent by domain warp | weight, frequency, octaves, warp, invert to carve valleys |
| Test-function basins | Classic optimisation landscapes sunk into the map: Rosenbrock (long curved valley), Rastrigin (egg crate of pits), saddle, Ackley (one deep hole in rough ground) | depth, size, per-function toggles |
| Blobs | The original generator: random Gaussian hills and pits on a plain | weight, large count, small count, pit share |
| Erosion | Droplet erosion carving branching channels | raindrops, strength |

Presets: **Hybrid (default)**: noise + ridges + Rastrigin and Ackley basins; Blobs; Layered noise; Ridged + warped; Test-function regions; Eroded noise. Changing any layer control switches the preset to Custom.

Each preset carries its own relief, calibrated so roughly 10–30% of the ground exceeds the stall slope of 4.5 (Hybrid about 10%).

### Fairness rules

- Flatten ground around the base (Gaussian blend to the local mean).
- Deepest point at least a set distance from the base (default 0.4).
- A stall-free route from base to the deepest point must exist (breadth-first search on the grid, 8-connected, blocked where uphill slope exceeds the stall limit).

If a rule fails, the seed is incremented and the world regenerated, up to 12 times.

### Why not cellular automata

The original notes considered cellular automata. With the grid gone as a gameplay concept, the idea survives only as a possible way to drive a dynamic world (hazards or an environment opponent): diffusion fields, Lenia/SmoothLife-style continuous automata, or agent swarms. None of that is built.

## 4. Nodes

A node is a combination of four parts, all paid for from one energy budget.

| Part | Options in the prototype |
| --- | --- |
| Movement after the drawn path | stop; descend (greedy steepest descent); roll (momentum); walk contour; split into 3 children |
| Sensor | endpoint only; trace (readings along the whole run, +15% travel cost) |
| Reporting | burst at end; live stream (+25% travel cost) |
| Steep rule | stop; slide downhill; sidestep |

Presets: Ping, Trace, Descender, Roller, Contour, Splitter.

**Decided:** a burst node that runs out of energy reports nothing; a streaming node keeps what it sent. Rules are chosen from menus, not written as code (for now). Energy is one shared pool for travel and sensing.

### Ideas not built

- **Planner (A\*)**: plans on the belief map, with an optimism setting for fog.
- **Annealer**: descent with random uphill steps, set by a temperature.
- **Relay pole**: static; extends reporting range or reduces noise ("poles as help").
- **Sonar**: line-of-sight rays; sees further from high ground.
- **Curvature probe**: reports bowl, dome, or saddle.
- Sensors for the full gradient (including the sideways component) as a premium option.

## 5. Energy

- Game: 60 reserve, 10 turns. Committing takes the slider value (1–12) from the reserve. No refunds or income.
- Per step (0.004 map widths): `distance × 2 × multiplier + climb × max(0, height gained)`. Climb cost defaults to 10. Downhill costs only distance.
- A share of committed energy can be set aside for an end scan; radius `0.03 + 0.025 × scan energy`, capped at 0.2. It fires only if the node ends alive.
- Stalling is independent of energy: any step with slope above the stall limit (default 4.5) triggers the steep rule.
- Leftover energy when a node stops is lost.
- Split divides remaining energy equally among 3 children.

## 6. Belief map

Gaussian process regression on height readings, written by hand in `fitBelief`.

- Squared-exponential kernel, length scale 0.075, prior variance `(0.5 × relief)²`, noise 0.003.
- Readings carry Gaussian noise (σ 0.02). Readings closer than 0.016 are merged; the total is capped at 480.
- Evaluated on a 33×33 grid and bilinearly interpolated for drawing and forecasts.
- Fog is the posterior standard deviation relative to the prior.
- The cost forecast runs the energy formula along the belief mean. Its ± band is a heuristic, not a derived quantity.

Known limits: slope readings are recorded but not used by the estimate; the fixed length scale smooths away narrow features; the forecast does not cover the automatic part of a run.

## 7. Interface

- **Play view:** belief map with fog, path drawing, node loadout, forecast, report chart (height, slope along heading, energy vs distance), run log, rules summary.
- **World settings view** (`#world`): preset, seed, layers, fairness rules, physics; live preview of true terrain with stall zones, a stall-free route, deepest point, and stats (steep %, flat %, pits, distance, route).
- World settings persist in `localStorage`.
- Art direction: a neon instrument console. Black panels, amber for the player's signals and interface chrome, phosphor green for terrain and readings, cyan for the slope trace, red for alerts, violet for flat ground. Hazard striping, scanlines, square corners, condensed uppercase display type with monospace readouts. The mood is taken from anime control-room displays (Neon Genesis Evangelion was the stated inspiration); no logos, names, or graphics from any existing work are used.
- The map is drawn as a dark green height ramp with bright contour lines; unexplored fog is black.

## 8. Reference games

- IMAGINARY "Gradient Descent" exhibit: probing a hidden terrain for its lowest point.
- Exapunks: small programmable agents moving through a network; scored on efficiency.
- Duskers: indirect control through commands and partial sensors.
- HighFleet: active sensing gives your position away (a model for player-versus-player).
- Radio Determination: terrain shapes signal strength; triangulation.
- Frozen Synapse (simultaneous planned turns) and Screeps (scripted units on an energy economy).

## 9. Open questions

1. **Goals and opponents.** Deliberately deferred. Notes: do not put the prize at the global minimum or plain descent solves the game; consider a moving target that responds; for player-versus-player, scanning should reveal the scanner.
2. **Balance.** Stall limit, climb cost, reserve, turn count, and fog clearing rate are untested guesses.
3. **Leftover energy.** Lost, refunded in part, or banked in the node?
4. **Belief upgrades as "cost of intelligence".** Raw dots → smoothed map → uncertainty shading → slope-aware estimate.
5. **Rules as code.** Menus now; a small scripting layer is a possible later direction.
6. **Dynamic world.** Hidden fields beyond height (interference, opponent signal) that appear only in reports.
7. **End state.** There is no win, loss, or scoring screen.
