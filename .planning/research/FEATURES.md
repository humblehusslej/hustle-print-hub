# Feature Research

**Domain:** Quoting + production tracking for a 3-person screen-print / embroidery shop
**Researched:** 2026-07-30
**Confidence:** MEDIUM-HIGH (pricing *structure* HIGH, specific dollar figures illustrative only; stage model HIGH; anti-features MEDIUM — reasoned from documented constraints, not external sources)

**Scope check:** Every recommendation below was checked against `.planning/PROJECT.md` → Out of Scope. Nothing here proposes payment processing, inventory, customer accounts/login, drop-shipping, native apps, or marketing. Where a recommendation sits near a boundary, the boundary is stated inline.

---

## 1. How decorated-apparel pricing actually works

This section exists because `FR-5` says "server-side pricing" but never says *what the pricing model is*. The prototype's `calculateManifestTotals()` (`index.html:672-725`) is a linear `qty × rate` function. Decorated apparel is not priced that way, and rebuilding the same math on the server just moves a wrong answer to a safer location.

### 1.1 The shape of a screen-print price

Screen printing has **two cost components with different shapes**, and conflating them is the single most common modeling error:

| Component | Shape | Scales with |
|---|---|---|
| **Setup** (burning screens, registration, ink mixing) | Fixed per job | Number of *screens* — i.e. colors × locations. Not quantity. |
| **Run** (per-piece press time + blank) | Per piece, **tiered downward** by quantity | Quantity, color count, garment |

Concretely:

```
screens = Σ over print locations of (ink_colors_at_location + underbase?)
setup_total = screens × screen_fee          # $15–$30 per screen is the common band
per_piece = blank_cost(brand, size)
          + print_rate[qty_tier][total_colors]
          + size_upcharge(size)
line_total = Σ over sizes (qty_size × per_piece) + setup_total + art_fee?
```

**Quantity price breaks** are the defining feature. Common tier boundaries: **12 / 24 / 48 / 72 / 144 / 288**. A representative per-color-per-piece ladder looks like `1–12: $3.50 → 13–24: $2.50 → 25–48: $1.75 → 49–72: $1.25 → 73–144: $0.90`. The delta between tiers is 3–4×.

> **The prototype has no quantity tiers at all.** `inkGrid = [0, 6.50, 7.50, 8.50, 9.50]` charges $6.50 for the first ink color whether the order is 12 pieces or 500. A 1-color Gildan tee quotes at `$6.25 + $6.50 = $12.75/pc` at *every* volume. Market rate at 250+ pieces is $4–6. **The tool systematically overquotes exactly the orders worth winning** and hands them to a competitor. This is a revenue bug, not a formatting bug, and it is the highest-stakes thing in the pricing engine.

**Size upcharges** are driven by blank cost and are near-universal: `S–XL: +$0`, `2XL: +$2`, `3XL: +$3`, `4XL: +$4–5`, `5XL+: +$6`. Youth blanks are *cheaper* and often carry a small credit. The prototype **already collects** 2XL/3XL/4XL and the full youth grid and then prices every size identically. On a 72-piece order with 20 extended sizes that is $40–80 of margin gone per order, and youth-heavy school orders are overquoted. **The data is already captured — only the price function is missing. Highest value-per-hour fix in the entire build.**

**Dark garments add a screen.** Plastisol needs a white underbase on black/navy/red to stay opaque, plus a flash-cure pass. A "1-color" logo on a black shirt is a 2-screen, 2-color job. The prototype has a free-text `Base Color` field that the pricing function never reads. Since most shop work is on dark garments, this systematically *under*quotes. Model as an explicit **"dark garment?"** boolean rather than parsing free text — colour names are unbounded and "Midnight Black" won't string-match reliably.

**Print locations, not front/back.** Real quotes need N locations, each with its own colour count: *full front, full back, left chest, right chest, sleeve (L/R), neck label, hem*. Each location is a separate screen, a separate setup, and a separate press pass. The prototype models exactly two integer fields (`frontColors`, `backColors`), which cannot express "left chest + sleeve" — a very common combo.

**Shared art across line items.** If a customer orders 48 tees and 24 hoodies with the *same* design, the shop burns screens once and runs 72 pieces — one setup charge, and the **72-piece tier**. If the designs differ, it's two setups and two independent tiers. This is a real margin question shops care about, and the prototype has no concept of it. Model as an optional *"same design as line #N"* grouping on each line.

### 1.2 The shape of an embroidery price

```
per_piece = blank_cost(style)
          + stitch_rate[qty_tier] × ceil(stitch_count / 1000)
one_time  = digitizing_fee    # only if this logo has never been digitized
per_run   = machine_setup     # small, per run / per thread-change
```

- Commercial rate: **$0.50–$1.50 per 1,000 stitches** in the US, dropping 10–25% above 50 units and up to 40% above 500.
- **Digitizing** — converting the logo to a stitch file (`.DST`) — is a **one-time $40–$100 fee that is never charged again for that logo**. It has a completely different lifecycle from a screen setup, which recurs per run.
- Reference stitch counts: left-chest logo 5k–8k, cap front 6k–10k, full back 15k–25k. Prices in practice: $3–10 chest, $4–8 cap, $10–30 full back.
- Caps price differently from flats (cap-specific hooping, 3D puff option) — worth a modest surcharge line.

> **The prototype charges one flat `$30` "logo setup" for both screen printing and embroidery** (`index.html:705-707`), keyed off a customer self-reported `logoOnFile` boolean. That single number is standing in for three genuinely different fees with three different lifecycles: per-screen setup (recurs, scales with colours × locations), digitizing (one-time, permanent), and art/vectorization (one-time, conditional on file quality). Collapsing them means reorders are overcharged and multi-colour multi-location jobs are massively undercharged.

Also: **the customer does not know their stitch count.** Asking for it (a 2-option dropdown: 5k / 8k) asks the wrong party the wrong question. Ask for *design size* and *location*; estimate stitch count server-side from a lookup; let the shop correct it during pre-press.

### 1.3 Rush

Industry standard turnaround is ~2 weeks; inside that is a rush. The prototype **detects** rush correctly at `< 14 days` (`validateTurnaroundTimeline()`, `index.html:409-424`) and renders a banner reading *"⚡ Rush Timeline Surcharge Applied"* — and then `calculateManifestTotals()` never applies one. The customer is told there is a surcharge and quoted a price without it. `ASSESSMENT.md` §5 even lists "rush-order percentage" as a metric to instrument. **The rush surcharge exists in the UI copy and in the metrics plan and nowhere in the math.**

### 1.4 What the quote form must capture to avoid the follow-up call

Ranked by how often its absence forces a phone call:

| Field | Why the call happens without it | In prototype? | In PRD FR-7? |
|---|---|---|---|
| Print **location(s)** + colours per location | Can't count screens → can't price at all | Partial (front/back only) | No |
| **Garment colour** (or at least light/dark) | Underbase changes screen count and price | Yes | **No — dropped from FR-7's field list** |
| Quantity **per size** | Extended-size upcharge; also the press schedule | Yes (adult + youth) | Yes |
| **Design size** (approx. inches) | Drives stitch count and screen size | No | No |
| Artwork file | Can't quote art fee, can't start pre-press | Broken | Yes (FR-1) |
| **In-hands date** (distinct from production date) | "Due date" is ambiguous — is that when you need it, or when we finish? | No (one date) | No |
| Garment brand/style code | Blank cost is the largest per-unit variable | Yes | Yes |
| Is this a **reorder**? | Skips digitizing/art fee entirely | Partial (`logoOnFile`) | No |
| Delivery: pickup vs ship vs deliver | Affects the last stage and the date math | No | No |
| Names/numbers on individual pieces | Per-piece personalization charge | No | No |

**In-hands vs. production date** is a real trade distinction and worth calling out: the in-hands date is when the customer must physically have the goods; the production date is when the shop finishes, with a deliberate buffer for misprints, short shipments, and damaged blanks. Scheduling to the in-hands date means nothing is allowed to go wrong. Two date fields, one of them buffered.

---

## 2. Artwork intake

### 2.1 What formats shops actually need, and why

| Format | Status | Why |
|---|---|---|
| `.ai`, `.eps` | **Preferred** | Native vector; editable; separations can be pulled per colour |
| `.pdf` (vector) | **Preferred** | Vector-preserving and universally openable |
| `.svg` | Accepted | Vector, but font/effect fidelity is unreliable from web tools |
| `.png` / `.tif` / `.psd` at 300 DPI **at final print size** | Accepted with caveats | Usable but cannot be cleanly colour-separated |
| `.jpg` | Accepted reluctantly | Lossy; compression artefacts become visible halftone noise |
| `.zip` | **Needed and missing everywhere** | Designers routinely send AI + fonts + preview as one archive |

The "300 DPI" rule is **300 DPI at the actual printed dimensions** — a 300 DPI file that's 2" wide is useless for an 11" full-front print. This is the single most common customer misunderstanding.

`FR-1` gets the format list right (`PDF, AI, EPS, SVG, PNG, JPG`) — a genuine improvement over the prototype's `accept="image/*"` (`index.html:606`), which excludes every format a print shop actually wants. Three implementation traps sit under it though, listed in §8.

### 2.2 What goes wrong with customer-supplied art

Ranked by frequency:

1. **Logo pulled off the customer's own website.** 72 DPI, a few hundred pixels wide, often with a white box behind it. Unusable at print size. This is *the* daily reality of the trade.
2. **Low-res JPEG.** Lossy artefacts amplify into visible noise in halftones; upscaling makes it worse, not better.
3. **Vector file with live text and no fonts.** Opens with substituted fonts; the logo is silently wrong.
4. **Wrong colour mode / no Pantone reference.** RGB screen colours don't map to plastisol ink; "make it the same blue" needs a PMS number.
5. **No file at all at quote time.** Customer wants a price before committing to art.

### 2.3 What a good intake flow does about it

- **Accept the file first, judge it second.** Blocking submission on file quality loses the lead. Take it, then flag it.
- **Pre-flight warning at upload, not rejection.** For raster uploads, read pixel dimensions and warn inline: *"This image is 480×200px. For an 11-inch front print we need roughly 3300px wide. We can usually still work with it — there may be an art fee to recreate it."* Sets the art-fee expectation *before* the customer is anchored on a price. Vector files can't be checked this way; just accept them.
- **Make artwork optional with an explicit state.** A quote with no art is still a quote; it's a job that cannot leave pre-press. The prototype's default label — `"No file attached"` — has the right instinct. Make "awaiting artwork" a visible board condition, not an empty field.
- **Multiple files per line item.** Front art, back art, a reference photo, a PMS swatch. One-file-per-line breaks in week one.
- **Per-line-item, not per-quote.** Tees carry logo A, hats carry logo B. The prototype already models art per manifest line (`handleManifestLineArtUpload(id, event)`); `FR-1` doesn't say either way, and getting this wrong is a schema change later.
- **A second, shop-side file slot.** The team produces a mockup/proof and print-ready separated art. Those belong on the job too. `FR-1` only models customer → shop.

---

## 3. Production stages

### 3.1 What real shop-management tools use

Printavo's recommended status set for a screen-printing shop:

> Quote sent → Awaiting approval → Approved → **Waiting on goods** → Ready for production → In production → Quality check → Ready for pickup / Shipped → Complete

Their stated design rule is worth adopting verbatim: **"a status should exist only when it reflects a real stage change, an ownership handoff, or a meaningful operational pause"** — with detailed work (confirm ink colours, order garments) handled by **task checklists inside a stage**, not by more stages. Most shops land at 7–10 statuses.

### 3.2 Verdict on the prototype's / PRD's model

`FR-9` proposes three columns: **Pending/Pre-Press → In Production → Ready/Complete**.

**This is the wrong split, and it fails the tool's own stated purpose.** `PRD.md` §1 says the board must answer *"what's in progress, what's **stuck**, and **who do we need to call**?"* The two places jobs actually get stuck in this trade are:

1. **Waiting on the customer** to approve the proof / mockup. Universal industry gate — no shop burns screens or digitizes before sign-off, and once a mockup is approved, errors in it become the customer's responsibility. Jobs sit here for days.
2. **Waiting on blanks** from the supplier. Shops don't stock inventory (correctly out of scope); they order from SanMar per job and wait 2–5 days.

Both stalls are **blocked on an external party, not on the shop** — and both are invisible inside "Pending/Pre-Press." A job waiting five days on a customer email looks identical to a job the worker is actively pre-pressing. The board cannot answer "what's stuck and who do we call," which is the reason it exists.

Meanwhile **pre-press and production are collapsed in the wrong direction.** In a 3-person shop, burning screens and running the press are usually the *same person on the same day* — an internal handoff with no operational pause. By Printavo's own rule that does not deserve its own column; it deserves a checklist. And the prototype **already has that checklist**, correctly specialized per decoration method (`index.html:1146`):

- Screen printing → `Art Approved, Blanks Staged, Sizes Verified, Color Checked, Screens Burnt, Inks Prepped`
- Embroidery → `Art Approved, Blanks Staged, Sizes Verified, Color Checked, DST Proofed, Threads Prepped`

`.DST` is the Tajima stitch-file format. That is real trade knowledge and it is exactly the right pattern. **Keep the checklists; fix the columns.**

### 3.3 Recommended stage model

One linear enum (simpler than a second "blocked" axis, and the team is non-technical):

| # | Stage | Blocked on | Notes |
|---|---|---|---|
| 1 | **New — Needs Quote** | Us | Submitted, nobody has priced or reviewed it |
| 2 | **Quote Sent — Awaiting Approval** | **Customer** | Price + mockup out; nothing happens until sign-off |
| 3 | **Approved — Awaiting Blanks** | **Supplier** | Goods on order from SanMar / Cotton Collective / Otto |
| 4 | **In Production** | Us | Pre-press *and* press/machine, driven by the existing per-method checklist |
| 5 | **Ready — Pickup / Delivery** | Customer | |
| 6 | *(Complete / archived)* | — | Can be a filter on stage 5 rather than its own column |

Five visible columns, one glanceable read of *customer / supplier / us*, and it fits inside Printavo's 7–10 guidance. `Needs Call` remains a separate side list as it is today.

**Constraint on proof approval:** with customer accounts and customer status links out of scope, v1 approval is **team-recorded, not customer-self-service** — a `proof_sent_at` date, an `approved_at` date, and a button. No customer-facing approval link. That is a deliberate v1 boundary, not an oversight, and it should be written down so it isn't re-litigated during build.

---

## 4. Table Stakes (shop can't operate without these)

| Feature | Why expected | Complexity | Notes |
|---|---|---|---|
| Multi-line-item quote | One job = tees + hoodies + hats. Single-garment quote forms fail immediately | MEDIUM | Prototype already does this — carry it forward (`FR-7` ✓) |
| Adult **and** youth size grids | Schools, teams, church groups. Half this shop's market | LOW | Prototype ✓, `FR-7` ✓ |
| Artwork that actually arrives | The entire premise (`PROJECT.md` Core Value) | MEDIUM | `FR-1` ✓ — but see §8 traps |
| **Multiple files per line item** | Front art + back art + reference. One-file breaks week one | MEDIUM | Not in `FR-1` |
| **Quantity price-break tiers** | Defines the trade's economics. Absent = lost volume orders | MEDIUM | Missing everywhere |
| **Extended-size upcharge (2XL+)** | Standard, ~$2/$3/$4. Data already collected | LOW | Missing everywhere |
| **Setup fee per screen** (colours × locations) | $15–30/screen. Flat $30 is wrong by 2–4× on real jobs | LOW (after locations exist) | Missing everywhere |
| **Digitizing fee, one-time and remembered** | $40–100, never charged twice for the same logo | MEDIUM | Conflated with screen setup |
| **Rush surcharge actually applied** | Currently promised in UI copy, absent from math | LOW | Detection already exists |
| **Print locations with per-location colours** | Can't count screens without it; can't price at all | MEDIUM | Front/back only |
| **Garment colour / dark-garment flag** | Underbase = extra screen + extra colour | LOW | Prototype ✓, **dropped from `FR-7`** |
| **Manual job entry (walk-in / phone)** | Most local-shop work walks in or calls | MEDIUM | Prototype ✓ (`openManualJobModal`), **absent from PRD entirely** |
| **Manual lead entry** | Leads come from conversations, not just the form | LOW | Prototype ✓, `FR-13` only describes reading/marking |
| **Edit an existing job** | Customers change size counts constantly | MEDIUM | **No FR exists.** Board diverges from reality on day 2 |
| Server-computed, tamper-proof price | `ASSESSMENT` §2.6 / §3.4 | MEDIUM | `FR-5` ✓ |
| Per-method production checklist | Screens burnt vs. DST proofed are different jobs | LOW | Prototype ✓, **no FR** |
| Bidirectional stage moves + confirm | Misclicks are a stated pain point | LOW | `FR-10` ✓ |
| Audit attribution | Three people, one board | MEDIUM | `FR-11` ✓ |
| Human-readable job number | It's what they say on the phone | LOW | Prototype uses `Math.random()` — collides |
| Invoice-sent / paid **status** toggles | `PRD` §2 says the team sees "payment status" | LOW | Prototype ✓, no FR. **Tracking ≠ processing — within scope** |
| Due-date + overdue visibility | `T9` | MEDIUM | `FR-15` ✓ |
| Honest failure states | `ASSESSMENT` §2.4 | LOW | `FR-4` ✓ |

## 5. Differentiators (real advantage worth building)

| Feature | Value proposition | Complexity | Notes |
|---|---|---|---|
| **"Who are we waiting on"** as a first-class board read | Directly answers the tool's stated purpose. Nothing at this price point does it well | LOW (it's just the stage enum) | Highest value-per-hour in the build |
| **Printable work order / pull sheet** | The physical ticket that walks with the job to the press. Likely the most-used artifact in the shop | LOW–MEDIUM | Print stylesheet on the job detail modal |
| **Reorder awareness / logo-on-file registry** | Shop-side record of which customer logos are digitized and screened. Kills the digitizing re-charge, pre-fills art, speeds repeat quotes | MEDIUM | Shop-side data keyed on customer — **no customer login, stays in scope** |
| **Artwork pre-flight warning at upload** | Sets the art-fee expectation before the customer anchors on a price. Defuses the #1 daily friction in the trade | MEDIUM | Raster only; can't inspect vectors |
| **Time-in-stage / stall highlight** | `ASSESSMENT` §5 asks for it and no FR delivers it. "3 days awaiting approval" on the card is the whole product | LOW–MEDIUM | Derive from audit trail (`FR-11`) |
| **Customer confirmation email** | Written record of what they asked for; doubles as proof the submission landed | LOW | Resend already in stack. **A one-shot receipt is not the deferred "customer status link"** |
| **Daily due/overdue digest** | Arguably more valuable than the new-quote email — nobody opens a calendar daily | LOW–MEDIUM | Vercel Cron + Resend |
| **MOQ soft-warning** | Screen print MOQ is ~12–24; a 3-shirt request is a lead, not a job. Route it to Needs Call instead of rejecting | LOW | Fits the existing Needs Call concept exactly |

## 6. Anti-Features (deliberately NOT building)

| Feature | Why it gets requested | Why it's wrong here | Instead |
|---|---|---|---|
| **Drag-and-drop kanban** | It's what a board "should" feel like | Directly hostile to the stated requirement. `PROJECT.md` Key Decisions: *"Confirmation modal on every stage change — misclicks were a stated pain point."* DnD *is* the misclick, and it's worse on the phone they'll actually use in the shop | Explicit move buttons + confirm modal (`FR-10`) |
| **Real-time collaborative board** | Three people on one board sounds like it needs sync | Already P2-deferred — reinforcing it. Three people in one building with ~20 jobs. Realtime adds subscription lifecycle, reconnection, and stale-cache bugs to buy nothing | Server Action revalidation; a refresh is fine |
| **Exact, binding automated quotes** | "Why can't the form just give the real price?" | The shop's own success copy says *"rough preliminary baseline… a specialist will review your artwork."* That's not a cop-out, it's the trade — final price depends on art the shop hasn't seen. Chasing exactness adds enormous edge-case logic to produce a number the shop overrides anyway | Good-faith estimate + explicit "estimate, not final" framing |
| **SanMar / blank catalog integration** (live style, colour, price lookup) | Would auto-fill blank costs | This *is* the separately-scoped drop-ship project (`PROJECT.md`, `ASSESSMENT` §8). Pulling it in as "just an API call" is exactly how the scope boundary dissolves | Link out to sanmar.com / cottoncollective.org / ottocap.com + free-text style code — what the prototype already does correctly |
| **Online design studio / clipart builder** | It's the headline feature of InkSoft and DecoNetwork | Months of work. This shop's customers send a logo file; they do not want to design in a browser | Artwork upload |
| **Barcode scanning / scan-in-scan-out job travelers** | YoPrint's marquee feature; feels professional | Requires hardware, printed travelers, and daily discipline from three people who can already see the whole shop | Checkbox checklist on the card |
| **Production scheduling / press capacity planning / gang-sheet optimization** | "Optimize the schedule" | Built for shops with multiple presses and shifts. One press, one worker. The schedule is a due-date list | Calendar (`FR-15`) |
| **Time tracking / per-job labour costing** | Margin analysis | Requires the worker to clock in and out of jobs. It will not happen, and half-entered data is worse than none | Nothing |
| **Reporting dashboards / BI charts** | Every SaaS competitor has them | ~20 active jobs. The report *is* the board. Charts over 20 rows are decoration | The handful of counters in `ASSESSMENT` §5 |
| **Role separation / approval gates** | Sounds like governance | Explicitly declined (`PROJECT.md`, `ASSESSMENT` §6.1). Attribution is the accountability mechanism | Audit trail (`FR-11`) |
| **Customer portal / accounts / self-service approval links** | Would cut "where's my order?" calls | Out of scope; status links are P2-deferred | Team records proof-sent / approved dates |
| **Automated customer follow-up sequences, SMS drips, review requests** | Marketing lift | Marketing is out of engagement scope; also a deliverability and consent surface a 3-person shop shouldn't own | The Needs Call list |
| **Names & numbers roster management** (per-piece personalization) | Real trade feature for sports teams | Meaningful schema and UI cost for an occasional order type at this shop's scale | Attach the customer's roster spreadsheet as a job file; note the per-name charge in notes. Revisit if team orders become routine |
| **Quote versioning / e-signature / quote expiry** | "Track the revision history" | Already P2-deferred. The audit trail covers *who changed what* adequately for three people | `FR-11` |
| **Multi-location / multi-shop support** | Future-proofing | One location. This is speculative generality with a real schema tax | Nothing |

---

## 7. What the prototype got right (carry forward deliberately)

`PROJECT.md` says the prototype *"demonstrably captures the shop's real workflow."* Confirmed — these are load-bearing and easy to lose in a rewrite:

- **Multi-item manifest.** One quote, N garment lines. Most naive quote forms model 1 quote = 1 garment and fail on the first real order.
- **Adult + youth size grids per line.**
- **Headwear breaks the size model correctly.** `isHatsSelected` swaps the S–4XL grid for a single OSFA quantity (`index.html:518-532`). Caps don't have shirt sizes. Correct conditional domain modeling.
- **Decoration-specific production checklists** — `Screens Burnt / Inks Prepped` vs `DST Proofed / Threads Prepped`. Genuine trade vocabulary, and structurally the right pattern (tasks inside stages, not more stages).
- **Supplier link-outs by decoration type** — SanMar/Cotton Collective for flats, Otto Cap for headwear. Correct, and correctly *not* an integration.
- **Garment tiering as the price axis** — Gildan economical / Next Level midweight / Bella+Canvas premium / specialty heavyweight. Right axis; blank cost dominates per-unit price.
- **14-day rush threshold.** Matches the industry standard 2-week turnaround.
- **"Rough estimate, a specialist will finalize"** framing. Honest, and correct for the trade.
- **Invoice-sent / paid as separate booleans**, with an "Awaiting Payment" panel distinct from the production board.
- **Leads carry size breakdowns and a follow-up date**, not just a phone number — a lead in this trade is a half-formed quote.

---

## 8. Gaps in the drafted PRD §4

Ordered by how fast the shop hits them.

### Would break in week one

| # | Gap | Detail |
|---|---|---|
| G1 | **No manual job entry FR** | Walk-ins and phone orders are most of a local shop's volume. The prototype has `openManualJobModal()` with a flat-price override — correct for phone quoting. With only the public form, the team will type customer data into the *public customer form* as a workaround, and the audit trail will record every job as customer-submitted. Add an FR. |
| G2 | **No "edit a job" FR** | Nothing in FR-1…FR-16 permits changing a job after creation. Customers revise size counts, colours, and dates constantly. Without edit, the board is wrong by day two and the team goes back to texting. Needs a decision on whether edits re-run pricing. |
| G3 | **Pricing model is undefined** | `FR-5` specifies *where* pricing runs (server, tamper-proof) and never *what it computes*. Per §1: no quantity tiers, no size upcharges, no per-screen setup, no digitizing distinction, no underbase, and the rush surcharge is promised in UI copy but never applied. `FR-5`'s acceptance criteria all pass with the current wrong formula. Needs its own FR with the pricing model as an explicit input. |
| G4 | **Garment colour dropped from FR-7** | `FR-7` says "preserving current capability" then enumerates fields — and garment colour isn't among them, though the prototype has it (`input-color`) and it drives underbase pricing. Enumerated lists override intent during build. |
| G5 | **Artwork cardinality unspecified** | `FR-1` never says whether artwork attaches per-quote or per-line-item, or whether more than one file is allowed. Both answers are schema decisions. Should be: per line item, multiple files, plus a shop-side slot for the proof and print-ready separations. |
| G6 | **AI/EPS MIME types will fail server-side validation** | `FR-1` requires type "validated **server-side**." `.ai` files report as `application/postscript`, `application/pdf`, or `application/octet-stream` depending on export vintage; `.eps` similarly. MIME-only validation rejects legitimate customer art and reproduces the original bug with better error messages. Validate on extension **plus** magic bytes. |
| G7 | **File size cap will bounce real art** | `FR-1` requires "a maximum file size" without a number. Designer AI/EPS/PDF files with embedded raster routinely run 50–200 MB. A typical web default of 5–10 MB rejects normal customer art. Recommend ≥50 MB and an explicit acceptance test with a large layered `.ai`. |
| G8 | **`.zip` not accepted** | Designers send AI + fonts + preview as one archive as a matter of course. Rejecting `.zip` sends the customer back to email — exactly the failure `FR-1` exists to fix. Store it; never expand it server-side. |
| G9 | **No print-location field** | Can't count screens → can't compute setup → can't price. Front/back integers can't express "left chest + sleeve." |

### Would break in month one

| # | Gap | Detail |
|---|---|---|
| G10 | **Stage model hides both real stall states** | See §3.2. `FR-9`'s three columns cannot answer "what's stuck / who do we call," which `PRD` §1 names as the tool's purpose. Needs Awaiting-Approval and Awaiting-Blanks stages. |
| G11 | **No proof / mockup approval tracking** | The universal production gate in this trade, and the origin of the "I approved that" dispute. Minimum: `proof_sent_at`, `approved_at`, `approved_by_name`. No customer-facing link needed in v1. |
| G12 | **Per-method production checklist has no FR** | The prototype's `tasks[]` array with decoration-specific labels is real domain knowledge that exists nowhere in the PRD and will simply be lost in a rewrite. |
| G13 | **One date field for two concepts** | In-hands date vs. production date. Scheduling to the in-hands date means nothing is allowed to go wrong. Also affects what `FR-15`'s calendar and the "overdue" flag actually mean. |
| G14 | **Payment status has no FR** | `PRD` §2 states team members see "pricing and **payment status**," and the prototype has invoice-sent / paid toggles plus an Awaiting Payment panel — but no FR covers it. Checked against out-of-scope: `PROJECT.md` excludes *payment processing* — "the tool quotes and tracks; it does not collect money." A boolean marked by hand is tracking. **In scope; needs an FR or an explicit decision to drop it.** |
| G15 | **No minimum order quantity** | `FR-6` validates against negatives but not against a 3-piece screen-print request. MOQ ~12–24. Soft-warn and route below-MOQ requests to Needs Call. |
| G16 | **Job IDs collide** | `"HPC-" + Math.floor(1000 + Math.random() * 9000)` is a 9,000-value space used as a primary key — a duplicate is likely inside the first ~100 jobs. `FR-3` specifies the table but not the identifier. Use a Postgres sequence and display it. |
| G17 | **No time-in-stage surfacing** | `ASSESSMENT` §5 calls it "the single most useful number a production board can produce" and `PRD` §6 lists it as a ship criterion — but no FR produces it. Derivable from `FR-11`; needs to be stated. |
| G18 | **No customer confirmation email** | `FR-14` notifies the team only. The customer gets an on-screen message and no record. Cheap on an already-provisioned Resend account, and it reinforces `FR-4`'s "never a false success." |
| G19 | **No "art not yet supplied" state** | A quote without art is legitimate but cannot leave pre-press. It needs to be visible on the board as a callback reason, not an empty field. |

### Worth a decision, not necessarily a build

| # | Gap | Detail |
|---|---|---|
| G20 | **DTF is not modeled** | Direct-to-film transfers have become the default method for small runs — no screens, no setup, no minimum, flat per-piece cost regardless of colour count, and most decorators now run it alongside screen printing to serve jobs their press can't take economically. The prototype offers Screen Printing / Embroidery / Marketing Materials only. **Ask the shop whether they run DTF.** If yes it's a third decoration type with a genuinely simpler price model (and it's what should absorb below-MOQ requests from G15). If no, record the answer so it isn't rediscovered mid-build. |
| G21 | **Rush surcharge amount undefined** | Detection exists; the multiplier or flat fee is a client input, alongside the price list `ASSESSMENT` §8 already lists as client-provided. |
| G22 | **"Marketing Materials" decoration type** | Banners, business cards, signage — priced per-unit with no size grid. Present in the prototype, absent from the PRD's decoration enum discussion. Confirm it stays. |
| G23 | **Artwork retention** | Already flagged as an open question in `PROJECT.md`. Note that it interacts with G-reorder value: keeping files indefinitely is what makes the logo-on-file registry work. |

---

## 9. Feature Dependencies

```
Print locations (per-location colour counts)
    └──required by──> Per-screen setup fee
                          └──required by──> Correct screen-print total
Dark-garment flag
    └──required by──> Underbase screen count ──> Correct screen-print total

Quantity price-break tiers
    └──required by──> Correct screen-print total
    └──required by──> Correct embroidery total
Size upcharge table ──required by──> Correct total (data already captured)

Logo-on-file registry (shop-side)
    └──required by──> One-time digitizing fee
    └──enhances────> Reorder quoting speed
    └──requires────> Artwork retention decision (PROJECT.md open question)

Stage: Awaiting Approval ──requires──> proof_sent_at / approved_at fields
Stage: Awaiting Blanks   ──requires──> nothing (enum value only)
Time-in-stage display    ──requires──> Audit trail (FR-11)
Stall highlighting       ──requires──> Time-in-stage + the split stages above

Manual job entry ──shares──> the quote form + pricing engine (build after FR-7/FR-5)
Edit job         ──shares──> the same form; ──requires──> FR-11 for field-level attribution
Work order print view ──requires──> Job detail modal (FR-12)
Daily due digest ──requires──> in-hands vs production dates (else it fires on the wrong day)

Artwork pre-flight ──requires──> upload pipeline (FR-1)
Multiple files/line ──must precede──> FR-1 schema, not follow it
```

**Ordering consequences for the roadmap:**

- **Print locations and the dark-garment flag must land in the same milestone as pricing** (proposed Milestone 2). They are pricing *inputs*; retrofitting them means re-migrating and re-pricing existing rows.
- **Artwork cardinality (per-line, multi-file, shop-side slot) must be decided before Milestone 3**, not during it.
- **Stage enum changes belong in Milestone 1's schema**, not Milestone 4. Adding enum values later is cheap; adding them after the board UI is built means rebuilding the board.
- **Manual job entry can follow the customer form** — it reuses the same components and pricing.

---

## 10. MVP Definition

### Launch with (add to P0/P1)

- [ ] **Pricing model FR** — quantity tiers, size upcharges, per-screen setup, digitizing as a distinct one-time fee, underbase, rush surcharge actually applied *(the current `FR-5` passes its acceptance criteria with the wrong formula)*
- [ ] **Print locations with per-location colour counts** *(pricing prerequisite)*
- [ ] **Garment colour + dark-garment flag** *(pricing prerequisite; restore to `FR-7`)*
- [ ] **Artwork: per line item, multiple files, `.zip` accepted, ≥50 MB cap, extension + magic-byte validation** *(amend `FR-1`)*
- [ ] **Manual job entry** for walk-ins and phone orders, with flat-price override
- [ ] **Manual lead entry** *(amend `FR-13`)*
- [ ] **Edit an existing job**, audited
- [ ] **Stage model split** — Awaiting Approval and Awaiting Blanks as distinct stages; pre-press folded into production behind the checklist *(amend `FR-9`)*
- [ ] **Per-method production checklist** carried forward from the prototype
- [ ] **Proof sent / approved dates**
- [ ] **In-hands date separate from production date**
- [ ] **Sequential human-readable job number**
- [ ] **MOQ soft-warning** routing below-minimum requests to Needs Call
- [ ] **Payment status booleans** *(or an explicit decision to drop them, given `PRD` §2 promises them)*

### Add after validation (v1.x)

- [ ] **Time-in-stage on the card** + stall highlight — trigger: first "how long has that been sitting there?"
- [ ] **Printable work order** — trigger: first time someone writes the job on paper to take to the press
- [ ] **Customer confirmation email** — trigger: first "did you get my order?" call
- [ ] **Daily due/overdue digest** — trigger: first missed in-hands date
- [ ] **Artwork pre-flight warning** — trigger: third art-fee argument
- [ ] **Logo-on-file / reorder registry** — trigger: first repeat customer re-charged for digitizing

### Defer (v2+)

- [ ] Names & numbers roster management — use a file attachment until team orders are routine
- [ ] Quote versioning, customer status link, live board sync — already P2 in `ASSESSMENT` §4
- [ ] DTF as a first-class decoration type — pending the G20 answer
- [ ] Anything in §6

### Prioritization matrix (net-new items only)

| Feature | User value | Cost | Priority |
|---|---|---|---|
| Quantity price-break tiers | HIGH | MEDIUM | P0 |
| Extended-size upcharge | HIGH | LOW | P0 |
| Rush surcharge applied | HIGH | LOW | P0 |
| Print locations + per-location colours | HIGH | MEDIUM | P0 |
| Dark-garment flag / underbase | HIGH | LOW | P0 |
| Per-screen setup + separate digitizing fee | HIGH | MEDIUM | P0 |
| Artwork cardinality + `.zip` + size cap + magic-byte validation | HIGH | MEDIUM | P0 |
| Manual job entry | HIGH | MEDIUM | P0 |
| Edit a job | HIGH | MEDIUM | P0 |
| Stage split (approval / blanks) | HIGH | LOW | P0 |
| Per-method checklist | MEDIUM | LOW | P0 |
| Sequential job number | MEDIUM | LOW | P0 |
| Proof sent / approved dates | MEDIUM | LOW | P1 |
| In-hands vs production date | MEDIUM | LOW | P1 |
| Manual lead entry | MEDIUM | LOW | P1 |
| MOQ soft-warning | MEDIUM | LOW | P1 |
| Payment status booleans | MEDIUM | LOW | P1 |
| Time-in-stage / stall highlight | HIGH | LOW–MED | P1 |
| Customer confirmation email | MEDIUM | LOW | P1 |
| Printable work order | HIGH | LOW–MED | P2 |
| Daily due digest | MEDIUM | LOW–MED | P2 |
| Artwork pre-flight warning | MEDIUM | MEDIUM | P2 |
| Logo-on-file registry | MEDIUM | MEDIUM | P2 |

---

## 11. Competitor feature comparison

| Feature | Printavo (~$400/mo) | YoPrint (~$149/mo) | This tool |
|---|---|---|---|
| Job statuses | Fully customizable, 7–10 recommended | Customizable | Fixed 5-stage enum — three non-technical users, must be obvious without training |
| Art approval | Emailed proof, customer approves online | Customer portal with approvals | Team-records sent/approved dates; no customer login (out of scope) |
| Quoting | Full line-item pricing matrices | Full pricing matrices + inventory | Server-computed estimate, explicitly non-binding |
| Customer portal | Yes | Yes — orders, quotes, approvals, balance | **No** — public link only, by design |
| Payments | Collects | Collects | **Tracks status only** — collection out of scope |
| Inventory / purchasing | Yes | Yes | **No** — out of scope |
| Barcode job travelers | — | Core feature | **No** — anti-feature at 3 people |
| Scheduling | Drag-and-drop calendar | Yes | Due-date calendar, no capacity planning |
| Cost | $400/mo | $149/mo | Barter build, no recurring licence |

**Positioning:** this is not a cheaper Printavo. It is the ~15% of Printavo that a 3-person shop touches daily, with the quote form as the front door and no monthly fee. Every anti-feature in §6 is something a competitor charges for that this shop would never open.

---

## Sources

**Pricing structure** (MEDIUM-HIGH — structure corroborated across many independent shop price lists; dollar figures vary regionally and are illustrative only, the client's real price list is a documented build input per `ASSESSMENT` §8)
- [Screen Printing Pricing Guide — Printable Press](https://www.printablepress.com/screen-printing-pricing-guide/)
- [How to price a screen printing job without leaving money on the table — InkTracker](https://www.inktracker.app/blog/how-to-price-a-screen-printing-job)
- [Screen Printing Pricing Guide: Cost Per Shirt by Color and Quantity — CraftsTrack](https://craftstrack.app/blog/screen-printing-pricing-guide)
- [Screen Printing Pricing Guide — Branded Reno](https://www.brandedscreenprinting.com/screen-printing-pricing/)
- [Custom Shirt Pricing FAQ — Main Street Shirt Company](https://mainstreetshirtcompany.com/pages/custom-shirt-pricing-faq) (2XL+ upcharge)
- [Why Screen Printing Costs More on Black Fabric — Tanners](https://tannersinc.net/why-is-it-more-expensive-to-screenprint-on-black-fabric/) (underbase as extra screen)
- [4 FAQs About Using A White Under Base — Sharprint](https://www.sharprint.com/blog/bid/104384/4-faqs-about-when-to-use-a-white-under-base-when-screen-printing)
- [Embroidery Charges Per Stitch — MaggieFrames](https://www.maggieframes.com/blogs/embroidery-blogs/embroidery-charges-per-stitch-the-complete-pricing-guide)
- [How Much Does Embroidery Cost? Pricing by Stitch Count — Arklavo](https://arklavo.com/blogs/custom-apparel-guide/how-much-does-embroidery-cost)
- [Embroidery digitizing cost per 1,000 stitches — Rise Digitizing](https://risedigitizing.com/embroidery-digitizing-cost-per-1000-stitches/)

**Artwork requirements** (HIGH — consistent across every shop artwork-guideline page reviewed)
- [Artwork Guidelines — Culture Studio](https://culturestudio.net/artwork-guidelines/)
- [Artwork Requirements — ML Screen Printing](https://www.mlscreenprinting.com/artwork-requirements)
- [Best File Formats for Screen Printing: AI, EPS, SVG or PDF? — Vector Tracing Pro](https://www.vectortracingpro.com/blogs/news/best-file-formats-for-screen-printing)
- [Fixing Bad Customer-Supplied Artwork — Graphics Pro](https://graphics-pro.com/feature/lets-talk-shop-fixing-bad-customer-supplied-artwork/)
- [Helping Custom Decorated Apparel Clients Understand Problematic Artwork — Impressions](https://impressionsmagazine.com/process-technique/how-can-custom-decorated-apparel-clients-better-understand-problematic-artwork/169984)

**Production stages and shop workflow** (HIGH for the recommended status list — sourced from Printavo's own guidance)
- [Print Shop Job Tracking Software: Build Better Statuses — Printavo](https://www.printavo.com/blog/print-shop-job-tracking-software/)
- [3 Ways To Use Statuses in Printavo](https://www.printavo.com/blog/3-ways-to-use-statuses-in-printavo/)
- [Use Proofs, Save Money in Your Screen Printing Business — Printavo](https://www.printavo.com/blog/use-proofs-save-money-in-your-screen-printing-business/)
- [Mockup & Proof Approval Policy — Tee Imprints](https://teeimprints.com/pages/mockup-proof-approval-policy)
- [How To Handle Production Scheduling In Your Print Shop — Printavo](https://www.printavo.com/blog/how-to-handle-production-scheduling-in-your-print-shop/) (in-hands vs production date)
- [A Guide to Industry Standard for Screen Print Placements — ScreenPrinting.com](https://www.screenprinting.com/blogs/news/a-guide-to-industry-standard-for-screen-print-placements-and-dimensions)

**Competitive landscape** (MEDIUM — vendor-published comparisons, read with that bias in mind)
- [Printavo vs YoPrint — YoPrint](https://www.yoprint.com/printavo-vs-yoprint)
- [Top 8 Screen Printing Shop Management Software Picks — DecoNetwork](https://www.deconetwork.com/top-8-screen-printing-shop-management-software-picks/)
- [Screen Printing Software (17 Tools) — ConvertCalculator](https://www.convertcalculator.com/blog/software-for-screenprinters/)

**DTF market context** (MEDIUM — vendor sources, directionally consistent; flagged as a question for the client rather than a recommendation)
- [DTF vs Screen Printing: Which Is Better for Small Orders? — Crystal DTF](https://crystaldtf.com/blogs/learn/dtf-vs-screen-printing-small-orders)
- [DTF vs Screen Printing: Which Is Right for Your Shop? — Transfer Superstars](https://www.transfersuperstars.com/blogs/news/seo-title-dtf-vs-screen-printing-which-is-right-for-your-shop-2026-guide)

**Project sources** (HIGH — read directly)
- `PRD-hustle-print-hub.md` §4 (FR-1…FR-16), §5 non-goals
- `.planning/PROJECT.md` (out-of-scope list, key decisions)
- `ASSESSMENT.md` §1, §2, §4, §5, §6, §8
- `index.html` — pricing engine `:672-725`, art upload `:471-485`, stages `:1054-1061`, per-method checklists `:1146`, manual job modal `:998-1024`, manual lead entry `:935-989`, rush detection `:409-424`

---
*Feature research for: decorated-apparel quoting and production tracking, 3-person shop*
*Researched: 2026-07-30*
