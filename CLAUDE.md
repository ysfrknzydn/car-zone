# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Two self-contained HTML files, each a full redesign concept/demo for **Car Zone**, a real used-car wholesale
dealership in Birmingham, AL (live site: carzonellc.com). They exist to be pitched to the dealership owner
before any real production build is commissioned — neither is wired to a backend, and the vehicle inventory
in both is sample/placeholder data modeled on the dealer's actual current listings, not live data.

- **`index.html`** — the first redesign pass (Bootstrap-flavored, navy/gold, sidebar-filter inventory).
- **`index-v2.html`** — a second, from-scratch visual pass sharing the same data/routing engine as
  `index.html` but with a distinct "wholesale hang-tag" design system built around real dealer-lot signage
  motifs (die-cut price/deal tags, mono ticket typography, a navy full-bleed "why buy here" band) instead of
  `index.html`'s flatter card-and-badge look. It was built by copying `index.html` verbatim and then rewriting
  the `<style>` block and select markup in place — the `RAW`/`enrich()` vehicle data, routing, filter state,
  and form-binding logic are unchanged between the two files. Everything below in this doc describes
  `index.html`'s architecture; `index-v2.html` follows the same patterns except where "Vendored Bootstrap" or
  the CSS-specific sections note otherwise (`index-v2.html` adds a proper `<!DOCTYPE html>`/`<head>`/`<body>`
  wrapper, which `index.html` is missing).
- The two are kept **side by side on purpose** so they can be A/B'd with the client before picking one —
  don't merge, delete, or rename either without the user asking. If asked to work on "the redesign" without
  specifying which file, ask which one (or both) is meant.

There is no build system, package manager, dependency list, or test suite in this repo — just these two HTML
files. Bootstrap 5.3.3 (CSS + JS bundle) is vendored, inlined in full inside each file (see "Vendored
Bootstrap" below) — this was an explicit exception granted by the user, not a default to build on. Do not
introduce anything beyond that (a bundler, another framework, npm dependencies) unless the user explicitly
asks; keep changes inside the single-file structure unless told otherwise.

## Running it

No build step. Either:
- Open `index.html` or `index-v2.html` directly in a browser, or
- Serve it locally (needed if testing anything that behaves differently under `file://`):
  ```
  python3 -m http.server 8765
  ```
  then visit `http://localhost:8765/index.html` or `http://localhost:8765/index-v2.html`.

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

**Data layer**: a `RAW` array of vehicle objects (stock #, year, make, model, price, mileage, status, and
occasionally an explicit `msrp` for the handful of vehicles marked as discounted) is run through `enrich()`,
which deterministically derives display-only fields (paint color, VIN, engine, transmission, fuel type,
options list, description) via `seedPick()` (index-seeded picks from fixed pools) so the same vehicle always
renders the same derived data. The result is `VEHICLES`; `MAKES` is the derived sorted list of distinct makes
used to populate the filter sidebar.

**Routing**: `routes` maps hash paths (`""`, `car-finder`, `contact`, `directions`, `disclaimer`) to
`render*()` functions; `vehicle/:id` is matched separately by regex and handled by `renderVehicleDetail()`.
`route()` runs on `hashchange`/`DOMContentLoaded`, rebuilds `#app` via `layout(inner, ...)` (which wraps the
view in the shared topbar/header/footer/newsletter-band/quick-contact-rail — see below), then calls the
matching `bind*()` function to attach event listeners for that view (`bindHome`, `bindVehicleDetail`,
`bindForm`, plus `bindNewsletter()` on every route). The old manual mobile-nav-toggle JS was removed in favor
of Bootstrap's `data-bs-toggle="collapse"` (delegated on `document`, so it works fine against the
innerHTML-replaced DOM without any rebinding).

**Inventory filtering**: the inventory section (`renderInventorySection()`) is a persistent left-hand filter
sidebar (`renderFilterSidebar()` — accordion sections for Make/Price/Transmission, plus a search box) next to
the results grid, with a "Sort By" dropdown pinned above the grid. Filter/sort/search state lives in the
module-level `filterState` object (`make`, `price`, `trans`, `sort`, `q`). Changing any filter control calls
`renderGrid()`, which re-renders only `#vehicle-grid` and `#result-count` in place — it does not go through
the router — so filtering/sorting stays fast and doesn't reset scroll position or other page state.
`syncFilterControls()` is the single place that pushes `filterState` back out to whichever controls exist in
the DOM (the sidebar radios/search box, the sort dropdown, or the hero quick-search fields) — always route
state changes through `filterState` + `syncFilterControls()` + `renderGrid()` rather than writing to specific
control elements directly. The hero "Quick Search" card writes into the same `filterState` before calling
`renderGrid()`, so all filter entry points stay in sync.

**Vehicle cards**: `vehicleCard()` renders each listing with a 2×2 icon+label spec grid (year / transmission /
fuel / mileage, built by `specGrid()` using the hand-authored `iconCalendar`/`iconGear`/`iconFuel`/
`iconGauge` glyphs) instead of a single meta line. A vehicle gets a rotated "Deal of the Week" corner ribbon
plus a struck-through original price whenever its `msrp` (set explicitly in `RAW`) is higher than its current
`price`. The ribbon is wrapped in a `.ribbon-wrap` self-clipping overlay so the rotated banner doesn't get cut
off oddly by `.card`'s `overflow:hidden`.

**Footer / newsletter band**: `layout()` always inserts `renderNewsletterBand()` between `<main>` and
`footer()` — an email-capture strip with its top edge cut on a diagonal via `clip-path` on `.newsletter-band`,
giving a deliberate angled transition out of whatever section precedes it (most routes hit a light `--bg`
section there, so the cut reads clearly; on the home route it follows the navy `.cta-band`, so the cut is a
subtler navy-on-navy fold — worth a second look if the client wants more contrast there). Submission is
handled by `bindNewsletter()` (called from `route()`), which mirrors `bindForm()`'s fake-submit/confirm-box
pattern but keeps its own confirm element (`#newsletter-confirm`) since `bindForm()`'s confirm-element lookup
is hardcoded to `contact-confirm`/`finder-confirm`.

**Sticky quick-contact rail**: `renderQuickRail()`, also always rendered by `layout()`, is a small
`position:fixed` icon strip pinned to the right edge of the viewport (vertically centered) with a call
shortcut (`tel:` link) and a directions shortcut (`#/directions`). No JS binding needed — plain links.

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
Bootstrap/Design-Lab-era work for how that line was drawn in practice.

**Icons**: all hand-authored inline SVG (phone/pin/clock/mail/social icons, a small set of spec-grid icons —
calendar/gear/fuel/gauge — plus a single reusable car glyph `<symbol id="car-icon">` referenced via `<use>`)
— there is no icon font or external asset dependency.

**Scroll/load-reveal animation**: a small `.reveal` utility (CSS `opacity`/`transform` transition + a single
shared `IntersectionObserver` in `initReveal()`) fades/slides elements in once, either on initial load (for
above-the-fold elements, since `observe()` fires immediately if the target is already in the viewport) or on
first scroll into view. `initReveal(app)` runs after every `route()` re-render and `initReveal(grid)` runs
after every `renderGrid()` re-render (so re-filtering the inventory replays the reveal on the fresh cards).
Stagger within a group is done with an inline `--i` custom property consumed by
`transition-delay:calc(var(--i, 0) * 60ms)` — capped (see `vehicleCard()`'s `Math.min(i, 7)`) so long lists
don't get a sluggish tail. Fully respects `prefers-reduced-motion: reduce` (disables the transition and shows
content immediately). This is deliberately plain CSS + vanilla JS, not a library — see "AI design/animation
skills" below for why GSAP was considered and not vendored.

## `index-v2.html`: the hang-tag visual system

`index-v2.html` reuses `index.html`'s JS almost verbatim (data layer, routing, filter state, forms are
byte-for-byte the same logic, same class-name vocabulary for most elements) but replaces the entire `<style>`
block and a handful of markup blocks to express a different design language. If you're touching `index-v2.html`,
know these specifics rather than assuming it's identical to `index.html`:

- **Tokens**: warm paper background (`--bg:#f6f2e8`) and warm ink (`--ink:#132038`) instead of `index.html`'s
  cool gray, plus a `--radius-sm` token — most surfaces are sharp-edged (0 radius) by design, with rounding
  reserved for inputs/buttons only. Locked navy/gold values (`--navy`, `--accent`, `--accent-ink`) are
  unchanged from `index.html` — still sampled from the real dealership brand, still don't touch without asking.
- **`.hang-tag`**: the signature component — a die-cut price-tag shape (`clip-path` notch + a punched-hole
  `::before` + a hanging-string `::after`, all pure CSS, no images). Applied via `class="quick-search hang-tag"`
  on the hero's quick-search card. The vehicle-card `.price-tag` and vehicle-detail `.price-block` borrow the
  same visual language (navy gradient, clip-path notch, punch-hole dot) but are separate CSS rules, not the
  `.hang-tag` class itself — they needed different geometry, not a shared selector.
- **Sharp edge**: `.price-tag` (top-right on the media area) and `.ribbon-wrap`/`.ribbon-deal` ("Deal of the
  Week" banner, also top-right) will visually collide if you move either one without checking the other —
  `.price-tag` is deliberately pinned to `bottom-right` of `.card-media` for exactly this reason. If you add
  another badge to the media area, check it against both.
- **Sharp edge**: elements inside `.why-band` (the full-bleed navy "why buy here" section) don't inherit a
  white text color from the section the way `.newsletter-band`/`footer.site` do (those set `color:#fff` on the
  section itself) — `.why-band` only sets it per-child (`.section-head h2`, `.section-sub`, `.why-col p`). Any
  new element added inside `.why-band` needs its own explicit light color or it'll render as unreadable dark
  text on navy.
- **New sections not present in `index.html`**: `.tag-strip`/`.tag-chip` (the "Family-Run · Wholesale Direct ·
  Birmingham, AL" badges under the hero copy) and `.why-band`/`.why-grid`/`.why-col` (a 3-column navy band
  replacing `index.html`'s emoji-icon `.trust-item` grid — emoji were dropped project-wide in favor of the
  mono/eyebrow typographic system).
- **Document structure**: `index-v2.html` has a proper `<!DOCTYPE html><html lang="en"><head>…</head><body>…
  </body></html>` wrapper with `charset`/`viewport` meta tags. `index.html` is missing all of this (relies on
  browser HTML5-parser auto-insertion) — a pre-existing gap in the first file that wasn't fixed there since
  that wasn't in scope for a visual-only pass; don't assume the two files' document shells match.

## AI design/animation skills

`.claude/skills/` has three third-party Claude Skills installed to inform design/UX/motion work on this repo:
`ui-ux-pro-max` (searchable design-rules database — run its `scripts/search.py`, e.g. `--design-system` for a
full recommendation or `--domain ux`/`--domain gsap`/etc. for a focused query), `frontend-design` (Anthropic's
official guidance for distinctive, non-templated visual design), and six `gsap-*` skills (official GreenSock
guidance for core tweens, timelines, ScrollTrigger, utils, performance, and plugins). All three were vetted
before installing (checked for network calls/`eval`/`subprocess` in any scripts, checked publisher legitimacy)
since skill content directly steers AI behavior.

`ui-ux-pro-max`'s own recommendation engine defaults to its own color/typography suggestions — **don't apply
those to this project**; the navy/gold palette is locked (see "Styling/theming" above). Use it for the
non-palette guidance instead: UX/accessibility checks, motion timing/easing, layout patterns.

The `gsap-*` skills are installed for reference/future use, but GSAP itself (the actual JS library) was
deliberately **not** vendored into `index.html` — that would add a new ~117KB third-party runtime dependency,
which is a bigger call than "install some guidance skills" and this site's actual animation needs (simple
scroll reveals, no scroll-scrubbing/pinning/timeline choreography) don't call for it. If a future need
justifies it, vendor `gsap.min.js`/`ScrollTrigger.min.js` from the official npm `gsap` package the same way
Bootstrap is vendored (verify the tarball integrity hash against npm's registry first) — don't pull from a
CDN, since that would violate the zero-external-request constraint below.

`shadcn` (from shadcn-ui/ui) and the `bergside/awesome-design-skills` list were evaluated but not installed:
`shadcn`'s every command assumes a Node/npm project with `components.json`/React/Tailwind, which this
single-file static-HTML project fundamentally isn't; the awesome-design-skills list requires running an
unvetted third-party puller (`npx typeui.sh pull <name>`) per entry and its 67 skills are prepackaged
copycat-brand visual styles, which clashes with this project's locked, real-brand palette anyway.

## Constraints worth preserving

- Zero external network requests (no CDN fonts, no map embeds, no analytics, no CDN-hosted Bootstrap) —
  everything is inlined, including the vendored Bootstrap build.
- All user-facing text/data that isn't pre-escaped goes through `esc()` before insertion into HTML strings.
- Non-ASCII characters (dashes, middle dots, emoji, etc.) inside JS string literals are written as `\uXXXX`
  escapes rather than literal characters, to avoid mojibake if the file's charset is ever misdetected. Keep
  using escapes for any new special characters added to strings, rather than pasting literal Unicode glyphs.
