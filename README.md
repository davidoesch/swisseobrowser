# swissEO Browser `BETA`

A minimal, browser-only viewer for swisstopo's **swissEO** satellite products, modelled on the
[Copernicus Browser](https://browser.dataspace.copernicus.eu/) workflow.

**Live version: <https://davidoesch.github.io/swisseobrowser/>**

> **Not for operational use.** This is an experimental beta prototype: values, renderings and statistics
> are computed in the browser, are not validated and may be wrong or incomplete. For authoritative data use
> the swisstopo products and services directly. The website was developed with the help of
> [Claude Code](https://claude.com/claude-code) (Anthropic), an AI coding assistant. Every layer is being
> checked in depth; see [Layer verification](#layer-verification).

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
| [swissEO S2-SR v200](https://www.swisstopo.admin.ch/en/satelliteimage-swisseo-s2-sr) | COGs on `data.geo.admin.ch`, catalogue via the STAC API `data.geo.admin.ch/api/stac/v0.9` (collection `ch.swisstopo.swisseo_s2-sr_v200`) | Sentinel-2 L2A surface reflectance mosaics (B01–B12, B8A, AOT, SCL, cloud mask, terrain mask, true color image TCI), 10/20/60 m, LV95 |
| [swissEO VHI v100](https://www.swisstopo.admin.ch/en/satelliteimage-swisseo-vhi) | Own collection: `wms.geo.admin.ch` (`ch.swisstopo.swisseo_vhi_v100`, `…_vegetation`) for display; for AOI statistics and analytical downloads the `forest-10m` / `vegetation-10m` COGs, addressed through the STAC items of `ch.swisstopo.swisseo_vhi_v100` | Vegetation Health Index for forest and all vegetation, relative to 1991–2020 |
| [swissEO NDVIdiff v100](https://www.swisstopo.admin.ch/en/satelliteimage-swisseo-ndvidiff) | `forest-10m` COGs of the STAC collection `ch.swisstopo.swisseo_ndvi_diff_v100` | NDVI change of the current year compared to the previous year, forest perimeter, 10 m; one product for the end of July, August and September each year (since 2018) |
| [swissEO NDVIz v100](https://www.swisstopo.admin.ch/en/satelliteimage-swisseo-ndviz) | `forest-10m` COGs of the STAC collection `ch.swisstopo.swisseo_ndvi_z_v100` | NDVI anomaly as z-score, forest perimeter, 10 m; end of July, August and September (since 2017) |
| Vegetation masks | The `forest-10m` / `vegetation-10m` COGs of swissEO VHI v100 (STAC collection `ch.swisstopo.swisseo_vhi_v100`, most recent item): inside the mask = value ≠ 255 | Forest / all-vegetation mask for every S2-SR layer, statistics and exports |
| Basemaps | `wmts.geo.admin.ch` (pixelkarte-farbe, swissimage, pixelkarte-grau) | Background maps on the native LV95 grid |
| Label overlay | `wmts.asit-asso.ch` (`asitvd.fond_pourortho`) | Roads and place names above the data |

The map runs in **EPSG:2056 (CH1903+ / LV95)**, the native CRS of the tile grid and the COGs, so nothing
is reprojected.

### Vegetation masks

The vegetation mask option restricts every S2-SR layer, the AOI statistics, time series and spectral
explorer, and the analytical and print exports to forest or to all vegetation. It is read from the
`forest-10m` and `vegetation-10m` COGs of **swissEO VHI v100**: the VHI processor writes 255 (no data)
outside its forest / vegetation mask and 110 where VHI is missing inside it, so *inside = value ≠ 255*.
The v100 mask is the same on every date, so the most recent VHI item is used for all dates.

> **Needed soon:** the VHI v200 processor ([topo-satromo-v2](https://github.com/swisstopo/topo-satromo-v2))
> uses versioned masks (`PRODUCT_VHI["vegetation_masks"]` = `s3://s3-topo-satromo-prod/data/MASKS/Vegetation/`,
> habitat map v1-0 / v1-1 / v1-2 chosen by date in `step1_processor_vhi.py`). Those files are plain
> 767 MB TIFFs (not cloud-optimised) in a bucket without CORS, so a web page cannot read them. To use
> them, the masks have to be published as **COGs** (tiled, with overviews) at a CORS-enabled location,
> e.g. with the VHI v200 STAC items. The browser then only needs the new URLs and the version-by-date
> rule (`readVegMask` in `swisseo-browser.html` already receives the date).

### Scaling and no-data

Physical value = DN × scale + offset, as documented in the swisstopo *Additional Content Information*
sheets ([S2-SR v200](https://www.swisstopo.admin.ch/dam/en/sd-web/-i2Y10KmboPf/swissEO_S2-SR_AdditionalContentInfo_v200.pdf),
[VHI](https://www.swisstopo.admin.ch/dam/en/sd-web/vtF6jZqH7L4V/swissEO_VHI_AdditionalContentInfo.pdf),
[NDVIdiff](https://www.swisstopo.admin.ch/dam/en/sd-web/bODfHY8ioFyS/swissEO_NDVIdiff_AdditionalContentInfo.pdf),
[NDVIz](https://www.swisstopo.admin.ch/dam/en/sd-web/mZTj7kABmTrD/swissEO_NDVIz_AdditionalContentInfo.pdf)):

| Asset | Scale | Offset | No-data / codes |
| --- | --- | --- | --- |
| B01–B12, B8A | 0.0001 | −0.1 | 0 |
| AOT | 0.001 | 0 | 0 |
| SCL | 1 | 0 | 0 (Sen2Cor classes 1–11) |
| Cloud mask (OmniCloudMask) | 1 | 0 | 0 clear · 1 thick cloud · 2 thin cloud · 3 cloud shadow |
| Terrain mask | 1 | 0 | 0–180 solar incidence angle (°) · 200 shadow · 255 no data |
| VHI | 1 | 0 | 0–100 index · 110 no data (class) · 255 no data (raster) |
| NDVIdiff (int16) | 0.001 | 0 | 32700 missing data (shown grey) · 32701 no data |
| NDVIz (int16) | 0.01 | 0 | 32700 missing data (shown grey) · 32701 no data |

**NDVIdiff / NDVIz overviews:** the internal overviews of these COGs average the missing-data code 32700
into valid values (e.g. 31630 or −9395 in the overviews, while the full resolution stays within
−0.8…0.65 ΔNDVI). The browser therefore treats overview values outside |ΔNDVI| ≤ 1 and |z| ≤ 20 as no
data; at full resolution (zoomed in, or AOI statistics on a grid finer than 14.8 m) every value is used,
and the statistics then match a rasterio read of the COG exactly. The overviews should be rebuilt with
32700 excluded (e.g. as an additional no-data value or a mask).

Indices (NDVI, NDWI, …) are computed from ρ after this scaling. Very dark pixels can fall slightly below
0 after the −0.1 offset; such negative reflectance is set to 0 before an index is computed, so ratio
indices stay within −1…1.

**Why indices are higher than in the Copernicus Browser:** since processing baseline 04.00 the Sentinel-2
L2A digital numbers contain a +1000 shift, which the offset −0.1 removes. The Copernicus Browser requests
its data with Sentinel Hub's `harmonizeValues`, which is numerically equivalent to leaving the offset out,
so its NDVI over dense vegetation is markedly lower (e.g. 0.56 instead of 0.87). CDSE confirmed that the
offset is correct and recommends keeping it
([forum thread](https://forum.dataspace.copernicus.eu/t/sentinel-2-baseline-04-00-offset-csde-stac-derived-ndvi-doesnt-match-copernicus-browser-ndvi/5417)).
The swissEO Browser therefore always uses the physical reflectance.

## Features

- **Date selection as in Copernicus Browser** — a link without a date opens on the latest date with data;
  ‹ › jump to the previous / next date with data, the arrow button jumps to the latest one. The calendar,
  ‹ › and the grey "no data" date only consider acquisitions that **reach the map view**: the swath edge
  runs diagonally through each mosaic, so the data footprint of every relative orbit (`ORBIT_NR`) is read
  once from the coarsest B04 overview (≈ 640 m) and the calendar updates when the map moves.
  The calendar marks days with data, has month / year pull-downs and a maximum cloud-coverage slider
  (cloud cover of the whole mosaic, from its `metadata.json`, as the tile cloud cover in Copernicus). Several orbits acquired on the same day are mosaicked
  (most recent on top).
- **Configurations** — as the themes of the Copernicus Browser, the *Configuration* pull-down lists the
  layers for a task; each layer keeps its own data source, and the calendar shows the dates of the
  selected layer's source:
  - *Default* — composites (true color, false color, urban, SWIR, agriculture, geology), indices (NDVI,
    NDWI, moisture index, NDSI, NBR, AOT, **swissEO VHI** forest and vegetation, **swissEO NDVIdiff**),
    masks & quality (scene classification, cloud mask, terrain);
  - *Agriculture* — as the Copernicus Browser Agriculture theme, with the TCI as true color and without
    custom scripts: true color, false color, NDVI, EVI, Barren Soil, Moisture Stress, moisture index,
    agriculture, SAVI (colour steps and scripts of the Copernicus layers);
  - *Drought* — NDVI, swissEO VHI forest / vegetation, swissEO NDVIdiff, swissEO NDVIz, NDDI, NDDI change
    vs. the previous year and the NDVI Analyst (Canton of Aargau);
  - *Drought 2026* — as the Copernicus Browser *Drought and Floods* theme: true color, moisture index,
    NDWI, Moisture Stress, NDVI, SWIR, urban and Highlight Optimized Natural Color, plus **highlights**.
- **Highlights** (*Drought 2026*) — story cards as in the Copernicus Browser (live thumbnail, title,
  dates, chevron). Clicking a card opens it on the map — the place at a fixed resolution and the TCI of the
  two dates as a split at 50 % (earlier date left), with a swipe handle — and the card turns blue; clicking
  another card switches. The chevron shows the text (English, at most ten lines), ending with *Show effects
  and advanced options*, which opens the compare settings. Choosing a layer leaves the highlight. Shared
  links reopen the highlight. Stories (defined in `CONFIGS.DROUGHT2026.highlights`: title, dates, layer,
  LV95 centre, m/px, text):

  | Story | Dates | Source of the text |
  | --- | --- | --- |
  | Baselland | 2026-05-23 ↔ 2026-07-04 | own |
  | Gürbetal and Aare valley | 2026-05-28 ↔ 2026-07-04 | [SRF Meteo](https://www.srf.ch/meteo/meteo-stories/trockenheit-in-der-schweiz-die-duerre-aus-der-vogelperspektive), 9 July 2026 (place identified from the SRF image) |
  | Great Aletsch Glacier | 2019-08-18 ↔ 2026-08-13 | [watson.ch](https://www.watson.ch/schweiz/klima/773592776-dieser-satellitenbilder-vergleich-zeigt-trockenheit-in-der-schweiz) |
  | Konstanz and Untersee, Zurich Airport, Forests near Brugg AG, Upper Lake Zurich, Wil SG, Lake Lucerne, Interlaken BE | 2025-08-08 ↔ 2026-08-08 | watson.ch, 10 August 2026 |
  | Lake Constance – Untersee | 2025-08-08 ↔ 2026-08-08 | [Blick](https://www.blick.ch/schweiz/ostschweiz/thurgau/untersee-faellt-auf-tiefsten-august-wert-seit-1930/49bvg7s), 12 August 2026 |
  | Lakes Neuchâtel and Murten | 2025-08-08 ↔ 2026-08-11 (no clear swissEO image on 8 August 2026 there) | watson.ch |
  | All of Switzerland | 2025-08-08 ↔ 2026-07-24 | watson.ch |

  The Tages-Anzeiger article on the same topic could not be read (paywall, archive not reachable).
- **Verification status** — behind every layer name a symbol shows whether the layer has been checked in
  depth (✓ *Verified*) or not yet (⏳ *To be verified*); see [Layer verification](#layer-verification).
- **Layers** — S2-SR: true color — the **TCI** (`tci_10m`, 8-bit RGB, JPEG / YCbCr) delivered with each mosaic,
  shown as delivered — and the composites computed from the bands (false color, urban, SWIR, agriculture,
  geology 12-8-2, Barren Soil, Highlight Optimized Natural Color); NDVI, NDWI, moisture index, NDSI, NBR,
  AOT, EVI, SAVI, Moisture Stress, NDDI; scene classification, cloud mask, terrain / solar incidence.
  VHI: forest and vegetation (WMS). Layers are listed with the Copernicus Browser preview thumbnails as icons.
- **Drought layers**
  - *swissEO NDVIdiff* and *swissEO NDVIz* are read from their COGs with the colour ranges of the content
    information sheets (missing data grey). There are only three products per year (end of July, August,
    September), so the product **nearest to the selected date** is shown — also in compare, timelapse,
    statistics and exports; the date line names the product used.
  - *NDDI* = (NDVI − NDWI) / (NDVI + NDWI), clipped to −1…1, with the palette of the
    [openEO drought notebook](https://github.com/Open-EO/openeo-community-examples/blob/main/python/50_thematic-notebooks/drought-nddi/drought-nddi.ipynb).
    By default the NDWI is computed from NIR and SWIR (B08, B11) as in
    [Gu et al. 2007](https://doi.org/10.1029/2006GL029127). The notebook (Awesome Spectral Indices) uses an
    NDWI from green and NIR (B03, B08); it is selectable as *Green*, with a warning: NDVI + NDWI is then
    close to zero over vegetation and the index saturates. Test over Aarau, 2026-07-04, cloud-free pixels:

    | NDWI | inside −1…1 | median | at +1 (vegetation, NDVI > 0.5) |
    | --- | --- | --- | --- |
    | green (notebook), with the −0.1 offset | 0.2 % | 19.5 | 86 % |
    | green (notebook), without the offset (DN / 10000) | 0.2 % | 23.2 | 100 % |
    | SWIR (Gu et al. 2007), with the offset | 74.5 % | 0.54 | 4 % |

    So the all-red map comes from the green NDWI, not from the reflectance offset.
  - *NDDI change vs. previous year* — NDDI of the date minus NDDI one year earlier (notebook palette,
    −1…1). The notebook compares May–July means of two years; here single dates are compared.
  - *NDVI Analyst (Kt. Aargau)* — port of [RaffiBienz/ndvi_analyst](https://github.com/RaffiBienz/ndvi_analyst):
    ΔNDVI = NDVI(date) − NDVI(reference) (negative reflectance set to 0), forest mask by default, change =
    ΔNDVI below a threshold (default −0.08) followed by a 3 × 3 majority filter against edge effects.
    *Display*: the difference, or the detected change. The AOI statistics add the report of the original
    script: forest area, compared area and affected area in ha and %, with and without edge-effect removal.
  - Reference of the two-date layers: the acquisitions within ±20 days of the same day one year earlier
    (or of a chosen reference date, which goes first), clearest first; up to six days form a per-pixel
    composite, each pixel taking the first acquisition that covers it without clouds, so orbit gaps and
    clouds are filled. Clouds and cloud shadows are masked on both sides by default.
- **Layer pull-down**
  - the Copernicus Browser title, description and "More info" link (Sentinel Hub custom-scripts page;
    swisstopo product page for True color L2A and the swissEO-specific layers); `</>` shows the formula
    and scaling;
  - legend with adjustable colour scale (min / max); NDSI is rendered as in the Copernicus Browser (snow
    above an adjustable threshold in blue, true color elsewhere);
  - masks: clouds and cloud shadows (OmniCloudMask), terrain shadow (terrain mask 200) and a vegetation
    mask (Off / Forest / All vegetation, from swissEO VHI v100);
  - *effects and advanced options*: gain, gamma, red / green / blue ranges and opacity for all colour
    layers (for the TCI applied on top of the delivered image); for the composites computed from the bands
    also **Look** — *Standard* stretch or *True
    color optimized* (the tone curve of the Copernicus Browser True color layer: contrast enhancement with
    highlight compression, saturation 1.2, sRGB); gain, gamma, **adaptive gain** (lowers the gain when the
    view is bright — snow, glaciers, clouds — so that the 98th percentile of the reflectance stays just
    below saturation; never brightens, at most a factor of 4, recomputed when the view changes; not needed
    with the optimized look). Statistics, spectral explorer and pixel inspector use the reflectance bands
    also for true color; the inspector additionally shows the TCI value.
- **Compare** — add layers to the compare list; opening the Compare panel (or the ⇄ map button) shows
  them with a split or opacity effect, going back to Layers shows the single layer again. Reorder,
  remove and zoom to each entry. With the split effect a handle on the map swipes the top layer, with the
  date of each side next to it.
- **Area of interest** — draw a rectangle (axis-aligned in LV95) or polygon, or import KML/KMZ, GPX, WKT,
  GeoJSON, a zipped Shapefile, an MGRS / GEOREF cell or a bounding box (WGS84, LV95 and LV03 coordinates
  are detected). Vertices stay editable and drawing again adds a polygon. The AOI is shown as an outline
  only; its toolbar copies the geometry, shows the area, centres the map and downloads the AOI as GeoJSON.
  For the area:
  - *Statistical info*: statistics and histogram for the selected date, and a time series over a date
    range (CSV download). The time series reads every date in the range — clouds are judged over the
    AOI itself, not by the cloud cover of the whole scene — least cloudy first (Stop keeps what has been
    read), and reports which dates were not reached by that day's orbit and which were fully masked. Clouds can additionally be excluded with the scene classification (SCL 3, 8, 9, 10),
    which catches clouds OmniCloudMask misses, and the area can be restricted to forest or to all
    vegetation (the share of valid pixels then refers to the masked area). VHI, NDVIdiff and NDVIz
    statistics are read from their COGs; the two-date layers report the difference (and the NDVI Analyst
    report) instead of a time series.
  - *Spectral explorer*: mean surface reflectance of all 12 bands (± σ), with CSV download.
- **Image download** — three tabs as in the Copernicus Browser:
  - *Basic*: the current view as PNG or JPG, with captions (datasource, date, scale bar) and an optional
    description, map overlays, legend, crop to AOI and an optional AOI outline; works in compare mode.
  - *Analytical*: any layers of the collection (visualised) and raw bands (B01–B12, B8A, AOT, SCL, cloud
    mask, terrain mask; VHI values) for the current view or the AOI, at LOW / MEDIUM / HIGH (40 / 20 /
    10 m) or a custom resolution, up to 2500 × 2500 px, as PNG, JPG or GeoTIFF (8-bit, 16-bit = stored DN,
    32-bit float = index or physical value) in CH1903+ / LV95, optionally with a dataMask band. Several
    files are zipped.
  - *High-res print*: the view re-rendered for a printed size (width / height in inches and DPI), with
    the same captions, legend, overlay and AOI options, as PNG or JPG.
- **Timelapse** — as in the Copernicus Browser, for the current layer over the area of interest (data
  clipped to it, no outline drawn) or the current view:
  - time range, *Select 1 image per* orbit / day / week / month / year, month filter, Search;
  - list of images with thumbnail, cloud cover and coverage of the timelapse area (AOI or view, computed
    from the cloud mask and the data footprint — not the cloud cover of the whole scene); per interval the image with the best coverage × clear sky is chosen; Max. cloud
    coverage and Min. tile coverage filter the list, images can be (de)selected;
  - preview player with speed (fps), transition None / Fade with fade duration, delay last frame,
    adaptive gain per frame for the band composites (snowy winter frames), legend, map overlays and captions (date label,
    scale bar, source);
  - download as GIF or as video (MP4 or WebM, depending on the browser), default 1024 px (up to 2048).
    VHI layers work too (via the WMS).
- **Loading indicator** — bottom left: the number of tiles still rendering and of requests in flight; after
  6 s of continuous loading it says that the connection is slow.
- **Pixel inspector** — band values (DN and physical value), index value, SCL, cloud mask, terrain mask
  and forest / vegetation mask at a clicked location; for the two-date layers the index on both sides and
  the difference; for VHI, NDVIdiff and NDVIz the product value.
- **Permalink** — the URL is updated on every change (configuration, view, date, cloud filter, layer,
  legend range, masks, look and effects, reference date, compare list, AOI); *Copy link* in the header
  copies it.
- **Info tab** — the *not for operational use* notice, the acquisitions of the selected date with their
  metadata, the scaling / no-data table, a description of the services used and links to all
  repositories and libraries the website builds on.
- Colours follow the Swiss Confederation scheme used by
  [swisstopo/topo-drought-briefing](https://github.com/swisstopo/topo-drought-briefing).

## Layer verification

The layers are being checked one by one, in depth: data access (assets, dates, nearest product), scaling and
no-data, the formula and its source, masks, rendering and legend, statistics and exports — compared with
reference implementations where possible (GDAL / rasterio reads of the COGs, the original scripts). Until a
layer has passed this check it is shown with ⏳ *To be verified*; afterwards with ✓ *Verified*.

**This checklist is the source of the symbols.** Tick a box (`[x]`) when a layer is verified and push: the
Pages workflow reads the list between the two markers below on every deploy (also when only this README
changed) and writes the checked layer ids into the published page. The workflow warns when a layer of the
page is missing here or an id here is unknown. A local copy of `swisseo-browser.html` served from the
repository reads this README itself.

<!-- verification:start -->
- [x] `true` — True color *(Default, Agriculture, Drought 2026)*
- [x] `nir` — False color *(Default, Agriculture)*
- [ ] `urban` — False color (urban) *(Default, Drought 2026)*
- [x] `swir` — SWIR *(Default, Drought 2026)*
- [ ] `agri` — Agriculture *(Default, Agriculture)*
- [ ] `geo` — Geology 12, 8, 2 *(Default)*
- [x] `ndvi` — NDVI *(Default, Agriculture, Drought, Drought 2026)*
- [ ] `ndwi` — NDWI *(Default, Drought 2026)*
- [ ] `ndmi` — Moisture index *(Default, Agriculture, Drought 2026)*
- [ ] `ndsi` — NDSI *(Default)*
- [ ] `nbr` — Normalized Burn Ratio (NBR) *(Default)*
- [ ] `aot` — Aerosol optical thickness *(Default)*
- [x] `scl` — Scene classification map *(Default)*
- [x] `cloud` — Cloud mask *(Default)*
- [x] `terrain` — Terrain / solar incidence *(Default)*
- [ ] `vhi` — swissEO VHI forest *(Default, Drought)*
- [ ] `vhiveg` — swissEO VHI vegetation *(Default, Drought)*
- [ ] `evi` — EVI *(Agriculture)*
- [ ] `savi` — SAVI *(Agriculture)*
- [ ] `barren` — Barren Soil *(Agriculture)*
- [ ] `mstress` — Moisture Stress *(Agriculture, Drought 2026)*
- [ ] `hon` — Highlight Optimized Natural Color *(Drought 2026)*
- [ ] `ndvidiff` — swissEO NDVIdiff *(Default, Drought)*
- [ ] `ndviz` — swissEO NDVIz *(Drought)*
- [ ] `nddi` — NDDI *(Drought)*
- [ ] `nddichg` — NDDI change vs. previous year *(Drought)*
- [ ] `analyst` — NDVI Analyst (Kt. Aargau) *(Drought)*
<!-- verification:end -->

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
on every push to `main` that changes `swisseo-browser.html` or `README.md`, and can be started manually from
the *Actions* tab. On every deploy it applies the [layer verification](#layer-verification) checklist of this
README to the page. It deploys only `swisseo-browser.html`, as `index.html` (and under its own name), plus
the layer thumbnails it references from `public/previews/`, so the upstream `index.html` and sources are
never published. The repository's *Settings → Pages → Source* must be set to **GitHub Actions**.

External libraries are loaded from CDNs: Leaflet, Proj4js, Proj4Leaflet, geotiff.js and Leaflet-Geoman;
the import formats additionally load togeojson, JSZip, shpjs and mgrs on demand, and the timelapse GIF
export loads gifenc.

## URL parameters

Parameter names follow the Copernicus Browser where an equivalent exists.

| Parameter | Example | Meaning |
| --- | --- | --- |
| `lat`, `lng`, `zoom` | `46.55`, `6.70`, `17` | Map centre (WGS84) and zoom level of the LV95 grid (8–27; ≈17–18 for a region; fractional values, as written by the highlights, give resolutions between the grid levels) |
| `fromTime`, `toTime` | `2026-10-05T00:00:00.000Z` | Selected date |
| `cloudCoverage` | `30` | Maximum cloud coverage in % |
| `themeId` | `DROUGHT` | Configuration: `DEFAULT`, `AGRICULTURE`, `DROUGHT`, `DROUGHT2026` (when missing or not containing the layer, the first configuration with the layer is used) |
| `layerId` | `ndvi` | `true`, `nir`, `urban`, `swir`, `agri`, `geo`, `barren`, `hon`, `ndvi`, `ndwi`, `ndmi`, `ndsi`, `nbr`, `aot`, `evi`, `savi`, `mstress`, `nddi`, `nddichg`, `analyst`, `scl`, `cloud`, `terrain` (S2-SR); `vhi`, `vhiveg` (VHI); `ndvidiff`, `ndviz` |
| `valueRange` | `[0,0.9]` | Colour-scale min/max of index and continuous layers |
| `threshold` | `0.5` | NDSI snow threshold (default 0.42; above it snow is shown in blue, otherwise true colour, as in the Copernicus Browser); NDVI Analyst change threshold (default −0.08) |
| `refDate` | `2025-07-01` | Reference date of the two-date layers (default: the same day one year earlier) |
| `displayMode` | `change` | NDVI Analyst display: `diff` (default) or `change` |
| `ndwi` | `green` | NDWI of the NDDI layers: `swir` (default, Gu et al. 2007) or `green` (openEO notebook) |
| `gain`, `gamma` | `1.2` | Effects for the composites (1 = default) |
| `autoGain` | `true` | Adaptive gain for the composites |
| `look` | `optimized` | *True color optimized* tone curve for the composites (default: standard stretch) |
| `redRange`, `greenRange`, `blueRange` | `[0,0.8]` | Advanced RGB effects |
| `opacity` | `70` | Layer opacity in % |
| `cloudMask`, `shadowMask`, `terrainMask` | `true` | Masks (written when they differ from the layer's default; the two-date layers mask clouds and shadows by default) |
| `vegetationMask` | `forest` | Vegetation mask: `off`, `forest` or `vegetation` (S2-SR layers; from swissEO VHI v100; the NDVI Analyst defaults to `forest`) |
| `basemap`, `labels` | `swissimage`, `true` | Basemap (`pixelkarte`, `swissimage`, `grey`, `none`) and label overlay |
| `compareLayers`, `comparedOpacity`, `comparedClipping`, `compareMode` | | Compare list (base64url JSON), per-layer opacity and split range; `compareMode` is `split` / `opacity` when comparing and `off` otherwise (links without it open in compare mode) |
| `aoi` | | Area of interest as base64url GeoJSON |

Example:
`swisseo-browser.html?zoom=17&lat=46.55&lng=6.70&fromTime=2026-10-05T00:00:00.000Z&toTime=2026-10-05T23:59:59.999Z&cloudCoverage=77&layerId=ndvi&valueRange=[0,0.9]`

## Limitations

- Not available compared with the Copernicus Browser: 3D, pins, product search and download, custom
  scripts, other satellite collections, sharing timelapses through a server; the image download has no
  OSM background and no choice of coordinate system (always LV95).
- The VHI layers are rendered by the WMS with its published styling, so their colour scale cannot be changed.
- NDVIdiff / NDVIz: see the note on their overviews above; zoomed out, implausible overview pixels are left out.
- NDDI and NDDI change compare single acquisitions, not the May–July means of the openEO notebook. With the
  notebook's green NDWI the NDDI saturates (see above); the default SWIR NDWI does not.
- The NDVI Analyst works on the swissEO S2-SR mosaics instead of the original script's own Sentinel-2
  processing; AOI reports use a grid of at least 10 m (coarser for large areas, which the report states).
- Statistics, time series, analytical downloads and high-res prints read the COGs in the browser; large
  areas, long date ranges and large prints take a while.

## Credits and terms

- Data © [swisstopo](https://www.swisstopo.admin.ch) —
  [open government data](https://www.swisstopo.admin.ch/ogd-conditions). swissEO S2-SR and swissEO VHI
  contain modified Copernicus Sentinel data. Label overlay © [ASIT VD](https://www.asit-asso.ch).
- User-interface workflow modelled on the [Copernicus Browser](https://browser.dataspace.copernicus.eu/)
  by the Copernicus Data Space Ecosystem ([eu-cdse/copernicus-browser](https://github.com/eu-cdse/copernicus-browser):
  themes, layer descriptions, preview thumbnails). This project is not affiliated with or endorsed by the
  Copernicus Data Space Ecosystem.
- Builds on: [swisstopo/topo-satromo-v2](https://github.com/swisstopo/topo-satromo-v2) (processing of the
  swissEO products), [swisstopo/topo-drought-briefing](https://github.com/swisstopo/topo-drought-briefing)
  (colour scheme), [RaffiBienz/ndvi_analyst](https://github.com/RaffiBienz/ndvi_analyst) (NDVI Analyst of
  the Canton of Aargau), [Open-EO/openeo-community-examples](https://github.com/Open-EO/openeo-community-examples)
  (NDDI drought notebook), [sentinel-hub/custom-scripts](https://github.com/sentinel-hub/custom-scripts)
  (index definitions and visualisations); libraries [Leaflet](https://github.com/Leaflet/Leaflet),
  [Proj4Leaflet](https://github.com/kartena/Proj4Leaflet), [proj4js](https://github.com/proj4js/proj4js),
  [Leaflet-Geoman](https://github.com/geoman-io/leaflet-geoman), [geotiff.js](https://github.com/geotiffjs/geotiff.js),
  [togeojson](https://github.com/placemark/togeojson), [shpjs](https://github.com/calvinmetcalf/shapefile-js),
  [mgrs](https://github.com/proj4js/mgrs), [JSZip](https://github.com/Stuk/jszip) and
  [gifenc](https://github.com/mattdesl/gifenc).
- Highlight texts after [watson.ch](https://www.watson.ch/schweiz/klima/773592776-dieser-satellitenbilder-vergleich-zeigt-trockenheit-in-der-schweiz)
  (Philipp Reich, 10 August 2026) and [SRF Meteo](https://www.srf.ch/meteo/meteo-stories/trockenheit-in-der-schweiz-die-duerre-aus-der-vogelperspektive)
  (Roman Brogli, Timothy Schwitter, 9 July 2026) and [Blick](https://www.blick.ch/schweiz/ostschweiz/thurgau/untersee-faellt-auf-tiefsten-august-wert-seit-1930/49bvg7s)
  (12 August 2026), shortened and translated; the images are swissEO S2-SR TCI.
- Developed with the help of [Claude Code](https://claude.com/claude-code) (Anthropic). Not for operational use.
- The repository keeps the upstream MIT license ([LICENSE.md](LICENSE.md)).
