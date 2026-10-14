# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **data-only** repository: no build, no package manager, no tests. It hosts JSON files that the NUPSI Angular frontend (sibling repo `../NUPSI-FE`) fetches **at runtime** from GitHub raw URLs. The repo is effectively a remotely-editable config store for the frontend:

- `NEWMENU.json` — the product catalogue (menu of satellite-derived map products).
- `TOURDESKTOP.json`, `TOURTABLET.json`, `TOURMOBILE.json` — the Intro.js guided-tour steps, one file per layout. Until 2026-10-14 there was a single `TOUR.json`; it was renamed to `TOURDESKTOP.json`, and `TOURTABLET.json` started as a copy of it.

`README.md` is empty. `.autogit/` and `.idea/` are editor tooling. `.idea/` is listed in `.gitignore` but its files are still tracked.

## How the frontend consumes these files

The URL each frontend fetches decides when an edit goes live:

- **NEWMENU.json**: `https://raw.githubusercontent.com/AliaSpaceSystems/NUPSI-FE-menu/main/NEWMENU.json`. Both the legacy FE (`../NUPSI-FE/OLD-FE-1.1.2`, `persistence.service.ts` / `query-compose.service.ts`) and the new FE (`../NUPSI-FE/NUPSI-FE-CESIUM`, `MenuService`) read it from **`main`**. Changes on a release branch (`nupsi-1.x.0`) take effect only after they are merged to `main`.
- **Tour files**: the frontends still request a single `TOUR.json`, which **no longer exists on `nupsi-1.2.0`**. The legacy FE reads `refs/heads/main/TOUR.json`. `NUPSI-FE-CESIUM` reads `RUNTIME_CONFIG.TOUR_URL` (an `env.js` override, defaulting in `runtime-config.ts` to a URL pinned to commit `5434610…/TOUR.json`). As of 2026-10-14 the FE has no per-device selection yet: wiring up `TOURDESKTOP`/`TOURTABLET`/`TOURMOBILE` means changing the FE (or `env.js`), and every tour edit needs the pinned URL bumped to the new commit before it shows up.

Because the files are fetched live, a malformed JSON file on `main` breaks the menu or tour in production. Validate before committing:

```bash
for f in NEWMENU.json TOUR*.json; do python3 -m json.tool "$f" > /dev/null || echo "BAD: $f"; done
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

All three files have the same 20 steps in the same order. Keep the step indexes aligned across the files so a change to one step can be carried over to the others.

Tour steps run JavaScript through `<img src='…' onerror='…' style='display:none;' />` at the start of `intro`. This works only because intro.js inserts `intro` as unsanitised HTML. It is a known XSS vector: anyone who can push to this repo can run JS in the app. Keep the JS inside a single-quoted HTML attribute with inner double quotes JSON-escaped (`\"`). Avoid `'` inside the JS.

The files use two different patterns:

- **`TOURMOBILE.json` (state-driven; tested with intro.js 8.5.0 on the mobile layout).** Step 1 defines `window.__nupsiTour(state)`. Every later step calls `window.__nupsiTour&&window.__nupsiTour("<state>")`, where the state is one of `"sidebar"`, `"burger"`, `"Layers"`, `"Statistics"`, `"Stories"`, `"Compare"` or `"none"`. The function reads the real UI state (`aria-pressed` on the hamburger and toolbar buttons, `.open` on `.menu-sidebar`), closes what isn't wanted and opens what's missing, then sends a synthetic `resize` after 350 ms so intro.js moves the highlight. That makes Prev and jumping between steps safe. Step 1 also registers, once (guarded by `window.__nupsiTourExit`), capture-phase listeners for Skip, Done, an overlay click and Esc that call `__nupsiTour("none")`. It uses `src='data:,'`, which triggers `onerror` without a 404 request. Limitations: the function only exists if the tour starts at step 1, and disabled toolbar buttons (e.g. Statistics/Compare for products without them) are highlighted but not opened. Step 17 (NUTS) has no `element`, because `.footer .vintage-badge` is always hidden on mobile. **Viewport margin:** step 1 also installs, once (`window.__nupsiGap`), a minimum distance `G=16` px between the tooltip and every viewport edge. It injects `<style id="nupsi-tour-gap">`, which caps the tooltip's `max-width`/`min-width` at `100vw - 2G`. A `MutationObserver` on `introjs*` style/class changes, together with `resize`/`transitionend`, re-runs `fit()`, which shifts `.introjs-tooltip` through the CSS `translate` property (not `transform`, so it doesn't conflict with intro.js) and subtracts its own previous offset before measuring. The observer disconnects when the tooltip disappears. To change the threshold, edit only `G`.
- **`TOURDESKTOP.json` / `TOURTABLET.json` (legacy).** These use direct, imperative hooks: `src='x'`, with `dataset.hooked*` listeners attached to `.introjs-nextbutton` on steps 4 and 11 and blind `.click()` calls on steps 13–16. They have known bugs that the mobile file fixes: the listeners stay attached and toggle the menu on every Next, Prev breaks the panel state, step 16 opens Stories instead of Compare, the month arrow uses the fragile `.month-nav:nth-child(3)` (mobile uses `[aria-label="Next month"]`), and step 18 is titled "Deselct". To fix them, port the `__nupsiTour` pattern rather than patching the hooks. Step 17's `.footer .vintage-badge` anchor is valid on desktop.

## Git workflow

- `main` is what production reads. Release branches are `nupsi-1.1.0` and `nupsi-1.2.0`, both also on `origin`.
- Commits are mostly small single-file edits made through the GitHub web UI (e.g. "Update TOUR.json"). Active work is on `nupsi-1.2.0`; local `main` is behind `origin/main`. Pull before editing, because collaborators push here directly.
