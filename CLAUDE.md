# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single self-contained HTML file (`index.html`) that is a redesign concept/demo for **Car Zone**, a real
used-car wholesale dealership in Birmingham, AL (live site: carzonellc.com). It exists to be pitched to the
dealership owner before any real production build is commissioned — it is not wired to a backend, and its
vehicle inventory is sample/placeholder data modeled on the dealer's actual current listings, not live data.

There is no build system, package manager, dependency list, or test suite in this repo — just the one HTML
file. Do not introduce a bundler, framework, or npm dependencies unless the user explicitly asks for a real
production build; keep changes inside the single-file, zero-dependency structure unless told otherwise.

## Running it

No build step. Either:
- Open `index.html` directly in a browser, or
- Serve it locally (needed if testing anything that behaves differently under `file://`):
  ```
  python3 -m http.server 8765
  ```
  then visit `http://localhost:8765/index.html`.

There is no lint or test command configured for this project.

## Architecture (all inside `index.html`)

The file is one document with an inline `<style>` block, an inline `<svg>` icon-symbol definition, an empty
`<div id="app">` mount point, and an inline `<script>` containing a single IIFE that renders a client-side,
hash-routed single-page app. There is no templating library — every view is built via plain string
concatenation and injected with `innerHTML`.

**Data layer**: a `RAW` array of vehicle objects (stock #, year, make, model, price, mileage, status) is run
through `enrich()`, which deterministically derives display-only fields (paint color, VIN, engine,
transmission, options list, description) via `seedPick()` (index-seeded picks from fixed pools) so the same
vehicle always renders the same derived data. The result is `VEHICLES`; `MAKES` is the derived sorted list of
distinct makes used to populate filters/chips.

**Routing**: `routes` maps hash paths (`""`, `car-finder`, `contact`, `directions`, `disclaimer`) to
`render*()` functions; `vehicle/:id` is matched separately by regex and handled by `renderVehicleDetail()`.
`route()` runs on `hashchange`/`DOMContentLoaded`, rebuilds `#app` via `layout(inner, ...)` (which wraps the
view in the shared topbar/header/footer), then calls the matching `bind*()` function to attach event
listeners for that view (`bindHome`, `bindVehicleDetail`, `bindForm`, plus `bindGlobal` for the mobile nav
toggle on every route).

**Inventory filtering**: filter/sort state lives in the module-level `filterState` object. Changing a filter
control calls `renderGrid()`, which re-renders only `#vehicle-grid` and `#result-count` in place — it does not
go through the router — so filtering/sorting stays fast and doesn't reset scroll position or other page state.
The hero "Quick Search" card writes into the same `filterState` and main filter `<select>` elements before
calling `renderGrid()`, so the two search entry points stay in sync.

**Forms** (Contact, Car Finder): fully client-side. `bindForm()` intercepts `submit`, prevents the default,
and renders a canned confirmation message — there is no backend, no network call, and no persistence.

**Styling/theming**: colors are CSS custom properties on `:root`, overridden under
`@media (prefers-color-scheme: dark)` and via `[data-theme="dark"]`/`[data-theme="light"]` for manual
overrides. The navy/gold palette (`--navy`, `--accent`, etc.) was sampled directly from the real dealership's
live banner/logo image to match their actual brand identity — don't replace it with an arbitrary palette
without checking with the user first, since matching the existing brand was an explicit requirement.

**Icons**: all hand-authored inline SVG (phone/pin/clock/mail/social icons, plus a single reusable car glyph
`<symbol id="car-icon">` referenced via `<use>`) — there is no icon font or external asset dependency.

## Constraints worth preserving

- Zero external network requests (no CDN fonts, no map embeds, no analytics) — everything is inlined.
- All user-facing text/data that isn't pre-escaped goes through `esc()` before insertion into HTML strings.
- Non-ASCII characters (dashes, middle dots, emoji, etc.) inside JS string literals are written as `\uXXXX`
  escapes rather than literal characters, to avoid mojibake if the file's charset is ever misdetected. Keep
  using escapes for any new special characters added to strings, rather than pasting literal Unicode glyphs.
