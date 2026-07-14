# Car Zone

Single-file, zero-dependency redesign concepts for **Car Zone**, a used-car wholesale dealership in
Birmingham, AL ([carzonellc.com](https://carzonellc.com)). Built to pitch a redesign to the dealership owner
before any production build is commissioned.

This is **not** wired to a backend, and the vehicle inventory shown is sample/placeholder data modeled on the
dealer's actual current listings — not live data.

## Two concepts, same repo

- **`index.html`** — the first redesign pass: a clean, Bootstrap-flavored take on the navy/gold brand
  (sidebar filters, rich vehicle cards, newsletter band).
- **`index-v2.html`** — a from-scratch second pass on top of the same data/routing engine, built around a
  "wholesale hang-tag" visual identity (die-cut price tags, mono/ticket typography, a navy "why buy here"
  band) instead of a generic card-and-badge layout. See `CLAUDE.md` for the full design rationale.

Both are complete, independently viewable sites kept side by side on purpose so they can be compared before
picking a direction with the client.

## Running it

No build step required. Either:

- Open `index.html` or `index-v2.html` directly in a browser, or
- Serve it locally (needed to test anything that behaves differently under `file://`):

  ```bash
  python3 -m http.server 8765
  ```

  then visit `http://localhost:8765/index.html` or `http://localhost:8765/index-v2.html`.

There's no lint or test command — this project has no build system, package manager, or dependency list.

## What's in the box

Both files are self-contained: a client-side, hash-routed single-page app (home/inventory, vehicle detail,
car finder, contact, directions, disclaimer) written as plain string-concatenated HTML with no templating
library or framework, plus a vendored copy of Bootstrap 5.3.3 (CSS + JS) for layout/components — inlined
directly in each file rather than loaded from a CDN, so the page still makes zero external network requests.
`index-v2.html` shares this same underlying app (data, routing, filters, forms) with `index.html` and only
diverges in visual design — see "Two concepts, same repo" above.

Car Zone's own navy/gold brand styling (sampled from the dealership's real signage) is layered on top of
Bootstrap and takes precedence for any shared component. See `CLAUDE.md` for a fuller architecture writeup
if you're working on this codebase with an AI coding assistant.

The inventory section uses a persistent left-hand filter sidebar (Make/Price/Transmission accordions plus a
search box) next to the results grid. Vehicle cards show a 2×2 icon spec grid (year/transmission/fuel/mileage)
and a "Deal of the Week" ribbon on discounted listings. A newsletter signup strip (with a diagonal clipped
transition) sits above the footer, and a small sticky quick-contact rail (call/directions) floats on the page
edge throughout. Sections and cards fade/slide in on load or first scroll into view via a small plain-CSS +
`IntersectionObserver` utility (no animation library).

`.claude/skills/` has a few installed Claude Skills (design-rules database, Anthropic's frontend-design
guidance, GSAP reference docs) used to inform this design work — see `CLAUDE.md` for what they are and why
GSAP itself wasn't added as an actual dependency.

## Constraints

- Zero external network requests — no CDN fonts, no map embeds, no analytics, no CDN-hosted Bootstrap.
- The navy/gold color palette matches the dealership's real brand identity and shouldn't be swapped out
  without checking with the client first.
