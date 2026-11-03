# swissEO Browser: notes for Claude Code

## Writing style

- **Never use em-dashes** (the long dash, U+2014), in any text: UI strings, layer descriptions, the Info tab, README, code
  comments, commit messages and replies. Use a colon, comma, semicolon, parentheses or a new sentence
  instead. En-dashes in number ranges (1991–2020, −1…1) are fine.
- UI and documentation texts are in English. Highlight stories have at most ten lines and credit their
  source.
- Keep the "Not for operational use" notice and the Claude Code note in the Info tab and the README.

## Repository

- The app is the single file `swisseo-browser.html` (no build, no backend). It belongs together with
  `README.md`, `CLAUDE.md`, `.github/workflows/pages.yml`, the issue forms in `.github/ISSUE_TEMPLATE/`, the
  help pages in `docs/` and the thumbnails it references in `public/previews/`. Everything else is the upstream Copernicus Browser, kept only as a reference:
  do not modify it.
- **Git: push only to the remote `origin`** (github.com/swisstopo/topo-swisseobrowser); local `main`
  tracks `origin/main`. Never commit for or push to `upstream` (eu-cdse/copernicus-browser); its push URL
  is disabled on purpose. `davidoesch` (github.com/davidoesch/swisseobrowser) is the former repository.
- The upstream `.gitignore` ignores `.github`, so workflow and issue-form changes need `git add -f`.
- Issues: the Info tab writes the issue text itself (`ISSUE_KINDS` / `issueBody()` in the page, passed as `title` and
  `body`; pre-filling issue form fields by id did not work), always starting with the permalink. Keep its sections in
  sync with the forms in `.github/ISSUE_TEMPLATE/` (each with a required `permalink` field) and keep
  `blank_issues_enabled: true`, otherwise GitHub sends the link to the template chooser and drops the text.
- Help and tutorial: `docs/index.html` (screenshots in `docs/img/`, JPEG, 1400 × 900 px views of the app), opened by
  the (i) button at the top right of the map. When a function is added or changed, update its section there too.
- A push to `main` that changes the page, the README or `docs/` deploys GitHub Pages
  (https://swisstopo.github.io/topo-swisseobrowser/).

## Conventions

- **Coordinates are LV95 (EPSG:2056)** wherever users see them: the permalink centre
  (`center=2703331.08,1133878.99`, as on map.geo.admin.ch), compare entries, the pixel inspector, the
  area of interest in the link and the import dialog defaults. WGS84 is used only internally (MapLibre
  LngLat, GeoJSON sources) and in GeoJSON downloads, which the GeoJSON standard requires.
- **Map**: MapLibre GL JS with the custom projection EPSG:2056 (`addProjection`, experimental: keep the version
  pinned). Its tile grid is a quad tree from the swisstopo origin at 2560 m/px ÷ 2^z (zoom 8 = 10 m/px), not the
  swisstopo WMTS levels; every raster tile is drawn by the page through the `swisseo://` protocol (`addRaster`).
  The `zoom` of links and compare entries stays the swisstopo level (`zoomToRes` / `resToZoom`), so old links keep
  working. Features of the view (`viewBox`, exports) assume the flat north-up map; in 3D (terrain from swissALTIRegio)
  they use the area around the centre, and the compare split is not available. The labels are swisstopo's vector tiles
  (Web Mercator), moved into the LV95 grid in MapLibre's workers (`swisseo-vt://`, `vtWorker`); their style's zoom
  stops are shifted by `VT_DZ`.
- **Reflectance**: ρ = DN × 0.0001 − 0.1 for the swissEO S2-SR bands; negative ρ is clamped to 0 before
  an index is computed. Sentinel Hub custom scripts that work on DN expect 10000 × ρ, i.e.
  `max(0, DN − 1000)` of the swissEO bands.
- **New layers**: add them to `LAYERS`, give them a description in `COP_META`, put them into a
  configuration in `CONFIGS`, and add them unchecked to the README verification checklist (the deploy
  warns about missing entries) and to the README layer lists.
- **Layer verification**: only the user ticks a layer as verified in the README checklist.
- Data that is not in STAC (e.g. NDVIdiff v200 BETA) gets a fixed item list in `SOURCES`.

## Testing

- Syntax-check the page script after every edit: it is an ES module with top-level `await`, so extract it into a
  `.mjs` file and run `node --check` on that.
- Serve the folder (`python -m http.server`) and open `swisseo-browser.html` in a browser. The map needs WebGL:
  headless Chromium (Playwright, `--use-angle=swiftshader`) renders it, clicks and drags included; the map object
  is module-scoped, so test through permalinks and the UI.
- Check data reads against an independent rasterio / GDAL read of the same COG where possible.
