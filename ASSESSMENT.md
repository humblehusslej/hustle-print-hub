# Hustle Print Hub — Technical Assessment & MVP Requirements

**Prepared:** July 2026
**Repo:** `hustle-print-hub` (single-file `index.html`, 1,184 lines)
**Engagement:** Rebuild to MVP. Barter — company apparel (hats, tees, polos).

---

## 1. What this tool is for

A shared production board for a 3-person shop: **owner, business partner, one worker.**

Three jobs it has to do well:

1. **Quote intake.** Customer opens a link, configures a job (garment, decoration, colors, sizes, due date), uploads artwork, submits.
2. **Production visibility.** Everyone on the team sees what's in the queue, what stage each job is at, what's moving, and what's stalled.
3. **Lead callback.** Interested customers who aren't yet jobs surface in a "Needs Call" list so nobody gets dropped.

Everything below serves those three. Anything that doesn't is out of scope.

> Not in scope for this engagement: the SanMar/Cotton Collective drop-ship platform. That is a separate project, separately priced.

---

## 2. Current state — verified findings

Each finding was confirmed by reading source or probing the live backend, not inferred from discussion.

### 2.1 Artwork uploads never happen — Critical

`index.html:471`

```js
function handleManifestLineArtUpload(id, event) {
  const file = event.target.files[0];
  row.artFileName = "✓ " + file.name;   // only the filename string is kept
  showToast('Artwork linked successfully');
}
```

The `File` object is read and discarded. Only the filename is retained, and it is concatenated into a free-text notes field (`:764`). **No file is transmitted anywhere** — not to a database, not to storage, not to email.

This is not a display problem on the dashboard side. There is nothing to display. Every customer who uploaded art saw a green "✓ linked successfully" confirmation for a file that was never sent.

**Consequence:** every quote submitted to date is missing its artwork, and the customer believes it was received.

### 2.2 No authentication — Critical

Admin access is granted by either:

- `?admin=true` in the URL (`:361`), or
- triple-clicking a page element (`:848`)

Both are client-side booleans. There is **no IP allowlist** and no server-side check of any kind. Any person who appends `?admin=true` to the URL sees the full production board: every customer's name, phone number, email address, order history, and payment status.

This is worth stating plainly because the operating assumption has been that the owner's "saved device" is what grants access. It is not. Nothing is checking who the visitor is.

### 2.3 The configured Supabase project no longer exists — Critical

The committed source points at a Supabase project (`rtgjovwxbunfbytheyjp.supabase.co`, publishable key inline at `:348`).

That hostname **does not resolve in DNS.** Verified from a machine that resolves `supabase.com` normally on the same lookup — so this is not a local network issue. The project is deleted or the reference is stale.

Per client direction, this project is being **abandoned and replaced with a new database.** Noted here because it explains §2.4 and because any existing data in it should be assumed unrecoverable.

### 2.4 Failed writes are reported to the customer as success — Critical

`syncCloudOrders` and `syncCloudLeads` (`:385-396`) perform Supabase inserts with **no error handling at all** — no return value checked, no `try`/`catch`.

`submitOrder` awaits them and then unconditionally renders the success screen (`:809`):

> *"Your combined multi-item package specifications and contact credentials have been logged successfully into our production queue."*

A failed insert is indistinguishable from a successful one. Combined with §2.3, this is active silent data loss: a customer completes a quote, is told it was received, and no record exists.

On the read side, `fetchCloudDatabaseState` catches errors and only `console.error`s them (`:381`), so the dashboard renders an empty board on failure. From the owner's chair, a broken backend and a quiet week look identical.

### 2.5 Duplicate records are structural, not user error — High

Every submission inserts into **both** `app_orders` and `app_manual_leads` (`:798-799`), once per line item:

```js
await syncCloudOrders(systemicCard, 'insert');
await syncCloudLeads(manualLeadCard, 'insert');
```

The duplication observed in the "Needs Call / Outreach" column is produced by the write path itself. It is not caused by customers clicking "Add New" twice.

Two tables hold overlapping copies of the same information with no foreign key relating them. They will drift.

**There is a second multiplier.** The `for (const item of customerManifestItems)` loop at `:739` encloses *both* inserts. A three-line quote therefore produced **six rows** — three orders and three leads — not two. Consolidating to a single table fixes the dual write but not the loop. Only a relational `quotes` 1:N `quote_line_items` structure fixes both, which makes the line-items table a correctness requirement rather than a scale preference.

### 2.6 The stored price is not the quoted price — High

The customer is shown a real calculated total (`completeBatchCost`, `:724`).

The database stores a flat placeholder (`:774`):

```js
estimated_total: totalCount * 11.50,
```

Every dashboard financial figure derives from that placeholder — including the "Balance Out" amount shown in payment tracking (`:1143`). Revenue totals, outstanding balances, and per-job value are all incorrect, and they do not match what the customer was quoted.

### 2.7 Stored cross-site scripting — High

The file contains **zero** output-escaping helpers (no `escapeHtml`, no `textContent` assignment, no sanitizer). User-supplied values are interpolated directly into `innerHTML` throughout, e.g. `:1143` and the calendar renderer at `:1069`.

A customer submitting a name containing a script tag gets that script executed in the owner's browser session when the dashboard is opened.

### 2.8 Pipeline is one-way — Medium

`advanceJob` (`:1053`) only moves forward:

```js
if (cur !== -1 && cur < stages.length - 1) { ... }
```

There is no way to move a job back a stage. A misclick is permanent. This was raised directly in the review meeting.

### 2.9 Production calendar is hardcoded to a fixed date — Medium

`:1065`

```js
const baseToday = new Date('2026-06-15T00:00:00');
```

The calendar renders a fixed 28-day window starting 15 June 2026 regardless of the current date. As of this writing that window closed roughly six weeks ago, so jobs due now do not appear on it.

### 2.10 Additional

- **Artwork file types are wrong for the trade.** The upload input accepts `image/*` only (`:606`), which excludes PDF, AI, EPS, and SVG — the formats print customers actually send.
- **Hard delete behind a single confirm.** `cancelOrder` (`:1048`) permanently removes the record. No recovery.
- **No input validation.** No schema, no bounds checking. Negative and absurd quantities are accepted.
- **Tailwind loaded from CDN** (`:8`). Not intended for production use.
- **No notifications of any kind.** New quotes arrive silently; the board must be checked manually.

---

## 3. Architecture

### 3.1 Recommendation

| Layer | Choice |
|---|---|
| Framework | **Next.js (App Router)** on Vercel |
| Auth | **Clerk** |
| Database + file storage | **Supabase** — new project |
| Validation | **Zod**, enforced server-side |
| Email | **Resend** |

### 3.2 Next.js over Vite + React

Next.js is the better call here, and the deciding reason is not SEO — it is the pricing requirement.

**It gives a real server boundary without a separate backend.** A Server Action runs on Vercel's server. The pricing logic and the price list live in a module that is never bundled to the browser. That is precisely the guarantee needed in §3.4, and it comes with no additional service to deploy, host, or maintain.

With Vite you would build a static SPA and then bolt separate serverless functions beside it — two build targets and two mental models to get the same result Next.js gives natively.

Secondary reasons:

- **SEO.** A Vite SPA ships an empty HTML shell; crawlers see nothing without extra work. Next.js server-renders. If the quote page should ever rank for local searches ("custom t-shirts <city>"), that capability is already there.
- **Vercel is Next.js's first-party platform.** Zero deployment configuration.

### 3.3 Clerk over Supabase Auth — endorsed, with one condition

Clerk is the right call for this build.

The usual objection to mixing Clerk with Supabase is that Supabase Row Level Security keys off `auth.uid()` from a Supabase-issued JWT, so using a different auth provider means bridging tokens. That objection **does not apply to database access here**, because the browser never queries the database (§3.4). Supabase never needs to know who the user is; Next.js does, and that is Clerk's job.

> **Qualified 2026-07-30.** One place this does bite: Supabase's documented resumable-upload path authenticates with a Supabase Auth session token, which this architecture does not have. Supabase supports a signed-token alternative, but the official example does not demonstrate it. Proving that path is the sole objective of the Phase 1 spike, and it is the one place the Clerk decision carries genuine risk.

Supporting facts:

- Clerk's free tier covers **50,000 monthly retained users** as of February 2026. This shop has three. Cost is not a factor.
- Clerk's drop-in components remove a meaningful chunk of build time versus assembling sign-in UI by hand.
- If customer-facing accounts are added later (§6, P2), Clerk extends to that cleanly.

**Condition:** if the design ever changes so the browser queries Supabase directly, this decision must be revisited. In that model RLS becomes the security boundary, and Clerk would then require Supabase's native third-party auth integration — note that Clerk's older Supabase **JWT template was deprecated in April 2025** and should not be used. Staying with the Server Action model in §3.4 avoids the question entirely.

### 3.4 How pricing is protected

This is the core security requirement and it is satisfied by one rule:

> **The client sends inputs. The server returns the price. No dollar amount originating from the browser is ever trusted or stored.**

Implementation:

**1. The price list is server-only.** It lives in a module marked `import 'server-only'`, which makes the build fail if it is ever imported into a client component. The price list is never in the JavaScript bundle and cannot be read or modified from the browser.

**2. The browser never holds a database key.** All database access goes through Server Actions using the Supabase **service role key**, held in a Vercel environment variable and never exposed to the client. There is no anon key in the browser and no public database endpoint to call.

**3. The submission payload contains no prices.** The client sends only configuration:

```ts
'use server'
import 'server-only'
import { PRICE_LIST } from '@/lib/pricing'

const QuoteInput = z.object({
  decorationType: z.enum(['screen_print', 'embroidery', 'marketing']),
  brand:          z.enum(BRANDS),
  frontColors:    z.number().int().min(0).max(8),
  sizes:          z.record(SizeKey, z.number().int().min(0).max(5000)),
  dueDate:        z.coerce.date().min(new Date()),
  // no price fields — by design
})

export async function submitQuote(raw: unknown) {
  const input = QuoteInput.parse(raw)        // reject malformed/hostile input
  const total = calculateTotal(input, PRICE_LIST)  // server computes
  const { error } = await db.from('quotes').insert({ ...input, total })
  if (error) return { ok: false, error: 'Could not save your quote.' }  // §2.4
  return { ok: true, total }
}
```

A customer can send any payload they like. The worst outcome is a Zod rejection. There is no field in which a price can be supplied.

**4. Row Level Security stays on as a backstop.** Default-deny on every table. It should never be the thing standing between a customer and the data, but it should be there if something upstream is misconfigured.

### 3.5 Tooling / MCP setup

- **Supabase MCP** currently points at an unrelated project (84 tables, a different SaaS product). It must be repointed at the new print-hub project before it is useful here, otherwise migrations risk landing in the wrong database.
- **Resend MCP** likewise — confirm the target account before wiring notifications.
- **Clerk** — machine-to-machine credentials for automated setup; no manual token plumbing required.
- Secrets (`SUPABASE_SERVICE_ROLE_KEY`, `CLERK_SECRET_KEY`, `RESEND_API_KEY`) live in Vercel environment variables. None are ever referenced from client components.

---

## 4. MVP requirements

### P0 — required to ship

| # | Requirement | Addresses |
|---|---|---|
| 1 | **Artwork upload to Supabase Storage.** Private bucket, server-issued signed URLs, download rather than inline preview. Accept PDF, AI, EPS, SVG in addition to raster images. Enforce a size cap and validate type server-side. | §2.1, §2.10 |
| 2 | **Clerk authentication on the entire dashboard.** Remove the `?admin=true` parameter and the triple-click handler outright. | §2.2 |
| 3 | **New Supabase project**, schema defined in migrations under version control. | §2.3 |
| 4 | **Error handling on every write**, with a visible failure state. The success screen renders only after a confirmed commit. | §2.4 |
| 5 | **Single `quotes` table.** One row per submitted job, with a stage enum and a `needs_outreach` flag. Eliminate the dual write. | §2.5 |
| 6 | **Server-side pricing engine** per §3.4. Store the real computed total. | §2.6, §3.4 |
| 7 | **Zod validation** at every server entry point. | §2.10 |
| 8 | **Output escaping.** Handled by React by default; no `dangerouslySetInnerHTML`. | §2.7 |

### P1 — requested in the review meeting

| # | Requirement | Addresses |
|---|---|---|
| 9 | **Email notification on new quote** (Resend) to the owner. | §2.10 |
| 10 | **Bidirectional stage movement** — move jobs backward as well as forward, behind a **confirmation modal**. | §2.8 |
| 11 | **Audit trail** — who changed what, when, attributed to an individual login. See §5. | — |
| 12 | **Clickable job cards opening a detail modal**, so the board stays uncluttered. | — |
| 13 | **Working production calendar** driven by the current date. | §2.9 |
| 14 | **Soft delete** with a restore path. | §2.10 |
| 15 | Replace CDN Tailwind with a proper build. | §2.10 |

### P2 — next iteration

| # | Requirement |
|---|---|
| 16 | **Quote versioning.** The current flow is "rough estimate → specialist finalizes." That is two states and should be modeled as such, with the customer's original preserved. |
| 17 | **Customer-facing status link** — a tokenized read-only URL, cutting inbound "where's my order?" calls. |
| 18 | **Live board updates** via Supabase Realtime, so three people watching the same board see each other's changes. |
| 19 | ~~Role separation~~ — explicitly declined for MVP per §6.1. Revisit only if headcount grows beyond the current three. |

---

## 5. Metrics worth instrumenting

Chosen for what actually helps run a 3-person shop, not for vanity.

**Operational**

- **Time-in-stage per job.** Surfaces where work stalls — the single most useful number a production board can produce.
- **Jobs due this week / overdue count.** What the calendar was meant to answer.
- **Lead response time.** How long a "Needs Call" lead waits before someone calls. This directly measures the stated goal of the outreach column.
- **Rush-order percentage.** A rush surcharge already exists in the UI; this shows whether it is actually being captured.

**Commercial**

- Lead → quote → won conversion rate
- Average order value (real, once §2.6 is fixed)
- Repeat customer rate

**System health** — these exist so a failure is never silent again (§2.4)

- Quote submission success rate
- Artwork upload success rate
- Unhandled server errors (Sentry free tier is sufficient)
- Email delivery success

**On the audit trail (P1 #11):** with an owner, a partner, and a worker sharing one board, "who moved this job and when" is not a nice-to-have. It is what prevents *"I thought you already did that."* Cheap to add now, awkward to retrofit.

---

## 6. Decisions and open questions

### Resolved — locked for MVP

1. **All authenticated users see financials.** Owner, partner, and worker all see pricing, totals, and payment status. No role separation in the MVP.
2. **Flat permission model.** Any logged-in user may move any job between stages. No approval gate.
3. **Individual logins per person.** No shared account. This is what makes the audit trail (P1 #11) meaningful — with a flat permission model and no gate, *attribution* is the accountability mechanism, so per-user identity is load-bearing rather than cosmetic.
4. **Stage changes require a confirmation modal.** Guards against misclicks on a board three people are touching.

5. **New-quote notifications go to the owner only.** Noted as a risk to revisit: it makes the owner a single point of failure on lead response time, which is one of the metrics being tracked (§5). Cheap to widen later.

### Still open

6. **Artwork retention.** How long are customer files kept? Affects storage cost and is worth a line in a privacy policy. Not blocking; decide before launch.
7. **Is there existing data in the old Supabase project worth recovering?** Assessment assumes no (§2.3).

---

## 7. A note on the data

It was suggested in review that nothing in this system is sensitive because there are no payments.

Every record contains a customer's **name, phone number, and email address**. That is personal information under Texas Business & Commerce Code § 521, and it is currently readable by anyone who appends a query parameter to the URL. The absence of payment data lowers the severity; it does not remove the obligation.

This is straightforward to resolve — P0 #2 closes it — but it should not be carried forward as "just quotes."

---

## 8. Scope boundary

This engagement is a barter arrangement (development in exchange for company apparel). Because there is no invoice to anchor scope, the boundary is worth stating explicitly — this protects both sides:

**In scope:** Sections 4 P0 and P1.

**Out of scope, quoted separately if wanted:**

- P2 items (§4)
- The SanMar / Cotton Collective drop-ship platform
- Marketing, ad creative, social campaigns
- Ongoing hosting, maintenance, or support beyond handover

**Assumed provided by client:** logo and brand assets, the finalized price list, the list of garment brands to offer, and timely review at the two checkpoints below.

**Suggested checkpoints:** one after P0 (working, secure, real uploads) and one after P1 (feature-complete). Two reviews, not open-ended revisions.

---

## 9. Recommended order of work

> Revised after research. See `PRD-hustle-print-hub.md` §8 for full rationale.

1. New Supabase project + **complete** schema migrations — including line items, audit events, and soft-delete columns from the first migration
2. Next.js scaffold, Clerk wired via `proxy.ts`, dashboard behind auth, data-access layer with error-unwrapping helper
3. **Artwork upload spike** — prove the browser can PUT to a server-issued signed Storage URL. This is the riskiest unknown and it constrains the form built in step 4
4. Quote form + **server-side pricing** + Zod
5. **Artwork upload, full feature**
6. Production board: stages both directions, audit trail, manual job entry, editing, detail modal
7. Outreach/leads view
8. Resend notifications — *begin domain verification during step 1*, as DNS propagation is wall-clock time
9. Calendar, soft delete, payment tracking, metrics
10. Handover

**Two sequencing changes from the original draft**, both driven by research findings:

- The **artwork spike moved to the front**. Vercel caps function request bodies at 4.5 MB, so bytes cannot pass through a Server Action. Discovering that after the quote form is built would invalidate it.
- The **schema is complete in the first migration** rather than grown per feature. Audit and soft-delete columns are load-bearing from the first query written, even though their interfaces ship much later.

---

*Findings in §2 were verified against source at the referenced line numbers and by probing the live backend. §3 recommendations reflect the Clerk and Supabase integration status as of July 2026.*
