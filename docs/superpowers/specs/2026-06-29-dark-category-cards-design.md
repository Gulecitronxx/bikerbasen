# Populære kategorier: dark stat-card redesign

## Context

The user shared two screenshots from an NFT marketplace template: a dark
"Featured NFTs" row (rounded dark cards with image, title, item count,
and a floor-price/avg-price stat row, with left/right arrow nav) and a
separate light-themed hero layout with a tilted card grid. The user
confirmed the second screenshot is inspiration only, not a build request.
For the first, they want the dark-card stat-row style applied to the
homepage's existing "Populære kategorier" row.

This supersedes the previous "denser 4-column grid" spec
(`2026-06-28-denser-listing-grid-design.md`), which the user explicitly
dropped in favor of this work. That spec is left in place as a historical
record but will not be implemented.

Through a visual mockup comparison (real category data, two color
variants), the user picked the variant with a green accent on the price
stat (tying into the site's existing brand color) over a plain-white
variant matching the screenshot exactly.

## Goal

Redesign the "Populære kategorier" row on `index.html` from its current
static, flat-white card style into dark stat-cards: each card shows the
category's real photo, its name, a live count of listings in that
category, and a live average price — replacing the swipe-only scroll with
explicit left/right arrow buttons.

## Non-goals

- The "Nyeste annoncer" row and the "Alle annoncer" grid are unaffected —
  this only touches the category row.
- No change to `listings.html`, `favoritter.html`, or `mine-annoncer.html`.
- The previously-dropped denser-grid work is not part of this spec.
- The second (light/tilted-grid) screenshot is not being built — inspiration
  only, no concrete deliverable from it.
- No new backend/Supabase queries — stats are computed client-side from
  data already loaded into the existing `motorcycles` array (same data
  source the rest of the homepage already uses).

## Current state

The category row is static HTML today (not JS-rendered): a
`<div class="popular-categories scroll-row">` containing six
`<div class="category-card" onclick="filterCategory('Cruiser')">` elements
(one per type: Cruiser, Sport, Adventure, Naked Bike, Retro, Touring),
each with an embedded base64 photo and a text label overlay
(`.category-card-title`). `filterCategory(type)` navigates to
`listings.html?type=...`. The six base64 photos are large (combined
~1.7MB) but unchanged by this work — they're reused, not replaced.

Real counts/avg prices computed from the current `motorcycles` array (for
reference — these will be computed live in code, not hardcoded):
Cruiser (4 listings, ~83.000 kr avg), Adventure (3, ~95.000 kr),
Naked Bike (3, ~88.000 kr), Sport (3, ~61.000 kr), Retro (3, ~75.000 kr),
Touring (0 listings — confirmed by the user to still show, with
"0 annoncer" and "–" for price rather than being hidden).

## Design

### Card content (per category)

- Photo (existing embedded base64 image, unchanged), with a dark overlay
  gradient added so the white title text stays readable against it
  (similar to the existing `.category-card-title` gradient treatment, just
  extended to cover more of the image since the whole card is now dark).
- Category name (white, bold) + "{count} annoncer" subtitle (light gray).
- A divider line.
- A two-column stat row: left column "Annoncer" (white count), right
  column "Gns. pris" (average price, green accent color — reusing the
  site's brand green, `#1B4332`, as the accent rather than introducing an
  unrelated green).
- Categories with zero listings (Touring today) still render a card:
  count shows "0 annoncer" / "0", average price shows "–" instead of
  computing `NaN` or `0 kr`.
- Clicking a card still calls the existing `filterCategory(type)` —
  unchanged behavior.

### Section header

Add explicit ← / → circular arrow buttons next to the "Populære
kategorier" title (matching the screenshot's header layout), in addition
to the existing native swipe/drag scroll (not a replacement — both work).
Each arrow click scrolls the row container horizontally by one card's
width (using `scrollBy({ left: ±cardWidth, behavior: 'smooth' })`), so
repeated clicks step through the row one card at a time.

### From static markup to JS-rendered

Today's category section is plain HTML with no JS involved. To show *live*
counts/averages (the user's explicit choice over hardcoding fixed
numbers), it must become JS-rendered: a new `renderCategoryRow()` function
that:

1. Defines the six categories in a fixed order (Cruiser, Sport, Adventure,
   Naked Bike, Retro, Touring) — order unchanged from today's row.
2. For each category, filters the current `motorcycles` array by `type`,
   computes `count = matches.length` and
   `avgPrice = count > 0 ? Math.round(matches.reduce((s,m) => s+m.price, 0) / count) : null`.
3. Renders each category's existing embedded photo (looked up from a
   small `category → base64 string` map built once from today's existing
   markup, not re-fetched or re-encoded) plus the stat markup described
   above, into a container `<div id="categoryRow">`.
4. Is called once from `init()`, after `displayListings(motorcycles)` and
   `renderLatestRow()` — same timing as the existing latest-listings row,
   so it reflects the same merged (demo + Supabase) data set.

This mirrors the existing `renderLatestRow()` pattern already in the
codebase (added in the previous homepage-rows feature), so the codebase
ends up with one consistent technique for "rows computed from current
listings data" instead of a mix of static and dynamic rows.

### Visual reference

Dark card background approximately `#1a1d24` (matching the screenshot's
near-black navy), white text, `#1B4332`-derived green for the price accent
(e.g. a lighter/brighter shade like `#4ade80` for sufficient contrast on a
dark background — exact shade to be finalized during implementation by
checking contrast against the dark background, not by eye).

## Error handling / edge cases

- Zero-listing category (Touring today, or any category if demo/Supabase
  data changes): renders "0 annoncer" and "–" for price, never `NaN`,
  `undefined`, or `0 kr`.
- If `motorcycles` is empty entirely (e.g. Supabase down and demo data
  somehow missing — not expected, but consistent with how
  `renderLatestRow()` already degrades): all six categories show "0
  annoncer" / "–", same zero-listing handling, no crash.
- Arrow button clicks at the start/end of the row: `scrollBy` naturally
  clamps at the container's scroll boundaries — no special-casing needed,
  consistent with how native scroll already behaves.

## Testing / verification

Same ad-hoc Playwright approach used throughout this project (no test
framework in this repo):

- Confirm 6 category cards render with the dark style, correct category
  names, and computed counts/prices matching what the actual
  `motorcycles` array produces at the time of the check (recompute
  expected values from the live data in the test script itself, rather
  than hardcoding the numbers from this spec — they'll drift as the
  Supabase data changes).
- Confirm the Touring (or any zero-listing) category shows "0 annoncer" /
  "–", not `NaN`/`undefined`.
- Confirm clicking a category card still navigates to
  `listings.html?type=...` exactly as before.
- Confirm clicking the → arrow scrolls the row right, ← scrolls left.
- Confirm the existing "Nyeste annoncer" row and "Alle annoncer" grid are
  visually and functionally unaffected.
