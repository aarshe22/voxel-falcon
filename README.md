# 🦅 Voxel Falcon

**A 32767 × 32767 procedural voxel world you can fly, storm, irradiate and laser — rendered entirely in Three.js cubes, from a single HTML file.**

Open `index.html` by double-clicking it. No build, no npm, no web server. *(Three.js itself loads from a CDN on first launch — after one online launch it stays in the browser cache.)*

---

## The world

The planet is **32,767 × 32,767** — 1.07 billion cells, which is far too large to store (≈4.3 GB of height data). Nothing is stored. Every chunk recomputes its own terrain, cities, rivers, lakes, roads, forests and clouds on demand, deterministically, from one integer seed, as you move. Fly 10 km in any direction and come back: everything is exactly where you left it, because it was re-derived, not remembered.

- **Island continent** ringed by open ocean: sandy coasts, rolling plains, dense forests, and **big mountain ranges** that step up into rock and permanent snowcaps.
- **Seven active volcanoes** per world — the primary peak plus secondaries scattered across the land. Cones, craters, glowing lava flows down their flanks, and plumes of rising smoke you can see from the horizon.
- **Rivers** carved by simulated steepest descent from the highlands to the sea, widening as they go.
- **Lakes** dotted across the lowlands — each one automatically gets a boat puttering around it.
- **Cities** (2× density) on a deterministic lattice, *including on uneven ground* — the generator relaxes over hills and slopes rather than demanding flatness, so towns climb valleysides. Street grids, roads connecting neighbours in curved lanes, and a causeway to a hut beside the volcano.
- **Architecture**: three built archetypes scattered by hash — single/two-storey homes with pitched roofs, brick apartment blocks with balcony rows and rooftop tanks, and glass office towers with stepped setbacks and masts. Larger cities get **one airport** (never more): runway with centre-line, terminal, control tower with red beacon, hangar and a parked jet.
- **Countryside**: 4 farms per city with crop rows (visible stalks), red barns, silos, windmills, water troughs and greenhouses, plus grazing herds.
- **Persistent ground editing**: craters, rims and burn scars are live features layered into the height and material functions — after a detonation or a laser run, fly away, come back: the damage is still there (within the last few crater scars kept).

## Life

The world is populated, and the population scales with what you discover:

| | |
|---|---|
| 👤 **People** | wander cities, stroll sidewalks on road shoulders |
| 🐕 **Dogs** | trail after the nearest person |
| 🐄 **Cows** | graze farmyards and wild herds, avoid water and lava |
| 🚗 **Cars** | six per road lane, kept to the driving side, U-turn at ends |
| 🚚 **Trucks** | longer box-loads cruising the highways |
| ✈️ **Planes** | a small airlane circling the region overhead |
| 🎈 **Hot-air balloons** | eight of them: rise, drift between targets, land, rest, relaunch |
| 🕊️ **Birds** | 18 flocks wheeling on thermals at head height |
| ⛵ **Boats** | river ferries, lake boats, an ocean belt-hopper, a volcano ferry |
| 🌋 **Smoke** | dense volcanic plumes from every nearby cone |

## Weather — buttons or keys `1`–`6`

`☀` clear · `🌧` rain · `❄️` snow · `⛈` thunderstorm · `🌩` lightning · `🌪` tornado

The sky, fog and sun dim/brighten smoothly between modes; cloud decks thicken and lower; up to 2,200 instanced rain streaks or 1,600 drifting snowflakes recycle around you. Storms throw jagged additive lightning bolts with a synchronized point-light flash, screen flash, and **synthesized thunder** — noise through a distance-attenuated low-pass, *delayed by distance ÷ 343 m/s so the boom arrives after the flash, like real life.* The tornado is a 400-piece debris funnel that wanders the ground and follows your region, with a looping low rumble.

## 🔆 Become the falcon — `F`

Third-person flight on the wings of a voxel peregrine: chestnut body, pale belly, black-hooded head with bandit malar stripes and a yellow hooked beak. **Wings flap** (deep fast strokes when climbing, lazy cruise glides otherwise), **head turns into every turn** (and scans the ground when flying straight), body banks and pitches with your inputs, chase camera rides behind. All flight/zoom/mouse controls continue to work; terrain never clips you; let go of the throttle and you auto-cruise.

## 🔆 Burn-laser — hold `B` (in falcon mode)

A 45° forward-sloping high-power beam from the falcon with crossed additive core/glow quads, aimed by flying. While held it marches to the ground and **carves a permanent charred scar** wherever it touches: terrain material turns to burnt glass, chunks remesh, trees and buildings in the path are destroyed (never regrow), anything that walks is killed, and the trail is left behind as **flicker-lit fire sites and rising columns of smoking ruins** that burn for a minute and a half. A synthesized laser hum runs while firing. Works from the toolbar button too (hold it).

## ☢ Nukes — `N`, press again to double the yield

Press `N`: a **15 kt device** drops at your position with a **10-second blinking countdown** (HUD banner + blinking bomb marker). **Press `N` again during the countdown: yield doubles.** And again: doubles again (up to 3.84 Mt — the banner counts `15KT → 30KT → 60KT → 120KT…`).

On expiry: full-screen flash, expanding shockwave ring, blinding decaying light, an incandescent fireball that fades into a rising **helical stem → toroidal billowing cap** mushroom cloud (~136 puffs) glowing white-hot through orange to ash-grey, drifting on the wind and dissipating over minutes. **Everything scales with the cube root of the chosen yield** — bigger craters, bigger kill zones, bigger clouds. The explosion leaves **permanent terrain**: a deep bowl with a raised, scorched rim ringed with **burning fires and long-smoking ruins**, buildings inside the blast flattened or half-razed, vegetation gone.

## Toolbar (full-width, bottom)

One strip, every function, with hover tooltips:

- **FIND** — `👤 🐕 🐄 🚗 ⛵ 🎈 🕊️ 🏙️ 🏔️ 🌋` one-click fly-to the **nearest** of anything (including undiscovered cities and peaks via spiral search)
- **WEATHER** — `☀ 🌧 ❄️ ⛈ 🌩 🌪`
- **WORLD** — `☢` nuke · `🌱` regenerate the planet · `☁` cloud layer · `🔆` laser (hold) · `❔` help

## Controls

| Key | Action |
|---|---|
| `← ↑ ↓ →` | turn / thrust (falcon: flight stick) |
| `z` / `x` | zoom (FOV) |
| `q` / `a` | climb / descend |
| `Shift` | 3× boost |
| mouse drag | look trim |
| `F` | falcon mode on/off |
| `B` (hold) | burn-laser (falcon mode) |
| `N` | arm nuke; **re-press while counting: double yield** |
| `1`–`6` | ☀ 🌧 ❄️ ⛈ 🌩 🌪 |

**📱 Mobile/touch:** drag the world to look, a glass thumb-pad (thrust/turn/climb/dive/boost/laser) appears bottom-right, the command strip is swipe-scrollable and tap-to-use — no keyboard needed.
| `C` | toggle clouds · `H` help · `R` new world |

## HUD & radar

Top-left: coordinates, altitude, current biome, nearest city tag, nearest volcano bearing + range. Top-right: a **192 px radar** (auto-shrinks on short viewports, never spills off-screen) with terrain materials, cities, every volcano, your position and your field-of-view cone.

## Under the hood

- One HTML file, ~900 KB; Three.js r150 from CDN; WebAudio for thunder/rumble/laser (synthesized, zero assets); works from `file://`
- **Seeded everything**: terrain (fBm + ridge fBm + island falloff + per-volcano cones + river/lake/city/scar feature grids), cities, roads, river sources, lakes, clouds (percentile-calibrated per seed), buildings and their windows
- **Chunk streaming** with per-frame millisecond budgets, frustum culling, incremental feature discovery, adaptive view radius (4–8 rings by FPS)
- Creatures, traffic, weather particles, tornado debris, fires and ruins are **InstancedMesh** batches; terrain chunks are merged vertex-coloured BufferGeometries with baked face lighting
- QA hook: open the console and inspect `VOXWORLD.dbg` (chunk counts, world stats, weather, nukes, lasers), plus `VOXWORLD.jump('city')`, `.wx('storm')`, `.hAt(x,z)`, `.arm()`, `.falconPos()`
- Verified by a headless harness (real page JS under a stubbed THREE + DOM): 50+ assertions across random seeds — streaming, controls, crater geometry, yield doubling, laser scarring, weather lifecycles, legend jumps

## Run it

```
git clone git@github.com:aarshe22/voxel-falcon.git
open voxel-falcon/index.html        # macOS: just double-click
```

Tip: first open needs internet (three.js CDN, then cached). If frames dip while exploring fast, the world auto-reduces its view radius — no settings required.
