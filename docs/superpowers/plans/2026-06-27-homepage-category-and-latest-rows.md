# Homepage Popular Categories + Latest Listings Rows Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a "Populære kategorier" row and a "Nyeste annoncer" row to the BikerBasen homepage (`index.html`), matching the segmented-rows pattern from bilbasen.dk, while leaving the existing "Alle annoncer" filterable grid untouched.

**Architecture:** Pure client-side change to a single static HTML file (`index.html`). No new dependencies, no backend changes. The "Populære kategorier" row is static markup relocated from inside the hero search box into its own section. The "Nyeste annoncer" row is rendered by new JavaScript that reuses the existing `motorcycles` data array and a newly-extracted shared card-rendering function.

**Tech Stack:** Vanilla HTML/CSS/JS (no build step, no framework). Verification uses a one-off Node + Playwright script (Playwright is already available in this environment from prior work in this project — `npm install playwright` + `npx playwright install chromium` if not).

## Global Constraints

- Repo: `c:/Users/pasca/bikerbasen-homepage` (a git repo cloned from `https://github.com/Gulecitronxx/bikerbasen`, deployed via GitHub Pages at `https://gulecitronxx.github.io/bikerbasen/`).
- Only `index.html` is modified by this plan. No other file changes.
- The existing "Alle annoncer" grid, its filters, and `listings.html`/`listing-detail.html` must continue to work exactly as before — this is an additive change.
- Danish UI copy only, matching the site's existing tone (e.g. "Populære kategorier", "Nyeste annoncer", "Se alle").
- No new npm dependencies committed to the repo. Playwright is a local dev-only verification tool, not a runtime dependency — do not add a `package.json` to the site repo.
- File uses LF line endings (verified: 0 CR bytes in the file). Don't introduce CRLF.
- `index.html` contains six very large single-line embedded base64 images (the category card images) inside the block this plan relocates. Do **not** attempt to read, retype, or paste those lines through the Edit tool — the line-relocation step uses a small Node script that moves them by reference (array slicing) instead, exactly to avoid this problem.

---

## File Structure

Only one file changes:

- **Modify:** `c:/Users/pasca/bikerbasen-homepage/index.html`
  - CSS (`<style>` block, roughly lines 1-200): add a small set of new generic "homepage row" classes; modify the existing `.popular-categories` / `.category-card` / `.category-card img` rules to support horizontal scrolling at a larger size.
  - Markup (`<body>`, roughly lines 200-380): relocate the existing "Populære søgninger" block out of the hero search box into its own new `<section class="homepage-row">`; add a second new `<section class="homepage-row">` for "Nyeste annoncer".
  - Script (`<script>` block, roughly lines 440-870): extract a shared `cardHTML(m, favs)` function out of `displayListings()`; add `renderLatestRow()`; call it from `init()`.

A temporary, **uncommitted** helper script (`scratch-restructure-categories.js`) is created in Task 1 to perform the markup relocation safely, then deleted before committing.

---

### Task 1: Relocate and restyle the "Populære kategorier" section

**Files:**
- Modify: `index.html` (CSS block and hero markup block)
- Create (temporary, not committed): `scratch-restructure-categories.js` in the repo root

**Interfaces:**
- Consumes: nothing new
- Produces: a new `<section class="homepage-row">` containing the relocated `<div class="popular-categories">...</div>` (the six `.category-card` divs, untouched internally), positioned between the hero `</section>` and `<div class="cta-section">`. New CSS classes `.homepage-row`, `.homepage-row-header`, `.homepage-row-title`, `.homepage-row-link` are available for Task 3 to reuse.

- [ ] **Step 1: Confirm current structure before touching anything**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
grep -n "Populære søgninger" index.html
```
Expected output: one line, e.g. `358:                    <h3 ...>Populære søgninger</h3>` (the exact line number doesn't matter, but there must be exactly one match). If there are zero or more than one matches, stop — the file doesn't match what this plan assumes; re-read the surrounding ~15 lines with the Read tool before proceeding.

- [ ] **Step 2: Write the relocation script**

Create `scratch-restructure-categories.js` in the repo root with this exact content:

```js
const fs = require('fs');

const filePath = 'index.html';
const lines = fs.readFileSync(filePath, 'utf8').split('\n');

const titleIdx = lines.findIndex(l => l.includes('Populære søgninger'));
if (titleIdx === -1) {
  throw new Error('Could not find "Populære søgninger" - has index.html already been changed?');
}

// Expected structure relative to titleIdx (see plan doc Task 1 for the source of these offsets):
//   titleIdx - 1 : <div style="border-top: ...">            (wrapper open)
//   titleIdx     : <h3>Populære søgninger</h3>
//   titleIdx + 1 : <div class="popular-categories">         (categories container open)
//   titleIdx + 2 .. titleIdx + 7 : six <div class="category-card" ...>...</div> lines
//   titleIdx + 8 : </div>                                    (categories container close)
//   titleIdx + 9 : </div>                                    (wrapper close)
const wrapperStart = titleIdx - 1;
const categoriesOpenIdx = titleIdx + 1;
const categoriesCloseIdx = titleIdx + 8;
const wrapperEnd = titleIdx + 9;

if (!lines[wrapperStart].includes('border-top')) {
  throw new Error('Unexpected structure: line above title is not the border-top wrapper');
}
if (!lines[categoriesOpenIdx].includes('popular-categories')) {
  throw new Error('Unexpected structure: popular-categories div not where expected');
}
if (!lines[categoriesCloseIdx].trim().startsWith('</div>')) {
  throw new Error('Unexpected structure: popular-categories close not where expected');
}
if (!lines[wrapperEnd].trim().startsWith('</div>')) {
  throw new Error('Unexpected structure: wrapper close not where expected');
}

// Extract the popular-categories div (and its six category-card children) completely intact.
const categoriesBlock = lines.slice(categoriesOpenIdx, categoriesCloseIdx + 1);

// Remove the entire old wrapper (h3 + popular-categories div) from inside the hero.
lines.splice(wrapperStart, wrapperEnd - wrapperStart + 1);

// Find the hero's closing </section> tag (the first one at or after where we just cut).
let heroCloseIdx = -1;
for (let i = wrapperStart; i < lines.length; i++) {
  if (lines[i].trim() === '</section>') { heroCloseIdx = i; break; }
}
if (heroCloseIdx === -1) {
  throw new Error('Could not find hero closing </section> after removing the old block');
}

const newSection = [
  '',
  '    <section class="homepage-row">',
  '        <div class="homepage-row-header">',
  '            <h2 class="homepage-row-title">Populære kategorier</h2>',
  '        </div>',
  ...categoriesBlock,
  '    </section>'
];

lines.splice(heroCloseIdx + 1, 0, ...newSection);

fs.writeFileSync(filePath, lines.join('\n'));
console.log('OK: relocated "Populære kategorier" section after the hero.');
```

- [ ] **Step 3: Run the script**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
node scratch-restructure-categories.js
```
Expected output: `OK: relocated "Populære kategorier" section after the hero.`
If it throws an error instead, stop and investigate — do not proceed to Step 4 with a half-modified file. (If it already ran successfully once, running it again will fail at Step 1's `findIndex` check inside the script itself, which is expected and fine — it just means don't run it twice.)

- [ ] **Step 4: Verify the relocation**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
grep -n "Populære søgninger\|Populære kategorier\|popular-categories\|homepage-row" index.html
```
Expected: zero matches for "Populære søgninger" (it's gone), exactly one match for "Populære kategorier" (the new `<h2>`), and the `popular-categories` div should now appear shortly after a `homepage-row` section opens — i.e. `homepage-row` line numbers should come immediately before the `popular-categories` line. Also run:
```bash
grep -c "category-card\"" index.html
```
Expected: `6` (all six category cards survived the move).

- [ ] **Step 5: Delete the temporary script**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
rm scratch-restructure-categories.js
```
This script must never be committed — it was a one-time migration tool.

- [ ] **Step 6: Add new CSS classes for homepage rows**

Use the Edit tool on `index.html`. Find this existing line (it's the `.cta-section` rule, used here only as a unique anchor — do not change it):

```
        .cta-section { background: white; padding: 2rem; border-radius: 6px 6px 0 0; text-align: center; max-width: 800px; margin: 0 auto 0 auto; box-shadow: 0 1px 3px rgba(0,0,0,0.08); border: 12px solid #f0f0f0; border-bottom: 12px solid #f0f0f0; }
```

Replace it with (adds four new rules immediately before the unchanged `.cta-section` rule):

```
        .homepage-row { max-width: 800px; margin: 0 auto 1.5rem auto; padding: 1.5rem 2rem; background: white; border-radius: 6px; box-shadow: 0 1px 3px rgba(0,0,0,0.08); }
        .homepage-row-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 1rem; }
        .homepage-row-title { font-size: 18px; font-weight: 700; color: #333333; }
        .homepage-row-link { font-size: 13px; font-weight: 600; color: #1B4332; text-decoration: none; }
        .cta-section { background: white; padding: 2rem; border-radius: 6px 6px 0 0; text-align: center; max-width: 800px; margin: 0 auto 0 auto; box-shadow: 0 1px 3px rgba(0,0,0,0.08); border: 12px solid #f0f0f0; border-bottom: 12px solid #f0f0f0; }
```

- [ ] **Step 7: Restyle `.popular-categories` and `.category-card` for horizontal scroll at a bigger size**

Use the Edit tool. Find:

```
        .popular-categories { display: grid; grid-template-columns: repeat(6, 1fr); gap: 1rem; margin-bottom: 2rem; }
        .category-card { background: white; padding: 0; overflow: hidden; border-radius: 10px; text-align: center; cursor: pointer; box-shadow: 0 2px 8px rgba(0,0,0,0.12); transition: all 0.3s; position: relative; }
        .category-card:hover { box-shadow: 0 6px 20px rgba(0,0,0,0.2); transform: translateY(-3px); }
        .category-card:hover img { transform: scale(1.05); }
        .category-card img { width: 200%; height: 100px; object-fit: cover; display: block; transition: transform 0.4s ease; }
        .category-card-title { position: absolute; bottom: 0; left: 0; right: 0; color: white; font-weight: 700; font-size: 15px; padding: 20px 16px 16px; background: linear-gradient(to top, rgba(0,0,0,0.75) 0%, rgba(0,0,0,0) 100%); letter-spacing: 0.5px; }
```

Replace with:

```
        .popular-categories { display: flex; gap: 1rem; overflow-x: auto; scroll-snap-type: x mandatory; padding-bottom: 0.5rem; margin-bottom: 0; }
        .category-card { flex: 0 0 220px; scroll-snap-align: start; background: white; padding: 0; overflow: hidden; border-radius: 10px; text-align: center; cursor: pointer; box-shadow: 0 2px 8px rgba(0,0,0,0.12); transition: all 0.3s; position: relative; }
        .category-card:hover { box-shadow: 0 6px 20px rgba(0,0,0,0.2); transform: translateY(-3px); }
        .category-card:hover img { transform: scale(1.05); }
        .category-card img { width: 100%; height: 160px; object-fit: cover; display: block; transition: transform 0.4s ease; }
        .category-card-title { position: absolute; bottom: 0; left: 0; right: 0; color: white; font-weight: 700; font-size: 16px; padding: 28px 16px 16px; background: linear-gradient(to top, rgba(0,0,0,0.75) 0%, rgba(0,0,0,0) 100%); letter-spacing: 0.5px; }
```

- [ ] **Step 8: Verify with a Playwright smoke check**

Run (from any writable working directory with Playwright installed — see prerequisite below):

```bash
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto('file://c:/Users/pasca/bikerbasen-homepage/index.html', { waitUntil: 'load' });
  await page.waitForTimeout(2000);
  const cardCount = await page.locator('.category-card').count();
  const titleText = await page.locator('.homepage-row-title').first().textContent();
  const heroHasCategories = await page.locator('section.hero .popular-categories').count();
  console.log('CATEGORY_CARD_COUNT=' + cardCount);
  console.log('FIRST_ROW_TITLE=' + titleText);
  console.log('CATEGORIES_STILL_INSIDE_HERO=' + heroHasCategories);
  await browser.close();
})();
"
```

Prerequisite if Playwright isn't already set up in a scratch folder: `npm install playwright && npx playwright install chromium` (run once in some scratch working directory, not inside the site repo).

Expected output:
```
CATEGORY_CARD_COUNT=6
FIRST_ROW_TITLE=Populære kategorier
CATEGORIES_STILL_INSIDE_HERO=0
```

- [ ] **Step 9: Commit**

```bash
cd "c:/Users/pasca/bikerbasen-homepage"
git add index.html
git status
```
Confirm `scratch-restructure-categories.js` is NOT listed (it was deleted in Step 5, so `git status` should show only `index.html` modified). Then:
```bash
git commit -m "$(cat <<'EOF'
Relocate Populære kategorier into its own bigger homepage row

Moves the small category-icon section out of the hero search box into
a standalone, larger horizontally-scrollable row between the hero and
the CTA section, matching bilbasen.dk's segmented homepage style.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Extract shared `cardHTML(m, favs)` function

**Files:**
- Modify: `index.html` (the `displayListings` function, currently around line 652-691 — re-check the exact line with `grep -n "function displayListings" index.html` since Task 1 shifted line numbers)

**Interfaces:**
- Consumes: nothing new (pure refactor of existing logic)
- Produces: `function cardHTML(m, favs)` — takes one listing object `m` (same shape as items in the `motorcycles` array: `{id, brand, model, type, year, km, price, images?, thumbnail?, isDealer?/is_dealer?, dealerName?/dealer_name?, dealerLogo?/dealer_logo?, ...}`) and the current favorites array `favs` (array of listing ids, from `getFavorites()`), and returns the HTML string for one `.listing-card` element. Task 3 depends on this function existing with this exact name and signature.

- [ ] **Step 1: Locate the current function**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
grep -n "function displayListings" index.html
```
Read the function with the Read tool starting at that line, for about 45 lines, to see its current exact content (it should match the body described below — if it doesn't, the file has diverged from what this plan assumes; stop and re-read more context before editing).

- [ ] **Step 2: Replace `displayListings` with `cardHTML` + a slimmer `displayListings`**

Use the Edit tool. Find (this is the current, un-refactored function — confirm it matches what Step 1 showed before editing):

```javascript
        function displayListings(items) {
            const favs = getFavorites();
            document.getElementById('listingsContainer').innerHTML = items.map(m => {
                const cachedImgs = JSON.parse(localStorage.getItem('listingImages_' + m.id) || '[]');
                const thumb = m.thumbnail
                    || (m.images && m.images.length > 0 ? m.images[0].data : null)
                    || (cachedImgs.length > 0 ? cachedImgs[0].data : null);

                const isDealer = m.isDealer || m.is_dealer;
                const dealerLogo = m.dealerLogo || m.dealer_logo || localStorage.getItem('dealerLogo_' + m.id);
                const dealerName = m.dealerName || m.dealer_name;

                const dealerHeader = isDealer ? `
                    <div style="padding:6px 10px;border-bottom:1px solid #f0f0f0;display:flex;align-items:center;gap:8px;background:white;min-height:38px;">
                        ${dealerLogo
                            ? `<img src="${dealerLogo}" style="max-height:24px;max-width:90px;object-fit:contain;">`
                            : `<span style="font-size:11px;font-weight:700;color:#333;">${dealerName || 'Forhandler'}</span>`
                        }
                    </div>` : '';

                return `
                <div class="listing-card" onclick="window.location.href='listing-detail.html?id=${m.id}'">
                    ${dealerHeader}
                    <div class="listing-image" style="overflow:hidden; position:relative;">
                        ${thumb
                            ? `<img src="${thumb}" alt="${m.brand} ${m.model}" style="width:100%;height:100%;object-fit:contain;background:white;">`
                            : `<div style="width:100%;height:100%;display:flex;align-items:center;justify-content:center;background:linear-gradient(135deg,#f0f0f0 25%,transparent 25%,transparent 50%,#f0f0f0 50%,#f0f0f0 75%,transparent 75%,transparent);background-size:20px 20px;font-size:36px;color:#ccc;">🏍️</div>`
                        }
                        <button class="favorite-btn ${favs.includes(m.id) ? 'active' : ''}" onclick="toggleFavorite(event, ${m.id})">${favs.includes(m.id) ? '❤️' : '🤍'}</button>
                    </div>
                    <div class="listing-info">
                        <div class="listing-title">${m.brand} ${m.model}</div>
                        <div class="listing-details">${m.year} • ${m.km.toLocaleString('da-DK')} km</div>
                        <div class="listing-price">${m.price.toLocaleString('da-DK')} kr</div>
                    </div>
                </div>`;
            }).join('');
            document.getElementById('bikeCount').textContent = items.length;
            document.getElementById('announceCount').textContent = items.length.toLocaleString('da-DK') + ' annoncer i dag';
        }
```

Replace with:

```javascript
        function cardHTML(m, favs) {
            const cachedImgs = JSON.parse(localStorage.getItem('listingImages_' + m.id) || '[]');
            const thumb = m.thumbnail
                || (m.images && m.images.length > 0 ? m.images[0].data : null)
                || (cachedImgs.length > 0 ? cachedImgs[0].data : null);

            const isDealer = m.isDealer || m.is_dealer;
            const dealerLogo = m.dealerLogo || m.dealer_logo || localStorage.getItem('dealerLogo_' + m.id);
            const dealerName = m.dealerName || m.dealer_name;

            const dealerHeader = isDealer ? `
                <div style="padding:6px 10px;border-bottom:1px solid #f0f0f0;display:flex;align-items:center;gap:8px;background:white;min-height:38px;">
                    ${dealerLogo
                        ? `<img src="${dealerLogo}" style="max-height:24px;max-width:90px;object-fit:contain;">`
                        : `<span style="font-size:11px;font-weight:700;color:#333;">${dealerName || 'Forhandler'}</span>`
                    }
                </div>` : '';

            return `
            <div class="listing-card" onclick="window.location.href='listing-detail.html?id=${m.id}'">
                ${dealerHeader}
                <div class="listing-image" style="overflow:hidden; position:relative;">
                    ${thumb
                        ? `<img src="${thumb}" alt="${m.brand} ${m.model}" style="width:100%;height:100%;object-fit:contain;background:white;">`
                        : `<div style="width:100%;height:100%;display:flex;align-items:center;justify-content:center;background:linear-gradient(135deg,#f0f0f0 25%,transparent 25%,transparent 50%,#f0f0f0 50%,#f0f0f0 75%,transparent 75%,transparent);background-size:20px 20px;font-size:36px;color:#ccc;">🏍️</div>`
                    }
                    <button class="favorite-btn ${favs.includes(m.id) ? 'active' : ''}" onclick="toggleFavorite(event, ${m.id})">${favs.includes(m.id) ? '❤️' : '🤍'}</button>
                </div>
                <div class="listing-info">
                    <div class="listing-title">${m.brand} ${m.model}</div>
                    <div class="listing-details">${m.year} • ${m.km.toLocaleString('da-DK')} km</div>
                    <div class="listing-price">${m.price.toLocaleString('da-DK')} kr</div>
                </div>
            </div>`;
        }

        function displayListings(items) {
            const favs = getFavorites();
            document.getElementById('listingsContainer').innerHTML = items.map(m => cardHTML(m, favs)).join('');
            document.getElementById('bikeCount').textContent = items.length;
            document.getElementById('announceCount').textContent = items.length.toLocaleString('da-DK') + ' annoncer i dag';
        }
```

- [ ] **Step 3: Verify the grid still renders identically**

Run:
```bash
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const errors = [];
  page.on('pageerror', e => errors.push(e.message));
  await page.goto('file://c:/Users/pasca/bikerbasen-homepage/index.html', { waitUntil: 'load' });
  await page.waitForTimeout(5000);
  console.log('GRID_CARD_COUNT=' + await page.locator('#listingsContainer .listing-card').count());
  console.log('ERRORS=' + JSON.stringify(errors));
  await browser.close();
})();
"
```
Expected: `GRID_CARD_COUNT=16` (or more if a real Supabase listing is reachable) and `ERRORS=[]`. This must match the card count you'd see before this task's edit — if it's `0`, the refactor broke something; re-check the replacement was pasted exactly.

- [ ] **Step 4: Commit**

```bash
cd "c:/Users/pasca/bikerbasen-homepage"
git add index.html
git commit -m "$(cat <<'EOF'
Extract shared cardHTML() from displayListings()

Pure refactor, no behavior change. Pulls the per-listing card markup
out of displayListings() into its own function so the new "Nyeste
annoncer" row (next commit) can reuse the exact same card rendering
instead of duplicating it.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Add the "Nyeste annoncer" row

**Files:**
- Modify: `index.html` (CSS block, markup between the new categories section and `.cta-section`, and the `init()` function)

**Interfaces:**
- Consumes: `cardHTML(m, favs)` from Task 2 (exact signature: `cardHTML(listingObject, favoritesIdArray) -> htmlString`), the existing global `motorcycles` array, the existing `getFavorites()` function.
- Produces: `function renderLatestRow()` (no params, no return value — renders directly into `#latestListingsRow`), called once from `init()`.

- [ ] **Step 1: Add markup for the new row**

Important: the relocated `</div>` from Task 1 keeps its *original* indentation from when it lived inside the hero (the Node script moves those lines by reference, it doesn't re-indent them) — it's indented 20 spaces, not the 8 you'd expect for content directly inside a top-level `<section>`. Verify before editing:

```bash
cd "c:/Users/pasca/bikerbasen-homepage"
sed -n '/<section class="homepage-row">/,/<div class="cta-section">/p' index.html | cat -A | grep -n '</div>\$\|</section>\$\|cta-section'
```
Expected: the line with `</div>$` immediately before `</section>$` shows 20 `$`-less leading space characters (i.e. `cat -A` prints 20 spaces then `</div>$`). If your editor/terminal shows a different count, use the actual count you observe instead of assuming — do not guess.

Use the Edit tool. Find the end of the categories section followed by the CTA section (this exact text, with 20 spaces before the first `</div>`, should now exist after Task 1):

```
                    </div>
    </section>

    <div class="cta-section">
```

Replace with (inserts a new `homepage-row` section in between):

```
                    </div>
    </section>

    <section class="homepage-row">
        <div class="homepage-row-header">
            <h2 class="homepage-row-title">Nyeste annoncer</h2>
            <a href="listings.html" class="homepage-row-link">Se alle →</a>
        </div>
        <div class="popular-categories" id="latestListingsRow"></div>
    </section>

    <div class="cta-section">
```

Note: this reuses the `.popular-categories` class purely for its now-generic horizontal-scroll-row CSS (flex + overflow-x + scroll-snap) — it has nothing to do with categories here, it's just the existing scroll-row styling. The `.category-card` fixed-width styling does NOT apply to `.listing-card` elements, so add a small CSS rule in Step 2 to give listing cards a fixed width inside this row.

- [ ] **Step 2: Add CSS for listing cards inside the scroll row**

Use the Edit tool. Find:

```
        .listing-price { font-weight: 700; color: #1B4332; font-size: 16px; }
```

Replace with:

```
        .listing-price { font-weight: 700; color: #1B4332; font-size: 16px; }
        .popular-categories .listing-card { flex: 0 0 220px; scroll-snap-align: start; }
```

- [ ] **Step 3: Add `renderLatestRow()` and call it from `init()`**

Use the Edit tool. Find:

```javascript
        function init() {
            renderBrands();
            displayListings(motorcycles);
        }
```

Replace with:

```javascript
        function init() {
            renderBrands();
            displayListings(motorcycles);
            renderLatestRow();
        }

        function renderLatestRow() {
            const favs = getFavorites();
            const latest = [...motorcycles].sort((a, b) => b.id - a.id).slice(0, 10);
            document.getElementById('latestListingsRow').innerHTML = latest.map(m => cardHTML(m, favs)).join('');
        }
```

- [ ] **Step 4: Verify with a Playwright smoke check**

Run:
```bash
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const errors = [];
  page.on('pageerror', e => errors.push(e.message));
  await page.goto('file://c:/Users/pasca/bikerbasen-homepage/index.html', { waitUntil: 'load' });
  await page.waitForTimeout(5000);
  const latestCount = await page.locator('#latestListingsRow .listing-card').count();
  const gridCount = await page.locator('#listingsContainer .listing-card').count();
  const sectionOrder = await page.evaluate(() => {
    const titles = Array.from(document.querySelectorAll('.homepage-row-title, .listings-title, .cta-title')).map(el => el.textContent.trim());
    return titles;
  });
  console.log('LATEST_ROW_CARD_COUNT=' + latestCount);
  console.log('GRID_CARD_COUNT=' + gridCount);
  console.log('SECTION_ORDER=' + JSON.stringify(sectionOrder));
  console.log('ERRORS=' + JSON.stringify(errors));
  await browser.close();
})();
"
```
Expected:
- `LATEST_ROW_CARD_COUNT=10` (there are 16 demo motorcycles, so the row is capped at 10 as designed)
- `GRID_CARD_COUNT=16` (or more, unchanged from Task 2's verification — the full grid is untouched)
- `SECTION_ORDER=["Populære kategorier","Nyeste annoncer","Alle annoncer","Vil du sælge din motorcykel?"]` — confirms the page order matches the spec (categories, then latest, then the CTA title comes after both, then the existing grid title). Note: `.cta-title` text appears in the array after both new row titles, and `.listings-title` ("Alle annoncer") last — if the array order differs, the sections are in the wrong order; re-check Step 1's insertion point.
- `ERRORS=[]`

- [ ] **Step 5: Click-through check (category card and "Se alle" link)**

Run:
```bash
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto('file://c:/Users/pasca/bikerbasen-homepage/index.html', { waitUntil: 'load' });
  await page.waitForTimeout(3000);
  await page.locator('.category-card').first().click();
  console.log('AFTER_CATEGORY_CLICK=' + page.url());
  await browser.close();
})();
"
```
Expected: `AFTER_CATEGORY_CLICK=` ending in `listings.html?type=...` (some category type) — confirms `filterCategory()` still works unchanged on the relocated cards.

- [ ] **Step 6: Commit**

```bash
cd "c:/Users/pasca/bikerbasen-homepage"
git add index.html
git commit -m "$(cat <<'EOF'
Add Nyeste annoncer homepage row

Shows the 10 most recently added listings (sorted by id descending) in
a horizontally-scrollable row between Populære kategorier and the
existing Alle annoncer grid, reusing the shared cardHTML() renderer.
Matches bilbasen.dk's segmented-homepage pattern.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Final full-page verification and push

**Files:** none modified — verification only.

**Interfaces:** none new.

- [ ] **Step 1: Full local smoke test**

Run:
```bash
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  const errors = [];
  page.on('pageerror', e => errors.push(e.message));
  page.on('console', m => { if (m.type() === 'error' && !m.text().includes('ERR_NAME_NOT_RESOLVED') && !m.text().includes('Supabase')) errors.push(m.text()); });
  await page.goto('file://c:/Users/pasca/bikerbasen-homepage/index.html', { waitUntil: 'load' });
  await page.waitForTimeout(5000);
  console.log('CATEGORY_CARDS=' + await page.locator('.category-card').count());
  console.log('LATEST_ROW_CARDS=' + await page.locator('#latestListingsRow .listing-card').count());
  console.log('GRID_CARDS=' + await page.locator('#listingsContainer .listing-card').count());
  console.log('HERO_STILL_HAS_CATEGORIES=' + await page.locator('section.hero .popular-categories').count());
  await page.click('#typeFilter'); // sanity: existing filter UI still present and clickable
  console.log('UNEXPECTED_ERRORS=' + JSON.stringify(errors));
  await browser.close();
})();
"
```
Expected: `CATEGORY_CARDS=6`, `LATEST_ROW_CARDS=10`, `GRID_CARDS=16` (or more), `HERO_STILL_HAS_CATEGORIES=0`, `UNEXPECTED_ERRORS=[]` (the filter on Supabase-related console noise is expected and fine — that's the known dead-Supabase-project fallback behavior from prior work, unrelated to this change).

If any of these don't match, do not proceed to push — go back to the relevant task and fix it first.

- [ ] **Step 2: Push to GitHub**

```bash
cd "c:/Users/pasca/bikerbasen-homepage"
git log --oneline -5
git push origin main
```
Confirm the push succeeds and shows the three commits from Tasks 1-3.

- [ ] **Step 3: Verify on the live deployed site**

Wait about a minute for GitHub Pages to redeploy, then run the same Step 1 script against the live URL instead of the local file:
```bash
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto('https://gulecitronxx.github.io/bikerbasen/', { waitUntil: 'load' });
  await page.waitForTimeout(6000);
  console.log('CATEGORY_CARDS=' + await page.locator('.category-card').count());
  console.log('LATEST_ROW_CARDS=' + await page.locator('#latestListingsRow .listing-card').count());
  console.log('GRID_CARDS=' + await page.locator('#listingsContainer .listing-card').count());
  await browser.close();
})();
"
```
Expected: same counts as Step 1 (`CATEGORY_CARDS=6`, `LATEST_ROW_CARDS=10`, `GRID_CARDS=16` or more). This confirms the live site, not just the local file, reflects the change.
