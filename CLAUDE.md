# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **data-only** repository: no build, no package manager, no tests. It hosts two JSON files that the NUPSI Angular frontend (sibling repo `../NUPSI-FE`) fetches **at runtime** from GitHub raw URLs. The repo is effectively a remotely-editable config store for the frontend:

- `NEWMENU.json` — the product catalogue (menu of satellite-derived map products).
- `TOUR.json` — the Intro.js guided-tour steps.

`README.md` is empty. `.autogit/` and `.idea/` are editor tooling and are git-ignored.

## How the frontend consumes these files

The URL each frontend fetches decides when an edit goes live:

- **NEWMENU.json**: `https://raw.githubusercontent.com/AliaSpaceSystems/NUPSI-FE-menu/main/NEWMENU.json`. Both the legacy FE (`../NUPSI-FE/OLD-FE-1.1.2`, `persistence.service.ts` / `query-compose.service.ts`) and the new FE (`../NUPSI-FE/NUPSI-FE-CESIUM`, `MenuService`) read it from **`main`**. Changes on a release branch (`nupsi-1.x.0`) take effect only after they are merged to `main`.
- **TOUR.json**: the legacy FE reads `refs/heads/main/TOUR.json`. `NUPSI-FE-CESIUM`'s `TourService.TOUR_URL` is **pinned to a commit SHA**, so after a TOUR.json change someone has to update `TOUR_URL` in that repo to the new commit before the new tour shows up there.

Because the files are fetched live, a malformed JSON file on `main` breaks the menu or tour in production. Validate before committing:

```bash
python3 -m json.tool NEWMENU.json > /dev/null && python3 -m json.tool TOUR.json > /dev/null && echo OK
```

## NEWMENU.json schema

A flat array of product objects. All values are **strings**, including the booleans. Fields:

- `id` — UUID, unique per entry.
- `level1` — category (e.g. "The air we breath", "Extraordinary Events"). The UI groups products by this exact string, so keep the spelling identical across entries, typos included.
- `level2` — product display name. `product_name` — long description.
- `input_name` — short product code (e.g. `S2 TCI`, `S5P L2 CH4`). The CESIUM FE splits it on whitespace to get `bandId`, dedupes the Satellite view by it, and seeds the default product from the first enabled `S2 TCI`.
- `satellite` — `Sentinel-1` / `Sentinel-2` / `Sentinel-3` / `Sentinel-5P`; drives the Satellite tab.
- `enabled` — `"true"` / `"false"` (string). Only `"true"` entries are shown.
- Images, all paths relative to the FE's `assets/`: `image` = category banner (one per `level1`), `image2` = product thumbnail, `image3` = always `assets/go.png`, `image4` = satellite image (one per `satellite`). Filenames may contain spaces; the FE URL-encodes them.
- Rendering scripts (empty string = not available):
  - `script` — **base64-encoded JSON** with a `postprocessing` object (band expressions, `stretch_range`, `colormap_*`). Decode with `base64 -d` to edit it, then re-encode.
  - `script_riotiler` — TiTiler/rio-tiler query string starting with `&` (`bands=`/`assets=`, `expression=`, `rescale=`, `color_formula=`, optional `colormap=` JSON, `asset_as_band=True`). The FE splits it on `&` and `=`, so values must not contain those characters unescaped.
  - `script_mosaics` — the same format, for mosaics.
  - `script_statistics` — the same format, for TiTiler `/stac/statistics`. A non-empty value is what turns on the Statistics button in the FE.

## TOUR.json schema

`{ "steps": [ { "title", "intro", "element"? } ] }` (Intro.js shape). `element` is a CSS selector into the FE's DOM. A step without `element`, or whose selector matches nothing (the CESIUM FE strips unmatched selectors), renders as a centred tooltip. Fix tour anchors **here**, not by adding ids or classes to FE components.

Some `intro` strings embed `<img src='x' onerror='...'>` to run JS during the tour (open panels, click the hamburger or sidebar buttons when Next is pressed, guarded with `dataset.hooked*` flags). This is deliberate. When editing it, keep the JS inside a single-quoted HTML attribute with JSON-escaped inner double quotes, and keep the hook flags unique per step.

## Git workflow

- `main` is what production reads. Release branches are `nupsi-1.1.0` and `nupsi-1.2.0`, both also on `origin`.
- Commits are mostly small single-file edits made through the GitHub web UI (e.g. "Update TOUR.json"). Pull before editing, because collaborators push here directly.
