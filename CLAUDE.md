# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Personal static website for Mike Renoe ([mikerenoe.com](https://www.mikerenoe.com)). Plain HTML/CSS/JS — no framework, no bundler, no test suite. Deployed on Vercel via `vercel.json`: `cleanUrls` serves `whereirun.html` at `/whereirun`, and `buildCommand` (`npm run build`) compiles the SCSS on deploy. Custom domains (`mikerenoe.com`) are managed in the Vercel dashboard, not in config.

## Pages

- `index.html` — home page.
- `whereirun.html` — "Where I Run" interactive US map (also served at `/whereirun`).

Both pages share the same header/nav markup and link the same compiled stylesheet (`scss/index.css`). When changing nav or shared layout, edit both files.

## Styling workflow

Styles are authored in `scss/index.scss` and compiled to `scss/index.css` (compressed, no source map). Never hand-edit `scss/index.css` — edit the `.scss`, then:

- `npm run build` — one-off compile (also runs automatically on Vercel deploy).
- `npm run watch` — recompile on save during local work.
- `npm run dev` — serve the site locally (`npx serve`).

Commit both the `.scss` and the regenerated `.css`. The custom `simpel-medium` font is loaded via `@font-face` from `scss/simpel-medium.woff2`.

## Run map (whereirun.html)

The map is rendered client-side with Raphael.js. Two vendored scripts must load in order before the inline `<script>`:
1. `scripts/raphael.min.js` (vendor library)
2. `scripts/us-map-svg.js` — defines the global `usMap` object mapping each state abbreviation to its SVG path string.

All race data lives **inline at the bottom of `whereirun.html`** as arrays of state abbreviations by distance: `runMarathon`, `runHalf`, `run10k`, `run5k`, and `runCaz`. To record a new race, add the state's two-letter abbreviation to the appropriate array. `drawMap()` colors each state by the **highest** distance run there (marathon → half → 10k → 5k → caz → not-yet), so order of the if/else chain encodes that priority. Each distance has a matching `*Draw` style object controlling fill color.
