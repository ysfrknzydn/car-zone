# Car Zone

A single-file, zero-dependency redesign concept for **Car Zone**, a used-car wholesale dealership in
Birmingham, AL ([carzonellc.com](https://carzonellc.com)). Built to pitch a redesign to the dealership owner
before any production build is commissioned.

This is **not** wired to a backend, and the vehicle inventory shown is sample/placeholder data modeled on the
dealer's actual current listings — not live data.

## Running it

No build step required. Either:

- Open `index.html` directly in a browser, or
- Serve it locally (needed to test anything that behaves differently under `file://`):

  ```bash
  python3 -m http.server 8765
  ```

  then visit `http://localhost:8765/index.html`.

There's no lint or test command — this project has no build system, package manager, or dependency list.

## What's in the box

Everything lives in `index.html`: a client-side, hash-routed single-page app (home/inventory, vehicle detail,
car finder, contact, directions, disclaimer) written as plain string-concatenated HTML with no templating
library or framework, plus a vendored copy of Bootstrap 5.3.3 (CSS + JS) for layout/components — inlined
directly in the file rather than loaded from a CDN, so the page still makes zero external network requests.

Car Zone's own navy/gold brand styling (sampled from the dealership's real signage) is layered on top of
Bootstrap and takes precedence for any shared component. See `CLAUDE.md` for a fuller architecture writeup
if you're working on this codebase with an AI coding assistant.

### Design Lab

The site currently has a small floating "Design Lab" widget (bottom-left) for comparing layout variants of
the inventory section side by side before settling on one — this is a temporary pitch/comparison aid, not a
permanent feature of the final design.

## Constraints

- Zero external network requests — no CDN fonts, no map embeds, no analytics, no CDN-hosted Bootstrap.
- The navy/gold color palette matches the dealership's real brand identity and shouldn't be swapped out
  without checking with the client first.
