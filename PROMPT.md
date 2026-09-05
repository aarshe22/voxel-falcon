# PROMPT.md — Clean-Room Re-Implementation Spec
## "FALCON OF DOOM" — a single-file 3D voxel world game

> You are rebuilding this game **from scratch, in one shot**, with no access to the original source. Build exactly what is specified. Where a constant is given, use it — the feel has been tuned. Where behavior is described loosely, use good judgment consistent with the rest.

---

## 1. The deliverable & hard constraints

- **One file: `index.html`.** It must run by **double-clicking it (`file://` URL)** — no web server, no build step, no bundler, no ES modules, no `fetch`, no assets (no images, fonts, audio files — sound is synthesized).
- Three.js via classic script tag: `<script src="https://cdn.jsdelivr.net/npm/three@0.150.0/build/three.min.js"></script>` (global `THREE`). If `window.THREE` is missing, replace body with a friendly "connect once, cached after" message.
- Everything else is inline `<script>` + inline CSS. Desktop Chrome/Firefox/Safari **and mobile touch browsers** must both work.
- Target: interactive frame rate on a mid laptop; the game self-tunes (see §12).

## 2. Art direction

Everything is **cubes and boxes**. No textures, no models, no smooth shading tricks: `MeshLambertMaterial` with **per-vertex colors** for terrain/buildings/creatures (sun + ambient + volcano point light), `MeshBasicMaterial` (some with `AdditiveBlending`) for water, fire, lightning, laser, clouds-glow, nuke fx. Terrain chunks are **merged BufferGeometries** (one draw call per chunk): every block is emitted as 6 quads with baked per-face light multipliers — top ×1.0, ±X ×0.86, ±Z ×0.72, bottom ×0.5. Fog (`THREE.Fog`) matches the sky color; fog far = adaptive view radius × chunk size. Sky/fog color lerps smoothly with weather (§9).

## 3. The world: 32767 × 32767, stored *nowhere*

`WS = 32767`, center `WC = 16383`. The map is **never stored** (a heightmap would be ~4 GB). Everything derives per-query from one integer `SEED` (random per session, `R` rerolls it):

```
hsh(x,y,s)   -> 32-bit integer hash of (x|0, y|0, s|0), uniform float 0..1
vn(x,y,s)    -> value noise: hash at 4 lattice corners, smoothstep blend
fbm(x,y,sp,oct,seed) -> fractal sum, octave i has spacing sp/2^i, weight 1/2^i, normalized
```

### 3.1 Base terrain (`rawH`)
```
d   = hypot(x-WC, z-WC) / LAND_R            // LAND_R = 11800
if d > 1.24: return -6                       // deep ocean margin
isl = smoothstep(clamp((1.10 - d) / 0.34))   // island falloff
e1  = fbm(x,z,150,5)                        // rolling land
e2  = fbm(x,z,47,4) * 0.68
m   = fbm(x,z,380,5); m = max(0,(m-0.455))/0.545; m = m*m*m^0.32   // ridges
h   = (e1*26 + e2*8)*(0.10+0.9*isl) + m*76*isl² - 4
      + oceanDips + vn(x/13,z/13)*0.42       // dips: (fbm(x,z,1100,2)-0.5)*6 when isl<0.35
```
Water at h < 0.52. Sea level ≈ 0.

### 3.2 Volcanoes (≤ 7 per world, all "active")
`placeVolcano()` at regen: primary = best of 900 hash-sampled sites (r = 2400 + h·6200 from center, base 14–38); up to 6 secondaries from 900 more samples (r 1300–10700, base 8–26), pairwise ≥ 1500 apart, radius 105–190 (primary 170). `baseH(x,z)` = rawH, then for each volcano within its radius:
```
cone = base + (r<150 ? 34 : 48) * (1-d/r)^1.15 - craterDip(d < 0.12r)
h = max(h, cone)
```
Each volcano gets **2 lava flows**: start offset ±(13–18) from the peak, walk 95 steps of 2.4 units with hash-jitter, steering toward the lowest of 8 sampled neighbors; register points into a spatial grid (bucket 48) with width `1.2+1.7(1-d/r)`; those cells render as bright emissive lava blocks (pulse the shared material color). Volcano plumes: smoke particles (§6, `ENT.smoke`) spawn over whichever volcano is nearest and within 2000 units, capped 160, rising 2–3.4 u/s with slight drift.

### 3.3 Rivers, lakes, roads, cities, craters — the feature layers applied on top of `baseH`
All are **point/feature grids** (`Map` of `"bx,by"` bucket → array; grids insert/query with 3×3 neighbor buckets) populated lazily as the camera explores (never recomputed twice — `seen` sets keyed by lattice cell).

- **Rivers**: source lattice cell 640; probe its 4 quadrant offsets (160/480); a probe qualifies at `baseH>28 && hsh(cell)<0.62` (fallback: the single highest probe > 10 in the ±5 ring — **every region gets at least one**). March ≤ 3000 steps of 2.1 units: 5-heading probe fan (±0.5 rad around heading), descend to lowest `baseH`, jitter heading by `hsh-0.5)*0.85`; stop when terrain 30 units ahead ≤ 0.6. If ≥ 70 points: commit channel points with width `1.7+3·min(1,n/600)`. In `hV`, any point within w forces `h ≤ -0.62`.
- **Lakes**: lattice 760; prob `hsh<0.5`; center jitter ±240; needs `1.5 < baseH < 13` and ≥ 500 units clear of every volcano; radius 55–130. Insert into a 128-bucket grid at center + 4 inner-corner offsets (dedupe-ish). `hV`: inside r → `min(h,-1.5)`; in a 34-wide skirt → lerp toward -1.5. **Each lake spawns a boat** (circular route at 0.62r).
- **Cities**: lattice `CC=560`, hash placement (`hsh(i,j)<0.9` keep), up to 18 jittered candidate probes (±250) needing `2.6 ≤ baseH ≤ 26`, flat test: 8 ring samples within **11 units** (they build on *uneven* ground), ≥ 620 from every volcano. Radius R = 7–14. City flatten in `hV`: hard `c.h` inside R, smooth 11-unit skirt. Streets: `((dx+R+off)&7)<2 || ((dz+R+off)&7)<2`; buildings at lattice positions where both `(lx,lz) ∈ {2,6}` and `hsh(x+dx·13, z+dz·7)<0.72`, floors `clamp(2+hsh·3+(1-d/R)·5, 2, 13)`. `off = hsh·8`.
- **Roads**: each city links lattice neighbors `(i+1,j) (i,j+1) (i+1,j+1)` **when the link endpoints stay within 3 cells of the region that requested them** (prevents infinite recursion); quadratic bezier with bend `(hsh-0.5)·L·0.4`, sampled every 3.5 into a 64-bucket grid; `matAt` makes cells within √6.9 road (bridge material if h < 0.9). One causeway links whichever cities are within 820 of a volcano to a hut point 200/170 off its flank.
- **Craters** (from nukes, keep last 8): bowl `h -= D·t²(0.55+0.45t)` inside r1, rim `h += rh·sin(π·u)` from r1→r2.
- **Scars** (from the laser, permanent, unbounded grid at bucket 24, radius 3.1) — see §7.

### 3.4 Materials (`matAt(x,z,h)` returns an index; check features last)
| idx | name | top color [r,g,b] | rule |
|---|---|---|---|
|0|water bed|[0.30,0.44,0.62]|h<0.52 (separate transparent water surface quads at y≈0)|
|1|sand|[0.80,0.72,0.54]|h<1.4|
|2/3|grass A/B|[0.35,0.55,0.27]/[0.22,0.33,0.18] hash split|default land|
|4|forest|[0.17,0.33,0.18]|`fbm(x,z,24,2)>0.56 && h<22 && slope<0.8`|
|5|rock|[0.45,0.44,0.42]|h>34|
|6|snow|[0.93,0.95,0.98]|h>41|
|7|road|[0.34,0.31,0.29]|road grid|
|8|farm|[0.66,0.58,0.32]|farm cells (4 per city, ellipse ~15×9)|
|9|lava|emissive orange pulse|lava grid (shared basic material, color `setRGB(1, 0.42+0.1·sin(t·3), 0.1+0.08·sin)`)|
|10|bridge|[0.55,0.40,0.26]|road grid & h<0.9 — deck box floats at 0.45..1.35|
|11|scorched|[0.20,0.20,0.22]|crater sc2 radius|
|12|charred|[0.08,0.075,0.08]|laser scar grid (overrides 11)|

## 4. Chunk mesh builder

`CH = 32` cells = 32×32 blocks of 1×1×1 units. On camera chunk-crossing: rebuild queue within `viewR` (start 7, adaptive), sorted by distance, built within a **time budget per frame** (18 ms during initial fill, ~7 ms steady). Per chunk:
- 33×33 column-height cache via `hV`; material per cell via `matAt` (slope from cache).
- Land: one top block `[hR-1, hR]`; **cliff bands** toward lower neighbors, 1-unit slices, depth ≤ 6, colors from a strata rule (surface color near top, dirt `[0.42,0.29,0.17]` / rock below) rendered as thin side slabs (0.12 thick, inset).
- Water: bed block + surface quad (shared transparent basic, gently bobbing group).
- **Trees** (cap 120/chunk): forest cells `hsh<0.30` (or grass clusters) → trunk box + 2 leaf boxes; skipped where blastKill/scarred.
- **Crop stalks**: farm cells `hsh(x·7,z·5)<0.32` → 3 tiny gold boxes.
- **City content** (drawn when the chunk is near a city): buildings per archetype (§4.1), houses, **rural outbuildings** (per farm, 2 hash-styled `farmfx`: red barn+silo / windmill+trough / greenhouses), and the **airport** (≤ 1 per city — cities with ≥ 24 builds get one at bearing `hsh(i,j)·2π`, distance r+42, axis-along dominant axis): 86×16 dark runway slab with yellow centerline, terminal with glass strip, control tower with glass cab + red beacon cap, hangar, and a parked plane (3 boxes).
- `purgeNear(x,z,r)` destroys built chunks in radius (craters/lakes/laser rebuild terrain) and forces a queue refresh.
- **Clouds**: separate chunked group, 16-unit cells; density `fbm(x+DX, z+DZ, 520, 3)` at centers, threshold `CLOTH` calibrated **per seed** by percentile of 160 samples (~78th pct), lowered/darkened by weather; altitude ~96 lowered by weather; drift offsets `DX,DZ` step by chunk size and re-mark dirty.

### 4.1 Building archetypes (per building, `bt = hsh(x·13, z·7)`)
- `bt<0.42` **homes**: 1–2 storey, w 1.3–1.9, 2 pitched roof tiers + chimney, wall palette (cream/adobe/slate), roof palette (terracotta/grey/red).
- `bt<0.78` **apartments**: 3–6 floors, brick/terracotta/concrete walls, front window bands + rear balcony bands per floor, parapet + rooftop tank.
- else **office towers**: 8–13 floors, two widths, glass palettes (navy/steel/dark), floor bands, **setback** upper third (inset 0.35), roof cap + mast when ≥ 8 floors.

## 5. Camera & movement (desktop)

`cam = {x, y, z, yaw, zoom, look}`; forward = `(sin yaw, cos yaw)`.
- `←/→` yaw ±1.35/s; `↑/↓` thrust along yaw; speed `(14 + y·1.4)`, ×3 while `Shift`.
- `q/a` altitude ±44/s; ground clamp `y ≥ hV+3.5` (12 in falcon mode); zoom `z/x`: `cam.zoom` 0.26–4.2 exponential, mapped to `fov = clamp(zoom·57.29°, 15°, 120°)`.
- Mouse drag on window: `yaw -= dx·0.0035; look += dy·0.002` (look clamped ±0.4/0.75, trims view pitch); first-person view pitch `clamp(max(0.06, y·0.0038) - look)`.
- `world→camera`: `camera.rotation.y = yaw+π` (camera looks -Z). A rig group carries directional sun + ambient so lighting follows the player. One PointLight sits at each volcano... at the *nearest* volcano position (primary) when in range.

## 6. Living things (counts per discovered city / caps; all `InstancedMesh`)

| thing | spawn | behavior | cap |
|---|---|---|---|
| 👤 people | 20/city + 3 road walkers per road | random wander inside city (±32), re-pick every 3–8 s; avoid water | 480 |
| 🐕 dogs | 8/city | trail nearest person (offset ±8/6) | 160 |
| 🐄 cows | 10/farm (4 farms) + 44 wild on grass rings | graze wander, avoid water/lava/high rock | 460 |
| 🚗 cars | 6/road lane | route ping-pong, **driving side** `1.8·dir` lateral, U-turn at ends, bridge deck height | 60×2 (red/blue) |
| 🚚 trucks | 1/road | same, slower (4.4), taller ride | 36 |
| ✈️ planes | 6 global | great-circle-ish rings radius 820–1720 around the *camera*, altitude 132–218, re-headed each frame, always overhead | 8 |
| ⛵ boats | 1/river, 1/lake, 1 ocean belt circle (R+320), 1 volcano ferry (R+380 if all-water) | ping-pong routes, bob, hide if grounded | 28 |
| 🎈 balloons | 8 | state machine: rise → drift to random target (≤ 2000 from cam) → descend → land → wait → relaunch; envelope = 3 stacked boxes + basket | 8 meshes |
| 🕊️ birds | 18 flocks × 7 | circling flock centers, member sin-flap `sy=0.45+0.6·|sin|` | 140 |
| 🌫️ volcano smoke | ≤ 160 particles | rise 2–3.4, grow, 16 s life, over nearest volcano | 280 pool |
| 🔥 fire sites / 🏚 ruin puffs | nukes (§8), laser (§7) | fire: flicker scale `(0.55+0.65|sin(t·11)|)·(0.35+0.65·remaining)`, additive 2-box geo, life ~20–140 s; ruin smoke: rising, growing, wind-drifted grey boxes, 45–520 s | 420 / 520 |

Far-entity pruning at ~2.8k; flocks respawn one per 24 s (max 18) so the hunt continues.

## 7. FALCON MODE (`F`) + BURN LASER (`B`, hold)

**Peregrine**: a `Group` of 4 meshes — body+belly+tail+neck boxes (chestnut `[0.44,0.30,0.18]`, pale belly), head (pale skull, **black hood cap**, black malar side-stripes, yellow beak + dark tip, 2 tiny eye boxes) on a pivot at (0, 0.30, 1.18), and 2 wing meshes pivoted at the shoulders (2 boxes: panel + dark tip). Group scale 1.35; forward is +Z; chase camera: `back = 5.4+min(4.6, zoom·3.6)`, `up = 2.5+min(2.6, zoom·1.7)`, view pitch `clamp(0.30 - look·0.9, -0.15, 0.9)`, `fov=(zoom+0.14)·57.29`. Ground floor 12. **Auto-cruise** 17 u/s along yaw when no thrust key. Controls unchanged otherwise (§5).
- Wings: `flap = sin(t·spd)·amp + sin(t·spd·2.1)·amp·0.22`; `spd` 13 climbing (q) / 10.5 thrusting / 6.8 cruise / 4.5 descending; `amp` 0.85 / 0.62 / 0.46. `wingR.rot.z = +flap`, `wingL = -flap`.
- Head: smoothed `clamp(yawRate·1.7, ±0.62)` into turns; when straight, idle scan `sin(t·0.9)·0.18`.
- Body: `rot.z = roll` smoothed toward `clamp(yawRate·0.55, ±0.5)`; `rot.x = pitch` smoothed toward climb −0.30 / dive +0.26 / thrust +0.10 / glide +0.04 (positive = nose down).

**Burn laser** (only while falcon mode + `b` held): origin (falcon + 2.6 forward, +0.7 up), direction `(±0.7071 forward, −0.7071 down)`; march geometric `t=4→700 (×1.09)` for first `hV(x,z) ≥ y`. Beam = **two meshes, each 12 verts** (two crossed quads along the beam: horizontal side = camera yaw vector, vertical side = axis×that), additive basic, widths 0.42 (white-blue) & 1.7 (faint blue). While held, at each ground hit ≥ 1.6 units from the last: insert a **scar point (r 3.1)** into the scar grid + spawn a burn site (fire ~20–45 s + ruin smoke ~70–150 s); kill life within radius 6 (counted §10) and cars/trucks (uncounted "vehicles"); every 0.3 s `purgeNear(hit, 52)` so the burnt terrain remeshes **permanently**. Synth hum: two oscillators (68 Hz saw + 143 Hz square) through gain 0.05.

## 8. NUKES (`N`)

`armOrBoost()`: if a device is already counting down, **double its yield** (15→30→60→… cap 3840 kt); else arm at the camera's ground point (`kt=15`). 10 s blinking countdown: HUD banner `☢ {kt}KT DEVICE ARMED  T-{rem}s · x,z` flashing red/cream, blinking marker box.
On expiry, `ps = min(9, (kt/15)^⅓)`:
1. Crater `{r1:(40+rnd9)·ps, r2:(83+rnd10)·ps, D:(16+rnd4)·ps, rh:(6+rnd2)·ps, kill1=55ps, kill2=170ps, scorch=60ps}` pushed (max 8 kept); `purgeNear(x, z, min(900, 175·ps))`.
2. **Kill everything** in kill1 (all) / kill2 (55%) incl. bird flocks → the LIFE tally (§10).
3. **42 rim burn sites** scattered on the r1..r2 annulus: long fires (60–140 s) + very long ruin smoke (300–520 s) + scar points r≈5 (the charred rim).
4. Full-screen white `#flash` div opacity 1→0 (fast then slow); fireball sphere (additive) `r=(5+g²·143+g·42)·min(2.6,ps)` over 0.75 s, hot-white→orange, gone by 2.4 s; PointLight `9·ps` decaying over 1.4 s; shockwave ring (`RingGeometry`, double-sided basic) radius `(10+p·175)·(1+(ps-1)·0.8)` gone at 2.2 s.
5. **Mushroom cloud** from 3 pools — 66 stem + 46 cap + 24 crown unit spheres:
   - stem `i`: born `0.35+0.055i`; `y=8+cr·56·(0.72+0.5s₂)` capped at capY; helix radius `(9+16·max(0,1-fy·1.7))·(0.75+0.6s₁)+sin(y·0.09+s₁·22)·3.2`; scale `2.4+fy·4.2+min(3.5, cr·1.9)` capped 8.5; puffs vanish at the cap.
   - cap `i`: born `2.55+0.048i`; radius `min(CAPR·(0.5+0.5s₁), 9+cr·30)·(0.78+0.32s₂)` with roll wobble; y = capY + dome + bob; scale `3.4+cr·3.8` capped 16.
   - crown: born from 4.3 s, rise from the dome at `4.6·(0.5+s₂)`, scale capped 11.
   - `capY = (168+18s₁)·min(1.8,ps)`, `capR = (40+26s₂)·min(2.2,ps)`.
   - Colors: shared `heatG = max(0, 1-(p-1.1)/4.5)` ramp from incandescent to ash `(0.47,0.46,0.45)`; per-puff `dark = 0.70+0.30·y/capY`; opacity ramps in ×2.4/s; after p>112 fade to 0 by 130; then full cleanup (meshes/materials disposed). Cloud drifts with the wind all the while (`dr = max(0,p-8)·0.42` applied as expanding offset + lateral wind).
   - Simulation timescale hook `window.__NUKE_TSCALE` multiplies `p` (tests run 12×).

## 9. WEATHER (toolbar or keys `1–6`)

`WX ∈ clear,rain,snow,storm,light,tornado`. Table `[skyHex, sunIntensity, ambientIntensity]`:
```
clear 0x9cc4e2 1.1 0.8 · rain 0x8d95a3 0.72 0.85 · snow 0xc7ced8 0.85 0.95
storm 0x474f5c 0.40 0.72 · light 0x5d6877 0.50 0.78 · tornado 0x6a7380 0.52 0.85
```
All four (sky, fog copy, sun, ambient) lerp at `dt·1.4`. Cloud coverage threshold `CLOTH` shifts by −0.135/−0.105/−0.06/−0.05/−0.04 (storm/light/rain/tornado/snow) and altitude by −26/−20/−10/−8/−6; every setWX re-dirties cloud chunks.
- **Rain**: 1900/2200/900 (rain/storm/tornado) instanced streak boxes (0.09×2.3), fall 165–237 u/s, recycled in a ±200 box around the camera onto per-column ground; **snow**: 1600, 8 u/s fall, `sin` sway, rotation.
- **Lightning** (storm & "light"): every `storm: 2.6+rnd6` / `light: 1.0+rnd3` s, strike 260–900 from camera: 7-segment jagged bolt of axis-aligned thin glowing boxes (drop pillar + horizontal jog per step), additive basic, life 0.22 s with `age%0.055<0.028 ? 0.55 : 0.3` flicker; PointLight 26→0 over 0.35 s; screen flash 0.13.
- **Thunder (synthesized)**: 2.4 s buffer, leaky-integrated noise ×(1−t)¹·⁴, lowpass `clamp(700−dist·0.4, 80, 700)` Hz, gain envelope `0.0001→clamp(600/(180+dist),0.15,1)@+0.08s→0.0001@+dur`, **scheduled at `ctx.currentTime + dist/343`** (boom after flash).
- **Tornado**: 380 debris instances — funnel `hgt=(i·1.37+t·54)%76`, radius `(6+hgt·(0.5+0.4·hsh))·2.2`, angular `t·(7.6−hgt·0.045)+i·2.399963`, scale `0.7+2.2·hsh(+1.6 at base)`; wanders 27 u/s with smoothed random heading; **kills life within 22 every 0.12 s**; loops to ±430 of the camera when 1500 away; 1.5 s brown-noise loop at 90 Hz lowpass, gain 0.3.
- AudioContext created lazily on first gesture (button/key), `resume()` when suspended, everything try/caught.

## 10. FALCON OF DOOM — the game loop & LIFE HUD

The mission: **decimate all life**. Track `KILLS={people,dogs,cows,birds,vehicles}`; `killLife(x,z,r)` (laser/tornado) and `killBlast(crater)` (nukes: full inside kill1, 55 % inside kill2; flocks count their 7 members). Second HUD strip (top-left, below the main one):
```
LIFE  👤n 🐕n 🐄n 🕊n  =  <b>total</b>   ☠ DOOMED  <b>n</b> (+ n vehicles)
```
Green normally, class `.doom` (red glow) under 300, and at total 0 with DOOMED > 0 append `— EXTINCTION ACHIEVED: THE FALCON OF DOOM PREVAILS ☠`. Regeneration (`R`) resets kills. Title tag, HUD prefix,·  help text and README all branded "FALCON OF DOOM".

## 11. UI

**Main HUD** (top-left, throttled 0.5 s): `FALCON OF DOOM · x… z… alt … · [BIOME] · @CityName · NEAREST VOLCANO ▲ n.nnkm`; second line = LIFE strip; third area (top-right, under radar): `fps · chunks · r<viewR>`.

**Radar** top-right: square, size `min(192, max(96, innerHeight−150, innerWidth−40))` (recomputed on resize; fps line repositioned below it; never off-screen). Samples terrain by `matAt` on a 24×24 grid over a ±1200 span (CSS color array), gold city dots, red volcano triangles, white view cone (`atan` of zoom, 72 % radius), yellow camera square.

**Toolbar** — a *single* full-width bottom strip, `overflow-x: auto`, `flex-wrap: nowrap`, hidden scrollbars, `touch-action: pan-x`, **no group labels**, every command is one emoji `b` element with a native `title` tooltip (order and tooltips preserved):
`👤 🐕 🐄 🚗 ⛵ 🎈 🕊️ 🏙️ 🏔️ 🌋` (jump-to-nearest; cities/mountains found by spiral search if undiscovered) · `☀ 🌧 ❄️ ⛈ 🌩 🌪` (weather; active one tinted `#ffe15a`) · `⏫ ⬇️-hold a… ⏫/⏬/🐇` hold-buttons (pointerdown adds key, up/leave/cancel removes; keys `q`,`a`,`Shift`) · `🔍+ 🔍−` zoom (×0.8 / ×1.25 click) · `🦅` falcon · `🔆` laser **hold** (onpointerdown/leave add/remove `b`) · `☢` nuke (audio unlock + armOrBoost) · `🌱` new seed · `🗺️` radar map show/hide (auto-hidden when `IS_TOUCH` or width<560; hiding it also re-pins the fps chip to top:12 and skips radar sampling — saves the 576 `matAt` probes per second on small screens) · `☁` toggle clouds.
Click suppression during drag: `pointerdown` on empty bar area captures `scrollLeft−clientX`; after >6 px movement set `__dragged` (click handlers return early; cleared on pointerup+0 ms); double-click on empty bar re-centers scroll. Tooltips include the hotkey, e.g. `"Arm a nuclear device — press again while counting to double the yield (key: n)"`.

**No·  help panel** — there must be ZERO persistent text overlays besides the HUD/life strip/fps line and the toolbar; every command is self-documenting via its tooltip (this matters: a text panel filled an iPhone screen and was deleted).

## 12. Mobile / touch adaptation

`IS_TOUCH = 'ontouchstart' in window || navigator.maxTouchPoints>0` → `body.touch`.
- Canvas `touch-action:none`; **drag the world to look**: canvas pointerdown/move with `pointerType!=='mouse'` reuses the look math with coefficients ×0.0045 (yaw) / 0.002 (pitch); any pointerup releases.
- **Thumb-pad** `#tpad`: CSS grid 3×3 × 50 px glass buttons, fixed right 10 px above the strip: `⤴`(q) `🐇`(Shift) / `◀` `⏩`(thrust) `▶` / `⤓`(a) `🔥`(laser hold) — each onpointerdown adds the key (+ lit state), onpointerup/cancel/leave removes; `preventDefault` so iOS doesn't scroll/zoom.
- Toolbar strip: native pan-x swipe scrolls; buttons get larger touch padding; tapping (not dragging) triggers.
- No keyboard assumed: every key-driven action has a strip button.

**Performance self-tuning**: EMA fps; <27 sustained 2.5 s → `viewR--` (min 4), >52 → grow (max 7); radar/HUD at 2 Hz; geometry merged per chunk; instances for everything mobile.

## 13. QA hooks (bottom of the same file)

`window.VOXWORLD = { get dbg(){…}, jump(kind), wx(name), hAt(x,z), arm(), touch, camZoom(), zoom(f), life(), kills(), firstLife(), killAt(x,z,r), falconPos(), camPos(), wingZ(), headY() }` — `dbg` exposes `SEED, cam, VOL, vols, chunks, cities, roads, rivers, lakes, scars, craters, fx, clouds, people/dogs/cows/cars/trucks/planes/boats/balloons, airports, farms, smoke, weather, rainCount/snowCount, tornOn, bolts, fire/ruin counts (IM), laserOn, nu (active yield), falconOn`. Headless verification runs the page JS in a `vm` context with stubbed THREE (Box/Sphere/Ring geometries are inert, materials store colors, renderer is a no-op) and a tiny DOM; it must assert: streaming ≥ 150 chunks, ≤ 1 airport per city, farms ≥ 12, souls in the hundreds, crater geometry (center drops > 14 units at 60 kt), yield doubling `15→30→60`, laser scar counts rising + fires + ruins, tornado spawns debris and kills, weather clears to zero particles, all jumps land within sight, and **every geometry vertex / matrix element is finite** (NaN tripwires).

## 14. Acceptance checklist

1. Double-click → runs from `file://`, console clean, internet once for three.js.
2. World looks like a *planet*: coasts, plains→forest→rock→snow mountains, 7 smoking volcanoes with lava tongues, rivers reaching the sea, lakes with boats, dense-with-routes cities of 3 building archetypes (+≥1 airport somewhere), 4-farm countryside with barns/windmills/greenhouses/crop rows.
3. All fauna alive and behaving; LIFE strip falls when you nuke (blast kills), laser (trail of permanent char + fire + ruin smoke), or tornado; DOOMED rises; extinction banner at zero.
4. Falcon mode: chase cam, wing flap input-dependent, head turns into turns, banking; laser carve while held, exact stop on release.
5. Nukes: countdown doubles on re-press, HUD shows kt; explosion scales with yield (crater, light, cloud height); rim burns for minutes; craters persist while revisiting.
6. Weather: all six modes visibly differ; thunder *arrives after* the flash; tornado wanders and shreds.
7. UI: single scrollable icon strip (mouse-drag scroll + native touch swipe, tooltips, tap works), thumb-pad only on touch, radar never clipped.
8. `R` gives a genuinely new planet (new volcano sites, cities, rivers) and zeroes the tally.
