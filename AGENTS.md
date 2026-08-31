# cadastre-finder — Agent Guide

## Stack

- **Framework**: Svelte 5 (runes: `$state`, `$derived`, `$effect`) + Vite 7
- **Language**: JavaScript with JSDoc type checking (`checkJs: true` in `jsconfig.json`)
- **Package manager**: npm (lockfile v3)
- **Map**: Leaflet 1.9.4
- **Deploy**: GitHub Pages via `gh-pages`

## Commands

| Command | Action |
|---------|--------|
| `npm run dev` | Start Vite dev server |
| `npm run build` | Production build to `dist/` |
| `npm run preview` | Preview production build |
| `npm run deploy` | Deploy `dist/` to GitHub Pages |

Production base path is `/cadastre-finder/` (set in `vite.config.js`).

## Quality

No test framework, linter, or formatter is configured.

## Architecture

- **Entry**: `src/main.js` mounts `App.svelte` to `#app`
- **No routing** — single-page app, no SvelteKit
- **External APIs called from the browser** (no backend):
  - `geo.api.gouv.fr` — commune search
  - `api-adresse.data.gouv.fr` — reverse geocoding
  - `cadastre.s3.rbx.io.cloud.ovh.net` — parcel GeoJSON (gzipped). Use the S3 bucket directly, not `cadastre.data.gouv.fr`: the latter 302s here without an `Access-Control-Allow-Origin` header on the redirect, which the browser blocks.
  - `nominatim.openstreetmap.org` — reverse geocoding (in Map.svelte)
  - OpenStreetMap / ArcGIS tile layers (in Map.svelte)

## Svelte 5 conventions

The project uses Svelte 5's runes. However, the main `App.svelte` uses traditional `let` declarations for reactive state (still valid in Svelte 5 `.svelte` files). The `Counter.svelte` example uses `$state()` explicitly. Either style is acceptable.

## Deployment

`npm run deploy` runs `vite build` then pushes `dist/` to the `gh-pages` branch. No CI/CD is configured.
