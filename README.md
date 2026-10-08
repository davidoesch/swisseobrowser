# swissEO Browser `BETA`

A minimal, browser-only viewer for swisstopo's **swissEO** satellite products, modelled on the
[Copernicus Browser](https://browser.dataspace.copernicus.eu/) workflow.

**Live version: <https://davidoesch.github.io/swisseobrowser/>**

The whole application is a single file: [`swisseo-browser.html`](swisseo-browser.html). There is no
build step, no backend and no login. The page reads the swissEO Cloud Optimized GeoTIFFs (COGs) and
the swisstopo WMS/WMTS services directly in the browser.

> **About the rest of this repository:** this repo started as a clone of
> [eu-cdse/copernicus-browser](https://github.com/eu-cdse/copernicus-browser). All files other than
> `swisseo-browser.html` (`src/`, `public/`, `package.json`, …) are the upstream Copernicus Browser
> sources. They are kept **only as a reference** for its user-interface behaviour and are not built,
> used or modified by the swissEO Browser. Changes are pushed to this repository only, never upstream.
> For the Copernicus Browser itself, see the [upstream README](https://github.com/eu-cdse/copernicus-browser#readme).

## Data

| Product | Source | Used for |
| --- | --- | --- |
| [swissEO S2-SR v200](https://www.swisstopo.admin.ch/en/satelliteimage-swisseo-s2-sr) | COGs on `data.geo.admin.ch`, catalogue via the STAC API `data.geo.admin.ch/api/stac/v0.9` (collection `ch.swisstopo.swisseo_s2-sr_v200`) | Sentinel-2 L2A surface reflectance mosaics (B01–B12, B8A, AOT, SCL, cloud mask, terrain mask), 10/20/60 m, LV95 |
| [swissEO VHI v100](https://www.swisstopo.admin.ch/en/satelliteimage-swisseo-vhi) | `wms.geo.admin.ch` (`ch.swisstopo.swisseo_vhi_v100`, `…_vegetation`) for display; `forest-10m` / `vegetation-10m` COGs for statistics | Vegetation Health Index for forest and all vegetation, relative to 1991–2020 |
| Basemaps | `wmts.geo.admin.ch` (pixelkarte-farbe, swissimage, pixelkarte-grau) | Background maps on the native LV95 grid |
| Label overlay | `wmts.asit-asso.ch` (`asitvd.fond_pourortho`) | Roads and place names above the data |

The map runs in **EPSG:2056 (CH1903+ / LV95)**, the native CRS of the tile grid and the COGs, so nothing
is reprojected.

### Scaling and no-data

Physical value = DN × scale + offset, as documented in the swisstopo *Additional Content Information*
sheets ([S2-SR v200](https://www.swisstopo.admin.ch/dam/en/sd-web/-i2Y10KmboPf/swissEO_S2-SR_AdditionalContentInfo_v200.pdf),
[VHI](https://www.swisstopo.admin.ch/dam/en/sd-web/vtF6jZqH7L4V/swissEO_VHI_AdditionalContentInfo.pdf)):

| Asset | Scale | Offset | No-data / codes |
| --- | --- | --- | --- |
| B01–B12, B8A | 0.0001 | −0.1 | 0 |
| AOT | 0.001 | 0 | 0 |
| SCL | 1 | 0 | 0 (Sen2Cor classes 1–11) |
| Cloud mask (OmniCloudMask) | 1 | 0 | 0 clear · 1 thick cloud · 2 thin cloud · 3 cloud shadow |
| Terrain mask | 1 | 0 | 0–180 solar incidence angle (°) · 200 shadow · 255 no data |
| VHI | 1 | 0 | 0–100 index · 110 no data (class) · 255 no data (raster) |

Indices (NDVI, NDWI, …) are computed from ρ after this scaling. Very dark pixels can fall slightly below
0 after the −0.1 offset; such negative reflectance is set to 0 before an index is computed, so ratio
indices stay within −1…1.

## Features

- **Date selection as in Copernicus Browser** — opens on today's date (greyed out when there is no
  acquisition), ‹ › jump to the previous/next date with data, the arrow button jumps to the latest one.
  The calendar marks days with data, has month/year pull-downs and a maximum cloud-coverage slider
  (cloud cover from each mosaic's `metadata.json`). Several orbits acquired on the same day are mosaicked.
- **Layers** — true colour and five false-colour composites; NDVI, NDWI, NDMI, NDSI, NBR and AOT; scene
  classification, cloud mask, terrain / solar incidence; swissEO VHI forest and vegetation (WMS); a
  server-rendered true-colour WMS.
- **Layer pull-down** — the Copernicus Browser title, description, preview thumbnail and "More info"
  link to the Sentinel Hub custom-scripts page (swisstopo product page for the swissEO-specific layers),
  legend, adjustable colour scale (min/max), masks for clouds and cloud shadows (OmniCloudMask) and
  terrain shadow (terrain mask 200), and *effects and advanced options* (gain, gamma, red/green/blue
  ranges, opacity). `</>` shows the formula and scaling.
- **Compare** — add layers to the compare list; opening the Compare panel (or the ⇄ map button) shows
  them with a split or opacity effect, going back to Layers shows the single layer again. Reorder,
  remove and zoom to each entry.
- **Area of interest** — draw a rectangle or polygon, or import KML/KMZ, GPX, WKT, GeoJSON, a zipped
  Shapefile, an MGRS/GEOREF cell or a bounding box (WGS84, LV95 and LV03 coordinates are detected). For
  the area: statistics and histogram for the selected date, a time series over a date range (CSV
  download) and a spectral explorer. The time series reads every date in the range, least cloudy first
  (Stop keeps what has been read), and reports which dates were not reached by that day's orbit and
  which were fully masked. Statistics can additionally exclude clouds flagged by the scene
  classification (SCL 3, 8, 9, 10), which catches clouds OmniCloudMask misses.
- **Image download** — PNG of the current view with scale bar, legend and caption; works in compare mode
  and can be clipped to the area of interest.
- **Pixel inspector** — band values (DN and physical value), index value, SCL, cloud mask and terrain
  mask at a clicked location.
- **Permalink** — the URL is updated on every change, so copying it reproduces the view.
- Colours follow the Swiss Confederation scheme used by
  [swisstopo/topo-drought-briefing](https://github.com/swisstopo/topo-drought-briefing).

## Running it

Use the live version at <https://davidoesch.github.io/swisseobrowser/>, or run it locally by serving the
folder with any static web server:

```bash
python -m http.server 8000
# then open http://localhost:8000/swisseo-browser.html
```

A current desktop browser (Firefox, Chrome, Edge) is required.

### Deployment

GitHub Pages is published by the workflow [`.github/workflows/pages.yml`](.github/workflows/pages.yml)
on every push to `main` that changes `swisseo-browser.html`, and can be started manually from the
*Actions* tab. It deploys only `swisseo-browser.html`, as `index.html` (and under its own name), plus the layer
thumbnails it references from `public/previews/`, so the upstream `index.html` and sources are never
published. The repository's *Settings → Pages → Source*
must be set to **GitHub Actions**.

External libraries are loaded from CDNs: Leaflet, Proj4js, Proj4Leaflet, geotiff.js and Leaflet-Geoman;
the import formats additionally load togeojson, JSZip, shpjs and mgrs on demand.

## URL parameters

Parameter names follow the Copernicus Browser where an equivalent exists.

| Parameter | Example | Meaning |
| --- | --- | --- |
| `lat`, `lng`, `zoom` | `46.55`, `6.70`, `17` | Map centre (WGS84) and zoom level of the LV95 grid (8–27; ≈17–18 for a region) |
| `fromTime`, `toTime` | `2026-10-05T00:00:00.000Z` | Selected date |
| `cloudCoverage` | `30` | Maximum cloud coverage in % |
| `layerId` | `ndvi` | `true`, `nir`, `urban`, `swir`, `agri`, `geo`, `ndvi`, `ndwi`, `ndmi`, `ndsi`, `nbr`, `aot`, `scl`, `cloud`, `terrain`, `vhi`, `vhiveg`, `tciwms` |
| `valueRange` | `[0,0.9]` | Colour-scale min/max of index and continuous layers |
| `threshold` | `0.5` | NDSI snow threshold (default 0.42; above it snow is shown in blue, otherwise true colour, as in the Copernicus Browser) |
| `gain`, `gamma` | `1.2` | Effects for the composites (1 = default) |
| `redRange`, `greenRange`, `blueRange` | `[0,0.8]` | Advanced RGB effects |
| `opacity` | `70` | Layer opacity in % |
| `cloudMask`, `shadowMask`, `terrainMask` | `true` | Masks |
| `basemap`, `labels` | `swissimage`, `true` | Basemap (`pixelkarte`, `swissimage`, `grey`, `none`) and label overlay |
| `compareLayers`, `comparedOpacity`, `comparedClipping`, `compareMode` | | Compare list (base64url JSON), per-layer opacity and split range; `compareMode` is `split` / `opacity` when comparing and `off` otherwise (links without it open in compare mode) |
| `aoi` | | Area of interest as base64url GeoJSON |

Example:
`swisseo-browser.html?zoom=17&lat=46.55&lng=6.70&fromTime=2026-10-05T00:00:00.000Z&toTime=2026-10-05T23:59:59.999Z&cloudCoverage=77&layerId=ndvi&valueRange=[0,0.9]`

## Limitations

- Not available compared with the Copernicus Browser: timelapse, 3D, pins, product search and download,
  custom scripts, other satellite collections.
- The VHI layers are rendered by the WMS with its published styling, so their colour scale cannot be changed.
- Statistics and time series read the COGs in the browser; large areas and long date ranges take a while.

## Credits and terms

- Data © [swisstopo](https://www.swisstopo.admin.ch) —
  [open government data](https://www.swisstopo.admin.ch/ogd-conditions). swissEO S2-SR and swissEO VHI
  contain modified Copernicus Sentinel data. Label overlay © [ASIT VD](https://www.asit-asso.ch).
- User-interface workflow modelled on the [Copernicus Browser](https://browser.dataspace.copernicus.eu/)
  by the Copernicus Data Space Ecosystem. This project is not affiliated with or endorsed by the
  Copernicus Data Space Ecosystem.
- The repository keeps the upstream MIT license ([LICENSE.md](LICENSE.md)).
