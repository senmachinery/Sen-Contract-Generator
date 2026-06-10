# CLAUDE.md — Sen Labels Web Tools (Contract Generator + CRM)

> Read this fully **before touching any file.** This project runs a real business.
> Breaking it costs the owner real money. When unsure, ask — do not guess.

---

## 0. What this is

Owner: **Max Goh** — Sen Labels Machinery Sdn. Bhd. (label-printing machinery, Johor, Malaysia).
(His name is **Max Goh**, English only. Never invent or use a Chinese name for him.)

Two browser tools, hosted on **GitHub Pages** at `senmachinery.github.io`:

| File | What it is | Current version |
|------|------------|-----------------|
| `index.html` | **Contract Generator** — builds quotes / sales contracts / invoices, prints PDF | **v10.88** |
| `crm.html` | **CRM** — customers, contract pipeline, tasks, inquiries, invoices | **v2.24** |

Backend: a **Google Apps Script web app** talking to a Google Sheet named **"Sen Labels Contract Record"**. The backend reports version **v10.62** (plus invoice handlers added 2026-06 — see §9). Its source is a `.gs` file that lives **outside** the Pages repo. Both HTML files call it via `fetch`.

- The backend URL is **stored in the browser (localStorage), never hardcoded.** Do not hardcode it. Do not print it.
- Both HTML files are **single-file** apps (all HTML + CSS + JS inline). There is no build step.

---

## 1. Max's Iron Rules — NEVER violate these

1. **Only ADD. Never break or remove existing behavior.** Every new feature must default **OFF**, and the "feature-not-used" path must behave exactly as before. This is the #1 rule.
2. **Self-verify before claiming done. Never lie about completion.** See §3. "It should work" is not acceptable — prove it.
3. **Bump the version number on every file you change** — in BOTH the `<title>` and the visible top-bar version span. Generator uses `v10.xx`; CRM uses `v2.xx`.
4. **Reply to Max in Simplified Chinese (简体中文).** Code/comments stay English; the conversation is Chinese.
5. **Go step by step. Don't skip steps.** For a big change, do it as **separate, sequential versions** (one feature per version bump) so each is testable and reversible — not everything at once.
6. **Minimize the change.** Surgical edits only. Guard new logic behind a flag so the diff is small and the old path is untouched. Do not "tidy up" or rewrite functions you weren't asked to touch.
7. **Always tell Max how to deploy** — exactly which GitHub file to paste into, and **whether a backend redeploy is needed** (see §2).
8. **Name the download/archive copy with the version** (e.g. `SEN Contract Generator v10.85.html`, `SEN CRM v2.21.html`). The file Max pastes into GitHub **keeps its name** `index.html` / `crm.html` — the versioned name is only the archive copy.
9. **Prefer a frontend-only solution.** A backend change forces a risky redeploy. Most things can be done in the HTML alone — do that unless truly impossible.

---

## 2. How a change ships (deployment)

**Frontend (index.html / crm.html) — the normal case:**
1. Edit the HTML.
2. Max opens GitHub, selects all, pastes the full new content into the matching file (`index.html` or `crm.html`).
3. Commit → wait **~2 minutes** for GitHub Pages to rebuild → **hard refresh** the page.

**Backend (the `.gs`) — avoid if possible:**
- Editing the Apps Script requires Max to **redeploy the web app as a new version**. This is separate, slower, and easy to get wrong. Only do it when no frontend path exists, and say so loudly.

When delivering, always state: which file(s) changed, the new version number(s), the exact paste/commit/refresh steps, and **"no backend redeploy needed"** (or, rarely, the redeploy steps).

---

## 3. Mandatory verification (the safeguard against rule #1)

There is **no build/test pipeline**, so you must verify by hand, every time, like this:

**Step A — Syntax.** Extract every `<script>` block and run `node --check` on each. A 7000-line file with one stray bracket is silently broken otherwise.

**Step B — "Nothing broke" proof (the important one).** Keep the **previous version** of the file (ask Max for it or back it up before editing). Load BOTH old and new in a headless JS context (Node `vm` with a stubbed `document`/`window`/`localStorage`, or `jsdom`), feed BOTH the **same realistic state with your new feature switched OFF**, call the render function (`buildContractHTML()` for the Generator, `renderInquiryCard()` etc. for the CRM), and assert the output is **byte-identical**. If it differs, you broke something — fix it before shipping.

**Step C — New feature works.** With the feature ON, render and assert the new behavior (e.g. package price shows, total is correct, photo URL appears).

> Notes for the headless harness: top-level `function` declarations are hoisted, so they're callable even if some load-time init throws under stubs. `state` is a top-level `let` — reassign it from an appended footer function in the SAME script (e.g. `function __render(s){ state = s; return buildContractHTML(); }`). Stub `localStorage`, `setTimeout`/`setInterval` (return 0), `matchMedia`, `navigator`, `location`, `document.getElementById` (return a chainable stub). The big page can **hang under full jsdom** — the lightweight `vm` + stub approach is faster and reliable.

Don't tell Max "done" until A, B and C pass, and say briefly what you verified.

---

## 4. Architecture you must respect

### 4.1 Contract Generator (`index.html`)

- One global `state` object (a `let`), serialized with `JSON.stringify(state)` for **Save JSON** and re-hydrated by **Load JSON** → `normalizeState(state)`. `normalizeState` backfills defaults for older saved files — **add a default here for any new state/machine field** so old contracts still load.
- `M()` returns the **active machine**. `state.machines[]` — each machine has `machineType` (`'other'` = a machine sale, `'parts'` = a spare-parts order), `condition`, `customTitle`, `extraSpecs[]`, `cwItems[]` (comes-with), `optItems[]`, `partItems[]`/`partGroups[]`, `images[]`, `unitPrice`/`unitQty`/`sellPrice`, `discountPct`/`discountAmt`, and `inPackage` (boolean).
- **`buildContractHTML()` has TWO rendering branches** — a **parts-order** layout and a **non-parts (machine-sale)** layout — plus a **5-column** summary table (#, Description, Qty, Unit Price, Total). **Any change to rendering or totals must be handled (or safely guarded) in BOTH branches.**
- **Package pricing (added v10.84):** per-machine `inPackage` + `state.packagePrice` + `state.packageLabel`. Marked machines hide their individual price; one combined "Package Price" line shows after the last marked machine; totals/deposit use the package price. ALL of it is gated by helpers **`pkgAnyActive()`** and **`pkgContractTotal()`** so that when nothing is marked, output is byte-identical to before. Follow this exact pattern for any future "modes".
- **Images:** `m.images[]` entries are **either base64 data URLs or Google Drive image URLs** — both render via `<img src>`. To store to Drive: call backend `uploadImage(folder,name,base64)` then `listImages(folder)` (the latter sets sharing and returns a usable thumbnail URL — see §4.3). `reloadImageThumbs()` re-renders the form strip and already handles both kinds.
- **URL params (CRM → Generator handoff):** `?customer=NAME&autofill=true` (new quote, auto-fills customer header from CustomerDB), optional `&inqimg=1` (pull inquiry photos from localStorage — see §4.2), `?contract=NO&load=true` (open existing), `?preview=NO` (silent PDF preview). All handled in `handleCrmUrlParams()`.
- **404 fix (v10.83) — do not regress:** `doPrintInner()` must capture the **full** original URL (path + search + hash) **before** it changes the URL for printing, and restore it afterward. Otherwise refreshing after a print 404s and the CRM link breaks.
- **Variant numbers (v10.88):** one contract number can have multiple contents — `SEN/SC/NR/26-2453` + `26-2453(B)` + `(C)`… The `+B` button (`makeVariantNo()`) next to the contract-no field strips any existing `(X)`, queries `action=list` for used letters, and sets the next free one. Variants are **separate records** (saveRecord/saveState match the exact string). `getNextNo` is safe with the suffix: `parseInt('2453(B)') === 2453`. ⚠️ Never use digit suffixes like `24530` for this — they DO inflate `getNextNo`.

### 4.2 CRM (`crm.html`)

- **Inquiries are stored as TASKS** (`crm_addTask` / `crm_updateTask`). The inquiry's fields (customer, machine, tel, email, address, attn, country, budget, source, detail, **images**) are **JSON-encoded inside the task `notes` string**. `rebuildInquiriesFromTasks()` parses `notes` → inquiry objects using a **field whitelist** — when you add a new inquiry field, you must add it there too, or it won't surface.
- **Inquiry photos (added v2.21):** stored as **Drive URLs inside the notes JSON** (URLs are tiny — safe for the cell limit, see §4.3). The attach UI compresses client-side (canvas, ~1600px JPEG) → `uploadImage` → `listImages` (for a shareable URL) → pushes the URL into `_inqImages` → saved with the inquiry. Thumbnails show on the inquiry card.
- **Convert to Quote:** `convertInquiry()` marks the inquiry converted and opens `index.html?customer=...&autofill=true`. Customer details auto-fill from the CustomerDB. Inquiry photos are handed to the Generator via a **same-origin `localStorage` key `sen_pending_quote_images`** = `{ images:[...urls], ts: <epoch ms> }`, plus `&inqimg=1` on the URL. The Generator consumes it **once** (removes the key) and **ignores it if older than 5 minutes** (stale guard). crm.html and index.html share an origin (GitHub Pages), so localStorage is shared.
- **API:** `apiGet(action, params)` (GET) and `apiPost(action, data)` (POST) both hit the backend URL from localStorage. **Every action the frontend calls must have a matching backend handler** — verify this when adding actions.

### 4.3 Backend (Apps Script `.gs`) — and the storage constraint that bites

- Routing: `doGet`/`doPost` dispatch by `action`. Anything starting `crm_` goes to `handleCrmGet` / `handleCrmPost`.
- `saveState(contractNo, state)` stores the **entire** contract state JSON in **ONE Sheet cell** (no chunking).
- ⚠️ **Google Sheets hard limit: 50,000 characters per cell.** A single base64 photo usually exceeds that. **Therefore: NEVER store base64 images in the Sheet** (cell overflow + bloat — the sheet is already large). **Store images in Google Drive and keep only the URL** in the Sheet/notes. This is exactly why inquiry/machine photos use Drive.
- `uploadImage(folder,name,base64)` creates a Drive file but **does NOT set sharing** → a fresh upload won't display via a public thumbnail URL. **`listImages(folder)` sets `ANYONE_WITH_LINK` sharing and returns the thumbnail URL** (`https://drive.google.com/thumbnail?id=<id>&sz=w1000`). So the correct sequence is **uploadImage → listImages** (this is how the Generator and CRM both do it; reuse it instead of changing the backend).

---

## 5. Data schemas (for generating JSON Max can paste in)

The Generator accepts pasted JSON in two places. **The source is authoritative — read it before generating JSON:**

- **Single/multiple machine import** (the "append machine" paste box): see `appendMachinesFromJSON()` in `index.html` for the exact machine object shape (fields listed in §4.1). For `machineType:'other'`, every spec row goes in `extraSpecs[]` (no auto Power/Dimension rows).
- **Full contract** (Save/Load JSON): see `saveStateJSON()` / `loadStateJSON()` / `normalizeState()` for the full `state` shape.

When unsure of a field name or default, **grep the source**; do not invent fields.

---

## 6. Pitfalls — do NOT do these

- ❌ Store base64 photos in the Google Sheet (50k cell limit + bloat). ✅ Use Drive URLs.
- ❌ Rewrite or "refactor" a whole function. ✅ Make a tiny guarded edit.
- ❌ Change the output of the existing/feature-off path. ✅ Prove it's byte-identical (§3).
- ❌ Change rendering/totals in only one of the two `buildContractHTML` branches. ✅ Handle both.
- ❌ Add a state/machine field without a default in `normalizeState`. ✅ Old saved files must still load.
- ❌ Add a frontend action with no backend handler.
- ❌ Hardcode the backend URL or treat the Sheet ID as a secret to expose.
- ❌ Redeploy/edit the backend when a frontend fix exists.
- ❌ Break the `?customer` / `?contract` / `?preview` URL handoffs or the convert localStorage handoff.
- ❌ Ship without bumping the version, or forget to tell Max the deploy steps.
- ❌ Use a Chinese name for Max.

---

## 7. Current version log

- **index.html — v10.88**: **Variant contract numbers (v10.88)** — `+B` button: same number, second content as `2453(B)`/`(C)`…, saved as separate records; numbering tool unaffected. · Invoice doc type (v10.87) · inquiry-photos→quote handoff (v10.85) · package pricing (v10.84) · print-refresh 404 fix (v10.83). Two render branches; image = base64 or Drive URL.
- **crm.html — v2.24**: invoice/collection UI in **English** (v2.24) · **Dashboard "Collected" summary** (v2.23) · **Invoices tab** (v2.22) — create invoice (auto-numbered by category), list with Unpaid/Partial/Paid/Overdue filters, record payment. · inquiry photo attachment + photo handoff on Convert to Quote. Inquiries live in task `notes` JSON.
- **backend `.gs` — v10.62 (+invoice handlers, +variant insert 2026-06-10)**: `saveState` = one cell; images via Drive (`uploadImage` + `listImages`); `crm_*` handlers + invoice handlers `crm_addInvoice` / `crm_getInvoices` / `crm_nextInvoiceNo` / `crm_updateInvoice` / `crm_importInvoices`, and a new `Invoices` tab. **`saveRecord` variant insert**: a new `…(B)`-style row is inserted directly below its base contract row (or the last earlier variant) instead of appended; base row missing → append as before. ⚠️ The local `AppScript_最终版_全部修复.txt` (Desktop, old folder) predates the invoice handlers — never paste it wholesale over the deployed script; patch single functions only.

---

## 8. Working agreement (quick recap for every task)

1. Read the relevant file/functions first. 2. Smallest possible, additive, **guarded** edit. 3. Add `normalizeState` defaults for new fields. 4. Bump the version (title + top-bar). 5. Verify A/B/C (§3). 6. Deliver the file + **versioned archive name** + exact deploy steps + "no backend redeploy needed" (or the redeploy steps). 7. Reply to Max in **简体中文**.

---

## 9. Invoice module (added 2026-06 — all phases live)

Completes the money half of the pipeline: **inquiry → quote → contract → invoice → payment**. Scope = **internal bookkeeping only** — no SST / MyInvois e-invoice yet (a `taxPct` field is reserved for it).

- **New `Invoices` tab** in the central Sheet, mirroring the colleague's existing `SalesActuals` columns (`Date · Month · InvoiceNo · Customer · Category · Currency · Amount · Value_RM_equiv`) **plus** payment fields (`ContractNo · DueDate · TaxPct · PaidAmount · Status · Payments · Items · Remark · CreatedAt · CreatedBy`).
- **Three invoice-number series by Category**, auto-continued (max+1), assigned **server-side under a lock** so concurrent staff never collide. Numbering scans **both** `Invoices` and the legacy `SalesActuals`, so it continues the existing sequence:
  - `Parts/Consumables` → `S{YY}-####` (e.g. `S26-1426`)
  - `New Machine` → `IN/N/{YY}-####` (e.g. `IN/N/26-0074`)
  - `Refurb Machine` → `IN/R/{YY}-####` (e.g. `IN/R/26-0078`)
- **Backend handlers** (all additive): `crm_addInvoice` (assigns number + appends), `crm_getInvoices` (normalises date cells back to `yyyy-MM-dd`), `crm_nextInvoiceNo` (preview), `crm_updateInvoice` (record payments → recompute Unpaid/Partial/Paid), `crm_importInvoices` (idempotent bulk history import). Editing the backend still requires a redeploy (§2).
- **Generator (`index.html`)**: `docType:'Invoice'` prints an invoice via the existing engine (no buyer signature; "Invoice No." label). Guarded by `isQuoteLike()`.
- **CRM (`crm.html`)**: an **Invoices tab** to create/record invoices (pick category → auto-number → save), a list with status filters + record-payment; the **Dashboard** shows a **Collected / Invoiced / Outstanding** summary (RM-equivalent) that follows the period filter.
- **History**: the historical `SalesActuals` rows were imported into `Invoices` (marked Paid), original numbers preserved.
- **UI language**: the app UI is **English** — do not add Chinese to the app UI. Only the chat with Max is 简体中文.
