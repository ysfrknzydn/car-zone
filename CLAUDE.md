# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single self-contained HTML file (`index.html`) that is a redesign concept/demo for **Car Zone**, a real
used-car wholesale dealership in Birmingham, AL (live site: carzonellc.com). It exists to be pitched to the
dealership owner before any real production build is commissioned — it is not wired to a backend, and its
vehicle inventory is sample/placeholder data modeled on the dealer's actual current listings, not live data.

There is no build system, package manager, dependency list, or test suite in this repo — just the one HTML
file. Bootstrap 5.3.3 (CSS + JS bundle) is vendored, inlined in full inside `index.html` itself (see
"Vendored Bootstrap" below) — this was an explicit exception granted by the user, not a default to build on.
Do not introduce anything beyond that (a bundler, another framework, npm dependencies) unless the user
explicitly asks; keep changes inside the single-file structure unless told otherwise.

## Running it

No build step. Either:
- Open `index.html` directly in a browser, or
- Serve it locally (needed if testing anything that behaves differently under `file://`):
  ```
  python3 -m http.server 8765
  ```
  then visit `http://localhost:8765/index.html`.

There is no lint or test command configured for this project. To sanity-check a change, load the page in a
real browser (or drive headless Chromium via Playwright) and click through the routes/filters/forms you
touched — see the project's `verify`/`run` skills for the pattern used in this repo (screenshot-driven,
since rendering bugs here don't show up in the markup source).

## Architecture (all inside `index.html`)

The file is one document with a vendored Bootstrap `<style>` block, the site's own inline `<style>` block,
an inline `<svg>` icon-symbol definition, an empty `<div id="app">` mount point, a vendored Bootstrap
`<script>` bundle, and the site's own inline `<script>` containing a single IIFE that renders a client-side,
hash-routed single-page app. There is no templating library — every view is built via plain string
concatenation and injected with `innerHTML`.

**Vendored Bootstrap**: `bootstrap.min.css` and `bootstrap.bundle.min.js` (5.3.3) are pasted verbatim into
their own `<style id="bootstrap-css">`/`<script id="bootstrap-js">` blocks near the top of the file, ahead of
Car Zone's own style/script blocks, specifically so the site's own rules win the cascade for any selector
they both touch. This keeps the "zero external network requests" constraint intact while still getting real
Bootstrap components (`navbar`/`collapse`, the `row`/`col`/`row-cols-*` grid, `nav-tabs`/`tab-content`,
`form-control`/`form-select`, `.btn`/`.card` base styles). Car Zone's own navy/gold theme is layered on top:
Bootstrap's CSS custom properties (e.g. `--bs-btn-bg`) aren't generally overridden; instead Car Zone's plain
CSS rules for the same class names (which come later in the file) win on specificity/source order. Watch for
one sharp edge: Bootstrap sometimes defines a **higher-specificity** rule for a combined state (e.g.
`.navbar-nav .nav-link.active`) than a same-named single-purpose override elsewhere in the custom stylesheet —
if a themed element looks right unstyled/hovered but wrong in its "active" state, this is the first thing to
check. Don't re-derive Bootstrap's CSS/JS from scratch or fetch it from a CDN if it's ever missing/truncated —
regenerate by re-vendoring the same version.

**Data layer**: a `RAW` array of vehicle objects (stock #, year, make, model, price, mileage, status) is run
through `enrich()`, which deterministically derives display-only fields (paint color, VIN, engine,
transmission, options list, description) via `seedPick()` (index-seeded picks from fixed pools) so the same
vehicle always renders the same derived data. The result is `VEHICLES`; `MAKES` is the derived sorted list of
distinct makes used to populate filters/chips.

**Routing**: `routes` maps hash paths (`""`, `car-finder`, `contact`, `directions`, `disclaimer`) to
`render*()` functions; `vehicle/:id` is matched separately by regex and handled by `renderVehicleDetail()`.
`route()` runs on `hashchange`/`DOMContentLoaded`, rebuilds `#app` via `layout(inner, ...)` (which wraps the
view in the shared topbar/header/footer and appends the Design Lab widget — see below), then calls the
matching `bind*()` function to attach event listeners for that view (`bindHome`, `bindVehicleDetail`,
`bindForm`, plus `bindDesignLab()` on every route). The old manual mobile-nav-toggle JS was removed in favor
of Bootstrap's `data-bs-toggle="collapse"` (delegated on `document`, so it works fine against the
innerHTML-replaced DOM without any rebinding).

**Design Lab (layout-variant comparison tool)**: `uiVariants` is a small state object (persisted to
`localStorage` under `cz-ui-variants` via `loadUiVariants()`/`saveUiVariants()`) that lets a section of the
site render as one of several interchangeable layout variants, toggled live from a floating widget
(`renderDesignLab()`/`bindDesignLab()`, bottom-left on every page) without needing a code change or reload.
This exists because the site is being used to A/B a from-scratch redesign of specific sections against
reference UX (currently: inventory filters, `classic` vs `sidebar` — see `renderInventorySection()` and
`renderFilterSidebar()`) before committing to one. Each variant key gets its own render branch but **shares
the same underlying state/logic** — e.g. both filter layouts read/write the same `filterState` object and
call the same `filteredVehicles()`/`renderGrid()`, so a filter chosen in one variant is still applied if you
switch to the other. When adding a new variant key: add a default to `DEFAULT_UI_VARIANTS`, add a toggle row
to `renderDesignLab()`, branch the relevant `render*()` function on `uiVariants.<key>`, and make sure
`bind*()` binds controls defensively (`if (el) el.onchange = ...`) since only one variant's controls exist in
the DOM at a time. This widget is a temporary pitch/comparison aid, not a permanent feature — once a variant
is chosen, the losing branch, its Design Lab row, and any now-dead bound elements should be deleted rather
than kept behind the toggle indefinitely.

**Inventory filtering**: filter/sort/search state lives in the module-level `filterState` object (`make`,
`price`, `trans`, `sort`, `q`). Changing any filter control calls `renderGrid()`, which re-renders only
`#vehicle-grid` and `#result-count` in place — it does not go through the router — so filtering/sorting stays
fast and doesn't reset scroll position or other page state. `syncFilterControls()` is the single place that
pushes `filterState` back out to whichever controls currently exist in the DOM (classic toolbar, sidebar
radios, or the hero quick-search fields) — always route state changes through `filterState` +
`syncFilterControls()` + `renderGrid()` rather than writing to specific control elements directly, so classic
and sidebar variants (and any future ones) stay in sync automatically. The hero "Quick Search" card writes
into the same `filterState` before calling `renderGrid()`, so all filter entry points stay in sync.

**Forms** (Contact, Car Finder): fully client-side, styled with Bootstrap's `form-control`/`form-select`
(themed to Car Zone's CSS variables so they still respect light/dark mode — see the `.form-control`/
`.form-select` rules). `bindForm()` intercepts `submit`, prevents the default, and renders a canned
confirmation message — there is no backend, no network call, and no persistence.

**Styling/theming**: colors are CSS custom properties on `:root`, overridden under
`@media (prefers-color-scheme: dark)` and via `[data-theme="dark"]`/`[data-theme="light"]` for manual
overrides. The navy/gold palette (`--navy`, `--accent`, etc.) was sampled directly from the real dealership's
live banner/logo image to match their actual brand identity — don't replace it with an arbitrary palette
without checking with the user first, since matching the existing brand was an explicit requirement. This
also means external design references (e.g. another dealer's site) should only ever be mined for
layout/UX/structural patterns, not for their color/brand treatment — see the git history around the
Bootstrap/Design Lab work for how that line was drawn in practice.

**Icons**: all hand-authored inline SVG (phone/pin/clock/mail/social icons, plus a single reusable car glyph
`<symbol id="car-icon">` referenced via `<use>`) — there is no icon font or external asset dependency.

## Constraints worth preserving

- Zero external network requests (no CDN fonts, no map embeds, no analytics, no CDN-hosted Bootstrap) —
  everything is inlined, including the vendored Bootstrap build.
- All user-facing text/data that isn't pre-escaped goes through `esc()` before insertion into HTML strings.
- Non-ASCII characters (dashes, middle dots, emoji, etc.) inside JS string literals are written as `\uXXXX`
  escapes rather than literal characters, to avoid mojibake if the file's charset is ever misdetected. Keep
  using escapes for any new special characters added to strings, rather than pasting literal Unicode glyphs.
