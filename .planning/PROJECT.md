# Hustle Print Hub

## What This Is

A quoting and production-tracking tool for a three-person screen-print and embroidery shop. Customers use a public link to configure a job — garment, decoration, colors, sizes, due date — upload their artwork, and get an estimated price. The owner, his business partner, and his worker use a shared internal board to see what's in the queue, move jobs through production stages, and track leads who need a callback.

This is a rebuild. A working prototype exists as a single 1,184-line `index.html`; it is being replaced rather than extended.

## Core Value

**A customer's quote — including their artwork — reliably reaches the team and appears on a board all three of them can trust.**

Everything else is secondary. The current tool fails this: artwork is never actually uploaded, and failed submissions are reported to customers as successes.

## Requirements

### Validated

(None yet — ship to validate)

The existing prototype is not treated as validated. Its three core paths — artwork intake, access control, and submission reliability — are each broken in ways that make the data untrustworthy. See `ASSESSMENT.md` §2.

### Active

- [ ] Customer can submit a quote with garment, decoration, colors, adult/youth sizes, and due date
- [ ] Customer can upload artwork that the team can actually retrieve
- [ ] Customer sees an accurate estimated price that cannot be tampered with
- [ ] Customer gets honest confirmation — or a clear error, never a false success
- [ ] Each team member logs in with their own account
- [ ] Team sees all jobs grouped by production stage
- [ ] Team moves jobs forward and backward between stages, behind a confirmation
- [ ] Team sees who changed what and when
- [ ] Team opens a job for full detail and downloads the artwork
- [ ] Team sees leads needing a callback, with no duplicates
- [ ] Owner is emailed when a new quote arrives
- [ ] Team sees what is due this week and what is overdue
- [ ] Deletes are reversible

### Out of Scope

- **General role hierarchy / permission tiers** — declined beyond the single owner/staff split described in Key Decisions. There are exactly two roles and they differ on two things only: visibility of money, and authority to approve a lead into production. Everything else stays flat. No per-feature permission matrix.
- **Payment processing** — the tool quotes and tracks; it does not collect money.
- **SanMar / Cotton Collective drop-ship platform** — a separate project, separately priced and scoped. Not part of this build.
- **Customer accounts / customer login** — customers interact through a public link only.
- **Inventory management** — not a shop need at this size.
- **Marketing, ad creative, SEO content** — out of engagement scope.
- **Native mobile apps** — responsive web is sufficient.
- **Quote versioning, customer status links, live board sync** — deferred to P2, valuable but not required to ship.

## Context

**Prior work.** The existing prototype is a single `index.html` with inline Tailwind from CDN and a Supabase client wired directly into the page. It demonstrably captures the shop's real workflow — stage names, size breakdowns, rush-timeline logic, and lead tracking all reflect how they actually operate. That domain knowledge is worth carrying forward even though the code is not.

**Why the rebuild.** A full current-state assessment is in `ASSESSMENT.md`. The blocking findings:

- Artwork uploads keep only the filename; no file is ever transmitted (§2.1)
- The dashboard has no authentication — `?admin=true` grants full access to all customer PII (§2.2)
- The configured Supabase project no longer resolves in DNS (§2.3)
- Failed database writes render the customer success screen anyway (§2.4)
- Every submission duplicates itself across two tables (§2.5)
- The stored price is a hardcoded placeholder, not the quoted price (§2.6)
- Unescaped user input reaches `innerHTML`, giving stored XSS (§2.7)

**Users.** Owner, business partner, one worker. Customers are local businesses, teams, and organizations ordering decorated apparel.

**Data sensitivity.** Every record holds a customer's name, phone, and email — personal information under Texas Bus. & Com. Code § 521, currently exposed to anyone with the URL.

## Constraints

- **Tech stack**: Next.js App Router on Vercel — gives a real server boundary for pricing without operating a separate backend, and server-renders for future SEO.
- **Auth**: Clerk, individual accounts, **two roles** — `owner` and `staff`, differing only on money visibility and lead approval. Fast to integrate, and keeping the *database* off the browser removes the usual RLS/JWT coupling problem. (Artwork upload is the one browser-to-Supabase path; see Security below.)
- **Data**: Supabase Postgres + Storage on a **new** project — the old one is gone and is being abandoned.
- **Security**: The browser never holds a **durable** Supabase credential. All database access goes through Server Actions using the service role key. Artwork upload is the sole browser-to-Supabase path, authorized by a short-lived signed token scoped to a single server-chosen path — never an API key. No client-supplied price is ever trusted.
- **Team size**: Three non-technical users — the tool must be obvious without training.
- **Commercial**: Barter engagement (development for company apparel). Scope boundary is explicit in `ASSESSMENT.md` §8 precisely because there is no invoice to anchor it.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Next.js over Vite + React | Server Actions provide the server-side pricing boundary with no separate backend to deploy; Vite would mean a static SPA plus bolted-on functions | — Pending |
| Clerk over Supabase Auth | The browser never queries the *database*, so RLS/`auth.uid()` coupling is irrelevant; Clerk's drop-in UI is faster and free at this scale (50k MRU). **Caveat surfaced 2026-07-30:** Supabase's documented TUS upload path authenticates with a Supabase Auth session, which this design does not have. The signed-token alternative must be proven in the Phase 1 spike — it is the one place the Clerk choice carries real risk | ⚠️ Revisit after spike |
| All DB access via Server Actions with service role key | Makes price injection structurally impossible — no database key and no price field ever reaches the client | — Pending |
| New Supabase project; abandon the old one | The configured project no longer resolves in DNS; assume prior data unrecoverable | — Pending |
| Single `quotes` table replacing the `app_orders`/`app_manual_leads` split | The dual write is the actual source of duplicate outreach entries | — Pending |
| **Owner-only pricing visibility.** Money fields — totals, setup fees, payment status — are visible to the owner (Josh) alone, not to the partner or worker | Client decision, made with the tradeoff stated explicitly. **Reverses an earlier "all users see financials" choice.** Must be enforced by omitting fields from the server payload, never by hiding them in the UI — conditional rendering leaves values readable in the network tab, which is `?admin=true` in a new costume | — Pending |
| **Owner-only lead approval.** A lead enters the Pending column only when the owner approves it | Keeps Pending meaningful — it holds qualified work, not raw enquiries | — Pending |
| Stage movement stays flat — any authenticated user moves any job | Making the owner a bottleneck on routine production flow would defeat the point of a shared board. Confirmation modal plus audit attribution is the safeguard | — Pending |
| **Three production stages retained** — Pending / Pre-Press → In Production → Ready / Complete | Client declined a researched 5-stage model as more structure than the work needs. `stage` stays `text` + `CHECK`, so expanding later is a one-line migration | — Pending |
| Individual logins, not a shared account | With no permission gate, per-user attribution in the audit trail *is* the accountability mechanism | — Pending |
| Confirmation modal on every stage change | Three people touching one board; misclicks were a stated pain point | — Pending |
| New-quote email to owner only | Owner's explicit preference. Watch this — it makes him a single point of failure on lead response time, which is a tracked metric | ⚠️ Revisit |
| RLS default-deny as backstop, not primary boundary | Defense in depth; the Server Action layer is the real gate | — Pending |

## Open Questions

- **Artwork retention** — how long are customer files kept? Affects storage cost and belongs in a privacy policy. Not blocking; decide before launch.
- **Old Supabase data** — assumed unrecoverable and not worth migrating. Confirm with the client.

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-07-30 after initialization*
