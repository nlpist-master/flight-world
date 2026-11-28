# CLAUDE.md – Flight World (טיסה סביב העולם)

Browser flight simulator over real-world terrain. Focus: realistic scenery and flight *feel*, not complex aircraft systems.
Deployed on Vercel at https://flight-world.vercel.app. The user writes in Hebrew; reply in Hebrew. UI text is Hebrew (RTL).

## Repo layout
- `versions/v0NN.html` – each version is ONE self-contained HTML file (CSS + JS inline, Three.js r128 from cdnjs). No build step, no package.json.
- `vercel.json` – rewrites `/` to the active version. Edited by hand to switch versions.
- `VERSIONS.md` – version table (what each version added). **Update it whenever a version is created or activated.** `README.md` (Hebrew, public) has a short version table + "active version" line – keep it in sync too.
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

## Current status & roadmap (updated 2026-11-28, v011 active)
- **Active (main URL): v011** (activated 2026-11-28). v010 (2026-11-25) added: road labels at `g+RDH` (2.5 m) + terrain occlusion `occ()`; `loadTick()` keeps the loading overlay until z13 under the plane, no terrain holes/pending refinement within 10 km (`vPend`), imagery on every shown tile (`vImg`, `e.img`), all z14 building tiles within osmTick's R (~3 km) built, and z8/z11 label tiles – 60 s safety cap (user: "flight must always start complete and impressive"); DEM load failures are retried with backoff (`e.fail`).
- **v011 (done)**: mountain material in `detail()` – `MTN` (color: snow from imagery `sI` + above snow line `sP`, uniform `uSL`=`snowLine(lat)`×1.35; procedural rock where normal.y<~.76 on the dominant vertical plane; no cyan – saturation boost reduced on snow) and `MTNN` (rock bump normals <3.5 km); sun moved to SE (`sun.position(.6,.42,.55)`, 1.6) + hemisphere .45; plane shadow follows sun; `n2()` noise now samples a 256² mipmapped noise texture `NT` (`uNT`) instead of hashing.
- **GPU measurement (2026-11-26, Aletsch view, 882×477)**: v010 23 ms, v011 30 ms per frame. Without the per-fragment log-depth write the same frame costs ~15 ms → dropping `logarithmicDepthBuffer` is the biggest perf lever (risk: road/water z-fighting; test carefully).
- **Next: v012** – cast mountain shadows (computed per tile at build time), sharper ridges (normals from full DEM / denser near grid), distant haze; user decides order.
- **Background (v011 plan)**: satellite imagery is top-down Sentinel-2 (~10 m; EOX z14 is missing/1×1 in places, z15–16 are just upsampled), so cliffs are smeared. Plan: in the terrain shader blend a procedural world-space (triplanar) rock/snow detail by slope (normal.y) and brightness; tone down the saturation boost that turns snow cyan. Alternative: sharper imagery source (Esri/MapTiler – needs key/terms check).
- **Stage B (structural, discuss first)**: split into modules + Web Worker for MVT decode / bldTile / trees (removes 50–130 ms stutters); far-building LOD ring (simple boxes of tall buildings to 8–10 km, also above 3000 m AGL) to remove the "buildings end" line; consider dropping `logarithmicDepthBuffer` (disables early-Z).

## Code map (v008, plus v009/v010 changes noted; code is minified-style, one-letter names)
- v009 added: `demRaw/demFor` (DEM with ancestor fallback), `imgFor` (imagery with ancestor fallback), button blur on click.
- v010 added: `get(z,x,y,pri)` + `tprio()` priority queue (12 concurrent, DEM+imagery in parallel); `bU` uniforms + `aT` attribute → building dissolve-in (time) and distance fade (`uR`) in `oMat.onBeforeCompile`; `osmTick` every 8 frames, nearest/ahead tile first, R=2000+min(speed*12,2600); trees get a per-tile bounding sphere (frustum culling); `labels()` candidate list `LC` rebuilt every 10 frames; `inT()` drops vector-tile buffer duplicates in `addLabels`.
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
- Network reality (measured 2026-11-25): an uncached AWS Terrarium request takes 1.5–4 s (no CDN); EOX ~0.7–1 s; OpenFreeMap ~0.3 s. Every request before takeoff is expensive – v010 requests only the 8-tile chain under the plane at top priority (`get(z,x,y,1)`), then a distance/heading priority queue (`tprio`).
- Measured frame cost in Manhattan (v009): terrain detail shader was the biggest GPU cost and scales with pixel count; trees were drawn even off-screen; labels() scanned ~8000 labels per frame (×9 duplicates from vector-tile buffers – fixed in v010 by keeping only in-tile points). Building *construction* (bldTile, 50–130 ms per tile on the main thread) causes the stutters – real fix = Web Worker (planned Stage B).
- Both tile providers have permanent "holes": AWS Terrarium and EOX sometimes return a 1×1 placeholder image. v009 handles this with `demFor()` / `imgFor()` (fall back to an ancestor tile). Any new tile source needs the same guard.

## Verification (required after EVERY fix – user's explicit request)
After each fix, test it in the browser and show it works (data check + screenshot, ideally before/after vs. the previous version at the same spot) before asking to commit.
- Local server: `preview_start` with name `static` (`.claude/launch.json`, python http.server on :8765), open `/versions/v0NN.html`.
- The browser pane is often hidden → `requestAnimationFrame` is throttled and the game doesn't advance. Drive it manually from `javascript_tool` (top-level `const`/`function`s are reachable):
  `window.sim=async(n,ms=150)=>{for(let i=0;i<n;i++){updateTiles();if(mode=='loading'){const g=ground(pos.x,pos.z,11);if(g!=null){pos.y=g+500;AGL=500;mode='fly';$('ov').style.display='none'}}else{const g=ground(pos.x,pos.z,0);AGL=pos.y-(g??0);compass();gauge();labels();osmTick();carsTick(.016)}render(.016);await new Promise(r=>setTimeout(r,ms))}}`
  then `start(lat,lon); await sim(60); ...; await sim(250,60)` and take a screenshot. Inspect `tiles`, `VT`, `stat`, `osmTxt()` for data checks; check console errors.
- When the pane is displayed the game runs by itself – just `start(...)` and wait (javascript_tool calls time out at 45 s; split long waits).
- Cold-load test (no browser cache): before `start()`, patch `HTMLImageElement.prototype.src` and `fetch` to append a unique `?cb=<token>` to tile URLs. Reference: Manhattan (40.75,-73.985) v009 = 80 s to takeoff, v010 < 5 s.
- GPU cost per layer: pause (`setPaused(true)`), wrap `R.render` with `EXT_disjoint_timer_query_webgl2`, toggle `osmGrp`/`carM`/tree groups (`e.trg`)/building meshes (`material===oMat`) and compare medians.
- Don't hammer the tile servers: many cache-busted runs made them slow (2–14 s per request) for a while.
