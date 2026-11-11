# Signal Search

A browser game prototype about pathfinding with imperfect information. You send signals ("nodes") along paths you draw through a terrain you cannot see, spend a limited energy reserve doing it, and rebuild a map of the world from the telemetry they send back.

The idea comes from gradient descent: at any point you only know the local slope, and the danger is getting stuck in a local minimum. The mood comes from movie-style hacking, where you route signals and read reports rather than steer a character.

## Play

Open `index.html` in any modern browser. There is no build step and no dependencies (fonts load from Google Fonts and fall back to system fonts offline).

To host it, enable GitHub Pages for this repo (Settings → Pages → deploy from branch `main`, folder `/`).

0. **Start screen.** Set rounds, energy reserve, energy per node, sensor noise, and world type, then start.
1. **Plot a path.** Tap the map for a straight line from the nearest node, or drag to draw. The forecast panel shows the estimated cost ± uncertainty against the energy you have committed.
2. **Pick a node loadout.** Use a preset (Ping, Trace, Descender, Roller, Contour, Splitter) or set the menus and sliders yourself.
3. **Commit.** The run plays out on the real terrain. The report chart shows height, slope along the heading, and energy against distance traveled. New readings lift the fog.
4. **Repeat** until your rounds or reserve run out (10 rounds and 60 energy by default). "Reveal truth" shows the real terrain and its deepest point.

The **World settings** tab controls terrain generation: presets, seed, five stackable layers, fairness rules, and physics, with a live preview.

## Repository layout

| Path | What it is |
| --- | --- |
| `index.html` | The whole game: markup, styles, and script in one file |
| `docs/DESIGN.md` | Design document: concept, mechanics as built, decisions, open questions |
| `AGENTS.md` | Handoff notes for an AI agent or developer picking this up |
| `CLAUDE.md` | Pointer to `AGENTS.md` for Claude Code |

## Status

Single-player prototype of the world and interface. Goals, objectives, and opponents are deliberately not designed yet. Balance numbers are first guesses. See `docs/DESIGN.md` for what is decided and what is open.
