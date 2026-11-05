# Beaver Valley Idle

An incremental ("cookie clicker") game played over a living three.js valley. You gnaw
logs with a button and spend them in the lodgebook; every purchase is applied to the
real valley behind the interface, which keeps its water simulation, dam wear, wildlife
and day/night cycle from the game this is forked from.

## How this differs from Beaver Valley

The valley, water and wildlife code is unchanged. What was taken out: the joystick and
keys, the context action (chew/build/patch/splash), the build ghost, the action HUD,
the journal sheet and the portable save codes. What replaced it:

- **The idle layer** (its own `// ====` section, just before Update): `BUILDINGS`,
  `UPGRADES`, `costOf`/`lpsOf`/`clickPower`, the lodgebook renderer, the save, and
  `gnaw()`.
- **Purchases drive the world.** `applyWorld()` makes the valley match what has been
  bought: `placeDams` walks an ordered list of channel nodes (`damStepList` — one dam
  spanned and raised to full height before the next one downstream), `placeLodge`,
  `placeKits` and `placeSaplings` do the rest. `damStepList` depends on the river, so it
  is only built *after* `buildWorld` has seeded the noise — `startWorld` resets it.
- **The beaver wanders.** `pickWander()` sends it between the dam, the lodge and the
  water; `update()` steers toward that instead of reading input.
- **The camera** keeps a slow circuit of whatever has been built, re-centring every two
  seconds on the dam sections and lodge.
- **Saves** are `localStorage['beaverValleyIdle1']`: logs, counts, upgrades, flood marks
  and a timestamp. The world is *derived* from the counts rather than stored, so loading
  is just `startWorld(save)`. Time away pays out at half rate, capped at eight hours.

## Layout

Everything is in **one self-contained file, `index.html`**: HTML, CSS and a single `<script type="module">`. three.js r160 loads from jsdelivr through an import map. There are no other assets: every texture is painted onto canvases at load time, and all sound is synthesised with WebAudio.

The main sections inside the script are marked with `// ====` banner comments, in this order:

| Section | What it does |
|---|---|
| Sound | `Snd`, a WebAudio synth (chomp, splash, quack, ambience) |
| Renderer, scene, lights, sky | Renderer, `LMat()`, the post-processing pass (watercolour/cel), `bakeDirect()`, the sky shader, stars |
| Procedural textures | Tileable canvas textures (grass, leaf litter, mud, rock, bark, leaves, fur noise, feathers…), `STD()`, `furShellMat()`/`furry()` (shell-rendered fur), `bakeByMaterial()` |
| Models | `latheZ()` body builder, logs, `makeBeaver()` |
| Trees | `makeTree(kind)`: trunks and limbs plus alpha-tested leaf cards (aspen, maple, willow, pine, snag) |
| Wildlife | Ducks, frog, dragonfly, heron, deer, fish |
| Procedural valley | Seeded noise, `genHeight()`, the water simulation (`computeWater`, `stepWater`), terrain and water meshes, terrain splat material |
| Instanced plants | Rocks, flowers, swaying grass clumps (`updateGrass`), lily pads, reeds |
| Game state … Update | Trees, dams, lodge, kits, goals, UI helpers, world setup, wildlife AI, the idle layer, main `update(dt)` loop |

## Interface

The UI is a small design system in the `<style>` block, in the spirit of Simon Willison's
*Tree Appreciator*: dark chrome over the scene (hairline translucent pills, `--frame`
on `--frame-ink`) and pale paper for sheets you read (`--paper` on `--ink`). Three
fonts from Google Fonts: **IM Fell English** for display (`--font-display`),
**Alegreya Sans** for UI text, **Alegreya Sans SC** for the letterspaced small-caps
labels. The title screen is type set straight over the valley behind a gradient scrim,
not a card; `#title .card` unsets the paper sheet that the journal uses. Every fallback
stack is system fonts, so a blocked CDN degrades rather than breaks.

Interface glyphs are line icons from an inline SVG sprite at the top of `<body>`
(`<symbol id="i-…">`, 24px grid, 1.6 stroke, `currentColor`), used through
`icon(name)` in the script or `<use href="#i-name">` in markup. The lodgebook (`#shop`) is the one paper sheet: a right-hand panel on wide
screens, a bottom sheet under 820px.

## Key mechanics

- **Grid:** the simulation grid is `N×N` (129) nodes at `SP` = 1.25 m. `H` is terrain height, `dam` is the dam level per node (0–4, `DAM_STEP` = 0.5 m) and `eff = H + dam`.
- **Water:** `computeWater()` runs a priority-flood from the south (outlet) edge over `eff`. It keeps only the pools connected to the stream (`river` nodes), then groups them into flat "components". Each component rises toward its target at `INFLOW / area`, clamped to `[RISE_MIN, RISE]`, so the stream delivers a fixed volume per second and a wide pond fills far slower than a narrow one. Call it after any dam change.
- **Flow:** `updateFlow()` writes an `aFlow` vec2 per node from the downhill gradient of the water surface, using wet neighbours only (a high bank is not a slope). The water shader advects its noise along that vector and whitens where the current is quick, so ponds sit glassy, the channel runs, and the step below a dam churns. Recomputed with the water mesh.
- **Dam wear:** every section carries `wear[i]` (0 sound, 1 washed out). `wearDams()` erodes it in proportion to `damHead[i]`, the drop across that section, faster in rain; `eff` subtracts the wear, so the crest sinks and the pond follows it down. Past `LEAK_AT` it spits water downstream; at 1 the section drops a level. A log patches it back to 0 through the build button, which reads `Patch dam` whenever `wear > PATCH_AT`. Kits mend the worst section at `KIT_MEND` each per second, so a family roughly offsets the wear on one section and a bigger family gains on it; they also pick dam spots to potter to (`kitDamSpot`). The HUD's Dam pill (`#damPill`, beside the goal in `#hud2`) reports the worst section as Sound / Wearing / Leaking / Breaking with a bar of how much of it is left, and hides itself when there is no dam. The water is only re-solved once the crests have drifted 5 cm (`wearSync` holds the baseline), not every frame.
- **Rendering:** the water mesh shares the node grid. Dry nodes next to water borrow a neighbour's level to form shorelines, but never across a dam.
- **Saves:** see the fork notes above — `localStorage['beaverValleyIdle1']`, counts not world state.
- **Trees:** saplings grow in on their own and when you buy them; nothing fells them here, so `treeFrac()` and the falling states are inherited but idle.
- **Looks:** realistic is the default; `#style-water` or `#style-cel` in the URL switches the post-process.

## Running it

Serve the folder with any static server (for example `python3 -m http.server`) and open `index.html`. Use the browser's phone emulation to check the bottom-sheet layout.

Space or Enter gnaws; everything else is the lodgebook.

## Notes

- Performance matters: it's meant for phones. The draw-call count is kept low by baking meshes per material, and there's adaptive pixel ratio in `loop()`. The grass count is `GRASS_MAX`.
- It's a good candidate for splitting into ES modules (with Vite) if it keeps growing.
