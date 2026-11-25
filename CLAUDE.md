# CLAUDE.md – Flight World (טיסה סביב העולם)

Browser flight simulator over real-world terrain. Focus: realistic scenery and flight *feel*, not complex aircraft systems.
Deployed on Vercel at https://flight-world.vercel.app. The user writes in Hebrew; reply in Hebrew. UI text is Hebrew (RTL).

## Repo layout
- `versions/v0NN.html` – each version is ONE self-contained HTML file (CSS + JS inline, Three.js r128 from cdnjs). No build step, no package.json.
- `vercel.json` – rewrites `/` to the active version. Edited by hand to switch versions.
- `VERSIONS.md` – version table (what each version added). **Update it whenever a version is created or activated.**
- `og-image.png` – link-preview image (referenced by absolute URL from the HTML meta tags).

## Versioning rules
- Never edit an older version file except to hot-fix the active one (and only after agreeing with the user).
- New work goes into a new file `versions/v0NN.html` copied from the latest. Every version stays reachable at `/versions/v0NN.html`, so a new version can be tested on production before `vercel.json` points to it.
- Activating = edit `vercel.json` destination + update `VERSIONS.md`, in its own commit (history convention: "הפעלת גירסה N").
- Commit messages are short Hebrew sentences.
- **Git workflow:** at the end of every step/fix, ASK the user whether to commit and push. Only after they approve: commit and push to `main` (this triggers a Vercel deploy; the main URL stays on the version in `vercel.json`). Never commit or push without that per-step approval.

## Running locally
Must be served over HTTP (tiles are fetched with CORS; `file://` fails – the HUD then shows "שגיאת טעינה (הרץ דרך שרת מקומי)").
E.g. `npx serve .` or `python -m http.server` from the repo root, then open `/versions/v0NN.html`.

## External data sources (all free, no keys)
| What | URL | Notes |
|---|---|---|
| Elevation | `s3.amazonaws.com/elevation-tiles-prod/terrarium/{z}/{x}/{y}.png` | Terrarium RGB encoding, z ≤ 13 |
| Imagery | `tiles.maps.eox.at/.../s2cloudless-2020_3857/...` | Sentinel-2 cloudless, note `{y}/{x}` order |
| Vector tiles | `tiles.openfreemap.org/planet` (TileJSON → tile URL) | OpenMapTiles schema: place, mountain_peak, building, transportation(_name), water, landcover |
| Geocoding | `nominatim.openstreetmap.org/search` | search box |
Attribution text lives in `#att` – keep it when adding sources.

## Code map (v008; code is minified-style, one-letter names)
Coordinates: Web-Mercator meters relative to a start origin (`setOrigin`), scaled by `K=cos(lat)`; world X=east, Z=south, Y=up (meters). `toLL(x,z)` → lat/lon. `TS(z)` = tile size in mercator meters.
- **Scene setup**: renderer (logarithmic depth), fog, sky sphere, sun + hemisphere light, procedural `plane` group, plane shadow `shG`.
- **Terrain tiles** (`tiles` Map, `get/pump/load/build/visit/updateTiles`): quadtree LOD from z4 to `MAXZ=13`, max 8 concurrent loads, DEM → 64/128-grid mesh with skirts, imagery as texture, `detail()` shader patch adds sharpening + near-field noise. Height × `verticalExaggeration=1.35` inside `build()`. Eviction when >420 tiles.
- **`ground(x,z,minZ)`**: terrain height lookup from the finest loaded tile – used everywhere (collision, labels, buildings, trees).
- **Flight model** `fly(dt)`: quaternion attitude, roll/pitch from keys or touch joystick, auto wing-leveling, altitude hold (`altH`), throttle `thr`, turbo ×8, ground clamp at 5 m AGL.
- **HUD**: `compass()`, `gauge()` (pitch ladder + roll), speed/alt/pos text every 15 frames.
- **Audio**: `audioInit/audioTick` – Web Audio oscillators + filtered noise.
- **Vector tiles** (`VT` Map, `vtGet/vtPump/vtLoad`, hand-written `mvt()` protobuf decoder, gzip via DecompressionStream), 5 concurrent, retry with backoff.
- **Labels** `addLabels/labels()`: HTML divs (pool `lpool`) projected each frame; places from z8/z11, peaks z11, street names z14; max 18 labels (12 roads), simple overlap rejection.
- **OSM 3D** (`// OSM-BEGIN … OSM-END`): `bldTile(e)` builds buildings, roads, water for a z14 tile into one group; heights via `g13()` (needs z13 terrain loaded – returns `null` and the whole tile is retried later). `osmTick()` builds at most one tile per 20 frames.
- **Trees / cars / water** (`// STAGE-B-BEGIN … STAGE-B-END`): water shader `wMat`; cars = one global `InstancedMesh` (cap 1800) moved along road polylines in `carsTick`; trees = per-tile InstancedMesh, 2 LODs, placed in OSM `wood` polygons or where the satellite pixel is green (`fmask/satF`).
- **Main loop** `frameLoop`: modes `menu → loading → fly`; `render()` handles chase/FPV camera, FOV by speed, shadow.
- **Start**: `PRE` preset locations, `start(lat,lon)`, Nominatim search.

## Known gotchas
- The terrain heights already include the ×1.35 exaggeration; anything comparing to real elevations must account for it (e.g. trees use `g/1.35`; peak `ele` labels do not).
- Many things are throttled by `frame % N` – behaviour depends on frame rate.
- Everything lives in global scope in one `<script>`; names are terse and reused (`V`, `F`, `q`, `R`…). Search carefully before renaming.
- There are no tests. Verify changes by running in a browser (built-in browser pane + local server) and checking the console.
