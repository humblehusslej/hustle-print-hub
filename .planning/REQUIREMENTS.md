# Requirements: Hustle Print Hub

**Defined:** 2026-07-30
**Core Value:** A customer's quote — including their artwork — reliably reaches the team and appears on a board all three of them can trust.

**Source documents:** `PRD-hustle-print-hub.md` (user stories, full acceptance criteria), `ASSESSMENT.md` (current-state findings), `.planning/research/SUMMARY.md` (research synthesis).

---

## v1 Requirements

### Foundation

- [ ] **FOUND-01**: Schema ships complete in the first migration — `quotes`, `quote_line_items`, `artwork_files`, `quote_events`, including `deleted_at` and `actor_id` columns whose interfaces land later
- [ ] **FOUND-02**: All database access routes through a data-access layer with an error-unwrapping helper that throws — `supabase-js` returns `{data, error}` and does not throw, which is how the prototype's silent-failure bug would recur
- [ ] **FOUND-03**: Production build is grepped for rate-card values; any hit fails the build
- [ ] **FOUND-04**: Vercel deployment protection enabled and environment variables scoped per environment
- [ ] **FOUND-05**: Direct browser-to-Storage upload proven against a **deployed preview**, not `next dev` — neither the Vercel 4.5 MB nor the Next.js 1 MB body cap is enforced locally
- [ ] **FOUND-06**: Resend sending domain verified — started in this phase because DNS propagation is wall-clock time

### Authentication

- [ ] **AUTH-01**: Each team member signs in with their own individual account
- [ ] **AUTH-02**: Every dashboard route requires an authenticated session and returns no data when signed out
- [ ] **AUTH-03**: Every dashboard Server Action calls `auth.protect()` as its first statement — middleware cannot protect actions, which are dispatched by ID rather than path
- [ ] **AUTH-04**: The `?admin=true` parameter and triple-click handler are absent from the codebase

### Quote intake

- [ ] **QUOTE-01**: Customer submits a multi-item quote specifying decoration type, garment brand, garment color, ink color count, print locations, stitch count, adult and youth size breakdowns, and due date
- [ ] **QUOTE-02**: Customer sees a success confirmation only after a confirmed database commit, and an actionable error with input preserved on failure
- [ ] **QUOTE-03**: One submission produces exactly one `quotes` row with N related line items — never one row per line item
- [ ] **QUOTE-04**: Every server entry point validates input with Zod, rejecting negative quantities, out-of-range values, unknown enums, and past due dates
- [ ] **QUOTE-05**: Each quote receives a unique sequence-backed human-readable job number — the prototype's random 4-digit id collides roughly 50% of the time by the 112th quote
- [ ] **QUOTE-06**: User-supplied text renders as literal text everywhere in the dashboard and never executes

### Pricing

- [ ] **PRICE-01**: The rate card lives in a `server-only` module and is absent from the client bundle
- [ ] **PRICE-02**: The submission payload contains no price fields; an injected price is ignored or rejected
- [ ] **PRICE-03**: The live estimate is served by a debounced, abortable Route Handler — not a Server Action, which Next.js dispatches serially and cannot abort
- [ ] **PRICE-04**: Pricing applies quantity break tiers, so a large run quotes at a materially lower per-piece rate than a small one
- [ ] **PRICE-05**: Pricing applies 2XL/3XL/4XL size upcharges — these counts are already collected by the prototype and never priced
- [ ] **PRICE-06**: Screen setup is charged per color per location rather than as one flat fee
- [ ] **PRICE-07**: Embroidery digitizing is charged as a distinct one-time fee, separate from screen setup
- [ ] **PRICE-08**: Rush orders are surcharged — currently promised in the UI copy and never applied by the math
- [ ] **PRICE-09**: The total stored in the database equals the total quoted to the customer, in integer cents

### Artwork

- [ ] **ART-01**: Artwork uploads directly from the browser to Supabase Storage via a server-issued signed upload URL, so bytes never pass through a Vercel function
- [ ] **ART-02**: File type is validated server-side by extension and magic bytes, accepting PDF, AI, EPS, SVG, PNG, JPG, and ZIP
- [ ] **ART-03**: Artwork attaches per line item and supports multiple files per line
- [ ] **ART-04**: No quote row can reference artwork whose bytes never landed — enforced by verify-then-commit plus a database constraint, with a standing invariant query returning zero rows
- [ ] **ART-05**: The storage bucket is not publicly listable; downloads require a valid signed URL
- [ ] **ART-06**: Upload failure blocks submission and tells the customer, never reporting false success
- [ ] **ART-07**: A team member downloads a byte-identical copy of the customer's file from the job

### Production board

- [ ] **BOARD-01**: All jobs are visible grouped by production stage
- [ ] **BOARD-02**: Jobs move forward and backward between stages, each move gated by a confirmation naming the job and destination
- [ ] **BOARD-03**: Every stage change, edit, and delete records actor, action, and timestamp against an individual identity, visible on the job
- [ ] **BOARD-04**: Board cards open a detail view showing full job specification, customer contact, and artwork download
- [ ] **BOARD-05**: A team member creates a job directly from the dashboard, including a flat-price override, distinguishable in the audit trail from customer-submitted jobs
- [ ] **BOARD-06**: A team member edits quantities, sizes, notes, and due date on an existing job; edits re-run pricing server-side and are recorded with before/after values

### Leads and outreach

- [ ] **LEAD-01**: Leads needing a callback are listed with contact details, each submission appearing exactly once
- [ ] **LEAD-02**: A team member marks a lead contacted, with attribution and timestamp
- [ ] **LEAD-03**: A team member adds a lead manually for walk-in and phone enquiries

### Notifications

- [ ] **NOTIF-01**: A new quote emails the owner with customer name, job summary, and a dashboard link
- [ ] **NOTIF-02**: Email delivery failure is logged and never blocks or fails the customer's submission

### Operations

- [ ] **OPS-01**: The production calendar is driven by the current date and marks overdue jobs — the prototype's window is hardcoded to a fixed date
- [ ] **OPS-02**: Deleted jobs are recoverable; every board query excludes soft-deleted rows
- [ ] **OPS-03**: A team member marks a job invoiced and paid, with attribution *(pending client confirmation — see Client Input below)*
- [ ] **OPS-04**: Time-in-stage is visible per job, derived from the audit trail

---

## v2 Requirements

Deferred. Several are cheap and high-value — noted so they are not forgotten.

### Board and workflow

- **V2-01**: "Who are we waiting on" as a first-class board read — distinguishing jobs blocked on the customer from those blocked on the supplier. *Low complexity; depends on the stage-model decision.*
- **V2-02**: Printable work order / pull sheet — the physical ticket that walks with the job to the press. *Likely the most-used artifact in a real shop.*
- **V2-03**: Stall highlighting — surfacing "3 days awaiting approval" directly on the card
- **V2-04**: Per-decoration-method production checklists

### Customer-facing

- **V2-05**: Customer confirmation email — a one-shot receipt proving the submission landed. *Low complexity; Resend is already in the stack. Distinct from the deferred customer status link.*
- **V2-06**: Tokenized read-only customer status link
- **V2-07**: Minimum-order soft warning routing below-minimum requests to the callback list rather than rejecting them

### Shop intelligence

- **V2-08**: Logo-on-file registry — which customer logos are already digitized or screened. *Kills repeat digitizing charges and speeds reorders.*
- **V2-09**: Artwork pre-flight warning at upload, setting art-fee expectations before the customer anchors on a price
- **V2-10**: Daily due/overdue email digest
- **V2-11**: Quote versioning preserving the customer's original against the finalized quote
- **V2-12**: Live board sync across simultaneous viewers

---

## Out of Scope

| Feature | Reason |
|---------|--------|
| Drag-and-drop kanban | Directly contradicts the confirmation-modal decision — dragging *is* the misclick that was the stated pain point |
| Role separation / permission tiers | Three trusted people who all see everything; attribution replaces restriction |
| Payment processing | The tool quotes and tracks; it does not collect money. Hand-marked status is in scope; taking payment is not |
| SanMar / Cotton Collective drop-ship integration | A separate project, separately priced. "It's just an API call" is how the scope boundary dissolves |
| Customer accounts or login | Customers interact through a public link only |
| Inventory management | Not a need at this size |
| Barcode job travelers, capacity planning, time tracking, BI dashboards | Built for shops an order of magnitude larger |
| Online design studio | Enormous surface area, unrelated to the core problem |
| Names-and-numbers roster management | Team-sports feature; not this shop's work |
| Native mobile apps | Responsive web is sufficient |
| Marketing, ad creative, SEO content | Outside engagement scope |

---

## Client Input Required

Not developer decisions. Each blocks specific work.

| # | Needed | Blocks | Note |
|---|--------|--------|------|
| 1 | Full rate card — volume tiers, size upcharges, per-screen setup, digitizing fee, underbase | PRICE-04 … PRICE-07 | Engine structure can be built in parallel with the numbers |
| 2 | Rush surcharge multiplier | PRICE-08 | Promised to customers today, charged to nobody |
| 3 | Does the shop run DTF (direct-to-film)? | QUOTE-01, pricing model | Now the default for small runs; absent from the prototype's decoration types |
| 4 | Production stage model — four stages or five? | BOARD-01 | **Does not block the schema.** Stage is `text` + `CHECK`, alterable in a one-line migration. It blocks the board UI, so the decision is needed before the board phase, not the foundation phase |
| 5 | Payment status tracking wanted? | OPS-03 | In scope by the letter of the exclusion list — tracking is not processing |
| 6 | Artwork retention period | ART-04 sweeper | Build configurable regardless |
| 7 | Anything worth recovering from the old Supabase project? | FOUND-01 | Assumed no |

---

## Traceability

Populated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| FOUND-01 … FOUND-06 | TBD | Pending |
| AUTH-01 … AUTH-04 | TBD | Pending |
| QUOTE-01 … QUOTE-06 | TBD | Pending |
| PRICE-01 … PRICE-09 | TBD | Pending |
| ART-01 … ART-07 | TBD | Pending |
| BOARD-01 … BOARD-06 | TBD | Pending |
| LEAD-01 … LEAD-03 | TBD | Pending |
| NOTIF-01 … NOTIF-02 | TBD | Pending |
| OPS-01 … OPS-04 | TBD | Pending |

**Coverage:**
- v1 requirements: 43 total
- Mapped to phases: 0 (roadmap pending)
- Unmapped: 43 ⚠️

---
*Requirements defined: 2026-07-30*
*Last updated: 2026-07-30 after initial definition*
