# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An interactive atlas of 34 Indian bridges and monuments. The whole site is one hand-written file, `index.html` (about 2,700 lines), with inline CSS and vanilla JS. It uses no libraries and no images: every scene, building, vehicle and person is SVG generated at runtime. The only external resource is Google Fonts, and the page still works without it. `README.md` is the user-facing documentation, and `screenshots/` holds the images it references.

## Running and checking changes

There is no build, lint, test suite or package manager. Open `index.html` in a browser. To jump to one structure, add its `id` as a hash, for example `#taj`, `#chenab` or `#pamban`.

The site is deployed with GitHub Pages from the root of `main` (remote `kuntal-hub/Bharat_in_Elevation`) and is live at https://bharat-in-elevation.kuntalmaity.in/. Pushing to `main` redeploys it, so keep the page at `index.html` in the repository root. The root `CNAME` file sets the custom domain; deleting it takes the site off that domain.

Check changes by eye in the browser. The layout switches at these widths, so test both sides of each:
- Bridge charts go narrow below 560 px of chart width.
- Monument charts go narrow below 480 px.
- The Heights plate starts as bars below 640 px.
- The timeline becomes a vertical list below 700 px.

Also check all three skies (Day, Sunset, Night) and both OS light and dark mode.

## Navigating the file

The file is too large to read in one pass. The script is split by `/* ================= name ================= */` banners: utilities, tooltip, bridge scene renderer, monument elevation renderer, cards, bridges, monuments, comparison plates, boot. Grep for a banner or a structure constant (`const HOWRAH =`, `const TAJ =`) and read that range with an offset.

Layout of the file: `<style>` (design tokens on `:root`, a dark-mode override, and scene/layer classes), then static HTML for the masthead, sticky bar and plate sections, then a hidden `<svg><defs>` sprite sheet (`#v-car`, `#v-bus`, `#v-train`, `#v-walker`, `#v-boat` and others, used with `<use href>`), then a single IIFE `<script>`.

## Rendering model

- **Data-driven.** Each structure is one definition object. Bridges are built in IIFEs (`const HOWRAH = (() => { ...; return {...}; })()`) so they can keep private geometry helpers. Monuments are plain objects. They are registered in the `BRIDGES` and `MONUMENTS` arrays, and `ALL` combines them. The card HTML, map pins, map list, picker, Heights, Spans and Timeline plates are all generated from these arrays.
- **Strings, not DOM.** Renderers build SVG markup as template strings and assign it with `innerHTML`. Nothing is updated in place. A card is fully re-rendered when its width changes (`ResizeObserver`), when the sky changes (`setSky` redraws every visible card), when full screen toggles, and while the overview brush is dragged.
- **Coordinates are real metres.** `renderBridge` passes a viewport `V` with `V.X(m)` and `V.Y(m)`, which map metres to pixels and can scale x and y differently. The `BU` helpers (`pier`, `band`, `pylon`, `fan`, `rect`, `L`) take metres and cull anything outside `V.x0..V.x1`. `renderMonu` passes `P = makeP(...)`, which uses one uniform scale. Its helpers (`rect`, `poly`, `arch`, `onion`, `halfEll`, `ell`, `cypress`, and `path`, which takes a flat array of SVG commands) also take metres, with the origin at ground level on the centre line. The vertical-exaggeration caption under each bridge is computed from `sy / sx`.
- **Colour pipeline (Safari-sensitive).** Generated markup uses `var(--token)`. Before it is injected, `skyColours` (bridges) or `sceneColours` (monuments) replaces tokens with concrete values from `SKIES[SKY.cur]`; monuments also use `MAT`. Then `fixVars` moves any remaining `var()` out of presentation attributes into `style`, because Safari ignores `var()` in SVG attributes. When you add a scene colour, add the key to all three entries of `SKIES` (`sunset`, `day`, `night`) or to `MAT`. Run any new SVG string that still contains `var()` through `fixVars`.
- **Layer toggles.** A legend item with `layer: 'x'` toggles the class `off-x` on the card, and CSS hides `.lyr-x` inside it. The CSS currently handles `line`, `dims`, `traffic`, `grid`, `person` and `core`. A new layer needs a matching rule in `<style>`.
- **Animation.** Vehicles move with SMIL `animateMotion`; sky, water, birds and clouds use CSS keyframes. An `IntersectionObserver` pauses cards that are off screen (the `.paused` class plus `svg.pauseAnimations()`). `REDUCE` (prefers-reduced-motion) turns most of the motion off.

## Structure definition fields

**Shared:** `id` (also the card's DOM id and hash link), `kind` (`'bridge'` or `'monument'`), `name`, `short`, `alt`, `type`, `year` (a number, negative for BCE, used on the timeline), `yearLabel`, `place`, `lat`/`lon` (map pin), `lede`, `stats` (`[[label, value], ...]`), `legend`, `note`, and optionally `action`.

**Bridges:**
- Geometry and view: `len`, `minX`/`maxX`, `win: [x0, x1]` (the visible window in metres), `ymin`/`ymax`.
- Drawing: `draw(V)`.
- Profile and readout: `line(x)` (the yellow profile line; return `null` to leave a gap), `deck(x)`, and `readout(x)` (tooltip HTML).
- Annotations: `sections` (a bracket array, or a function of `V`) and `marks`.
- Traffic and lights: `lanes` and `lamps`.
- Terrain: `ground(x)`, `water: false`, `rocky`, and the scene flags `mountains`, `hills` and `city`.
- Comparison plates: `metric: {span, total, spanNote}` feeds the Spans plate. `sky: {h, what, glyph}` feeds the Heights plate, where `glyph` is `'pylon'`, `'lean'`, `'cant'` or `'arch'`; a bridge without `sky` is left out of Heights.
- Long bridges: setting `segs` + `segKey` + `ends` + `ovMax` turns on the draggable overview strip.
- Action: `action: {label, on}` toggles a `.raised` class on the card (this is the Pamban lift span).

**Monuments:**
- Geometry: `height`, `bw` (half the base width), `hw` (half the drawing width), `ymin`/`ymax`.
- Drawing: `draw(P)`.
- Parts: `parts: {key: {n, d, y: [lo, hi]}}`.
- Dimensions: `dims(D, P)`, using `D.v`, `D.h` and `D.note` from `dimHelpers`.
- Legend: `monuLegend([[material, label], ...], extra)`.
- Material argument to `P.*` helpers: a material is a token name (a `MAT` key). Append `*` to add cylindrical shading. Use `'void'` for openings and `'none'` for no fill.
- Part highlighting: the third argument to a `P.*` helper is a part key. It becomes `data-p`, which drives the hover highlight and the `parts` tooltip. When the cursor isn't over a drawn element, the `parts[...].y` range is used as a fallback hit test.
- Scene settings are assigned separately with `Object.assign(X, { scene: {...} })` in a block just above `const MONUMENTS`:
  - `ground`: `lawn`, `paving`, `pool` or `sea`.
  - `back`: any of `trees`, `palms`, `casuarina`, `gopurams`, `bodhi`, `mountains`, `hills`, `city`.
  - `traffic` tuples: `[sprite, real length m, sprite px, speed m/s, count]`.
- Monuments that should span both card columns are listed in the `.wide` array next to that block.
- Optional extras: `panel: {html, init(card)}` adds a side interactive (Konark, Jantar Mantar, Lotus). `action: {label, momentary: ms}` is a one-shot button (the Gol Gumbaz clap). `echo(P)`.

## Adding or changing a structure

Copy the most similar existing definition, change the figures and add it to `BRIDGES` or `MONUMENTS`. Most of the page updates itself, but these hard-coded values do not:
- The `tally` line in the boot section ("2,275 years", "359 m highest deck", "21.8 km longest crossing").
- The plate ranges in the section headings ("Plates V–XIX", "Plates XX–XXXVIII").
- The "Twenty-three centuries" timeline heading and the `TBANDS` year bands.
- The count of 34 and the structure tables in `README.md`.

## Data conventions

These come from the README and the page footer:
- Figures come from published sources, mainly each structure's Wikipedia article, last checked October 2026.
- Prefix indicative values with `≈`.
- Where sources disagree, the chart's `note` or lede says which figure is used.
- Truss panels, cable counts and pier spacing are schematic.
- The map (`INDIA` outline, `MP` projection) is a stylised orientation outline, not an authoritative boundary.
