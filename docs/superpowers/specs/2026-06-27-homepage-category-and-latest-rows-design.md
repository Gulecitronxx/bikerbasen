# Homepage: Popular Categories + Latest Listings rows

## Context

The user asked for the BikerBasen homepage to take feature inspiration from
bilbasen.dk (a Danish car marketplace), specifically calling out segmented
homepage sections instead of one mixed listings grid. Bilbasen splits its
homepage into curated rows (latest vehicles, private sellers, etc.) above a
full searchable grid.

Scope for this round: add two new homepage rows. Browse-by-brand pages,
location filtering, and a footer revamp are explicitly out of scope and
deferred to future rounds.

## Goals

1. Add a "Populære kategorier" (Popular categories) row — bigger, more
   prominent than today's small icon row inside the hero search box.
2. Add a "Nyeste annoncer" (Latest listings) row showing the most recently
   added listings, separate from the full filterable grid.
3. Keep the existing "Alle annoncer" full grid (with all search filters)
   exactly as it works today, positioned below the new rows.
4. Remove the small existing category-icon section from inside the hero
   search box (superseded by the new row — avoid duplicate UI).

## Non-goals

- Footer revamp / newsletter signup (deferred)
- Location/region filter (deferred)
- Browse-by-brand pages (deferred)
- Buy vs. lease toggle (deferred)
- Any new backend/Supabase changes — both rows render from data already
  loaded into the existing `motorcycles` array.

## Page layout (top to bottom, after this change)

1. Header (unchanged)
2. Hero / search filters (unchanged, minus the small category-icon section)
3. **Populære kategorier** (new)
4. **Nyeste annoncer** (new)
5. CTA "Vil du sælge din motorcykel?" (unchanged)
6. **Alle annoncer** full filterable grid (unchanged)
7. Footer (unchanged)

## Design: Populære kategorier

- Full-width section, title "Populære kategorier", horizontally scrollable
  row (CSS `overflow-x: auto; scroll-snap-type: x mandatory`, no JS
  carousel library).
- 6 cards: Cruiser, Sport, Adventure, Naked Bike, Retro, Touring — same six
  categories and same base64 images already embedded in the page today
  (currently used by the small hero icon row being removed), rendered
  bigger (~200x140px) with a text label overlay at the bottom of each card.
- Clicking a card calls the existing `filterCategory(type)` function
  (already defined, navigates to `listings.html?type=...`) — no new
  click-handling logic needed.
- Replaces the existing small category section inside the hero search box
  (that markup block is deleted).

## Design: Nyeste annoncer

- Full-width section, title "Nyeste annoncer" with a "Se alle →" link to
  `listings.html`.
- Horizontally scrollable row (same CSS scroll-snap technique), showing the
  10 most recently added listings.
- "Most recently added" = sorted by `id` descending. This is a deliberate
  choice distinct from the existing "Nyeste først" sort option elsewhere in
  the codebase (`listings.html`), which actually sorts by motorcycle model
  year, not by when the ad was posted. `id` descending is the correct proxy
  for "recently added" — it works for both real Supabase rows (which have
  `created_at`, but `id` order tracks it since rows are inserted in order)
  and hardcoded demo data (sequential ids, no `created_at` field).
- Cards reuse the exact same markup/behavior as the existing "Alle
  annoncer" grid (image/placeholder, dealer header if applicable, favorite
  heart button, click-through to `listing-detail.html`).

## Technical approach

`displayListings()` currently builds card HTML inline inside its own
`.map()`. Extract that per-card template into a standalone function,
`cardHTML(m, favs)`, that both `displayListings()` and the new "Nyeste
annoncer" renderer call. This avoids duplicating the (fairly involved)
card markup — dealer header, image fallback, favorite button state — in
two places.

New functions added to `index.html`'s existing `<script>` block:

- `cardHTML(m, favs)` — returns the HTML string for one listing card
  (extracted from `displayListings`).
- `renderCategoryRow()` — renders the 6 category cards into a new
  container, called once from `init()`.
- `renderLatestRow()` — sorts `motorcycles` by `id` descending, takes the
  top 10, renders them via `cardHTML` into a new horizontally-scrollable
  container, called once from `init()`.

`init()` is called after Supabase/localStorage data has been merged into
`motorcycles` (same timing as today), so both new rows reflect real data
the same way the existing grid does — no new data fetching, no new global
state.

## Error handling

No new failure modes: both rows render from data already validated/merged
by the existing `loadSupabaseListings()` flow (including its existing
timeout fallback). If `motorcycles` is empty for some reason, both new
rows simply render no cards (same graceful-empty behavior as the existing
grid).

## Testing / verification

Manual verification via the existing local Playwright check pattern used
earlier in this project:
- Open `index.html` locally, confirm both new rows render with expected
  card counts (6 category cards, up to 10 latest-listing cards).
- Confirm clicking a category card navigates to the right filtered
  `listings.html` URL.
- Confirm the existing "Alle annoncer" grid and its filters still work
  unchanged.
- Confirm the small hero category-icon section is gone with no leftover
  dead CSS/markup.
