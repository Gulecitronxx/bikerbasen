# Denser 4-column listing grid (Bilbasen-style density)

## Context

The user shared a screenshot of the real bilbasen.dk homepage and asked for
BikerBasen to be made "as cool" visually. Comparing the screenshot to the
current site surfaced 5 candidate differences: listing card style (price
badge overlay), a denser 4-column grid, a two-banner CTA section, header
icon/avatar polish, and a small blue section label above the grid.

Through brainstorming (including a visual mockup comparison), the user
picked the listing card style as their first priority, then explicitly
**kept the current card style** (3 stacked lines, no price badge — option A
in the mockup, confirmed by clicking it twice including after trying the
badge alternative). With the card style itself settled as "no change," the
user chose the denser grid as the next concrete item to build.

This spec covers only the denser grid. The other candidate differences
(CTA banners, header polish, section label) are deferred — not rejected,
just not in scope for this round.

## Goal

Make the listing grids on `index.html`, `listings.html`, and
`favoritter.html` show 4 cards per row at normal desktop width instead of
3, without changing each page's existing overall content width (confirmed
via visual mockup: user picked "Option A — keep width, shrink cards" over
"Option B — widen the page").

## Non-goals

- `mine-annoncer.html` is explicitly out of scope. It uses a horizontal
  row layout (thumbnail + info + action buttons in a flex row), not a card
  grid — structurally inappropriate for a "4-column grid" change, and the
  user confirmed leaving it as-is.
- No change to the card's internal content/style (title, details, price
  text, favorite button) — only the grid's column count and gap, and a
  proportional image-height tweak.
- No new CTA banners, header icons, or section labels (deferred to a future
  round per the priority decision made during brainstorming).
- No build step or new tooling — this stays within the existing plain
  HTML/CSS approach the site already uses.

## Current state (per file)

| File | Container | Current grid rule | Current gap |
|---|---|---|---|
| `index.html` | `.main-container`, max-width 800px, padding 2rem | `.listings { grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); }` | 1.5rem |
| `listings.html` | `.container`, max-width 900px, padding 2rem | `.listings-grid { grid-template-columns: repeat(3, 1fr); }` | 2rem |
| `favoritter.html` | `.page-wrapper`, max-width 900px, padding 0 1rem | `.listings-grid { grid-template-columns: repeat(3, 1fr); }` | 1.5rem |

`favoritter.html` already has mobile breakpoints at 700px (2 columns) and
500px (1 column) for `.listings-grid` — these stay untouched.

`index.html` and `listings.html` currently have **no** mobile breakpoints
for their grids at all. `index.html`'s `auto-fill`/`minmax()` pattern is
already naturally responsive without one (column count recalculates as the
viewport narrows). `listings.html`'s current `repeat(3, 1fr)` is NOT
responsive — at narrow viewports it would currently squeeze 3 fixed-ratio
columns into a small viewport rather than dropping to fewer columns. This
spec fixes that as a side effect of switching it to the same `auto-fill`
pattern.

## Design

Keep the responsive `grid-template-columns: repeat(auto-fill, minmax(Npx, 1fr))`
pattern (already proven in `index.html`) on all three files — don't switch
to a fixed `repeat(4, 1fr)`, which would require adding new media queries
to avoid breaking mobile layout. Instead, tune `N` (the minimum card width)
and the gap down on each file just enough that 4 columns fit at that page's
existing desktop content width, while the same rule continues to collapse
naturally to fewer columns as the viewport narrows — zero new media queries
needed for `index.html` or `listings.html`, and `favoritter.html`'s
existing ones remain valid extra safety nets.

Target values (content width = container max-width minus its horizontal
padding; chosen gap leaves room for exactly 4 columns at that width):

| File | Content width | New min card width | New gap | Resulting columns at full width |
|---|---|---|---|---|
| `index.html` | 736px | 170px | 1rem (16px) | 4 |
| `listings.html` | 836px | 180px | 1.25rem (20px) | 4 |
| `favoritter.html` | 868px | 190px | 1.25rem (20px) | 4 |

Each page's column width differs slightly (170-190px) because each page's
container width differs (800 vs 900px) and its padding differs — this is
expected and fine; the goal is "4 columns, denser" on each page
individually, not pixel-identical card widths across pages.

Also scale down each file's `.listing-image` height proportionally to its
new card width, so the photo area's aspect ratio stays consistent with how
it looks today rather than becoming disproportionately tall. All three
files already use the same selector name, `.listing-image`, but with very
different current heights — each needs its own scaled value:

| File | Old card width (approx) | Old `.listing-image` height | New card width | New `.listing-image` height |
|---|---|---|---|---|
| `index.html` | 220px | 150px | 170px | 116px |
| `listings.html` | 257px | 280px | 180px | 196px |
| `favoritter.html` | 273px | 200px | 190px | 139px |

("Old card width" is the approximate rendered width today, derived from
each file's current content width split across its current column count —
used only to compute a proportional scale factor, not itself a value being
changed.)

## Files changed

- `index.html`: `.listings` rule (grid-template-columns + gap),
  `.listing-image` height (150px → 116px).
- `listings.html`: `.listings-grid` rule (grid-template-columns + gap),
  `.listing-image` height (280px → 196px).
- `favoritter.html`: `.listings-grid` rule (grid-template-columns + gap),
  `.listing-image` height (200px → 139px).
- `mine-annoncer.html`: no changes.

## Error handling / edge cases

- Fewer than 4 listings: `auto-fill` already handles this gracefully today
  (unfilled grid cells simply don't render) — no new edge case introduced.
- Very narrow viewports (mobile): `auto-fill`/`minmax()` naturally drops to
  fewer columns; `favoritter.html`'s existing explicit breakpoints continue
  to apply as before, now controlling roughly the same column counts they
  already did since the new minmax floor is still in a similar range to
  trigger before those breakpoints.

## Testing / verification

Same ad-hoc Playwright verification approach used in the previous homepage
rows feature (no test framework in this repo):

- For each of the 3 files, load it locally and assert the grid container
  shows exactly 4 columns at a representative desktop viewport width (e.g.
  1280px), via `getComputedStyle(...).gridTemplateColumns` reporting 4
  space-separated track values, or by checking that 4 cards' bounding
  boxes share the same `y` position (same row).
- Confirm card click-through, favorite button, and existing filters/sort
  still work unchanged (these are CSS-only changes, but worth a smoke
  check per file since each file's JS independently re-renders this grid).
- Visually confirm via screenshot that card photos don't look distorted
  at the new size.
