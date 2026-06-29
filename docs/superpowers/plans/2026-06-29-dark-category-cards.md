# Dark Stat-Card Populære Kategorier Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert `index.html`'s static "Populære kategorier" row into JS-rendered dark stat-cards showing each category's live listing count and average price, with functional left/right arrow navigation.

**Architecture:** A one-time Node script extracts the six existing embedded base64 category photos out of static markup into a JS data map (avoiding ever retyping or re-reading that data through the Edit tool, which chokes on this file's huge embedded image lines). The row then becomes JS-rendered at `init()` time from the existing `motorcycles` array, following the same pattern already used for the "Nyeste annoncer" row (`renderLatestRow()`).

**Tech Stack:** Vanilla HTML/CSS/JS, no build step, no new dependencies. Verification via one-off Node + Playwright scripts (already used and available in this environment from prior work in this repo).

## Global Constraints

- Repo: `c:/Users/pasca/bikerbasen-homepage`, deployed via GitHub Pages at `https://gulecitronxx.github.io/bikerbasen/`.
- Only `index.html` is modified by this plan. No other file changes.
- No new Supabase queries — stats are computed client-side from the existing `motorcycles` array (the same array `displayListings`/`renderLatestRow` already use), not fetched separately.
- A category with zero matching listings must render `"0 annoncer"` and `"–"` for its average price — never `NaN`, `undefined`, or `"0 kr"`.
- The green accent color for the price stat must come from the site's existing brand green family (`#1B4332`) — use `#4ade80` (a brighter/lighter shade of the same hue, chosen for contrast against the new dark card background) for the actual stat value text, applied to the price value only (not the count).
- The six existing embedded base64 category photos must be reused byte-for-byte unchanged — never retyped, re-encoded, or read through a tool that requires loading the whole line into context (this file has six single lines around 200-350KB each).
- `filterCategory(type)` navigation behavior (click a card → `listings.html?type=...`) must be unchanged.
- This repo's `index.html` is CRLF throughout — that's the actual pre-existing convention here, not a defect (confirmed by a prior task review in this project). Don't "fix" it to LF.
- Category order is fixed: Cruiser, Sport, Adventure, Naked Bike, Retro, Touring (same order as today's static markup).

---

## File Structure

Only one file changes:

- **Modify:** `c:/Users/pasca/bikerbasen-homepage/index.html`
  - CSS (`<style>` block, lines ~105-116): replace the old flat-white `.category-card`/`.category-card-title` rules with dark stat-card rules; add `.row-arrows`/`.arrow-btn` rules for the new nav buttons.
  - Markup (`<body>`, lines ~366-378): the "Populære kategorier" section's header gets two new arrow buttons; its six static `.category-card` divs are replaced by one empty `<div id="categoryRow">` that JS fills in.
  - Script (`<script>` block): a new `CATEGORY_IMAGES` data map (six entries, inserted by a script in Task 1 — never typed by hand), a new `CATEGORY_ORDER` array, three new functions (`categoryCardHTML`, `renderCategoryRow`, `scrollCategoryRow`), and one new call from `init()`.

A temporary, **uncommitted** helper script (`scratch-extract-categories.js`) is created in Task 1 to perform the image extraction safely, then deleted before committing — same technique used successfully in the prior homepage-rows feature's Task 1.

---

### Task 1: Extract category images into a JS data map; replace static markup with an empty container

**Files:**
- Modify: `index.html` (hero section markup, and the script block's top)
- Create (temporary, not committed): `scratch-extract-categories.js` in the repo root

**Interfaces:**
- Consumes: nothing new
- Produces: `const CATEGORY_IMAGES = { "Cruiser": "data:image/png;base64,...", "Sport": "...", "Adventure": "...", "Naked Bike": "...", "Retro": "...", "Touring": "..." };` inserted into the script (exact six keys, exact existing image data as values — Task 3 depends on this object existing with these exact keys). Also produces an empty `<div class="popular-categories scroll-row" id="categoryRow"></div>` replacing the old six static cards, and two new arrow buttons in the section header (Task 3 depends on `id="categoryRow"` existing, and on the arrow buttons' `onclick` handlers calling a function named exactly `scrollCategoryRow`, which Task 3 will define).

- [ ] **Step 1: Confirm current structure before touching anything**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
grep -n 'class="popular-categories scroll-row"' index.html
grep -c 'class="category-card" onclick="filterCategory' index.html
grep -n 'const motorcycles = \[' index.html
```
Expected: exactly one match for the first grep, `6` for the second, and exactly one match for the third (the line `const motorcycles = [`). If any of these don't match, stop — the file doesn't match what this plan assumes; read more context with the Read tool (using `offset`/`limit` on a *small* range — do not Read the whole file, it will exceed the token limit because of the embedded images) before proceeding.

- [ ] **Step 2: Write the extraction script**

Create `scratch-extract-categories.js` in the repo root with this exact content:

```js
const fs = require('fs');

const filePath = 'index.html';
const lines = fs.readFileSync(filePath, 'utf8').split('\n');

const containerOpenIdx = lines.findIndex(l => l.trim() === '<div class="popular-categories scroll-row">');
if (containerOpenIdx === -1) {
  throw new Error('Could not find the popular-categories scroll-row container - has index.html already been changed?');
}

const cardLineRe = /<div class="category-card" onclick="filterCategory\('([^']+)'\)"><img src="(data:image\/[a-zA-Z]+;base64,[^"]+)"[^>]*><div class="category-card-title">[^<]*<\/div><\/div>/;

const images = {};
const order = [];
let i = containerOpenIdx + 1;
while (true) {
  const line = lines[i];
  const m = line && line.match(cardLineRe);
  if (!m) break;
  images[m[1]] = m[2];
  order.push(m[1]);
  i++;
}

if (order.length !== 6) {
  throw new Error('Expected 6 category cards, found ' + order.length + ': ' + order.join(', '));
}

const containerCloseIdx = i;
if (lines[containerCloseIdx].trim() !== '</div>') {
  throw new Error('Expected closing </div> for popular-categories at line ' + (containerCloseIdx + 1) + ', found: ' + JSON.stringify(lines[containerCloseIdx]));
}

// Replace the whole block (open div + 6 cards + close div) with one empty JS-rendered container.
lines.splice(containerOpenIdx, containerCloseIdx - containerOpenIdx + 1,
  '                    <div class="popular-categories scroll-row" id="categoryRow"></div>');

// Add arrow buttons right after the "Populære kategorier" header title.
const headerTitleIdx = lines.findIndex(l => l.includes('<h2 class="homepage-row-title">Populære kategorier</h2>'));
if (headerTitleIdx === -1) {
  throw new Error('Could not find the Populære kategorier header title line');
}
lines.splice(headerTitleIdx + 1, 0,
  '            <div class="row-arrows">',
  '                <button class="arrow-btn" onclick="scrollCategoryRow(-1)" aria-label="Forrige">←</button>',
  '                <button class="arrow-btn" onclick="scrollCategoryRow(1)" aria-label="Næste">→</button>',
  '            </div>');

// Insert the CATEGORY_IMAGES map right before the motorcycles array.
const motoIdx = lines.findIndex(l => l.trim() === 'const motorcycles = [');
if (motoIdx === -1) {
  throw new Error('Could not find "const motorcycles = [" anchor');
}
const imagesLines = ['        const CATEGORY_IMAGES = {'];
order.forEach((name, idx) => {
  const comma = idx < order.length - 1 ? ',' : '';
  imagesLines.push('            ' + JSON.stringify(name) + ': ' + JSON.stringify(images[name]) + comma);
});
imagesLines.push('        };');
imagesLines.push('');
lines.splice(motoIdx, 0, ...imagesLines);

fs.writeFileSync(filePath, lines.join('\n'));
console.log('OK: extracted ' + order.length + ' category images: ' + order.join(', '));
```

Note: the `←`/`→` escapes are the ← and → arrow characters, written as escapes here so this plan document stays plain ASCII — the script will write the literal arrow characters into `index.html`.

- [ ] **Step 3: Run the script**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
node scratch-extract-categories.js
```
Expected output: `OK: extracted 6 category images: Cruiser, Sport, Adventure, Naked Bike, Retro, Touring`
If it throws an error instead, stop and investigate before proceeding — do not run it a second time on an already-modified file (it will correctly fail at Step 1's check inside the script, since the container it looks for no longer exists in its original form).

- [ ] **Step 4: Verify the extraction**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
grep -c 'class="category-card" onclick="filterCategory' index.html
grep -c 'id="categoryRow"' index.html
grep -c 'CATEGORY_IMAGES' index.html
grep -c 'scrollCategoryRow' index.html
```
Expected: `0`, `1`, `1`, `2` (the two arrow buttons' `onclick` attributes — `scrollCategoryRow` itself isn't defined as a function yet, that's Task 3; clicking the buttons right now would throw, which is fine since this is an intermediate state).

Then verify the six images survived intact and the six keys are exactly right, without ever loading the huge values into your own context:
```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const m = html.match(/const CATEGORY_IMAGES = \{([\s\S]*?)\n        \};/);
if (!m) { console.log('FAIL: CATEGORY_IMAGES block not found'); process.exit(1); }
const obj = eval('({' + m[1] + '})');
const keys = Object.keys(obj);
console.log('KEYS=' + JSON.stringify(keys));
console.log('ALL_START_WITH_DATA_URI=' + keys.every(k => obj[k].startsWith('data:image/png;base64,')));
console.log('LENGTHS=' + JSON.stringify(keys.map(k => obj[k].length)));
"
```
Expected: `KEYS=["Cruiser","Sport","Adventure","Naked Bike","Retro","Touring"]`, `ALL_START_WITH_DATA_URI=true`, and six lengths each in the tens-of-thousands-of-characters range (these are the same large images from before — exact lengths aren't predictable in advance, just confirm none is `0` or suspiciously short).

- [ ] **Step 5: Delete the temporary script**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
rm scratch-extract-categories.js
```
This must never be committed — it was a one-time migration tool.

- [ ] **Step 6: Commit**

```bash
cd "c:/Users/pasca/bikerbasen-homepage"
git add index.html
git status
```
Confirm `scratch-extract-categories.js` is NOT listed. Then:
```bash
git commit -m "$(cat <<'EOF'
Extract category images into a JS data map for dark-card redesign

Moves the six embedded category photos out of static markup into a
CATEGORY_IMAGES map, and replaces the static category cards with an
empty JS-rendered container plus arrow nav buttons. The row renders
nothing yet - rendering logic lands in a follow-up commit.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Dark stat-card CSS

**Files:**
- Modify: `index.html` (CSS block)

**Interfaces:**
- Consumes: nothing new
- Produces: CSS classes `.category-card` (redefined), `.category-card-img`, `.category-card-body`, `.category-card-name`, `.category-card-count`, `.category-card-divider`, `.category-card-stats`, `.category-stat-label`, `.category-stat-value` (and `.category-stat-value.accent`), `.row-arrows`, `.arrow-btn`. Task 3's HTML template depends on these exact class names.

- [ ] **Step 1: Locate the current category CSS rules**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
grep -n '\.category-card\|\.category-card-title' index.html
```
Read the matched lines with the Read tool (small range, e.g. `offset` a few lines before/after) to confirm they match exactly what's shown in Step 2 below before editing — if the file has diverged, stop and re-read more context rather than forcing the edit.

- [ ] **Step 2: Replace the CSS**

Use the Edit tool. Find:

```
        .category-card { background: white; padding: 0; overflow: hidden; border-radius: 10px; text-align: center; cursor: pointer; box-shadow: 0 2px 8px rgba(0,0,0,0.12); transition: all 0.3s; position: relative; }
        .category-card:hover { box-shadow: 0 6px 20px rgba(0,0,0,0.2); transform: translateY(-3px); }
        .category-card:hover img { transform: scale(1.05); }
        .category-card img { width: 100%; height: 160px; object-fit: cover; display: block; transition: transform 0.4s ease; }
        .category-card-title { position: absolute; bottom: 0; left: 0; right: 0; color: white; font-weight: 700; font-size: 16px; padding: 28px 16px 16px; background: linear-gradient(to top, rgba(0,0,0,0.75) 0%, rgba(0,0,0,0) 100%); letter-spacing: 0.5px; }
```

Replace with:

```
        .category-card { background: #1a1d24; padding: 0; overflow: hidden; border-radius: 12px; text-align: left; cursor: pointer; box-shadow: 0 2px 8px rgba(0,0,0,0.12); transition: all 0.3s; position: relative; }
        .category-card:hover { box-shadow: 0 6px 20px rgba(0,0,0,0.2); transform: translateY(-3px); }
        .category-card:hover .category-card-img { transform: scale(1.05); }
        .category-card-img { width: 100%; height: 120px; object-fit: cover; display: block; transition: transform 0.4s ease; filter: brightness(0.7); }
        .category-card-body { padding: 0.9rem 1rem; }
        .category-card-name { color: white; font-weight: 700; font-size: 15px; margin-bottom: 4px; }
        .category-card-count { color: #9a9ea7; font-size: 12px; margin-bottom: 0.75rem; }
        .category-card-divider { border-top: 1px solid #2c2f38; margin-bottom: 0.75rem; }
        .category-card-stats { display: flex; justify-content: space-between; }
        .category-stat-label { color: #9a9ea7; font-size: 10px; text-transform: uppercase; margin-bottom: 3px; letter-spacing: 0.5px; }
        .category-stat-value { color: white; font-weight: 700; font-size: 13px; }
        .category-stat-value.accent { color: #4ade80; }
        .row-arrows { display: flex; gap: 0.5rem; }
        .arrow-btn { width: 32px; height: 32px; border-radius: 50%; background: white; border: 1px solid #ddd; display: flex; align-items: center; justify-content: center; cursor: pointer; font-size: 16px; color: #666; transition: background 0.2s; }
        .arrow-btn:hover { background: #f0f0f0; }
```

- [ ] **Step 3: Commit**

```bash
cd "c:/Users/pasca/bikerbasen-homepage"
git add index.html
git commit -m "$(cat <<'EOF'
Add dark stat-card CSS for Populære kategorier

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

(No functional verification in this task — the new classes aren't applied to any rendered element until Task 3. A visual/functional check happens at the end of Task 3.)

---

### Task 3: Render category cards with live stats; wire up arrows

**Files:**
- Modify: `index.html` (script block: new data/functions, and the `init()` function)

**Interfaces:**
- Consumes: `CATEGORY_IMAGES` (from Task 1, exact keys: `"Cruiser"`, `"Sport"`, `"Adventure"`, `"Naked Bike"`, `"Retro"`, `"Touring"`), the existing global `motorcycles` array, the existing `filterCategory(type)` function, the CSS classes from Task 2.
- Produces: `function renderCategoryRow()` (no params, renders into `#categoryRow`), `function scrollCategoryRow(direction)` (takes `-1` or `1`), called from `init()`. The two arrow buttons added in Task 1 already call `scrollCategoryRow(-1)` / `scrollCategoryRow(1)` — this task makes that function exist.

- [ ] **Step 1: Locate `renderLatestRow` and `init`**

Run:
```bash
cd "c:/Users/pasca/bikerbasen-homepage"
grep -n 'function renderLatestRow\|function init()' index.html
```
Read both with the Read tool (small ranges) and confirm they match what's shown in Steps 2 and 3 below before editing.

- [ ] **Step 2: Add the category rendering functions**

Use the Edit tool. Find (this is the existing `renderLatestRow` function, unchanged by prior tasks):

```javascript
        function renderLatestRow() {
            const favs = getFavorites();
            const latest = [...motorcycles].sort((a, b) => b.id - a.id).slice(0, 10);
            document.getElementById('latestListingsRow').innerHTML = latest.map(m => cardHTML(m, favs)).join('');
        }
```

Replace with (keeps it unchanged, adds three new functions after it):

```javascript
        function renderLatestRow() {
            const favs = getFavorites();
            const latest = [...motorcycles].sort((a, b) => b.id - a.id).slice(0, 10);
            document.getElementById('latestListingsRow').innerHTML = latest.map(m => cardHTML(m, favs)).join('');
        }

        function categoryCardHTML(type) {
            const matches = motorcycles.filter(m => m.type === type);
            const count = matches.length;
            const avgPrice = count > 0 ? Math.round(matches.reduce((sum, m) => sum + m.price, 0) / count) : null;
            const priceText = avgPrice !== null ? avgPrice.toLocaleString('da-DK') + ' kr' : '–';
            return `
            <div class="category-card" onclick="filterCategory('${type}')">
                <img class="category-card-img" src="${CATEGORY_IMAGES[type]}" alt="${type}">
                <div class="category-card-body">
                    <div class="category-card-name">${type}</div>
                    <div class="category-card-count">${count} annoncer</div>
                    <div class="category-card-divider"></div>
                    <div class="category-card-stats">
                        <div><div class="category-stat-label">Annoncer</div><div class="category-stat-value">${count}</div></div>
                        <div><div class="category-stat-label">Gns. pris</div><div class="category-stat-value accent">${priceText}</div></div>
                    </div>
                </div>
            </div>`;
        }

        function renderCategoryRow() {
            document.getElementById('categoryRow').innerHTML = CATEGORY_ORDER.map(categoryCardHTML).join('');
        }

        function scrollCategoryRow(direction) {
            document.getElementById('categoryRow').scrollBy({ left: direction * 236, behavior: 'smooth' });
        }
```

- [ ] **Step 3: Define `CATEGORY_ORDER` and call `renderCategoryRow()` from `init()`**

Use the Edit tool. Find:

```javascript
        const motorcycles = [
```

Replace with:

```javascript
        const CATEGORY_ORDER = ['Cruiser', 'Sport', 'Adventure', 'Naked Bike', 'Retro', 'Touring'];

        const motorcycles = [
```

Then use the Edit tool again. Find:

```javascript
        function init() {
            renderBrands();
            displayListings(motorcycles);
            renderLatestRow();
        }
```

Replace with:

```javascript
        function init() {
            renderBrands();
            displayListings(motorcycles);
            renderLatestRow();
            renderCategoryRow();
        }
```

- [ ] **Step 4: Verify with a Playwright check**

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

  const cardCount = await page.locator('#categoryRow .category-card').count();
  console.log('CATEGORY_CARD_COUNT=' + cardCount);

  // Recompute expected stats from the page's own live data - don't hardcode numbers here,
  // they'll drift as Supabase/demo data changes.
  const expected = await page.evaluate(() => {
    const out = {};
    CATEGORY_ORDER.forEach(type => {
      const matches = motorcycles.filter(m => m.type === type);
      const count = matches.length;
      const avg = count > 0 ? Math.round(matches.reduce((s, m) => s + m.price, 0) / count) : null;
      out[type] = { count, avg };
    });
    return out;
  });
  console.log('EXPECTED=' + JSON.stringify(expected));

  const rendered = await page.evaluate(() => {
    return Array.from(document.querySelectorAll('#categoryRow .category-card')).map(card => ({
      name: card.querySelector('.category-card-name').textContent,
      count: card.querySelector('.category-stat-value').textContent,
      price: card.querySelector('.category-stat-value.accent').textContent,
    }));
  });
  console.log('RENDERED=' + JSON.stringify(rendered));

  // Click a card, confirm navigation.
  await page.locator('#categoryRow .category-card').first().click();
  console.log('AFTER_CLICK_URL=' + page.url());

  console.log('ERRORS=' + JSON.stringify(errors));
  await browser.close();
})();
"
```
Expected:
- `CATEGORY_CARD_COUNT=6`
- For each category in `RENDERED`, its `count` matches `EXPECTED[name].count`, and its `price` matches either `EXPECTED[name].avg.toLocaleString('da-DK') + ' kr'` (if `avg` isn't null) or `'–'` (if `avg` is null — this is the zero-listing case, e.g. Touring with no demo/Supabase listings).
- `AFTER_CLICK_URL` ends in `listings.html?type=...` for whichever category was first in `CATEGORY_ORDER` (Cruiser).
- `ERRORS=[]`

If any category shows `NaN`, `undefined`, or `0 kr` instead of `–` for a zero-listing category, the null-check in `categoryCardHTML` isn't working — fix it before proceeding.

- [ ] **Step 5: Verify the arrow buttons scroll the row**

Run:
```bash
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto('file://c:/Users/pasca/bikerbasen-homepage/index.html', { waitUntil: 'load' });
  await page.waitForTimeout(5000);
  const before = await page.locator('#categoryRow').evaluate(el => el.scrollLeft);
  await page.locator('.arrow-btn').nth(1).click(); // second arrow = right/forward
  await page.waitForTimeout(500);
  const after = await page.locator('#categoryRow').evaluate(el => el.scrollLeft);
  console.log('SCROLL_BEFORE=' + before + ' SCROLL_AFTER=' + after);
})();
"
```
Expected: `SCROLL_AFTER` is greater than `SCROLL_BEFORE` (the row scrolled right). If they're equal, the arrow button's `onclick` or `scrollCategoryRow` isn't wired correctly.

- [ ] **Step 6: Commit**

```bash
cd "c:/Users/pasca/bikerbasen-homepage"
git add index.html
git commit -m "$(cat <<'EOF'
Render Populære kategorier as live dark stat-cards

Each category card now shows a live count and average price computed
from the current motorcycles data (same data source as the existing
Nyeste annoncer row), with zero-listing categories showing "0
annoncer" / "–" instead of NaN or a misleading 0 kr. Arrow buttons in
the section header scroll the row left/right.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Final verification and push

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
  console.log('CATEGORY_CARDS=' + await page.locator('#categoryRow .category-card').count());
  console.log('LATEST_ROW_CARDS=' + await page.locator('#latestListingsRow .listing-card').count());
  console.log('GRID_CARDS=' + await page.locator('#listingsContainer .listing-card').count());
  console.log('UNEXPECTED_ERRORS=' + JSON.stringify(errors));
  await browser.close();
})();
"
```
Expected: `CATEGORY_CARDS=6`, `LATEST_ROW_CARDS=10`, `GRID_CARDS` at least `16` (the existing "Nyeste annoncer" and "Alle annoncer" rows must be unaffected by this change), `UNEXPECTED_ERRORS=[]`.

If any of these don't match, do not push — go back to the relevant task and fix it first.

- [ ] **Step 2: Push to GitHub**

```bash
cd "c:/Users/pasca/bikerbasen-homepage"
git log --oneline -5
git push origin main
```
Confirm the push succeeds and shows the three commits from Tasks 1-3.

- [ ] **Step 3: Verify on the live deployed site**

Wait about a minute for GitHub Pages to redeploy, then poll the live URL (GitHub Pages can take longer than expected — don't give up after one check):
```bash
i=0; until [ $i -ge 8 ]; do
  result=$(node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  await page.goto('https://gulecitronxx.github.io/bikerbasen/', { waitUntil: 'load', timeout: 30000 });
  await page.waitForTimeout(6000);
  console.log('CATEGORY_CARDS=' + await page.locator('#categoryRow .category-card').count());
  await browser.close();
})();
" 2>&1)
  echo "[attempt $i] $result"
  if echo "$result" | grep -q "CATEGORY_CARDS=6"; then echo READY; break; fi
  i=$((i+1))
  sleep 15
done
```
Expected: `CATEGORY_CARDS=6` within the polling window. Once confirmed, also re-run Step 1's full script against the live URL instead of the local file to confirm `LATEST_ROW_CARDS=10` and `GRID_CARDS` still check out live, not just locally.
