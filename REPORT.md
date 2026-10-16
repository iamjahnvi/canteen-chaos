## CC-01 — "The search suggestions are behind everything"

**Reproduced**: Typed 2+ characters in the menu search box. The suggestion
dropdown appeared but only its top ~4px sliver was clickable; the rest was
visible yet dead — clicks fell through to whatever was underneath.

**Cause:** A CSS stacking-context trap in `frontend/style.css`. The dropdown
lives inside `.search-wrap` (`position: relative; z-index: 1`), but its sibling
`.cat-tabs` is `position: relative; z-index: 40` with an opaque background and
comes later in the DOM. The dropdown's own `z-index: 100` cannot escape its
parent's `z-index: 1` stacking context, so the whole suggestion list painted
*below* the category bar.

**Fix:** Raised `.search-wrap` to `z-index: 60` and lowered `.cat-tabs` to
`z-index: 1`. The parent filter bar (`z-index: 40`) was already above the menu
grid, so no other layer needed to move.

**Checked:** Dropdown now paints above the category bar and the dish grid; every
row is clickable. No other header/filter overlap changed.

**Time:** about 25 minutes, most of it confirming the parent-context trap rather
than the dropdown's own z-index.

**Code previous** (`frontend/style.css`):

```css
.search-wrap { position: relative; z-index: 1; flex: 1; min-width: 200px; }
```

```css
.cat-tabs {
  display: flex;
  gap: 6px;
  overflow-x: auto;
  padding: 12px 16px 12px;
  scrollbar-width: thin;
  position: relative;
  z-index: 40;
  background: var(--orange-soft);
}
```

**Code new** (`frontend/style.css` — what changed is marked in the comments):

```css
.search-wrap { position: relative; z-index: 60; flex: 1; min-width: 200px; } /* FIX CC-01: suggestions must paint above cat-tabs/menu (was z-index:1, trapped below .cat-tabs z:40) */
```

```css
.cat-tabs {
  display: flex;
  gap: 6px;
  overflow-x: auto;
  padding: 12px 16px 12px;
  scrollbar-width: thin;
  position: relative;
  z-index: 1; /* FIX CC-01: was z-index:40, painted over the suggest dropdown (search-wrap z:60 now wins) */
  background: var(--orange-soft);
}
```

---

## CC-02 — "Can't read anything in dark mode"

**Reproduced:** Opened the site with `prefers-color-scheme: dark` (or toggled
Dark). Dish names and prices were near-invisible dark-on-dark; the grey
category/rating line under the name rendered fine.

**Cause:** `.dish-body` in `frontend/style.css` hardcoded `color: #2b2118`
(the light-theme ink). In dark mode the card background becomes `#201913` via
`var(--card)`, so name + price (which inherit the body colour) turned into dark
text on a dark card. The meta line looked fine because it uses
`var(--muted)`, which *does* adapt.

**Fix:** One line — `.dish-body` now uses `color: var(--ink)`, which flips with
the theme like everything else.

**Checked:** Light mode unchanged (`--ink` is `#2b2118` there); dark mode names
and prices render in `#f2e8df` and are readable.

**Time:** about 10 minutes.

**Code previous** (`frontend/style.css`):

```css
.dish-body { padding: 12px; display: flex; flex-direction: column; gap: 7px; flex: 1; color: #2b2118; }
```

**Code new** (`frontend/style.css` — what changed is marked in the comment):

```css
.dish-body { padding: 12px; display: flex; flex-direction: column; gap: 7px; flex: 1; color: var(--ink); } /* FIX CC-02: was hardcoded #2b2118, invisible on dark card #201913; use theme ink */
```

---

## CC-03 — "The menu is wider than my phone"

**Reproduced:** Narrow viewport (≤760px, worst on phones). Page scrolled
sideways and the right column's Add to Cart buttons were cut off. Longest with
long dish names (e.g. "Paneer Butter Masala with Butter Naan").

**Cause:** Two rules in the `@media (max-width: 760px)` block of
`frontend/style.css`: `.name-btn { width: max-content; }` forbade the dish name
from wrapping, so each card grew to the full intrinsic width of its longest
name and forced the grid (and page) wider than the viewport. On top of that,
`.dish-card img { width: auto; min-width: 100%; }` let images overflow their
card instead of shrinking with it.

**Fix:** `.name-btn` becomes fluid (`width: auto; max-width: 100%;
white-space: normal; overflow-wrap: break-word`) so names wrap inside the card;
cards get `min-width: 0; max-width: 100%` (grid children default to
`min-width: auto`, which also blocks shrinking); images become `width: 100%;
min-width: 0`.

**Checked:** At 360px wide the grid fits with no sideways scroll; long names
wrap; images stay inside their cards.

**Time:** about 30 minutes.

**Code previous** (`frontend/style.css`):

```css
@media (max-width: 760px) {
  .menu-grid { grid-template-columns: repeat(2, 1fr); }
  .dish-name { font-size: 17px; }
  .name-btn { width: max-content; }
  .dish-card img { width: auto; min-width: 100%; height: 170px; aspect-ratio: 3 / 2; }
```

**Code new** (`frontend/style.css` — what changed is marked in the comments):

```css
@media (max-width: 760px) {
  .menu-grid { grid-template-columns: repeat(2, 1fr); }
  .dish-name { font-size: 17px; }
  /* FIX CC-03: .name-btn was width:max-content (forced card wider than viewport, sideways scroll).
     Let the name wrap inside the card instead. */
  .name-btn { width: auto; max-width: 100%; white-space: normal; overflow-wrap: break-word; }
  /* FIX CC-03: img was width:auto + min-width:100%, could overflow the card. Fluid image instead. */
  .dish-card { min-width: 0; max-width: 100%; }
  .dish-card img { width: 100%; min-width: 0; height: 170px; aspect-ratio: 3 / 2; }
```

---

## CC-04 — "The buttons don't work on my tablet"

**Reproduced:** Only in the 761–900px viewport band (tablets). Add to Cart and
the ★ favourite buttons looked normal but no click registered — on laptop or
phone widths everything worked.

**Cause:** The `@media (min-width: 761px) and (max-width: 900px)` block in
`frontend/style.css` adds invisible `::after` overlays: a 58px-tall strip over
the card bottom (exactly where Add to Cart / steppers sit) and a 52×52 block
over the image's top-right corner (exactly where the ★ button sits). They have
no `pointer-events: none`, so they swallow every tap while looking like nothing.

**Fix:** Added `pointer-events: none` to both `::after` overlays so clicks pass
through to the real buttons. Visuals untouched (backgrounds were already
transparent).

**Checked:** Buttons clickable again at 800px wide; laptop/phone widths
unaffected (the media query doesn't apply there).

**Time:** about 15 minutes.

**Code previous** (`frontend/style.css`):

```css
  .dish-card::after,
  .img-wrap::after {
    content: "";
    position: absolute;
    background: transparent;
    transition: background 0.2s;
  }
```

**Code new** (`frontend/style.css` — what changed is marked in the comment):

```css
  .dish-card::after,
  .img-wrap::after {
    content: "";
    position: absolute;
    background: transparent;
    transition: background 0.2s;
    pointer-events: none; /* FIX CC-04: invisible 58px/52px overlays sat over Add-to-Cart + fav star and swallowed all taps on tablet */
  }
```

---

## CC-05 — "The category bar scrolls away on my phone"

**Reproduced:** On ≤480px viewports the filter + category bar scrolled off with
the page instead of sticking to the top like on desktop.

**Cause:** The `@media (max-width: 480px)` block sets `.view { overflow-x:
hidden; }`. Any non-visible `overflow` on an ancestor makes it a scroll
container, and `position: sticky` descendants (here `.filters`) then stick
within *that* container instead of the viewport — i.e. they scroll away.

**Fix:** Changed to `overflow-x: clip`, which clips sideways overflow the same
way but does *not* create a scroll container, so `sticky` keeps working. This is
the right place for the fix (rather than un-sticking the bar), because the
clipping is still needed for CC-03-style overflow on small screens.

**Checked:** Category bar stays pinned while scrolling on a 360px viewport;
no sideways scrolling reappeared.

**Time:** about 20 minutes.

**Code previous** (`frontend/style.css`):

```css
@media (max-width: 480px) {
  .view { overflow-x: hidden; }
```

**Code new** (`frontend/style.css` — what changed is marked in the comment):

```css
@media (max-width: 480px) {
  /* FIX CC-05: overflow-x:hidden creates a scroll container that disables position:sticky,
     so the filter/category bar scrolled away on phones. overflow-x:clip clips without breaking sticky. */
  .view { overflow-x: clip; }
```

---

## CC-06 — "I ordered more than they had"

**Reproduced:** Dish with stock 2 (e.g. French Fries showed "Only 2 left").
Ordered qty 5 → `POST /api/orders` returned 201 and stock went negative. The
cart even warns ("cart quantity exceeds stock") but the server allowed it.

**Cause:** Two gaps in `backend/logic/validation.js`. `validateLine()` only
rejected `stock <= 0` (sold out) and never compared `qty` against `stock`; and
`reserveStock()` subtracted blindly with no sufficiency check, so concurrent
requests could also both take the last plate.

**Fix:** Added a `qty > stock` rejection (`"<name> has only <n> left"`) in
`validateLine()`, and made `reserveStock()` re-check `item.qty > dish.stock`
per line and return `{ ok: false, ... }` (which the route already turns into a
409) instead of driving stock negative.

**Checked:** Ordering 5 of a dish with 2 left now returns 400 with "has only 2
left"; `smoke.js` "qty over stock rejected" passes; legitimate orders still
pass.

**Time:** about 30 minutes.

**Code previous** (`backend/logic/validation.js`):

```js
  const blocked = blockedReason(dish, now);
  if (blocked && isServedNow(dish, now) === false) errors.push(`${dish.name}: ${blocked}`);
  else if (dish.disabled) errors.push(`${dish.name} is off the menu today`);
  else if (Number(dish.stock) <= 0) errors.push(`${dish.name} is sold out`);

  return errors;
```

```js
  items.forEach((item) => {
    const dish = next.find((d) => d.id === Number(item.dishId));
    if (!dish) return errors.push(`dish ${item.dishId} vanished`);
    dish.stock -= item.qty;
  });
```

**Code new** (`backend/logic/validation.js` — what changed is marked in the comments):

```js
  const blocked = blockedReason(dish, now);
  if (blocked && isServedNow(dish, now) === false) errors.push(`${dish.name}: ${blocked}`);
  else if (dish.disabled) errors.push(`${dish.name} is off the menu today`);
  else if (Number(dish.stock) <= 0) errors.push(`${dish.name} is sold out`);
  // FIX CC-06: previously only sold-out (stock<=0) was rejected, so ordering 5 when only 2 left passed validation.
  else if (qty > Number(dish.stock)) errors.push(`${dish.name} has only ${dish.stock} left`);

  return errors;
```

```js
  items.forEach((item) => {
    const dish = next.find((d) => d.id === Number(item.dishId));
    if (!dish) return errors.push(`dish ${item.dishId} vanished`);
    // FIX CC-06: re-check stock at reserve time so two concurrent requests cannot both take the last plate.
    if (Number(item.qty) > Number(dish.stock)) {
      errors.push(`${dish.name} has only ${dish.stock} left`);
      return;
    }
    dish.stock -= item.qty;
  });
```

---

## CC-07 — "Cancelling makes it worse"

**Reproduced:** Placed an order for 1 unit (stock N → N−1), then cancelled it
via `PATCH /api/orders/:token/status { "status": "cancelled" }`. Stock went to
N−2 instead of back to N. Repeated cancels drained the menu while the kitchen
was full.

**Cause:** Sign error in `releaseStock()` in `backend/logic/validation.js`:
it did `dish.stock - line.qty` (taking stock off *again*) instead of adding
the cancelled quantity back.

**Fix:** One character — `-` → `+`.

**Checked:** Cancel now restores stock exactly (`smoke.js` "cancel puts stock
back" passes: mid-stock + 1); placing still decrements.

**Time:** about 5 minutes.

**Code previous** (`backend/logic/validation.js`):

```js
/** Put stock back when an order is cancelled. */
function releaseStock(items, menu) {
  return menu.map((dish) => {
    const line = items.find((i) => Number(i.dishId) === dish.id);
    if (!line) return dish;
    return { ...dish, stock: dish.stock - line.qty };
  });
}
```

**Code new** (`backend/logic/validation.js` — what changed is marked in the comment):

```js
/** Put stock back when an order is cancelled. */
function releaseStock(items, menu) {
  return menu.map((dish) => {
    const line = items.find((i) => Number(i.dishId) === dish.id);
    if (!line) return dish;
    return { ...dish, stock: dish.stock + line.qty }; // FIX CC-07: was `- line.qty`, cancelling reduced stock again instead of restoring it
  });
}
```

---

## CC-08 — "An old coupon still works"

**Reproduced:** `POST /api/cart/quote` with `couponCode: "FRESHERS24"`
(`expiresAt: 2024-09-30`) returned a Rs. 30 discount with no error, and an order
placed with it kept the discount.

**Cause:** `applyCoupon()` in `backend/logic/pricing.js` checked
`usesLeft`, `minOrder`, `vegOnly` and `slots` — but never `expiresAt`. Only the
`GET /api/coupons` *listing* filtered by expiry, so the coupon was hidden from
the hints yet still honoured at billing time.

**Fix:** Added an expiry check in `applyCoupon()` (the single choke point both
quote and order placement go through): if `new Date(coupon.expiresAt) <= now`,
return `{ valid: false, ..., reason: "<CODE> has expired" }`. The route already
rejects orders carrying `couponError`, so both quote and placement are covered.

**Checked:** `FRESHERS24` now yields `couponDiscount: 0` + "FRESHERS24 has
expired"; valid coupons (BYTE10 etc.) still discount; `smoke.js` "expired
coupon refused" and "only usable coupons listed" pass.

**Time:** about 15 minutes.

**Code previous** (`backend/logic/pricing.js`):

```js
  if (coupon.usesLeft <= 0) {
    return { valid: false, discount: 0, reason: 'This coupon is fully used' };
  }
```

**Code new** (`backend/logic/pricing.js` — what changed is marked in the comment):

```js
  if (coupon.usesLeft <= 0) {
    return { valid: false, discount: 0, reason: 'This coupon is fully used' };
  }

  // FIX CC-08: expiry was never checked here (only the /api/coupons list filtered it),
  // so FRESHERS24 (expired 2024-09-30) still gave a discount at quote/order time.
  if (coupon.expiresAt && new Date(coupon.expiresAt) <= now) {
    return { valid: false, discount: 0, reason: `${coupon.code} has expired` };
  }
```

---

## CC-09 — "The menu shows more dishes than it should"

**Reproduced:** `GET /api/menu?limit=5&page=1` returned the full filtered list
(30+ dishes) instead of 5; `pages`/`hasMore` were computed correctly but
meaningless. The frontend asks `limit=5` with Load-more + infinite scroll, so
the whole menu dumped at once and paging never engaged.

**Cause:** In `paginate()` in `backend/logic/search.js` the slice was computed
into `items` but the return statement said `items: list` — the uncut array.

**Fix:** Return the sliced `items` instead of `list`.

**Checked:** Page 1 now holds 5 dishes, page 2 differs, `pages ===
ceil(total/5)`; `smoke.js` "pages differ" and "page count is right" pass.

**Time:** about 10 minutes.

**Code previous** (`backend/logic/search.js`):

```js
  return {
    items: list,
    page: pageNum,
    limit: perPage,
    total: list.length,
    pages: Math.ceil(list.length / perPage) || 1,
    hasMore: start + perPage < list.length,
  };
```

**Code new** (`backend/logic/search.js` — what changed is marked in the comment):

```js
  return {
    items, // FIX CC-09: was `items: list` (returned the whole menu, ignoring page/limit, so infinite-scroll never paged)
    page: pageNum,
    limit: perPage,
    total: list.length,
    pages: Math.ceil(list.length / perPage) || 1,
    hasMore: start + perPage < list.length,
  };
```

---

## CC-10 — "Sorting by price is backwards"

**Reproduced:** `GET /api/menu?sort=price-asc` returned most-expensive first;
`price-desc` returned cheapest first. Exactly swapped, both directions.

**Cause:** The two comparators in `SORTERS` in `backend/logic/search.js` were
transposed: `'price-asc'` did `b.price - a.price` (descending) and
`'price-desc'` did `a.price - b.price`.

**Fix:** Swapped them back.

**Checked:** `price-asc` now yields a non-decreasing sequence, `price-desc`
non-increasing; `smoke.js` "sort by price works" passes; other sorters
untouched.

**Time:** about 5 minutes.

**Code previous** (`backend/logic/search.js`):

```js
const SORTERS = {
  'price-asc': (a, b) => b.price - a.price,
  'price-desc': (a, b) => a.price - b.price,
```

**Code new** (`backend/logic/search.js` — what changed is marked in the comment):

```js
const SORTERS = {
  'price-asc': (a, b) => a.price - b.price, // FIX CC-10: comparators were swapped (asc sorted desc and vice versa)
  'price-desc': (a, b) => b.price - a.price,
```

---

## Extra credit — "Ready orders could never be picked up"

**How I noticed it:** The full `smoke.js` run (open-canteen clock) failed 5
checks that had nothing to do with the 10 reported bugs: "ready -> picked
allowed", "history is recorded", and the three picked-only rating checks. All
five traced to one line.

**Reproduced:** `PATCH /api/orders/:token/status { "status": "picked" }` on a
`ready` order returned 409 "A ready order cannot change". Consequence: no order
could ever complete, and since ratings require `status === 'picked'`, nothing
could ever be rated either.

**Cause:** `TERMINAL` in `backend/logic/status.js` listed `'ready'` as a
terminal state. That made `allowedNext('ready')` return `[]` and
`canTransition('ready', ...)` refuse everything (except the `picked` carve-out,
which only helps orders *already* picked). `ready` is plainly a mid-flow state
— the food is cooked but not yet handed over.

**Fix:** Removed `'ready'` from `TERMINAL` (now `['picked', 'cancelled',
'refunded']`). `allowedNext('ready')` now yields `['picked']` via `FLOW`, and
`canTransition` allows it. No other behaviour changes: `queueLength` only
counts placed/accepted/preparing, and `autoAdvance` has no timer for `ready`,
so nothing starts moving on its own.

**Checked:** Full `smoke.js`: **90 passed, 0 failed**, including the rating
flow end to end. How I know it's safe: `git diff` confirms this file was
untouched by the CC-01–CC-10 fixes, so the 5 failures were pre-existing and the
fix introduces no interaction with them.

**Time:** about 20 minutes, mostly confirming the 5 failures shared one root
cause instead of being 5 separate bugs.

**Code previous** (`backend/logic/status.js`):

```js
const FLOW = ['placed', 'accepted', 'preparing', 'ready', 'picked'];
const TERMINAL = ['ready', 'picked', 'cancelled', 'refunded'];
```

**Code new** (`backend/logic/status.js` — what changed is marked in the comment):

```js
const FLOW = ['placed', 'accepted', 'preparing', 'ready', 'picked'];
const TERMINAL = ['picked', 'cancelled', 'refunded']; // FIX (extra): 'ready' is not terminal — food still has to be handed over. With 'ready' in here, allowedNext('ready') was [] so ready->picked was refused and ratings (picked-only) were unreachable.
```

---

## UI polish — "Recommended dropdown arrow too close to edge"

**Reproduced:** Opened the menu view and looked at the sort `<select id="sortSelect">` ("Recommended"). The dropdown arrow sat tight against the right edge of the pill, as in the screenshot.

**Cause:** `.search, .select` in `frontend/style.css` share `padding: 9px 14px` and use the browser-native `<select>` arrow. First attempt (`padding-right: 28px` only) did nothing visible — Chrome/Edge draw the native arrow outside the padding box, so padding cannot shift it. The arrow position is not controllable until the native arrow is replaced.

**Fix:** Hid the native arrow (`appearance: none`) and drew our own chevron via `background-image` (inline SVG), positioned with `background-position: right 18px center`. 18px from the edge = a bit left of the native ~8-10px, without crowding the "Recommended" label. Separate SVG stroke colours per theme (`#2b2118` light / `#f2e8df` dark) so it stays readable; `padding: 9px 38px 9px 16px` reserves room for it. To move it further left/right, change only the `18px` value.

**Checked:** `#sortSelect` now shows the chevron 18px in from the edge in both light and dark themes. Hard-refresh required (CSS cache). No other `.search` input affected; `.select.small` untouched.

**Time:** about 15 minutes, including finding out padding alone cannot move a native select arrow.

**Code previous** (`frontend/style.css`):

```css
.select { cursor: pointer; }
```

**Code new** (`frontend/style.css`):

```css
.select {
  cursor: pointer;
  appearance: none;
  -webkit-appearance: none;
  padding: 9px 38px 9px 16px;
  background-color: var(--card);
  background-image: url("data:image/svg+xml,...chevron #2b2118...");
  background-repeat: no-repeat;
  background-position: right 18px center;
  background-size: 12px;
}
html[data-theme="dark"] .select {
  background-image: url("data:image/svg+xml,...chevron #f2e8df...");
}
```

---

## Could not fix

Nothing outstanding from the reported list — all 10 reproduced, fixed, and
verified. One observation left as a note rather than a change: `POST /api/orders`
decrements coupon `usesLeft` with `{ ...c, usesLeft: c.usesLeft }` (a no-op,
`routes/orders.routes.js`), so coupons never run out. I deliberately did not
touch it: changing billing-adjacent counters without a dedicated test risks
breaking the "minimum order / cap" checks, and it was not reported. A one-line
fix (`usesLeft: c.usesLeft - 1`) plus a smoke assertion would be the follow-up.
