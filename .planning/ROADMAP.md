# Roadmap: Hustle Print Hub

## Overview

This is a rebuild, not an extension. The prototype captures the shop's real workflow but fails on all three paths that matter: artwork is never uploaded, the dashboard has no lock on it, and failed writes are reported to customers as successes. The roadmap rebuilds those paths in the order they carry risk.

Phase 1 is deliberately heavy — it ships the complete schema, the auth pattern, the CI guards that make the prototype's bug classes unrepresentable, and a spike proving that a customer's 60 MB artwork file can actually reach storage on a deployed preview. Everything after that is safe to build on. Phase 2 makes a customer's quote land exactly once, honestly. Phase 3 makes their artwork land with it and come back byte-identical — at which point the first half of the core value is true. Phase 4 makes the price correct and untamperable. Phase 5 turns the list of jobs into the board all three of them can trust. Phase 6 stops leads being dropped and gets the owner off the board. Phase 7 makes the shop's day legible — what's late, what's stuck, and how to undo a mistake.

Pricing sits at Phase 4 on purpose: five of its nine requirements need dollar figures only the shop owner can supply, and putting it late maximises his response window. It is built and verified against a documented placeholder rate card so it cannot stall the build regardless.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation & Access Control** - Complete schema, locked dashboard, CI guards, and the artwork upload path proven on a deployed preview
- [ ] **Phase 2: Quote Intake** - A customer's quote reaches the team exactly once, complete, with an honest result either way
- [ ] **Phase 3: Artwork Pipeline** - The customer's actual file lands with the quote and comes back byte-identical
- [ ] **Phase 4: Pricing** - The price is the shop's own, correct across every dimension, and the same number the customer saw
- [ ] **Phase 5: Production Board** - One board all three trust, with confirmed moves and named attribution
- [ ] **Phase 6: Leads & Notifications** - Nobody who asks gets dropped, and the owner stops checking the board
- [ ] **Phase 7: Operations & Recovery** - What's late, what's stuck, and how to undo a mistake

## Phase Details

### Phase 1: Foundation & Access Control
**Goal**: The three of them can sign in to a trustworthy empty shell — nobody else can get in, the database has room for every piece of the story from day one, and the riskiest unknown in the build is already proven
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: FOUND-01, FOUND-02, FOUND-03, FOUND-04, FOUND-05, FOUND-06, AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05
**Success Criteria** (what must be TRUE):
  1. Each of the three team members signs in with their own account and reaches the dashboard; signed out, every dashboard page refuses and returns no customer data at all
  2. Adding `?admin=true` to any address grants nothing, and neither it nor the triple-click shortcut exists anywhere in the code
  3. A 60 MB file uploads from a browser directly into storage on the live preview site — proving the customer's artwork path works before a single form is built on top of it
  4. The shop's sending domain is verified with Resend, so quote emails can go out the moment they are wired up
  5. A build that leaks a rate-card number or a secret key into the browser is rejected and never reaches the site
**Plans**: TBD
**UI hint**: yes

**Notes:**
- This phase carries far more than "scaffold + auth" by design. Five separate prototype-class failures are prevented here, not fixed later: unprotected Server Actions, unthrown database errors, leaked secrets, middleware bypass, and preview environments pointed at production data.
- **FOUND-01 ships the whole schema in the first migration** — `quotes`, `quote_line_items`, `artwork_files`, `quote_events`, plus `deleted_at`, `actor_id`, and the intake-session key — even though the audit UI (Phase 5) and restore UI (Phase 7) land much later. A quote created before `quote_events` exists has no creation event, so its history starts mid-story and time-in-stage has no `t=0`. Every board query needs `where deleted_at is null` from the first query written.
- **AUTH-03 is delivered as an enforced pattern plus a CI check, not a hardening pass.** No Server Action can ship without `auth.protect()` because the build fails if one does. This is what makes the constraint hold in Phases 3, 5, 6 and 7 where the actual dashboard actions get written.
- `stage` ships as `text` + `CHECK` seeded with the **existing 3-stage model**: `pending` (Pending / Pre-Press) → `production` (In Production) → `ready` (Ready / Complete). **Client decision made — input #4 is closed.** Research proposed a 5-stage model; the shop declined it as more structure than the work needs. Keeping `text` + `CHECK` means adding stages later remains a one-line migration if that changes.
- **A lead is not a job.** Leads sit in the outreach list and only enter `pending` when approved. That approval is an explicit transition, not a stage — modelled as the `needs_outreach` flag clearing plus an approval event, so the board keeps exactly three columns.

**Research/spike flag:** FOUND-05 warrants dedicated spike time during planning. The exact `createSignedUploadUrl` → browser `PUT` request shape, progress reporting, and whether a single-shot PUT is reliable at the chosen size cap (versus needing resumable uploads) are the only genuinely unproven mechanics in the build. Everything else in this phase is well-documented pattern work.

**Client input:** #7 (anything worth recovering from the old Supabase project — assumed no) and #3 (does the shop run DTF) should be answered before this phase closes. Neither blocks it: both decoration type and stage are `text` + `CHECK`.

### Phase 2: Quote Intake
**Goal**: A customer's quote reaches the team exactly once, carrying everything the shop needs to price and produce it, and the customer is never lied to about whether it landed
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: QUOTE-01, QUOTE-02, QUOTE-03, QUOTE-04, QUOTE-05, QUOTE-06
**Success Criteria** (what must be TRUE):
  1. A customer configures a multi-garment quote — decoration type, brand, garment colour, ink colours, print locations, stitch count, adult and youth size counts, due date — and submits it in one go
  2. That one submission produces exactly one job with one line per garment; ten submissions produce ten jobs, never twenty and never sixty
  3. When the save fails, the customer sees a clear error explaining what to do, with everything they typed still on the screen — never a success message
  4. Every job carries its own readable job number and no two jobs ever share one
  5. A customer who types `<script>` into their company name appears on the dashboard as exactly that text, and nothing runs

**Plans**: TBD
**UI hint**: yes

**Notes:**
- **QUOTE-03 is the correctness spine of this phase.** The prototype's duplicate-record bug has two independent causes — a dual write across two tables *and* a loop enclosing both inserts, so a 3-line quote produced 6 rows. Consolidating the tables fixes the first; only a real `quotes` → `quote_line_items` relationship fixes the second. Both must be closed here.
- This phase adds **a minimal authenticated list of submitted quotes** as its verification surface — without it, criteria 2, 4 and 5 are only checkable in a database console, which is not something the shop owner can confirm. It is deliberately *not* the board: jobs are not grouped by stage, because the stage model is a pending client decision. The board is Phase 5, and it grows out of this list rather than replacing throwaway work.
- Print locations (N locations each with its own colour count, not two integers) and garment colour land here because they are pricing inputs. Collecting them before Phase 4 consumes them means no re-migration and no re-pricing of existing rows.
- The `quote.created` audit event is written here, against the table that has existed since Phase 1. Only the audit *display* waits for Phase 5.

### Phase 3: Artwork Pipeline
**Goal**: The customer's actual file reaches the team byte-for-byte — or the customer is told plainly that it did not, and no job ever points at a file that isn't there
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: ART-01, ART-02, ART-03, ART-04, ART-05, ART-06, ART-07
**Success Criteria** (what must be TRUE):
  1. A customer attaches a 60 MB PDF to their quote, and a team member opens that job and downloads a file identical to the one that was sent
  2. A designer's `.ai` file and a `.zip` containing art plus fonts are both accepted, while a file that is not really artwork is refused no matter what it's named
  3. A quote covering three different garments carries three separate sets of artwork, with more than one file allowed per garment
  4. When an upload fails, the customer is told and the quote does not submit — no job ever appears on the board pointing at artwork that was never received
  5. An artwork link without a valid signature returns nothing, and the storage bucket cannot be browsed by anyone who guesses at it

**Plans**: TBD
**UI hint**: yes

**Notes:**
- This is the full build of the architecture spiked in Phase 1. Bytes go browser → storage directly; they never pass through a server function, because Vercel caps function request bodies at 4.5 MB and that ceiling cannot be raised on any plan.
- **Criterion 1 is the core value's other half, and it is why the download surface lands here rather than in Phase 5.** Research flagged this as an open roadmap decision: the artwork phase cannot honestly close without proving a file comes back, and the prototype's defining failure was recording a filename with no file behind it. Resolution adopted: the download indirection route (short-lived signed URL minted at click time, authorisation rechecked at that moment) is permanent infrastructure built here and attached to Phase 2's job list. Phase 5's full job detail view consumes it rather than rebuilding it.
- Downloads are forced as attachments, not rendered inline. SVG is an executable document format and the accepted-format list opens that door directly.
- The server independently confirms the stored object exists and records its real byte size before any job may claim it has artwork. The client's word that an upload succeeded is never sufficient.

**Research/spike flag:** worth re-verifying the upload path under real production artwork — large layered `.ai` files and `.eps` with embedded rasters — once the size cap and format list are final. The Phase 1 spike proves the mechanism; this confirms it against what customers actually send.

**Client input:** #6 (artwork retention period) informs the orphan-sweeper's age threshold. Non-blocking — build it configurable and set the value when the answer arrives.

### Phase 4: Pricing
**Goal**: The price a customer is quoted is the shop's own price, correct on every dimension that moves money, impossible to tamper with, and exactly the number stored against the job
**Mode:** mvp
**Depends on**: Phase 2 (the intake form and submission path). Independent of Phase 3 — if artwork work stalls, this can proceed
**Requirements**: PRICE-01, PRICE-02, PRICE-03, PRICE-04, PRICE-05, PRICE-06, PRICE-07, PRICE-08, PRICE-09, PRICE-10
**Success Criteria** (what must be TRUE):
  1. A 288-piece run quotes at a visibly lower per-piece rate than a 12-piece run of the same garment
  2. A run containing 2XL and 3XL shirts prices higher than the identical run in standard sizes, and a job inside the rush window prices higher than the same job outside it
  3. A three-colour front plus one-colour sleeve charges four screen setups, and an embroidery job charges digitizing once as its own separate fee — not one flat charge standing in for both
  4. The estimate on the customer's screen updates as they change the job and matches, to the cent, the total the team sees on that job afterwards
  5. No rate-card number appears anywhere in the browser, and a request with a price injected into it is refused rather than quietly accepted

**Plans**: TBD

**Notes:**
- This is the highest-value fix in the build. The prototype prices linearly — a 12-piece order and a 500-piece order quote identically — never charges the 2XL/3XL counts it already collects, uses one flat $30 fee in place of three genuinely different charges, and displays "Rush Timeline Surcharge Applied" while charging nobody. That last one is a live revenue leak, not a defect.
- **Client input #1 (full rate card) and #2 (rush multiplier) are needed for this phase to be *signed off*, but do not block it from being *built or completed*.** The engine, every tier and upcharge dimension, the estimate endpoint and the fixture tests are built and verified against a documented placeholder rate card. Swapping in the shop's real numbers is a data-only change with no code change behind it, so it can happen at any point — including after later phases have shipped. Criteria 1, 2 and 3 are all demonstrable with placeholder rates.
- The live estimate is a Route Handler, not a Server Action. Server Actions dispatch one at a time per client and cannot be cancelled, so per-keystroke estimates on that transport would queue up and strand the customer's actual submit behind stale requests.
- The estimate and the submission call the same pricing function. That is what makes criterion 4 true by construction rather than by remembering to keep two copies in step.

### Phase 5: Production Board
**Goal**: All three of them look at one board and trust what it says — where every job is, who moved it, what it actually specifies, and that a misclick cannot quietly corrupt it
**Mode:** mvp
**Depends on**: Phase 3 (artwork download) and Phase 4 (totals to display)
**Requirements**: BOARD-01, BOARD-02, BOARD-03, BOARD-04, BOARD-05, BOARD-06, BOARD-07
**Success Criteria** (what must be TRUE):
  1. Every job sits on the board under its production stage, and a team member can see what's in each stage at a glance
  2. Moving a job asks "move job #1042 to In Production?" before anything happens, jobs move backward as readily as forward, and cancelling the question changes nothing
  3. Clicking a job opens its full detail — sizes, colours, notes, the customer's phone and email, and the artwork to download
  4. The job's history shows who moved it, who edited it, and when, by individual name
  5. A team member takes a walk-in order straight into the dashboard with a negotiated flat price, and later corrects a customer's size counts — the total re-prices itself and the change is recorded with what it was before

**Plans**: TBD
**UI hint**: yes

**Notes:**
- **Client input #4 is resolved — this phase is no longer blocked.** The board keeps the shop's existing three columns: Pending / Pre-Press, In Production, Ready / Complete. The 5-stage proposal was declined as more structure than the work needs.
- **Completed jobs sort newest-first**, so the most recently finished job is always at the top of the Ready / Complete column.
- **Lead approval is a gate into the board, not a column on it.** Leads live in the outreach list until approved; only then do they appear in Pending. This is what keeps the Pending column meaningful — it holds real work, not unqualified enquiries.
- Manual job entry (criterion 5) restores capability the prototype already has. Without it the team types customer data into the *public* form as a workaround, and the audit trail then misattributes every walk-in as customer-submitted.
- Editing an existing job re-runs pricing server-side. Customers revise size counts constantly; without this the board diverges from reality within days, which defeats the core value.
- No drag-and-drop. Dragging *is* the misclick that was the stated pain point, and it directly contradicts the confirmation-modal decision.
- The audit trail displayed here reads events that have been written since Phase 2. Nothing needs backfilling.

### Phase 6: Leads & Notifications
**Goal**: Nobody who asks for a quote gets dropped, and the owner learns about new work without watching the board
**Mode:** mvp
**Depends on**: Phase 2 (leads are a view over quotes) and Phase 1 (Resend domain verified there). Can run before Phase 5 if the stage-model decision is delayed
**Requirements**: LEAD-01, LEAD-02, LEAD-03, LEAD-04, NOTIF-01, NOTIF-02
**Success Criteria** (what must be TRUE):
  1. Every lead needing a callback is listed with name, phone and email, and ten submissions produce ten entries — not twenty
  2. A team member marks a lead as called, and the list shows who called and when
  3. A team member adds a walk-in or phone enquiry to the callback list by hand
  4. A new quote puts an email in the owner's inbox with the customer's name, a summary of the job, and a link that opens it on the dashboard
  5. With email delivery broken, the customer's quote still submits successfully and the delivery failure is recorded for the team to see

**Plans**: TBD
**UI hint**: yes

**Notes:**
- **"Needs a callback" is a flag on a quote, not a separate table.** The prototype's duplicate-outreach bug came from splitting `app_orders` and `app_manual_leads` and writing to both. The temptation to give leads their own table "because leads have different fields" is exactly how criterion 1 breaks. Manual leads (criterion 3) are quotes flagged for outreach, entered by a team member.
- Email is sent after the database commit is confirmed, on its own path, where it cannot affect what the customer sees. Criterion 5 is a deliberate test of that boundary, not an edge case.
- The owner is the only recipient by decision, and it is flagged in PROJECT.md as worth revisiting — it makes him a single point of failure on lead-response time, which is a metric the shop wants to track.

### Phase 7: Operations & Recovery
**Goal**: The shop can see what's late, what's stuck, and where the money stands — and a mistake made on the board can be undone
**Mode:** mvp
**Depends on**: Phase 5
**Requirements**: OPS-01, OPS-02, OPS-03, OPS-04
**Success Criteria** (what must be TRUE):
  1. Opening the calendar on any day shows this week's jobs against their due dates, and a job due yesterday reads clearly as overdue
  2. A deleted job disappears from the board, the calendar and the callback list, and a team member restores it intact
  3. Every job shows how long it has been sitting in its current stage
  4. A team member marks a job invoiced and paid, and the other two see it with the change attributed by name

**Plans**: TBD
**UI hint**: yes

**Notes:**
- The prototype's calendar window is hardcoded to a fixed date, so it silently stops being about today. Criterion 1 is written to catch exactly that regression.
- The `deleted_at` column has existed since Phase 1 and every query written since has excluded soft-deleted rows. Only the restore interface is new here — this phase must not be the moment queries get audited for it.
- Time-in-stage is derived from the audit events written since Phase 2, which is why criterion 3 works for jobs created long before this phase.
- **Client input #5 (payment status tracking wanted, yes or no) affects criterion 4 only.** In scope by the letter of the exclusion list — PROJECT.md excludes payment *processing*, not payment *tracking* — but it needs an explicit answer rather than a default. If the answer is no, OPS-03 and criterion 4 drop and the phase is otherwise unaffected.
- Artwork retention (client input #6) is applied to the sweeper threshold here if it was answered after Phase 3.

## Deviations from PRD §8

PRD §8 proposes six milestones and describes itself as a proposal. Two changes, both justified below; the ordering skeleton is otherwise kept because the research dependency analysis independently confirms it.

**1. PRD Milestone 2 ("Secure quote intake", carrying FR-4 through FR-8b) is split into Phase 2 (Quote Intake) and Phase 4 (Pricing).**

As drafted it carries 15 of 47 requirements — roughly a third of the build in one milestone — and welds the only client-blocked work in the project onto the critical path of the customer form. Splitting them:
- Isolates the rate-card dependency so a slow client answer cannot hold the intake form hostage
- Lets QUOTE-03 (one submission, one row, N line items — the correctness requirement the whole rebuild turns on) be proven on its own rather than buried among nine pricing requirements
- Gives each half a distinct verification surface: intake is confirmed by data shape and honest failure, pricing by fixture equality and bundle inspection

The research warning this could violate — that print locations and the dark-garment flag must not be retrofitted after pricing — is satisfied, because both are collected by the Phase 2 form and present in the Phase 1 schema, well before Phase 4 reads them.

**2. Artwork (PRD M3) moves ahead of pricing.**

PROJECT.md states the core value as "a customer's quote — *including their artwork* — reliably reaches the team," and adds "everything else is secondary." Artwork intake is also the prototype's single worst failure and the build's largest technical unknown. Putting it at Phase 3 means the core value's first half is true four phases from the end rather than three, and it pushes the client-blocked pricing work later, which maximises the owner's window to produce his rate card. Pricing depends only on Phase 2, so the two are independent and can be reordered again if the rate card arrives early.

**3. Two open research questions resolved here rather than deferred.**

- *Where artwork retrieval is verified* (research Contradiction #3): the download route is permanent infrastructure built in Phase 3 and attached to Phase 2's job list, so the artwork phase can close honestly. Phase 5's detail view consumes it. Neither of the two options research offered is throwaway work under this arrangement.
- *When the upload path is proven* (research Contradiction #1): treated as one continuous item — a narrow spike against a deployed preview in Phase 1 (FOUND-05), then the full implementation in Phase 3. Not competing recommendations.

## Client Input Tracker

Seven items await the shop owner. None are developer decisions. Their placement is designed so a delay on any of them stalls at most one phase's sign-off, never the build.

| # | Needed | Blocks | Hard block? |
|---|--------|--------|-------------|
| 1 | Full rate card — tiers, size upcharges, per-screen setup, digitizing | PRICE-04…07 sign-off | No — Phase 4 builds and verifies on placeholder rates; real numbers are a data-only change |
| 2 | Rush surcharge multiplier | PRICE-08 sign-off | No — same as above |
| 3 | Does the shop run DTF? | Decoration-type values | No — `text` + `CHECK`; wanted before Phase 1 closes |
| 4 | Production stage model — four stages or five? | Phase 5 board UI | **Yes, for Phase 5 only.** Needed before Phase 5 planning; does not touch Phases 1–4 |
| 5 | Payment status tracking — build or drop? | OPS-03 | No — drops cleanly from Phase 7 if unwanted |
| 6 | Artwork retention period | Sweeper threshold value | No — built configurable in Phase 3 |
| 7 | Anything worth recovering from the old Supabase project? | Whether any import work exists at all | No — assumed none; confirm before Phase 1 closes |

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6 → 7

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation & Access Control | 0/TBD | Not started | - |
| 2. Quote Intake | 0/TBD | Not started | - |
| 3. Artwork Pipeline | 0/TBD | Not started | - |
| 4. Pricing | 0/TBD | Not started | - |
| 5. Production Board | 0/TBD | Not started | - |
| 6. Leads & Notifications | 0/TBD | Not started | - |
| 7. Operations & Recovery | 0/TBD | Not started | - |

---
*Roadmap created: 2026-07-30*
*Coverage: 47/47 v1 requirements mapped*
