# Findings

13 defects found by reading the source, each confirmed with a test that asserts the *correct* behaviour and therefore **fails today**.
Every finding below has the same parts: what happens, steps to reproduce, what should happen, impact, the failing test, and then
the **original code**, the **updated code**, and **why** the update is the smallest correct fix.

- Failing tests: `tests/bugs/` (one file per finding, `F01`...`F13`). Run with `npm run test:bugs`.
- Every bug file also holds **control tests** that pass today (for example "an admin can still publish"), so a lazy fix that
  breaks normal behaviour cannot turn the file green.
- `npm run test:regression` runs everything else and should be green.
- The updated code is **not applied** to the main source, so the failing tests stay as evidence. It is in `fixes/fixes.patch`
  (11 files, about 110 lines added). To see the bug tests turn green: `git apply fixes/fixes.patch && npm run test:bugs`.

| # | Finding | Severity | Type |
|---|---|---|---|
| F01 | Public search answers for draft and expired catalogues | **High** | Security / data exposure |
| F02 | Public search returns the price of "Price on Request" products | **High** | Security / data exposure |
| F03 | Staff can delete a catalogue | **High** | Authorization |
| F04 | Staff can publish by creating a catalogue with "go live" | **High** | Authorization |
| F05 | Staff can publish by PATCHing `status` | **High** | Authorization |
| F06 | Listing PATCH/DELETE ignore the catalogue in the URL | Medium | Access control (IDOR) |
| F07 | Enquiries are accepted for draft and expired catalogues | Medium | Business rule |
| F08 | Enquiry reference collides and the buyer's lead is lost | **High** (grows with volume) | Data loss |
| F09 | Admin status badge is inverted (Live and Expired swapped) | Medium | Logic |
| F10 | Import auto-mapping puts MRP into offer price, MOQ into stock | Medium | Data corruption |
| F11 | `validUntil` accepts impossible dates | Low-Medium | Validation |
| F12 | Archived products can be added to a catalogue | Low-Medium | Business rule |
| F13 | Import writes an offer price onto Price on Request products | Low-Medium | Data integrity (enables F02) |

**What was actually run, and what was not.** The sandbox this was written in had no network, no `node_modules` and no database.
- *Run against the real source:* F09 and F10 (bug tests fail before the fix, pass after), and F01, F02, F07 and F08 (the real route
  handlers were executed against an in-memory database; all four show the bug before the fix and correct behaviour after). The F08 retry
  loop was tested with two forced clashes (saved on the third try) and with endless clashes (stops after 5 attempts).
- *Checked in Node:* F08's collision arithmetic and F11's date behaviour.
- *Not executed:* the F03, F04, F05, F06, F12 and F13 tests and fixes (traced through the code only), and the whole Vitest and Playwright suites.
  Run `npm run test:bugs` before and after applying the patch to confirm.

---

## F01. Public search answers for draft and expired catalogues

**What happens**
`GET /api/catalogues/[slug]/search` looks the catalogue up by slug and never checks `status` or `expiresAt`. The catalogue page itself refuses draft and expired slugs, but this endpoint, which needs no login, returns product ids, names, SKUs and offer prices for any catalogue that exists.

**Steps to reproduce**
1. Seed the database (`npm run db:seed`) and start the app.
2. Without signing in, open `http://localhost:3001/catalogue/festive-overstock-2026`. It shows a 404, as intended (it is a draft).
3. Still signed out, open `http://localhost:3001/api/catalogues/festive-overstock-2026/search?q=a`.
4. The response lists the draft's products with prices. Repeat with `monsoon-clearance-2026` (expired): same.

**What should happen instead**
A slug that is not a live catalogue behaves as if it does not exist: 404, no products.

**Impact**
The seed data calls the draft "Confidential. Holding pricing agreed with Northstar Retail pending sign-off." Anyone who can guess a slug (they are human-readable: `festive-overstock-2026`) can read negotiated pricing before it is approved. For the expired catalogue, stale prices that "must not be honoured" are still published. Nothing is logged. High, because the leak is silent and cannot be taken back.

**Failing test**
`tests/bugs/F01-public-search-leaks-draft-and-expired.test.ts` – *does not reveal the products of a DRAFT catalogue…* and *…EXPIRED catalogue…*

**Original code and updated code**

`src/app/api/catalogues/[slug]/search/route.ts`

Original code:

```ts
  const catalogue = await prisma.catalogue.findUnique({
    where: { slug },
    select: { id: true, status: true, expiresAt: true },
  });
  if (!catalogue) {
    return NextResponse.json({ error: "Catalogue not found." }, { status: 404 });
  }
```

Updated code:

```ts
  const catalogue = await prisma.catalogue.findUnique({
    where: { slug },
    select: { id: true, status: true, expiresAt: true },
  });
  // Same rule as the catalogue page: only a published, unexpired catalogue exists
  // for the public. Draft and expired look exactly like "not found".
  const isLive =
    catalogue !== null &&
    catalogue.status === "published" &&
    !(catalogue.expiresAt && catalogue.expiresAt < new Date());
  if (!catalogue || !isLive) {
    return NextResponse.json({ error: "Catalogue not found." }, { status: 404 });
  }
```

**Why this fix**

The route already selected `status` and `expiresAt` but never used them, as if the check had been deleted. It now applies the same rule as the public catalogue page. A draft, an expired catalogue and a missing one all return the same 404, so the API does not confirm that a secret catalogue exists.

---

## F02. Public search returns the price of "Price on Request" products

**What happens**
The search response includes `offerPrice` for every match. The catalogue page hides that price for products flagged Price on Request (`getPublishedCatalogue` maps it to `null`); the search route does not look at the flag.

**Steps to reproduce**
1. Have a product with *Price on Request* on and an offer price stored (F13 shows how import does this, and seed data can too).
2. Add it to a live catalogue.
3. Signed out, call `/api/catalogues/<slug>/search?q=<part of the name>`.
4. `matches[].offerPrice` contains the number.

**What should happen instead**
`offerPrice` is `null` (or omitted) whenever `priceOnRequest` is true, matching the catalogue page.

**Impact**
Price on Request exists so sales can quote individual buyers. A competitor or a buyer can read the "hidden" price with one unauthenticated GET and screenshot it. Commercial harm, no authentication needed. High.

**Failing test**
`tests/bugs/F02-public-search-leaks-price-on-request.test.ts` – *returns no price for a Price on Request product*

**Original code and updated code**

Change 1 of 2 - `src/app/api/catalogues/[slug]/search/route.ts`

Original code:

```ts
      product: { select: { name: true, sku: true, offerPrice: true } },
```

Updated code:

```ts
      product: { select: { name: true, sku: true, offerPrice: true, priceOnRequest: true } },
```

Change 2 of 2 - `src/app/api/catalogues/[slug]/search/route.ts`

Original code:

```ts
      offerPrice: listing.product.offerPrice,
```

Updated code:

```ts
      // Never expose the price of a Price on Request product.
      offerPrice: listing.product.priceOnRequest ? null : listing.product.offerPrice,
```

**Why this fix**

The catalogue page hides this price by mapping it to `null`; the search endpoint skipped that step. It now selects `priceOnRequest` and returns `null` for those products. Same behaviour as the page.

---

## F03. Staff can delete a catalogue

**What happens**
The `deleteCatalogue` server action checks that a user is signed in but never checks the role. `src/auth-guards.ts` says publishing and deleting a catalogue are admin-only; the UI hides the button (`canManage`), but a server action is a callable POST endpoint, so hiding the button is not access control.

**Steps to reproduce**
1. Sign in as `staff@catalogue.test` / `Staff#2026`. Confirm there is no delete button on *Catalogues*.
2. Call the action directly: in an admin session, capture the `Next-Action` POST that the delete button sends (its action id and body), then replay it with the staff session cookie. (Easiest proof: run the failing test.)
3. The catalogue and its listings are deleted.

**What should happen instead**
`{ error: "Only an admin can delete a catalogue." }` and no database call, like `setCatalogueStatus` already does.

**Impact**
A staff account (or a stolen staff session) can destroy catalogues. Deletion is only stopped when enquiries reference the catalogue (foreign-key error), so any catalogue without leads goes with all its listings, display order and badges. Not recoverable from the UI. High: it violates an explicit written rule.

**Failing test**
`tests/bugs/F03-staff-can-delete-catalogue.test.ts` – *rejects a staff user and does not touch the database*

**Original code and updated code**

`src/app/admin/actions.ts`

Original code:

```ts
export async function deleteCatalogue(catalogueId: string) {
  const actor = await currentActor();
  if (!actor) return { error: "You need to sign in again." };
  if (!z.string().uuid().safeParse(catalogueId).success) return { error: "Invalid catalogue." };
```

Updated code:

```ts
export async function deleteCatalogue(catalogueId: string) {
  const actor = await currentActor();
  if (!actor) return { error: "You need to sign in again." };
  if (!isAdmin(actor)) return { error: "Only an admin can delete a catalogue." };
  if (!z.string().uuid().safeParse(catalogueId).success) return { error: "Invalid catalogue." };
```

**Why this fix**

One line, using the `isAdmin` helper that already exists. `setCatalogueStatus` has exactly this check; `deleteCatalogue` was simply missing it. The check comes before any database call.

---

## F04. Staff can publish by creating a catalogue with "go live"

**What happens**
`createCatalogueWithProducts({ publish: true })` creates a `published` catalogue for any signed-in user. Only `setCatalogueStatus` has the admin check; this second path to the same outcome has none. The create dialog also shows the "publish" option to staff.

**Steps to reproduce**
1. Sign in as staff.
2. *Catalogues → New catalogue*, fill in name and link, tick the publish option, and choose the go-live button.
3. The catalogue is live at `/catalogue/<slug>` immediately.

**What should happen instead**
Staff cannot create a live catalogue: either the server rejects `publish: true` for non-admins or saves a draft. (The test accepts either.)

**Impact**
This defeats the publish rule. Pricing goes onto the public internet without admin approval. High.

**Failing test**
`tests/bugs/F04-staff-can-publish-via-create.test.ts` – *does not store a published catalogue for a staff user*

**Original code and updated code**

`src/app/admin/catalogues/actions.ts`

Original code:

```ts
export async function createCatalogueWithProducts(input: CatalogueCreateInput): Promise<CreateCatalogueResult> {
  const session = await auth();
  if (!session) return { error: "You need to sign in again." };
  const parsed = catalogueCreateSchema.safeParse(input);
```

Updated code:

```ts
export async function createCatalogueWithProducts(input: CatalogueCreateInput): Promise<CreateCatalogueResult> {
  const session = await auth();
  if (!session) return { error: "You need to sign in again." };
  // Going live is an admin decision, same rule as setCatalogueStatus.
  if (input?.publish && session.user?.role !== "admin") {
    return { error: "Only an admin can publish a catalogue. Save it as a draft instead." };
  }
  const parsed = catalogueCreateSchema.safeParse(input);
```

**Why this fix**

This was a second door to the same room. `setCatalogueStatus` blocked staff, but 'create and go live' did not. The server now refuses `publish: true` for non-admins with a clear message. I chose to reject rather than quietly save a draft, because silently changing what the user asked for is confusing.

---

## F05. Staff can publish by PATCHing `status`

**What happens**
`PATCH /api/admin/catalogues/[id]` accepts `status: "published"` from any signed-in user. It runs the incomplete-products check but no role check.

**Steps to reproduce**
1. Sign in as staff and open the browser console on any admin page.
2. Run (replace the id with a draft catalogue's id):
   ```js
   fetch("/api/admin/catalogues/<id>", { method: "PATCH", headers: { "content-type": "application/json" },
     body: JSON.stringify({ name: "Festive overstock 2026", slug: "festive-overstock-2026", description: "",
       category: "", notifyNumber: "", status: "published", validUntil: null, banners: [] }) }).then(r => r.status)
   ```
3. Status `200`. The draft is now public.

**What should happen instead**
`403` when a non-admin changes `status` from anything to `published` (or moves a live catalogue to draft). Staff must still be able to edit other fields; the tests include that control.

**Impact**
Same as F04, and it hits the most sensitive object in the system, the draft with negotiated pricing. High.

**Failing test**
`tests/bugs/F05-staff-can-publish-via-patch.test.ts` – *returns 403 and writes nothing*

**Original code and updated code**

Change 1 of 2 - `src/app/api/admin/catalogues/[id]/route.ts`

Original code:

```ts
import { auth } from "@/auth";
import { catalogueEditSchema } from "@/lib/schemas/catalogue";
```

Updated code:

```ts
import { auth } from "@/auth";
import { currentActor, isAdmin } from "@/auth-guards";
import { catalogueEditSchema } from "@/lib/schemas/catalogue";
```

Change 2 of 2 - `src/app/api/admin/catalogues/[id]/route.ts`

Original code:

```ts
    if (!existing) return NextResponse.json({ error: "Catalogue not found." }, { status: 404 });

    if (input.status === "published" && existing.status !== "published") {
```

Updated code:

```ts
    if (!existing) return NextResponse.json({ error: "Catalogue not found." }, { status: 404 });

    // Putting a catalogue live, or pulling it offline, is an admin decision.
    // Staff can still edit everything else, including a catalogue that is already live.
    const changesLiveState = (input.status === "published") !== (existing.status === "published");
    if (changesLiveState && !isAdmin(await currentActor())) {
      return NextResponse.json({ error: "Only an admin can change what is published." }, { status: 403 });
    }

    if (input.status === "published" && existing.status !== "published") {
```

**Why this fix**

The fix compares 'was live' with 'will be live'. If those differ and the caller is not an admin, the answer is 403. Staff can still edit the name, description, banners and so on, even on a live catalogue, because the live state does not change. This also avoids breaking legacy `inactive` rows, which read as draft.

---

## F06. Listing PATCH/DELETE ignore the catalogue in the URL

**What happens**
`/api/admin/catalogues/[id]/listings/[listingId]` checks that catalogue `[id]` exists, then updates or deletes the listing by `listingId` alone (`where: { id: listingId }`). Nothing ties the listing to the catalogue in the path. The sibling `/listings/order` route does check ownership, so this is an inconsistency, not a missing pattern.

**Steps to reproduce**
1. Get the id of catalogue A and the id of a listing that belongs to catalogue B.
2. `PATCH /api/admin/catalogues/<A>/listings/<B's listing>` with `{ "isVisible": false }`.
3. `200 OK`; the product disappears from catalogue B.

**What should happen instead**
`404 Listing not found`: the `where` must include `catalogueId: id`.

**Impact**
Today every signed-in user may edit every catalogue, so the extra power is small. It is still a real integrity bug: the response reports success for the wrong catalogue, and it removes a safety net that any future per-catalogue permission, or a mistaken client id, would rely on. Medium.

**Failing test**
`tests/bugs/F06-listing-routes-not-scoped-to-catalogue.test.ts` – *PATCH…cannot hide catalogue B's listing* and *DELETE…*

**Original code and updated code**

Change 1 of 2 - `src/app/api/admin/catalogues/[id]/listings/[listingId]/route.ts`

Original code:

```ts
    const result = await prisma.catalogueListing.updateMany({
      where: { id: ids.listingId },
      data: parsed.data,
```

Updated code:

```ts
    const result = await prisma.catalogueListing.updateMany({
      where: { id: ids.listingId, catalogueId: ids.id },
      data: parsed.data,
```

Change 2 of 2 - `src/app/api/admin/catalogues/[id]/listings/[listingId]/route.ts`

Original code:

```ts
    const result = await prisma.catalogueListing.deleteMany({
      where: { id: ids.listingId },
```

Updated code:

```ts
    const result = await prisma.catalogueListing.deleteMany({
      where: { id: ids.listingId, catalogueId: ids.id },
```

**Why this fix**

Adding `catalogueId` to the `where` clause means a listing can only be changed through its own catalogue. If the ids do not match, no row is updated and the route returns its existing 404. The sibling `/listings/order` route already worked this way.

---

## F07. Enquiries are accepted for draft and expired catalogues

**What happens**
`POST /api/enquiries` verifies that the products are visible listings of the given catalogue id, but never that the catalogue is published or unexpired.

**Steps to reproduce**
1. Take the id of the expired catalogue (visible in the admin URL `/admin/catalogues/<id>`).
2. `POST /api/enquiries` with that `catalogueId`, a valid visible product and quantity, name and phone.
3. `201 Created`; a lead appears in the inbox against an expired campaign.

**What should happen instead**
`404`/`409`, no lead. The public page already refuses these catalogues, so the API should agree with it.

**Impact**
Sales chase leads for prices "that must not be honoured". Needs the catalogue's UUID, so it is not a casual attack, but ids appear in admin links and screenshots, and a buyer holding a page from before expiry can still submit. Medium.

**Failing test**
`tests/bugs/F07-enquiry-accepted-for-draft-or-expired-catalogue.test.ts`

**Original code and updated code**

`src/app/api/enquiries/route.ts`

Original code:

```ts
      prisma.catalogue.findUnique({ where: { id: lead.catalogueId }, select: { notifyNumber: true } }),
    ]);
    if (listings.length !== items.length)
```

Updated code:

```ts
      prisma.catalogue.findUnique({
        where: { id: lead.catalogueId },
        select: { notifyNumber: true, status: true, expiresAt: true },
      }),
    ]);
    // Same rule as the public page: only a published, unexpired catalogue takes enquiries.
    const catalogueIsLive =
      catalogueRecord !== null &&
      catalogueRecord.status === "published" &&
      !(catalogueRecord.expiresAt && catalogueRecord.expiresAt < new Date());
    if (!catalogueIsLive) {
      return NextResponse.json({ error: "This catalogue is not accepting enquiries." }, { status: 409 });
    }
    if (listings.length !== items.length)
```

**Why this fix**

The route already loaded the catalogue for `notifyNumber`; it now also reads `status` and `expiresAt` and applies the same live rule as the public page. The check runs before anything is written, so no lead is saved against a draft or expired catalogue.

---

## F08. Enquiry reference collides and the buyer's lead is lost

**What happens**
The reference is `ENQ-` plus the last 7 digits of `Date.now()`. Only 10,000,000 values exist, and the sequence repeats every 10,000 seconds (2 h 46 m 40 s). The column is unique, so a new enquiry that lands on an existing reference fails to insert and the route returns 500 "Could not save the enquiry".

**Steps to reproduce**
1. Create an enquiry at time T (reference `ENQ-0000123`).
2. Create another at T + 10,000 s, or two within the same millisecond. Same reference.
3. The second buyer sees "Could not send your enquiry"; nothing is stored.

**What should happen instead**
Every valid enquiry is saved. Generate the reference from a random or sequential source and retry on `P2002`.

**Impact**
This is silent revenue loss on the core flow. Probability that a stored set of N enquiries contains a duplicate reference is about `1 − e^(−N²/20,000,000)`: 5 % at 1,000, 36 % at 3,000, 71 % at 5,000. Each *new* enquiry fails with probability roughly `existing/10,000,000` (0.1 % at 10,000 stored). Rare early, unavoidable later, and nobody is told. High.

**Failing test**
`tests/bugs/F08-enquiry-reference-collisions-lose-leads.test.ts` – *still saves the enquiry when the generated reference is already taken*. The test mocks a constant `Date.now()`, so the fix must add randomness (a retry that re-reads `Date.now()` would not pass, which is the point).

**Original code and updated code**

Change 1 of 5 - `src/app/api/enquiries/route.ts`

Original code:

```ts
  const reference = `ENQ-${Date.now().toString().slice(-7)}`;
```

Updated code:

```ts
  let reference = newReference();
```

Change 2 of 5 - `src/app/api/enquiries/route.ts`

Original code:

```ts
export async function POST(request: Request) {
```

Updated code:

```ts
// Random 7 digits (was: last 7 digits of the clock, which repeats every ~2h46m and
// collides for two enquiries in the same millisecond). Collisions that remain are
// retried below.
function newReference() {
  return `ENQ-${String(randomInt(0, 10_000_000)).padStart(7, "0")}`;
}

export async function POST(request: Request) {
```

Change 3 of 5 - `src/app/api/enquiries/route.ts`

Original code:

```ts
    await prisma.enquiry.create({
      data: {
        reference,
        catalogueId: lead.catalogueId,
```

Updated code:

```ts
    const enquiryFields = {
      catalogueId: lead.catalogueId,
```

Change 4 of 5 - `src/app/api/enquiries/route.ts`

Original code:

```ts
      },
    });
    const summary = items
```

Updated code:

```ts
    };
    for (let attempt = 1; ; attempt += 1) {
      try {
        await prisma.enquiry.create({ data: { reference, ...enquiryFields } });
        break;
      } catch (error) {
        const duplicate = typeof error === "object" && error !== null && "code" in error && error.code === "P2002";
        if (!duplicate || attempt >= 5) throw error;
        reference = newReference(); // unique-reference clash: pick another, never lose the lead
      }
    }
    const summary = items
```

Change 5 of 5 - `src/app/api/enquiries/route.ts`

Original code:

```ts
import { after, NextResponse } from "next/server";
```

Updated code:

```ts
import { randomInt } from "node:crypto";
import { after, NextResponse } from "next/server";
```

**Why this fix**

Two changes. (1) The reference is now 7 random digits instead of the last 7 digits of the clock, so it no longer repeats every 2h46m or clashes within a millisecond. The `ENQ-1234567` format is unchanged. (2) If the database still reports a duplicate (`P2002`), the route picks a new reference and retries up to 5 times, so a clash can never lose a buyer's lead. Any other error is still thrown as before.

---

## F09. Admin status badge is inverted

**What happens**
`effectiveStatus` returns `"expired"` when `expiresAt > new Date()` – that is, while the validity date is still **in the future**. The comparison is the wrong way round.

**Steps to reproduce**
1. Publish a catalogue with a validity date next month.
2. *Catalogues*: its badge says **Expired**.
3. Open a catalogue whose date has passed: it says **Live**, while buyers see "This catalogue has expired".

**What should happen instead**
`expiresAt < now` means expired. The public page (`getPublishedCatalogue`) and the dashboard's "active catalogues" count already use the correct direction, so the three screens disagree today.

**Impact**
The people running campaigns cannot trust the status column: they may renew campaigns that are live and leave dead ones marked Live. Also affects the global search results. Medium (no data loss; wrong information on every admin screen).

**Failing test**
`tests/bugs/F09-effective-status-inverted.test.ts` (2 tests). *Run and confirmed failing.*

**Original code and updated code**

`src/lib/catalogue-status.ts`

Original code:

```ts
  if (status === "published" && expiresAt && expiresAt > new Date()) return "expired";
```

Updated code:

```ts
  if (status === "published" && expiresAt && expiresAt < new Date()) return "expired";
```

**Why this fix**

One character. Expired means the validity date is in the past, so the comparison must be `<`. The public page and the dashboard already used the correct direction.

---

## F10. Import auto-mapping puts MRP into offer price and MOQ into stock

**What happens**
`autoMap` uses `header.includes(synonym)` and checks fields in a fixed order, and each field can be used once. The broad synonyms `price` (offer price) and `qty` (quantity) win over the more specific MRP and MOQ synonyms, so:
- `List Price` / `Retail Price` → **offer price** (and a real `Offer Price` column falls through to "attribute");
- `Min Qty` → **quantity** (and `Available Quantity` falls through to "attribute").

**Steps to reproduce**
1. *Products → Import*, upload a sheet with headers `SKU, Name, List Price, Offer Price, Qty` (or `…Min Qty, Available Quantity`).
2. Look at the pre-selected mapping.

**What should happen instead**
Prefer exact header matches over substring matches, and try specific fields before general ones.

**Impact**
The mapping is pre-filled, so a busy user confirming the dialog imports MRP as the selling price (no discount shown) or MOQ as stock. Wrong prices reach the public catalogue and get through the publish gate, because a value exists. Medium. A user who reads the dialog can fix it.

**Failing test**
`tests/bugs/F10-import-automap-misroutes-columns.test.ts` (3 tests). *Run and confirmed failing.*

**Original code and updated code**

Change 1 of 2 - `src/lib/import-mapping.ts`

Original code:

```ts
  offerPrice: ["offerprice", "landingprice", "sellingprice", "saleprice", "price", "offer"],
  mrp: ["mrp", "listprice", "retailprice", "maximumretailprice"],
  quantity: ["availablequantity", "available", "quantity", "qty", "stock", "units"],
  moq: ["moq", "minqty", "minimumorder", "minorderqty", "minimum"],
```

Updated code:

```ts
  // Specific fields (mrp, moq) come before the broad ones (offerPrice, quantity):
  // "price" and "qty" would otherwise swallow "List Price" and "Min Qty".
  mrp: ["mrp", "listprice", "retailprice", "maximumretailprice"],
  offerPrice: ["offerprice", "landingprice", "sellingprice", "saleprice", "price", "offer"],
  moq: ["moq", "minqty", "minimumorder", "minorderqty", "minimum"],
  quantity: ["availablequantity", "available", "quantity", "qty", "stock", "units"],
```

Change 2 of 2 - `src/lib/import-mapping.ts`

Original code:

```ts
  for (const header of headers) {
    const normalized = header.toLowerCase().replace(/[^a-z0-9]/g, "");
    let match: FieldKey = "attribute";
    for (const [field, keys] of Object.entries(SYNONYMS) as [FieldKey, string[]][]) {
      if (used.has(field)) continue;
      if (keys.some((key) => normalized === key || normalized.includes(key))) {
        match = field;
        break;
      }
    }
    if (match !== "attribute") used.add(match);
    mapping[header] = match;
  }
  return mapping;
```

Updated code:

```ts
  const normalize = (header: string) => header.toLowerCase().replace(/[^a-z0-9]/g, "");

  const claim = (header: string, matchesKey: (normalized: string, key: string) => boolean) => {
    const normalized = normalize(header);
    for (const [field, keys] of Object.entries(SYNONYMS) as [FieldKey, string[]][]) {
      if (used.has(field)) continue;
      if (keys.some((key) => matchesKey(normalized, key))) {
        used.add(field);
        mapping[header] = field;
        return true;
      }
    }
    return false;
  };

  // Pass 1: exact header matches win, so "List Price" is MRP and "Offer Price" is the offer price.
  const unmatched = headers.filter((header) => !claim(header, (normalized, key) => normalized === key));
  // Pass 2: looser "contains" matching for what is left ("Unit Price", "Item Qty" ...).
  for (const header of unmatched) {
    if (!claim(header, (normalized, key) => normalized.includes(key))) mapping[header] = "attribute";
  }
  return mapping;
```

**Why this fix**

Two changes. (1) Specific fields (`mrp`, `moq`) are listed before broad ones (`offerPrice`, `quantity`). (2) Matching now runs in two passes: exact header matches claim their field first, and only the leftovers use looser 'contains' matching. So `List Price` is MRP and `Offer Price` is the offer price, whatever order the columns come in.

---

## F11. `validUntil` accepts impossible dates

**What happens**
`validUntil` is only checked against `^\d{4}-\d{2}-\d{2}$`. `2026-02-30` passes and is silently rolled to **2 March 2026** by `new Date(...)`, so the catalogue lives two days longer than typed. `2026-13-45` gives an Invalid Date and the save fails with a generic 500 instead of a field error. (Both behaviours checked in Node 22.)

**Steps to reproduce**
`PATCH /api/admin/catalogues/<id>` with `"validUntil": "2026-02-30"` → `200` and `expiresAt` = 2 March. With `"2026-13-45"` → `500 Could not update the catalogue`.

**What should happen instead**
`400` with a "Pick a valid date" field error for any date that is not a real calendar day.

**Impact**
Low for the UI, whose date picker cannot produce these values; it matters for the API and for anything that automates it. The silent two-day extension is the worse half.

**Failing test**
`tests/bugs/F11-valid-until-accepts-impossible-dates.test.ts`

**Original code and updated code**

Change 1 of 2 - `src/lib/schemas/catalogue.ts`

Original code:

```ts
  validUntil: z
    .string()
    .regex(/^\d{4}-\d{2}-\d{2}$/, "Pick a valid date.")
    .nullable(),
```

Updated code:

```ts
  validUntil: z
    .string()
    .regex(/^\d{4}-\d{2}-\d{2}$/, "Pick a valid date.")
    .refine(isRealCalendarDate, "Pick a valid date.")
    .nullable(),
```

Change 2 of 2 - `src/lib/schemas/catalogue.ts`

Original code:

```ts
export const catalogueEditSchema = z.object({
```

Updated code:

```ts
// "2026-02-30" matches the yyyy-mm-dd shape but is not a real day; JavaScript would
// silently turn it into 2 March. Reject anything that does not round-trip.
function isRealCalendarDate(value: string) {
  const [year, month, day] = value.split("-").map(Number);
  const date = new Date(Date.UTC(year, month - 1, day));
  return date.getUTCFullYear() === year && date.getUTCMonth() === month - 1 && date.getUTCDate() === day;
}

export const catalogueEditSchema = z.object({
```

**Why this fix**

The old pattern only checked the shape `yyyy-mm-dd`. The new check builds the date and confirms the year, month and day survive unchanged, so `2026-02-30`, `2026-13-45` and `2027-02-29` are rejected while `2028-02-29` (a real leap day) is accepted. The admin now gets a normal 'Pick a valid date.' field error.

---

## F12. Archived products can be added to a catalogue

**What happens**
"Archived products are retired and can't be added to catalogues" is enforced only in the *picker's search query*. `POST /api/admin/catalogues/[id]/listings` inserts whatever product ids it receives.

**Steps to reproduce**
1. Archive a product.
2. `POST /api/admin/catalogues/<id>/listings` with `{ "productIds": ["<archived id>"] }` (or add it from a picker tab that was open before the archive).
3. `201`; the retired product is on the catalogue.

**What should happen instead**
Archived ids are rejected or dropped.

**Impact**
Retired stock returns to a live page and can be enquired about. Low–Medium: it needs a direct call or a stale tab.

**Failing test**
`tests/bugs/F12-archived-product-can-be-added-to-catalogue.test.ts`

**Original code and updated code**

`src/app/api/admin/catalogues/[id]/listings/route.ts`

Original code:

```ts
  try {
    const { _max } = await prisma.catalogueListing.aggregate({
```

Updated code:

```ts
  // Archived products are retired: the picker hides them, and the server must refuse them too.
  const wanted = [...new Set(parsed.data.productIds)];
  const active = await prisma.product.findMany({
    where: { id: { in: wanted }, archivedAt: null },
    select: { id: true },
  });
  if (active.length !== wanted.length) {
    return NextResponse.json(
      { error: "One or more products are archived or no longer exist." },
      { status: 409 },
    );
  }

  try {
    const { _max } = await prisma.catalogueListing.aggregate({
```

**Why this fix**

The rule 'archived products can't join a catalogue' lived only in the picker's search query. The endpoint that actually saves now enforces it too: if any requested product is archived or does not exist, nothing is inserted and the route returns 409. Duplicate ids in the request are collapsed first so the count comparison is correct.

---

## F13. Import writes an offer price onto Price on Request products

**What happens**
`importProducts` sets `offerPrice` from the sheet for existing products without reading `priceOnRequest`. Everywhere else the two are kept consistent (`toData`, `updateProductPrices` write `null` for Price on Request).

**Steps to reproduce**
1. Have a product with Price on Request on, whose SKU appears in a sheet with an *Offer Price* column.
2. Import with "update existing products".
3. The product now has an offer price stored again.

**What should happen instead**
The offer price is left `null` for Price on Request products.

**Impact**
It recreates exactly the data that F02 leaks, and shows in admin lists as a real price. Low–Medium alone; it is the mechanism that makes F02 reachable in normal operation.

**Failing test**
`tests/bugs/F13-import-writes-price-on-price-on-request-products.test.ts`

**Original code and updated code**

Change 1 of 3 - `src/app/admin/products/import-actions.ts`

Original code:

```ts
                offerPrice: row.offerPrice !== undefined ? row.offerPrice.toFixed(2) : undefined,
                quantity: locatedSkus.has(row.sku) ? undefined : row.quantity,
```

Updated code:

```ts
                offerPrice: priceOnRequestSkus.has(row.sku)
                  ? null
                  : row.offerPrice !== undefined
                    ? row.offerPrice.toFixed(2)
                    : undefined,
                quantity: locatedSkus.has(row.sku) ? undefined : row.quantity,
```

Change 2 of 3 - `src/app/admin/products/import-actions.ts`

Original code:

```ts
          select: { sku: true, archivedAt: true, _count: { select: { stocks: true } } },
```

Updated code:

```ts
          select: { sku: true, archivedAt: true, priceOnRequest: true, _count: { select: { stocks: true } } },
```

Change 3 of 3 - `src/app/admin/products/import-actions.ts`

Original code:

```ts
        // Products split by city keep their breakdown; an import must never
        // silently overwrite it with one flat number.
        const locatedSkus
```

Updated code:

```ts
        // Price on Request products never carry a stored offer price.
        const priceOnRequestSkus = new Set(
          existing.filter(({ priceOnRequest }) => priceOnRequest).map(({ sku }) => sku),
        );
        // Products split by city keep their breakdown; an import must never
        // silently overwrite it with one flat number.
        const locatedSkus
```

**Why this fix**

The import now also reads `priceOnRequest` for existing products and writes `offerPrice: null` for those. That is the rule the rest of the code already follows (`toData`, `updateProductPrices`). It removes the way a hidden price could be stored again, which is what made F02 reachable.

---

## Observations that are not bugs I can prove

These need a product decision, so I did not write a test that would just copy today's behaviour.

- **What should Staff be allowed to do?** The code states one rule (no publish, no delete). Staff can also archive and permanently delete products, run imports that overwrite prices, and change lead status. If those should be admin-only, say so and I'd add the tests.
- **No buyer basket.** The API supports up to 100 items per enquiry; the UI supports one product per enquiry. Either the UI or the assessment brief ("add items to an enquiry") is ahead of the other.
- **Buyer phone is 6–10 digits**, and the UI truncates to 10, so international numbers longer than 10 digits cannot be entered.
- **No rate limiting** on `POST /api/enquiries`, on login, or on newsletter subscribe (which at least has a honeypot). An automated script can flood the leads inbox.
- **Public search returns `offerPrice` as a JSON string** (Prisma `Decimal`) while every other endpoint returns numbers.
