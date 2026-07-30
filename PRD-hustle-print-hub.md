# PRD — Hustle Print Hub

**Type:** PRD
**Status:** Draft for milestone planning
**Companion doc:** `ASSESSMENT.md` — current-state findings, architecture rationale, and scope boundary. This PRD does not restate them.

---

## 1. Problem

A three-person screen-print and embroidery shop runs quoting and production tracking through a single-file web app. Three things are broken badly enough to block daily use:

- Customer artwork is never actually received (`ASSESSMENT.md` §2.1).
- The production board is publicly accessible to anyone with the URL (§2.2).
- Failed submissions are reported to customers as successes (§2.4).

The rebuild delivers a tool the owner, his business partner, and his worker can all rely on to answer: *what needs doing, what's in progress, what's stuck, and who do we need to call?*

## 2. Users

| Actor | Who | Access |
|---|---|---|
| **Customer** | Anyone sent the quote link | Public quote form only. No account. |
| **Team member** | Owner, business partner, worker | Full dashboard. Individual login. |

**Permission model: two roles, differing on two things only.**

| | Owner (Josh) | Staff (partner, worker) |
|---|---|---|
| See jobs, stages, specs, artwork, customer contact | ✅ | ✅ |
| Move jobs between stages | ✅ | ✅ |
| Create and edit jobs | ✅ | ✅ |
| **See prices, totals, payment status** | ✅ | ❌ |
| **Approve a lead into Pending** | ✅ | ❌ |

Everything not listed as owner-only is flat. Accountability for shared actions comes from per-user attribution in the audit trail rather than from restricting access — which is why individual logins are load-bearing.

> Money visibility is enforced by **omitting price fields from the server payload** for staff accounts. Hiding them in the UI is not hiding them: conditionally-rendered values remain readable in the browser's network tab, which is the same client-side gating failure as the prototype's `?admin=true` (`ASSESSMENT.md` §2.2).

## 3. User stories

### Customer

- **C1** — As a customer, I submit a quote request specifying garment, decoration type, colors, sizes, and due date, so the shop has what it needs without a phone call.
- **C2** — As a customer, I upload my artwork with my request, so I don't have to email it separately.
- **C3** — As a customer, I see an accurate estimated price before submitting, so I know roughly what I'm committing to.
- **C4** — As a customer, I get clear confirmation my request was received — and a clear error if it wasn't.

### Team member

- **T1** — As a team member, I log in with my own account, so board changes are attributable to me.
- **T2** — As a team member, I see all jobs organized by production stage, so I know the state of the shop at a glance.
- **T3** — As a team member, I move a job forward or backward between stages, confirming first, so misclicks don't corrupt the board.
- **T4** — As a team member, I open a job to see full detail — sizes, colors, notes, customer contact — without cluttering the board.
- **T5** — As a team member, I download the customer's artwork file from the job.
- **T6** — As a team member, I see which leads need a callback, so interested customers aren't dropped.
- **T7** — As a team member, I am emailed when a new quote arrives, so I don't have to poll the board.
- **T8** — As a team member, I see who last changed a job and when.
- **T9** — As a team member, I see what's due this week and what's overdue.

---

## 4. Functional requirements

Priorities align with `ASSESSMENT.md` §4. Acceptance criteria are the exit condition for each — a requirement is not done until its criteria demonstrably pass.

### P0 — required to ship

**FR-1 — Artwork upload and retrieval** *(C2, T5)*

> **Revised after research.** The original version routed bytes `browser → Server Action → Storage`. That is unbuildable: Vercel caps function request bodies at **4.5 MB** (`FUNCTION_PAYLOAD_TOO_LARGE`) and Next.js caps Server Action bodies at 1 MB by default. Neither limit is enforced by `next dev`, so the naive version passes locally and fails on the first real customer file. Three of four research agents reached this independently.

**Architecture.** A Server Action issues a Supabase **signed upload URL** — path-scoped, short TTL, `upsert:false`. The browser PUTs bytes directly to Storage with a plain `fetch`. Bytes never traverse a Vercel function. This preserves the locked constraint: the browser holds a single-use capability token for one server-chosen path, never a Supabase key.

- Accepts PDF, AI, EPS, SVG, PNG, JPG, **and ZIP** (designers routinely send AI + fonts + preview as one archive).
- Type validated server-side by **extension *and* magic bytes** — `.ai` and `.eps` report as `application/postscript` or `application/octet-stream`, so naive MIME checking rejects legitimate artwork.
- Size ceiling set well above web defaults; production artwork commonly runs 50–200 MB.
- **Artwork is per line item, multi-file.** A quote with three garments has three independent artwork sets.
- Upload failure blocks submission and tells the customer — never a false success.
- **Orphan guard:** the quote row cannot record an artwork path whose bytes never landed. Enforced structurally — `storage.info(path)` verify-then-commit before insert, plus a DB check constraint requiring `uploaded_at` and `byte_size` for any row claiming `uploaded`, where `byte_size` comes from a server-side HEAD of the stored object and never from the client's claim.
- **Verify:** a customer uploads a 60 MB PDF; a team member opens that job and downloads a byte-identical file.
- **Verify:** a `.ai` file uploads successfully rather than being rejected on MIME type.
- **Verify:** the storage bucket is not publicly listable, and a raw object URL without a valid signature is refused.
- **Verify:** the standing invariant query for rows referencing non-existent objects returns zero.

**FR-2 — Authentication** *(T1)*
Clerk protects the entire dashboard. Each team member has an individual account.

- All dashboard routes require an authenticated session.
- `?admin=true` and the triple-click handler are removed from the codebase entirely.
- **Verify:** requesting any dashboard route while signed out redirects to sign-in and returns no data.
- **Verify:** `?admin=true` on any route grants nothing.

**FR-3 — Database and schema**
New Supabase project. Schema in version-controlled migrations.

- A single `quotes` table replaces the `app_orders` / `app_manual_leads` split.
- One row per submitted job, with a stage enum and a `needs_outreach` flag.
- Row Level Security default-deny on every table.
- **Verify:** one submission produces exactly one row.
- **Verify:** migrations apply cleanly to an empty database.

**FR-4 — Reliable submission** *(C4)*
Every write is error-checked, with a visible failure state.

- Success confirmation renders only after a confirmed commit.
- Write failure shows an actionable error and preserves the customer's entered data.
- **Verify:** with the database unreachable, submitting shows an error — not the success screen — and the form retains its input.

**FR-5 — Pricing security** *(C3)*
Pricing is computed on the server per `ASSESSMENT.md` §3.4.

- Price list lives in a `server-only` module, absent from the client bundle.
- The submission payload contains no price fields.
- The stored total equals the total quoted to the customer.
- The live estimate is served by a **Route Handler** (`POST /api/estimate`), debounced with `AbortController` — *not* a Server Action. Next.js dispatches Server Actions one at a time and they are not abortable, so per-keystroke estimates would head-of-line-block the queue and strand a submit behind stale estimates.
- **Verify:** `next build`, then grep `.next/static` for rate-card sentinel values — zero hits. (`server-only` catches direct imports; the grep catches a shared barrel re-export.)
- **Verify:** a hand-crafted request with an injected price field is rejected or ignored, and the server-computed total is stored.
- **Verify:** the total shown to the customer matches the row in the database exactly.

**FR-5b — Pricing model correctness** *(C3)*

> **Added after research.** The original FR-5 specified *where* pricing runs and never *what it computes*. It would have passed its own acceptance criteria while quoting wrong. The prototype's formula is linear `qty × rate` (`index.html:691-707`); decorated apparel is not priced that way.

The engine must support these dimensions. **Rates are client-provided input, not developer assumptions** — see §9.

- **Quantity price breaks.** Tiered by volume (12/24/48/72/144/288). Linear pricing overquotes exactly the large orders worth winning.
- **Size upcharges** for 2XL/3XL/4XL. *The prototype already collects these counts and never prices them* — highest value-per-hour fix in the build.
- **Per-screen setup** charged per color per location, not one flat fee.
- **Digitizing** as a distinct one-time embroidery fee with a different lifecycle from screen setup.
- **White underbase** on dark garments. The garment color field exists and the pricing function never reads it.
- **Rush surcharge.** The UI already tells customers *"⚡ Rush Timeline Surcharge Applied"* and the math never applies it — the shop has been quoting rush jobs at standard rates. This is a live revenue leak, not just a defect.
- **Verify:** a 288-piece order quotes at a materially lower per-piece rate than a 12-piece order of the same garment.
- **Verify:** a run containing 2XL garments prices higher than the identical run in standard sizes.
- **Verify:** a job inside the rush window prices higher than the same job outside it.

**FR-6 — Input validation**
Zod schemas at every server entry point.

- Rejects negative quantities, out-of-range values, unknown enum values, and past due dates.
- **Verify:** a payload with `quantity: -5` is rejected and writes nothing.

**FR-7 — Quote intake form** *(C1)*
Rebuild of the customer-facing form, preserving current capability: multi-item manifest, adult and youth size breakdowns, decoration type, garment brand, **garment color**, ink color count, **print locations**, stitch count for embroidery, due date with rush indication.

> Garment color and print locations are called out explicitly because the original wording said "preserving current capability" and then gave a field list that omitted them. During a build, an enumerated list overrides stated intent. Both are **pricing inputs** — garment color drives white underbase, locations drive per-screen setup — so they must land with the schema, not later.

- **Verify:** a multi-item quote with mixed adult and youth sizes submits and renders correctly on the board.
- **Verify:** garment color and print locations persist and reach the pricing engine.

**FR-8 — Output escaping**
React default escaping; no `dangerouslySetInnerHTML`.

- **Verify:** a quote submitted with `<script>` in the customer name displays as literal text on the dashboard and does not execute.

**FR-8b — Server Action authorization** *(T1)*

> **Added after research.** All three technical researchers independently flagged this. Clerk middleware **cannot** protect Server Actions: actions are POST endpoints dispatched by action ID, and middleware matches paths. A route matcher would give the appearance of protection over exactly the layer it does not cover. Clerk now ships a migration guide away from `createRouteMatcher()` for this reason, and two CVSS-9.1 middleware bypasses in 13 months reinforce it.

- `await auth.protect()` is the first statement of **every** dashboard Server Action — authorization at the resource, not the perimeter.
- Public actions (quote submission, upload URL issuance) live in a separate directory from dashboard actions so the distinction is reviewable at a glance.
- **Verify:** invoking a dashboard Server Action directly while unauthenticated is refused.
- **Verify:** every file under the dashboard actions directory begins with the auth guard — automatable as a CI check.

**FR-8c — Manual job and lead entry** *(T2, T6)*

> **Added after research.** Both exist in the prototype (`index.html:858`, `:935`) and were missing from this PRD entirely. Walk-ins and phone orders are most of a local shop's volume. Without this the team types customer data into the *public* form, and the audit trail labels every job customer-submitted. Restores existing capability rather than adding scope.

- A team member can create a job directly from the dashboard, including a flat-price override for negotiated pricing.
- A team member can add a lead to the callback list manually.
- Manually-created records are distinguishable from customer-submitted ones in the audit trail.
- **Verify:** a job created from the dashboard appears on the board and is attributed to the creating user, not to a customer.

### P1 — feature-complete

**FR-9 — Production board** *(T2)*
Jobs grouped by stage: Pending/Pre-Press, In Production, Ready/Complete, plus a Needs Call list.

- **Verify:** a submitted quote appears in the correct starting column without a manual refresh cycle.

**FR-10 — Bidirectional stage movement with confirmation** *(T3)*
Jobs move forward and backward. Every stage change opens a confirmation modal naming the job and the destination stage.

- Cancelling the modal leaves the job unchanged.
- **Verify:** a job moves Production → Pre-Press and back.
- **Verify:** dismissing the confirmation makes no database write.

**FR-11 — Audit trail** *(T8)*
Every stage change, edit, and delete records actor, action, and timestamp.

- Attribution uses the individual Clerk identity.
- Visible on the job detail view.
- **Verify:** two different users each move a job; both entries appear correctly attributed.

**FR-12 — Job detail modal** *(T4)*
Board cards are clickable, opening full job detail including artwork download and customer contact.

**FR-13 — Leads / outreach view** *(T6)*
Leads needing a callback, with contact details and a way to mark one as contacted.

- Each submission appears exactly once. No duplicates.
- **Verify:** ten submissions produce ten outreach entries, not twenty.

**FR-14 — New quote notification** *(T7)*
Email via Resend on submission.

- Delivery failure is logged and never blocks the customer's submission.
- **Verify:** a submitted quote produces one email containing customer name, job summary, and a dashboard link.

**FR-15 — Production calendar** *(T9)*
Calendar driven by the current date, showing jobs by due date, with overdue clearly marked.

- **Verify:** the visible window includes today; a job due yesterday reads as overdue.

**FR-16 — Soft delete**
Deletes are reversible.

- The `deleted_at` column ships in the **first migration**, not this milestone — every board query needs `where deleted_at is null` from the first query written. Only the restore *UI* is deferred to here.
- **Verify:** a deleted job disappears from the board and is restorable.

**FR-17 — Edit a job after creation** *(T2)*

> **Added after research.** No requirement permitted editing a submitted job. Customers revise size counts constantly; without this the board diverges from reality on day two, which defeats the core value.

- A team member can edit quantities, sizes, notes, and due date on an existing job.
- Edits are recorded in the audit trail with before/after values.
- Editing a priced job re-runs the pricing engine server-side; the stored total never diverges from the current configuration.
- **Verify:** changing a size count updates the stored total, and the change appears in the job's history.

**FR-18 — Payment status tracking** *(T2)* — *pending client confirmation, see §9*

The prototype has invoiced/paid toggles and §2 of this document promises the team sees payment status, but no requirement covered it. Checked against the out-of-scope list: `PROJECT.md` excludes payment **processing**, not payment **tracking**, so hand-marked booleans are in scope.

- A team member can mark a job invoiced and paid.
- Changes are attributed in the audit trail.
- **Verify:** marking a job paid persists and is visible to the other two users.

---

## 5. Non-goals

Explicitly out of scope. Listed so they don't drift in mid-build:

- Role separation or permission tiers (declined — `ASSESSMENT.md` §6.1)
- Payment processing or payment collection
- The SanMar / Cotton Collective drop-ship platform — separate project, separately priced
- Customer accounts or a customer login
- Inventory management
- Marketing, ad creative, SEO content
- Native mobile apps
- Quote versioning, customer status links, and live board sync — P2, deferred

## 6. Success metrics

**Ships when:** all P0 and P1 acceptance criteria pass, and the team runs a full week of real quotes without falling back to the old tool.

**Working when:**

- Zero artwork files lost — every quote with an upload has a retrievable file
- Zero silent submission failures
- Time-in-stage visible for every job
- Lead callback response time measurable
- Owner stops manually checking the board for new quotes

Instrumentation detail in `ASSESSMENT.md` §5.

## 7. Locked constraints

Decided; not open for re-litigation during build. Rationale in `ASSESSMENT.md` §3.

| Decision | Value |
|---|---|
| Framework | Next.js App Router, deployed on Vercel |
| Auth | Clerk — individual accounts, flat permissions |
| Database & storage | Supabase — **new** project |
| Data access | Server Actions with service role key. **Browser never holds a Supabase key.** |
| Pricing | Server-computed. No client-supplied price is ever trusted. |
| Validation | Zod, server-side |
| Email | Resend |

## 8. Proposed milestones

> **Revised after research.** Milestone 1 is substantially heavier than originally drafted, and the artwork spike moved into it. Rationale below the table.

| # | Milestone | Contents | Exit criteria |
|---|---|---|---|
| 1 | **Foundation** | Next.js scaffold, Supabase project, **complete schema** (quotes + line items + `quote_events` + `deleted_at` + `actor_id`), Clerk auth with `proxy.ts`, DAL with `must()` unwrapper, post-build secret grep, Vercel deployment protection + env scoping, Resend domain verification started, **artwork upload spike** | Signed-in user reaches an empty dashboard; signed-out refused; a file PUTs successfully to a signed Storage URL |
| 2 | **Secure quote intake** | FR-4, FR-5, FR-5b, FR-6, FR-7, FR-8, FR-8b | A quote submits end-to-end with a server-computed, tamper-proof, **correctly-tiered** price |
| 3 | **Artwork pipeline** | FR-1 | Customer uploads a 60 MB file; team member downloads it byte-identical |
| 4 | **Production board** | FR-9, FR-10, FR-11, FR-12, FR-8c, FR-17 | Jobs move both directions with confirmation and attribution; team can create and edit jobs |
| 5 | **Leads & notifications** | FR-13, FR-14 | New quotes email the owner; outreach list is duplicate-free |
| 6 | **Operational polish** | FR-15, FR-16, FR-18, metrics | Calendar accurate, deletes reversible, metrics visible |

**Why Milestone 1 carries this much:**

- **The artwork spike belongs here, not in M3.** Discovering the 4.5 MB wall during M3 would invalidate the quote form built in M2. The riskiest unknown gets proven first, and it is a spike — proving the browser can `fetch`-PUT to a signed Supabase URL — not the full feature.
- **The whole schema ships in the first migration.** `quote_events`, `deleted_at`, and `actor_id` are load-bearing from the first query even though their UIs land in M4 and M6. Quotes created before the audit table exists have no creation event, so job history starts mid-story and lead-response-time has no `t=0`.
- **Line items are a correctness requirement, not a scale preference.** The prototype's duplicate-record bug has two causes: the dual write *and* a loop at `index.html:739` that wraps both inserts, so a 3-line quote produced **6 rows**. Consolidating tables fixes the first; only `quotes` 1:N `quote_line_items` fixes the second.
- **Resend DNS verification is wall-clock time** that cannot be compressed. Start it in week one or it blocks M5.

**Ordering note.** M2 still precedes M3, but not for the reason originally given. Uploads do *not* need a quote row to exist — they are scoped to an intake session, which is what makes large files work. The real reasons are that the *claim* step lives inside `submitQuote`, and FR-1's download half is not verifiable until a detail view exists in M4. Worth deciding whether to pull a minimal job-detail page forward into M3 or move that verification criterion to M4.

## 9. Client input required

These block specific milestones. None are developer decisions.

| # | Needed | Blocks | Notes |
|---|---|---|---|
| 1 | **Full rate card** — volume break tiers, 2XL/3XL/4XL upcharges, per-screen setup per color per location, digitizing fee, underbase cost | M2 | FR-5b cannot be built or verified without real numbers. The engine structure can be built in parallel. |
| 2 | **Rush surcharge multiplier** | M2 | Detection already exists in the UI; the multiplier exists nowhere. Currently promised to customers and never charged. |
| 3 | **Does the shop run DTF (direct-to-film)?** | M1 (schema), M2 | Now the default for small runs — no screens, no setup, no minimum, flat per-piece regardless of color count. Absent from the prototype's decoration types. If yes it is a fourth type with a simpler price model, and it is what should absorb below-minimum requests. Record the answer either way so it isn't rediscovered mid-build. |
| 4 | **Production stage model** — keep 3 columns or move to 5? | M1 (schema), M4 | Research argues the current three collapse the two places jobs actually stall — awaiting customer proof approval and awaiting blanks from the supplier — both of which are blocked on an *external* party. The stage enum ships in M1's schema; adding values after the board UI exists means rebuilding the board. |
| 5 | **Payment status tracking — in or out?** | M6 | FR-18. In scope by the letter of the exclusion list (tracking ≠ processing), but confirm it's wanted. |
| 6 | **Artwork retention period** | M3 | Determines the orphan-sweeper age threshold. Build it configurable regardless. |
| 7 | **Old Supabase project — anything worth recovering?** | M1 | Assumed no. |
