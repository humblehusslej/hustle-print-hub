# Architecture Research

**Domain:** Quoting + production-tracking tool for a 3-person screen-print / embroidery shop
**Researched:** 2026-07-30
**Confidence:** HIGH (stack constraints locked; all load-bearing platform claims verified against Next.js, Vercel, Clerk and Supabase primary sources — see Sources)

---

## 0. Three findings that change the design

Read these first. Each one invalidates an assumption present in `PRD-hustle-print-hub.md` or `ASSESSMENT.md`.

### F1 — `browser → Server Action → Storage` cannot carry a 12 MB PDF (blocking)

`PRD` FR-1 acceptance criterion: *"a customer uploads a 12 MB PDF; a team member opens that job and downloads a byte-identical file."*

That file cannot travel through a Server Action on Vercel.

- Next.js caps Server Action request bodies at **1 MB by default** (`serverActions.bodySizeLimit`), configurable.
- Vercel caps **any** function request body at **4.5 MB**, returning `413 FUNCTION_PAYLOAD_TOO_LARGE`. This is a platform limit and is **not** configurable.

So the stated flow tops out at 4.5 MB no matter what `next.config.js` says. The fix keeps every constraint intact: the Server Action issues a **short-lived, path-scoped signed upload URL**, and the browser `PUT`s the bytes straight to Supabase Storage. The bytes never enter a Vercel function. See §4. **The browser still holds no Supabase key** — it holds a single-use capability token for one server-chosen path.

### F2 — the live estimate must be a Route Handler, not a Server Action (design-deciding)

Next.js dispatches **Server Actions one at a time per client**, and they are not abortable. The official docs say so explicitly and recommend Route Handlers for non-mutation requests:

> "Next.js dispatches Server Actions one at a time per client. If a user triggers three actions in quick succession, the second waits for the first to finish, then the third waits for the second. […] If you need parallel work […] use a Route Handler for non-mutation requests."
> — Next.js, *Guides: Server Actions*

A debounced live price estimate is a high-frequency **read**. Putting it on a Server Action makes every keystroke burst head-of-line-block the queue, and there is no `AbortSignal` to cancel superseded requests. §3 resolves this.

### F3 — the prototype's duplicate bug has two causes, not one

`ASSESSMENT.md` §2.5 names the dual write. Reading `index.html:735-800` shows a second, larger multiplier: the submit handler wraps **both** inserts in `for (const item of customerManifestItems)`.

```js
for (const item of customerManifestItems) {     // index.html:747
  ...
  await syncCloudOrders(systemicCard, 'insert');   // :798
  await syncCloudLeads(manualLeadCard, 'insert');  // :799
}
```

A 3-line quote therefore produced **3 `app_orders` rows + 3 `app_manual_leads` rows = 6 records**, each carrying a separately randomised `HPC-####` id, for one customer submission.

Consolidating to a single `quotes` table fixes the dual write but **not** the loop. The loop is only fixed by modelling `quotes` **1:N** `quote_line_items`. This is the primary justification for a relational line-item table in §1 — it is a correctness requirement, not a scale preference.

There is also a third, latent duplicate source the prototype never hit because it never wrote successfully: **double-click submit**. Addressed with an idempotency key in §1.

---

## Standard Architecture

### System Overview

```
┌───────────────────────────────────────────────────────────────────────────┐
│  BROWSER                                                                   │
│                                                                            │
│  ┌────────────────────────────┐        ┌──────────────────────────────┐   │
│  │  PUBLIC  /quote            │        │  PROTECTED  /dashboard/**     │   │
│  │  (no auth, no Clerk UI)    │        │  (Clerk session required)     │   │
│  │  • config form (client)    │        │  • board, detail, leads, cal  │   │
│  │  • size grid (client)      │        │  • React Server Components    │   │
│  │  • estimate panel (client) │        │  • no Supabase key, no prices │   │
│  │  • file picker (client)    │        │    computed here              │   │
│  └───────┬────────┬───────┬───┘        └──────────────┬───────────────┘   │
└──────────┼────────┼───────┼───────────────────────────┼───────────────────┘
           │        │       │  (3) raw bytes, direct    │
      (1)  │   (2)  │       │      PUT to signed URL    │  (4) Server Actions
   debounced   Server│      └──────────────┐            │      (auth-checked)
   POST /api/  Action│                     │            │
   estimate    (sign)│                     │            │
           │        │                      │            │
┌──────────┼────────┼──────────────────────┼────────────┼───────────────────┐
│  VERCEL (Next.js App Router)             │            │                   │
│          ▼        ▼                      │            ▼                   │
│  ┌────────────────────────────────────┐  │  ┌──────────────────────────┐  │
│  │  middleware.ts — clerkMiddleware   │  │  │  Server Actions          │  │
│  │  protects /dashboard(.*) only      │  │  │  auth.protect() FIRST    │  │
│  └────────────────────────────────────┘  │  └────────────┬─────────────┘  │
│  ┌────────────────────────────────────┐  │               │                │
│  │  src/server/**  — import 'server-only'                 │                │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │                │
│  │  │ pricing/     │ │ db/          │ │ audit/       │◄──┘                │
│  │  │ rate-card.ts │ │ service-role │ │ log.ts       │                    │
│  │  │ calculate()  │ │ client       │ │ append-only  │                    │
│  │  └──────────────┘ └──────┬───────┘ └──────────────┘                    │
│  │  ┌──────────────┐        │         ┌──────────────┐                    │
│  │  │ storage/     │        │         │ email/       │                    │
│  │  │ sign upload  │        │         │ resend       │                    │
│  │  │ sign download│        │         │ (never       │                    │
│  │  └──────┬───────┘        │         │  blocking)   │                    │
│  └─────────┼────────────────┼─────────┴──────┬───────┴────────────────────┘
└────────────┼────────────────┼────────────────┼─────────────────────────────┘
             │ service role   │ service role   │ API key
             ▼                ▼                ▼
   ┌──────────────────┐ ┌──────────────┐ ┌──────────┐   ┌──────────┐
   │ Supabase Storage │ │ Supabase PG  │ │ Resend   │   │ Clerk    │
   │ bucket 'artwork' │ │ quotes       │ │          │   │ 3 users  │
   │ PRIVATE          │ │ line_items   │ └──────────┘   └──────────┘
   │ mime + size caps │ │ artwork_files│
   │ enforced by      │ │ quote_events │
   │ bucket config    │ │ RLS: deny-all│
   └────────▲─────────┘ └──────────────┘
            │
            └── (3) direct PUT from browser, signed single-path token
                    ↑ this edge is why 12 MB works
```

### Component Responsibilities

| Component | Owns | Implementation |
|-----------|------|----------------|
| `middleware.ts` | Route-level gate on `/dashboard(.*)` | `clerkMiddleware` + `createRouteMatcher` |
| Public quote route | Config capture, file picking, estimate display | Server Component shell + client leaf components |
| `POST /api/estimate` | Live price, concurrent + abortable | Route Handler; imports the server-only pricing module |
| Public Server Actions | Issue upload targets, finalize uploads, submit quote | `'use server'`, no auth, Zod-gated, session-scoped |
| Dashboard Server Actions | Stage moves, task toggles, lead contact, soft delete/restore | `'use server'`, `auth.protect()` as first statement |
| `src/server/pricing` | Rate card + `calculateQuote()` — **single source of price truth** | `import 'server-only'` |
| `src/server/db` | Service-role Supabase client + all queries | `import 'server-only'` |
| `src/server/storage` | Signed upload targets, signed downloads, existence verification | `import 'server-only'` |
| `src/server/audit` | Append `quote_events` rows | `import 'server-only'` |
| `lib/schemas` | Zod input shapes — **shared** client/server | Plain module; contains no secrets |
| Supabase Postgres | System of record | RLS deny-all; reached only by service role |
| Supabase Storage | Artwork bytes | Private bucket + bucket-level mime/size caps |

---

## 1. Data model

### Design decisions, stated up front

| Question | Decision | Why |
|---|---|---|
| Line items: JSONB or table? | **Relational table**, with JSONB *inside* it for the size vector | Artwork needs a real FK to a line; audit needs row-level before/after; §5 metrics need `GROUP BY decoration_type`. See below. |
| Size breakdown | **JSONB** (`sizes`) | 12 sparse keys, shape varies by decoration type, never filtered on individually |
| Money | **Integer cents** | Prototype used floats (`totalCount * 11.50`). Cents removes rounding drift entirely |
| Stage | **`text` + `CHECK`**, not a PG `ENUM` type | Satisfies FR-3's "stage enum" while staying trivially alterable in a migration |
| Quote id | **Sequence-backed `quote_number`** | Prototype's `"HPC-" + random(1000..9999)` collides ~50% by the 112th quote |
| Drafts | **No draft rows in `quotes`** | Keeps "one submission = exactly one row" (FR-3) literally true with no `WHERE` filter to forget |

**On JSONB vs. a line-items table.** At this scale (tens of quotes per month) both perform identically, so throughput is not the argument. The argument is referential integrity: `artwork_files` must point at a specific line item, and you cannot put a foreign key on an element of a JSONB array. Storing an array index or a client-minted id inside a JSON blob with no constraint reintroduces precisely the orphaned/mismatched-reference class of bug this rebuild exists to eliminate. Two secondary reasons: the audit log's before/after becomes a whole-document diff if a line edit rewrites the blob; and `ASSESSMENT.md` §5's rush-rate, decoration-mix and average-order-value metrics are one-line SQL against columns and awkward against JSON.

The size vector is the opposite case and JSONB is right there: a closed, Zod-validated set of ~12 keys that is always read and written as a unit, never queried by individual size, and whose shape legitimately varies (`{S,M,L,...}` for garments, `{OSFA}` for hats, `{FLAT}` for marketing goods).

### DDL — `supabase/migrations/0001_init.sql`

```sql
create extension if not exists pgcrypto;   -- gen_random_uuid()

-- =====================================================================
-- quotes — ONE row per customer submission.
-- Replaces app_orders + app_manual_leads. Kills both the dual write
-- and the per-line-item loop (see F3).
-- =====================================================================
create sequence quote_number_seq start with 1000;

create table quotes (
  id                 uuid primary key default gen_random_uuid(),
  quote_number       integer not null unique default nextval('quote_number_seq'),

  -- Idempotency. Minted once per form session; also namespaces artwork
  -- uploads before the quote exists (§4). Makes a double-clicked submit
  -- a no-op instead of a duplicate.
  intake_session_id  uuid not null unique,

  -- Customer. Denormalised: customers have no accounts (PRD §5 non-goal).
  customer_name      text not null check (length(btrim(customer_name)) between 1 and 200),
  customer_email     text not null check (customer_email ~* '^[^@[:space:]]+@[^@[:space:]]+\.[^@[:space:]]+$'),
  customer_phone     text not null check (length(btrim(customer_phone)) between 7 and 40),
  company_name       text,

  -- Scheduling. is_rush is SERVER-derived from due_date at submit time
  -- (prototype: <14 days). Frozen at submit so it stays true as time passes.
  due_date           date not null,
  is_rush            boolean not null default false,

  -- Production
  stage              text not null default 'pending'
                       check (stage in ('pending','prepress','production','ready')),
  stage_changed_at   timestamptz not null default now(),

  -- Outreach. Replaces app_manual_leads outright — one row, one lead.
  needs_outreach     boolean not null default true,
  contacted_at       timestamptz,
  contacted_by       text,

  -- Money. Integer cents. Written ONLY by the pricing engine.
  subtotal_cents     integer not null check (subtotal_cents  >= 0),
  setup_fee_cents    integer not null default 0 check (setup_fee_cents >= 0),
  rush_fee_cents     integer not null default 0 check (rush_fee_cents  >= 0),
  total_cents        integer not null check (total_cents >= 0),

  -- Which rate card produced the above. Required to explain an old quote
  -- after prices change, and the foundation for P2 quote versioning.
  pricing_version    text  not null,
  price_breakdown    jsonb not null default '{}'::jsonb,

  total_quantity     integer not null check (total_quantity >= 0),
  customer_notes     text,
  internal_notes     text,

  invoice_sent       boolean not null default false,
  is_paid            boolean not null default false,

  created_at         timestamptz not null default now(),
  updated_at         timestamptz not null default now(),
  deleted_at         timestamptz,          -- soft delete, present from day one

  constraint quotes_total_is_sum
    check (total_cents = subtotal_cents + setup_fee_cents + rush_fee_cents),
  constraint quotes_contact_pair
    check ((contacted_at is null) = (contacted_by is null))
);

-- =====================================================================
-- quote_line_items — N per quote. THIS is what removes the loop in F3.
-- =====================================================================
create table quote_line_items (
  id                 uuid primary key default gen_random_uuid(),
  quote_id           uuid not null references quotes(id) on delete cascade,
  line_number        smallint not null check (line_number >= 1),

  decoration_type    text not null
                       check (decoration_type in ('screen_print','embroidery','marketing')),
  product_category   text not null
                       check (product_category in
                         ('tee','polo','hoodie','jacket','hat',
                          'banner','business_card','signage')),
  brand              text not null,
  garment_color      text,
  placement_notes    text,

  -- screen_print
  front_colors       smallint not null default 0 check (front_colors between 0 and 8),
  back_colors        smallint not null default 0 check (back_colors  between 0 and 8),
  -- embroidery
  stitch_count       integer check (stitch_count is null or stitch_count between 0 and 200000),
  -- both decorated types: false => setup fee (prototype charged $30)
  logo_on_file       boolean not null default true,

  -- Size vector. Garments: {"S":12,"M":12,...,"YXL":0}
  -- Hats (OSFA):  {"OSFA":24}    Marketing: {"FLAT":250}
  sizes              jsonb not null default '{}'::jsonb
                       check (jsonb_typeof(sizes) = 'object'),
  quantity           integer not null check (quantity >= 0),

  unit_price_cents   integer not null check (unit_price_cents >= 0),
  setup_fee_cents    integer not null default 0 check (setup_fee_cents >= 0),
  line_total_cents   integer not null check (line_total_cents >= 0),

  -- 6-slot production checklist; labels vary by decoration_type in the UI
  -- (index.html:1146). Always read/written whole -> JSONB is correct here.
  tasks              jsonb not null
                       default '[false,false,false,false,false,false]'::jsonb,

  created_at         timestamptz not null default now(),
  updated_at         timestamptz not null default now(),

  unique (quote_id, line_number),
  constraint line_total_is_consistent
    check (line_total_cents = unit_price_cents * quantity + setup_fee_cents),
  constraint embroidery_declares_stitches
    check (decoration_type <> 'embroidery' or stitch_count is not null)
);

-- =====================================================================
-- artwork_files — the false-success fix lives in this table's constraints.
-- =====================================================================
create table artwork_files (
  id                 uuid primary key default gen_random_uuid(),

  -- Set at signing time, before a quote exists. Claimed at submit.
  intake_session_id  uuid not null,
  quote_id           uuid references quotes(id) on delete cascade,
  line_item_id       uuid references quote_line_items(id) on delete set null,

  storage_bucket     text not null default 'artwork',
  storage_path       text not null unique,   -- SERVER-generated, never user input

  original_filename  text not null,          -- display only, never a path
  extension          text not null
                       check (extension in ('pdf','ai','eps','svg','png','jpg','jpeg')),
  declared_mime      text not null,          -- what the browser claimed
  verified_mime      text,                   -- what magic-byte sniffing found
  byte_size          bigint check (byte_size is null or byte_size between 1 and 52428800),
  checksum_sha256    text,                   -- supports FR-1 "byte-identical"

  status             text not null default 'pending'
                       check (status in ('pending','uploaded','failed','rejected')),
  failure_reason     text,

  created_at         timestamptz not null default now(),
  uploaded_at        timestamptz,
  deleted_at         timestamptz,

  -- *** The structural anti-false-success guarantee. ***
  -- A row cannot claim 'uploaded' unless the SERVER observed the object:
  -- byte_size is written from a server-side HEAD of the stored object,
  -- never from the client's declared size.
  constraint uploaded_requires_server_evidence
    check (status <> 'uploaded'
           or (uploaded_at is not null and byte_size is not null)),
  constraint failure_is_explained
    check (status not in ('failed','rejected') or failure_reason is not null)
);

-- =====================================================================
-- quote_events — audit trail. Append-only.
-- With no permission tiers, attribution IS the accountability mechanism
-- (PROJECT.md Key Decisions), so unattributed team actions are rejected.
-- =====================================================================
create table quote_events (
  id             bigserial primary key,
  quote_id       uuid not null references quotes(id) on delete cascade,
  line_item_id   uuid references quote_line_items(id) on delete set null,

  action         text not null check (action in (
                   'quote.created','quote.updated','quote.deleted','quote.restored',
                   'stage.changed','line_item.updated','task.toggled',
                   'artwork.uploaded','artwork.failed',
                   'lead.contacted','lead.reopened',
                   'payment.updated','invoice.sent',
                   'email.sent','email.failed')),

  actor_type     text not null check (actor_type in ('team','customer','system')),
  actor_user_id  text,   -- Clerk user id
  -- Snapshotted, NOT joined: the audit trail must still name the person
  -- after they are removed from Clerk. That is the point of an audit trail.
  actor_name     text,
  actor_email    text,

  before         jsonb,          -- null for creates
  after          jsonb,          -- null for deletes
  summary        text not null,  -- pre-rendered, e.g. 'Production -> Pre-Press'

  created_at     timestamptz not null default now(),

  constraint team_actions_are_attributed
    check (actor_type <> 'team'
           or (actor_user_id is not null and actor_name is not null))
);

-- Append-only enforcement. A log you can edit is not a log.
create function forbid_mutation() returns trigger language plpgsql as $$
begin
  raise exception 'quote_events is append-only';
end $$;

create trigger quote_events_immutable
  before update or delete on quote_events
  for each row execute function forbid_mutation();

-- =====================================================================
-- Indexes — every board/lead read filters deleted_at, so partial indexes.
-- =====================================================================
create index quotes_board_idx    on quotes (stage, due_date)  where deleted_at is null;
create index quotes_due_idx      on quotes (due_date)         where deleted_at is null;
create index quotes_outreach_idx on quotes (created_at desc)  where deleted_at is null
                                                                and needs_outreach;
create index line_items_quote_idx on quote_line_items (quote_id);
create index artwork_quote_idx    on artwork_files (quote_id)          where deleted_at is null;
create index artwork_orphan_idx   on artwork_files (created_at)        where quote_id is null;
create index events_quote_idx     on quote_events (quote_id, created_at desc);

-- =====================================================================
-- updated_at
-- =====================================================================
create function touch_updated_at() returns trigger language plpgsql as $$
begin new.updated_at = now(); return new; end $$;

create trigger quotes_touch     before update on quotes
  for each row execute function touch_updated_at();
create trigger line_items_touch before update on quote_line_items
  for each row execute function touch_updated_at();

-- =====================================================================
-- RLS: default-deny backstop (PROJECT.md: "backstop, not primary boundary")
-- Enabled with ZERO policies => every role is denied. service_role bypasses
-- RLS by design, so Server Actions keep working. If a key ever leaks or an
-- anon endpoint is ever accidentally exposed, this table is still shut.
-- =====================================================================
alter table quotes           enable row level security;
alter table quote_line_items enable row level security;
alter table artwork_files    enable row level security;
alter table quote_events     enable row level security;
```

### Storage bucket — `0002_storage.sql`

```sql
insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values (
  'artwork', 'artwork',
  false,                       -- private: no public URL, not listable
  52428800,                    -- 50 MB; FR-1 needs 12 MB, this is headroom
  array[
    'application/pdf',
    'application/postscript',    -- .eps, and many .ai
    'application/illustrator',   -- some .ai
    'image/svg+xml',
    'image/png',
    'image/jpeg'
  ]
);
```

These two settings are enforced by the Storage service itself, on the upload request, independently of application code. That matters because with direct-to-storage uploads (§4) the bytes never pass through our server — this is the layer that cannot be bypassed by a hostile client.

**Gotcha:** `.ai` files are PDF- or PostScript-wrapped and report inconsistent MIME types across browsers and OSes; `.eps` is `application/postscript`. Validate on **extension + magic bytes**, and treat the browser-declared MIME as advisory only.

### What carries over from the prototype

Verified against `index.html` — the domain modelling is sound and should be preserved:

| Domain fact | Source | Lands in |
|---|---|---|
| 3 decoration types: Screen Printing, Embroidery, Marketing Materials | `:653-657` | `decoration_type` |
| SP brands: Gildan, Next Level, Bella+Canvas, Specialty Blank | `:654` | `brand`, rate-card tiers |
| EMB brands: Hats, Polos, Hoodies, Jackets | `:655` | `brand` + `product_category` |
| MKT: Banners, Business Cards, Metal Signage | `:657` | `brand` + `product_category` |
| Adult sizes S–4XL, youth YXS–YXL | `:437` | `sizes` JSONB |
| Hats are OSFA, single pooled qty, min 12 | `:530` | `sizes = {"OSFA": n}` |
| Marketing uses a flat qty, default 250 | `:749` | `sizes = {"FLAT": n}` |
| `logoOnFile = false` adds a setup fee | `:704-706` | `logo_on_file`, `setup_fee_cents` |
| Rush = due date inside 14 days | `:413-419` | `is_rush` |
| 4 stages: pending, prepress, production, ready | `:1056` | `stage` |
| 6-task checklist, labels vary by decoration | `:1146` | `tasks` JSONB |
| Artwork is attached **per line item** | `:471` | `artwork_files.line_item_id` |

**Flag for the rate card:** the prototype displays *"⚡ Rush Timeline Surcharge Applied"* (`:416`) but **never adds a rush charge** — `calculateManifestTotals` (`:692-707`) has no rush term. `ASSESSMENT.md` §5 wants rush-capture measured. The `rush_fee_cents` column exists; the multiplier is a **client input** the rate card needs (`ASSESSMENT.md` §8: "Assumed provided by client: the finalized price list"). Surface this before the pricing phase or it will block.

---

## 2. Route and directory structure

```
src/
├── middleware.ts                        # clerkMiddleware — the ONLY route gate
├── app/
│   ├── layout.tsx                       # <ClerkProvider> (root, wraps everything)
│   │
│   ├── (public)/                        # route group — no auth, no Clerk UI
│   │   ├── layout.tsx
│   │   ├── page.tsx                     # landing (SSR, keeps the SEO option open)
│   │   └── quote/
│   │       ├── page.tsx                 # Server Component shell
│   │       └── _components/             # leading _ => never routable
│   │           ├── quote-form.tsx           'use client'
│   │           ├── line-item-row.tsx        'use client'
│   │           ├── size-grid.tsx            'use client'
│   │           ├── artwork-uploader.tsx     'use client'
│   │           ├── estimate-panel.tsx       'use client'  # calls /api/estimate
│   │           └── submission-result.tsx    'use client'  # renders ONLY from ok:true
│   │
│   ├── (dashboard)/                     # route group — Clerk required
│   │   ├── layout.tsx                   # await auth.protect()  (second gate)
│   │   └── dashboard/
│   │       ├── page.tsx                 # board (RSC)
│   │       ├── @modal/(.)jobs/[id]/     # intercepting route => FR-12 modal
│   │       ├── jobs/[id]/page.tsx       # full detail (deep-linkable fallback)
│   │       ├── leads/page.tsx           # FR-13
│   │       └── calendar/page.tsx        # FR-15
│   │
│   ├── sign-in/[[...sign-in]]/page.tsx  # Clerk <SignIn/>
│   │
│   └── api/
│       └── estimate/route.ts            # POST — live pricing. See §3.
│
├── server/                              # EVERY file starts: import 'server-only'
│   ├── pricing/
│   │   ├── rate-card.ts                 # the price list. Never bundled.
│   │   ├── calculate.ts                 # calculateQuote() — single price truth
│   │   └── index.ts
│   ├── db/
│   │   ├── client.ts                    # service-role Supabase client
│   │   ├── quotes.ts
│   │   ├── artwork.ts
│   │   └── events.ts
│   ├── storage/
│   │   ├── sign-upload.ts               # createSignedUploadUrl
│   │   ├── sign-download.ts             # createSignedUrl (download: true)
│   │   └── verify-object.ts             # server-side existence + magic bytes
│   ├── audit/log.ts
│   ├── email/resend.ts
│   └── actions/
│       ├── public/                      # NO auth — reachable by anyone
│       │   ├── start-intake.ts          # mint intake_session_id
│       │   ├── artwork.ts               # createUploadTarget / finalizeUpload
│       │   └── submit-quote.ts
│       └── dashboard/                   # auth.protect() as first statement
│           ├── change-stage.ts
│           ├── toggle-task.ts
│           ├── contact-lead.ts
│           ├── update-payment.ts
│           └── soft-delete.ts
│
└── lib/
    ├── schemas/                         # Zod — SHARED client + server
    │   ├── quote-input.ts
    │   ├── artwork.ts
    │   └── enums.ts                     # brands, sizes, decoration types
    ├── format.ts                        # cents -> "$1,234.50"
    └── types/database.ts                # generated from the schema
```

### Where the middleware boundary falls

```ts
// src/middleware.ts
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server'

const isProtected = createRouteMatcher(['/dashboard(.*)'])

export default clerkMiddleware(async (auth, req) => {
  if (isProtected(req)) await auth.protect()
})

export const config = {
  matcher: ['/((?!_next|[^?]*\\.(?:html?|css|js|jpe?g|png|svg|webp|woff2?)).*)', '/(api|trpc)(.*)'],
}
```

Public: `/`, `/quote`, `/api/estimate`, `/sign-in`. Protected: `/dashboard/**`.

`clerkMiddleware` is public-by-default and you opt routes in — this is the current API. (Clerk's older `authMiddleware` is superseded; do not use it. Separately, Clerk's Supabase **JWT template was deprecated in April 2025** — irrelevant here only because the browser never talks to Supabase, per `ASSESSMENT.md` §3.3.)

### Middleware alone does NOT protect Server Actions

This is the single most important security note in this document.

A Server Action is a publicly reachable POST endpoint addressed by an opaque action id. Middleware matches on the **URL path**, not on the action. Clerk's own guidance is explicit: *"Route Handlers and Server Actions should enforce authorization on the server… the recommendation is to also perform explicit authorization checks within the Server Action itself rather than relying solely on middleware-based protection."*

Therefore every action under `server/actions/dashboard/` opens the same way:

```ts
'use server'
import { auth } from '@clerk/nextjs/server'

export async function changeStage(input: unknown) {
  const { userId } = await auth.protect()   // FIRST statement. Non-negotiable.
  const parsed = ChangeStageInput.parse(input)
  ...
}
```

Actions under `server/actions/public/` are deliberately unauthenticated (customers have no accounts) and are therefore defended by Zod + rate limiting + session scoping instead. Keeping the two sets in **separate directories** makes the distinction reviewable at a glance and lint-enforceable.

### Where Zod schemas live, and the exact line

`lib/schemas/` is imported by **both** client components and server code. That is safe and desirable — sharing means the customer sees inline validation instantly and the server re-validates identically, with no drift.

The line is precise:

- **Shareable:** input shapes, enum members (brand names, size keys, decoration types). These are already visible in the form's own dropdowns. Nothing is leaked.
- **Never shareable:** `server/pricing/rate-card.ts`. Base rates, ink-count grid, stitch rate, setup fee, rush multiplier.

---

## 3. The pricing engine boundary — resolving the live-estimate tension

### The two candidate answers, and why one loses

**Rejected: compute a "safe subset" client-side.**

There is no safe subset. Every input to `calculateQuote` maps to a rate-card entry, so any client-side calculation that produces a number the customer would accept must ship enough of the rate card to reconstruct it. The prototype proves this — `calculateManifestTotals` (`:692-707`) inlines the entire price list in the browser, and anyone can read `base = brand.includes('Specialty') ? 15.00 : ...` in DevTools. That directly violates FR-5 (*"searching the production client bundle for price-list constants returns nothing"*). It also creates two implementations of the same rules that will silently drift, which is a variant of the §2.6 bug — the number shown and the number stored diverge.

**Accepted: debounced server round-trip — but over a Route Handler, not a Server Action.**

Per F2, Next.js serialises Server Action dispatch per client and provides no abort. A price estimate fires on nearly every input change; that traffic pattern is exactly what the framework docs tell you to route through a Route Handler. Concretely, a Server Action estimate would mean a burst of size-grid edits queues up behind itself, and a submit fired mid-burst waits behind stale estimates.

A Route Handler is a plain HTTP endpoint: concurrent, and cancellable via `AbortController`. It is server code, so `import 'server-only'` holds and the rate card stays out of the bundle.

**The split:**

| Concern | Transport | Auth | Why |
|---|---|---|---|
| Live estimate (read) | `POST /api/estimate` Route Handler | none | Concurrent + abortable; high frequency |
| Submission (mutation) | Server Action `submitQuote` | none | Queueing is *desirable* here; integrates with `useActionState`; progressive enhancement |
| Dashboard mutations | Server Actions | `auth.protect()` | Queueing prevents interleaved board writes |

Both paths call **the same** `calculateQuote()`. That shared call is what makes FR-5's *"the total shown to the customer matches the row in the database exactly"* structurally true rather than a coincidence to be tested for.

### The module

```ts
// src/server/pricing/rate-card.ts
import 'server-only'          // build FAILS if a client component imports this

export const PRICING_VERSION = '2026-08-01'

export const RATE_CARD = {
  screenPrint: {
    base: { gildan: 625, next_level: 725, bella_canvas: 875, specialty: 1500 }, // cents
    inkByColorCount: [0, 650, 750, 850, 950],   // index = colour count, capped at 4
  },
  embroidery: {
    base: { hat: 1000, polo: 1000, hoodie: 1000, jacket: 4200 },
    digitizing: 350,
    perThousandStitches: 150,
  },
  marketing: { business_card: 30, banner: 4500, signage: 4500 },
  setupFeeNoLogoOnFile: 3000,
  rushMultiplierBps: 0,      // TODO(client): prototype never charged rush. See §1.
} as const
```

```ts
// src/server/pricing/calculate.ts
import 'server-only'
import { RATE_CARD, PRICING_VERSION } from './rate-card'
import type { QuoteInput } from '@/lib/schemas/quote-input'

export type PricedQuote = {
  pricingVersion: string
  lines: { lineNumber: number; quantity: number
           unitPriceCents: number; setupFeeCents: number; lineTotalCents: number }[]
  subtotalCents: number
  setupFeeCents: number
  rushFeeCents: number
  totalCents: number
  totalQuantity: number
  isRush: boolean
}

/** The ONLY function permitted to produce a dollar figure. */
export function calculateQuote(input: QuoteInput): PricedQuote { /* ... */ }
```

Note `PricedQuote` carries **totals, never rates**. The `/api/estimate` response is a computed result; it is not the rate card in disguise.

### The estimate endpoint

```ts
// src/app/api/estimate/route.ts
import 'server-only'
import { NextResponse } from 'next/server'
import { calculateQuote } from '@/server/pricing'
import { EstimateInput } from '@/lib/schemas/quote-input'

export async function POST(req: Request) {
  const parsed = EstimateInput.safeParse(await req.json())
  if (!parsed.success) {
    return NextResponse.json({ ok: false, error: 'Invalid configuration' }, { status: 400 })
  }
  const priced = calculateQuote(parsed.data)
  return NextResponse.json({
    ok: true,
    totalCents: priced.totalCents,
    totalQuantity: priced.totalQuantity,
    isRush: priced.isRush,
    lines: priced.lines.map(l => ({ lineNumber: l.lineNumber, lineTotalCents: l.lineTotalCents })),
  })
}
```

`EstimateInput` is `QuoteInput` **minus** contact details and `.partial()`-tolerant on a half-filled form, so an incomplete configuration still returns a running total. Neither schema has a price field — there is no key into which a price could be injected.

### The client side

```tsx
'use client'
// _components/estimate-panel.tsx
export function EstimatePanel({ config }: { config: DraftConfig }) {
  const [estimate, setEstimate] = useState<Estimate | null>(null)
  const [state, setState] = useState<'idle'|'calculating'|'error'>('idle')
  const abortRef = useRef<AbortController | null>(null)
  const debounced = useDebouncedValue(config, 350)

  useEffect(() => {
    abortRef.current?.abort()               // cancel the superseded request
    const ctrl = new AbortController()
    abortRef.current = ctrl
    setState('calculating')

    fetch('/api/estimate', {
      method: 'POST', signal: ctrl.signal,
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify(debounced),
    })
      .then(r => r.json())
      .then(d => { if (d.ok) { setEstimate(d); setState('idle') } else setState('error') })
      .catch(e => { if (e.name !== 'AbortError') setState('error') })

    return () => ctrl.abort()
  }, [debounced])

  // While recalculating, the old number is dimmed — never presented as current.
  // On error: "Estimate unavailable — your quote can still be submitted."
}
```

`AbortController` does double duty: it saves server work and it makes stale responses impossible, so no out-of-order reply can overwrite a newer estimate.

### Closing the last gap: estimate vs. stored total

A debounce leaves one race — the customer can hit Submit while a recalculation is in flight and briefly see a stale figure. Resolution:

> **The confirmation screen renders the total returned by `submitQuote`, never the last value from `/api/estimate`.**

The estimate is advisory UI. The submit response is authoritative and is the same number written to `quotes.total_cents` in the same call. FR-5's *"the total shown to the customer matches the row in the database exactly"* then holds by construction.

### Making FR-5 an automated check, not a manual one

FR-5 says to verify by searching the production bundle. Automate it in CI:

```bash
next build
# The rate card must not appear in any client chunk.
if grep -rlE "inkByColorCount|perThousandStitches|setupFeeNoLogoOnFile" .next/static/; then
  echo "FAIL: pricing constants leaked into the client bundle"; exit 1
fi
```

`server-only` already fails the build on a direct import; this catches the indirect path (a shared barrel file re-exporting pricing). Both belong in the pipeline.

---

## 4. Artwork upload flow

### The failure modes being designed out

`ASSESSMENT.md` §2.1 and §2.4 are **two distinct false successes**:

- **§2.1** — a green *"Artwork linked successfully"* toast fired on file **selection** (`index.html:471-485`). The `File` object was discarded; only the filename string survived, concatenated into a notes field.
- **§2.4** — the success screen rendered unconditionally after unchecked inserts (`:809`).

Three rules kill both:

1. **No success language before the server has evidence.** Picking a file yields *"Ready to upload"*, never *"uploaded"* or *"linked"*.
2. **The server verifies the object itself.** `status='uploaded'` is written only after the server observes the stored object; `byte_size` comes from that observation, never from the client's claim. The DB `CHECK` refuses any other combination.
3. **Submission refuses to complete while any attached file is unverified.** No partial success is representable.

### End-to-end flow

```
┌── BROWSER ──────────────┐   ┌── VERCEL ────────────┐   ┌── SUPABASE ────────┐
│                         │   │                      │   │                    │
│ 0. form mounts          │──▶│ startIntake()        │   │                    │
│                         │◀──│  -> intake_session_id│   │                    │
│                         │   │                      │   │                    │
│ 1. user picks file      │   │                      │   │                    │
│    cheap local checks:  │   │                      │   │                    │
│    extension, size      │   │                      │   │                    │
│    UI: "Ready to upload"│   │                      │   │                    │
│    (NEVER "uploaded")   │   │                      │   │                    │
│                         │   │                      │   │                    │
│ 2. createUploadTarget   │──▶│ Zod: ext allowlist,  │   │                    │
│    {filename,mime,size} │   │ mime allowlist,      │   │                    │
│                         │   │ size <= cap,         │   │                    │
│                         │   │ files/session <= 10  │   │                    │
│                         │   │        │             │   │                    │
│                         │   │        ▼             │   │                    │
│                         │   │ path = {session}/    │   │                    │
│                         │   │        {uuid}.{ext}  │   │                    │
│                         │   │ SERVER-generated —   │   │                    │
│                         │   │ user filename never  │   │                    │
│                         │   │ touches the path     │   │                    │
│                         │   │        │             │   │                    │
│                         │   │ INSERT artwork_files │──▶│ status='pending'   │
│                         │   │        │             │   │                    │
│                         │   │ createSignedUploadUrl│──▶│ token, 2h TTL,     │
│                         │◀──│ {fileId,url,token}   │◀──│ scoped to ONE path │
│                         │   │                      │   │ upsert: false      │
│                         │   │                      │   │                    │
│ 3. PUT bytes ───────────┼───┼──────────────────────┼──▶│ object stored      │
│    plain fetch/XHR      │   │  BYPASSES the 4.5 MB │   │ bucket re-checks   │
│    progress bar         │   │  Vercel body limit   │   │ mime + size        │
│    NO Supabase key      │   │  (F1)                │   │                    │
│                         │   │                      │   │                    │
│ 4. finalizeUpload       │──▶│ HEAD the object ─────┼──▶│ exists? size?      │
│    {fileId}             │   │ Range bytes 0-1023 ──┼──▶│ magic-byte sniff   │
│                         │   │        │             │   │                    │
│                         │   │  exists & plausible? │   │                    │
│                         │   │   yes -> 'uploaded'  │   │                    │
│                         │   │          + byte_size │   │                    │
│                         │   │          + uploaded_at                        │
│                         │   │   no   -> 'failed'   │   │                    │
│                         │   │          + reason    │   │                    │
│                         │◀──│ {ok, status}         │   │                    │
│                         │   │                      │   │                    │
│ 5. UI shows "Uploaded"  │   │                      │   │                    │
│    ONLY on ok+uploaded  │   │                      │   │                    │
│                         │   │                      │   │                    │
│ 6. submitQuote          │──▶│ Zod validate         │   │                    │
│    {config, sessionId}  │   │ REFUSE if any file   │   │                    │
│                         │   │  for this session is │   │                    │
│                         │   │  not 'uploaded'      │   │                    │
│                         │   │        │             │   │                    │
│                         │   │ calculateQuote()     │   │                    │
│                         │   │        │             │   │                    │
│                         │   │ TXN: quotes          │──▶│                    │
│                         │   │    + line_items      │   │                    │
│                         │   │    + claim artwork   │   │                    │
│                         │   │    + quote.created   │   │                    │
│                         │   │        │             │   │                    │
│                         │◀──│ {ok:true, number,    │   │                    │
│                         │   │  totalCents}         │   │                    │
│ 7. success screen       │   │  OR {ok:false,error} │   │                    │
│    renders ONLY on      │   │                      │   │                    │
│    ok:true; form data   │   │ (email fired after   │   │                    │
│    preserved on error   │   │  commit, non-blocking)│  │                    │
└─────────────────────────┘   └──────────────────────┘   └────────────────────┘
```

### Retrieval (dashboard)

```
Team member opens /dashboard/jobs/[id]
    ↓
RSC — auth.protect() at the (dashboard) layout
    ↓
db.artwork.listForQuote(id)  ->  rows with status='uploaded'
    ↓
Render filename, size, type. Download button posts to a Server Action.
    ↓  (on click — NOT at render time)
getArtworkDownloadUrl(fileId)   'use server'
    ↓  auth.protect()
storage.from('artwork').createSignedUrl(path, 60, { download: original_filename })
    ↓
60-second URL returned; browser navigates to it; file downloads
    ↓
audit: 'artwork.downloaded'  (optional but cheap)
```

Two deliberate choices:

- **Sign on click, not on render.** A signed URL embedded in server-rendered HTML lives as long as its TTL in the page source, browser history, and any screenshot. Minting on demand with a 60-second TTL keeps the exposure window at seconds.
- **`download: true` forces `Content-Disposition: attachment`.** `PRD` FR-1 says *"download rather than inline preview"*; the reason is that SVG is an executable document format. An SVG served inline from a storage origin is a stored-XSS vector — the exact §2.7 class of bug, relocated. Forcing attachment closes it. Never render customer artwork in an `<img>` or `<iframe>`.

### Why an `intake_session_id` rather than a draft quote row

The artwork exists before the quote does. Two ways to bridge that:

- **Draft rows in `quotes`** — real FKs throughout, but now every board, lead, calendar and metrics query must carry `WHERE submitted_at IS NOT NULL`. One forgotten filter puts an abandoned half-filled draft, with its customer's PII, on the production board. That is the same shape of bug as the one being fixed.
- **Session-scoped uploads (chosen)** — `artwork_files.quote_id` is nullable until submit claims it. `quotes` then means *"a real submitted quote"* with no qualifier, so FR-3's *"one submission produces exactly one row"* is checkable as `select count(*) from quotes` with nothing to remember.

Orphan cleanup is one scheduled job:

```sql
-- daily: delete unclaimed uploads older than 24h (storage objects deleted first)
select id, storage_path from artwork_files
where quote_id is null and created_at < now() - interval '24 hours';
```

**Abuse surface, and its bounds.** `intake_session_id` is an unauthenticated capability, so: the **server** mints it (never accept a client-chosen id); cap files per session (10) and total bytes; rate-limit `createUploadTarget` by IP; the signed upload token is scoped to **one server-chosen path** with `upsert: false`, so it cannot overwrite another quote's artwork or write outside its prefix; the bucket's own mime and size limits apply regardless; and the 24-hour sweep bounds accumulation.

### Where each validation happens, and why there

| Check | Where | Why there |
|---|---|---|
| Extension in allowlist | Client (UX) **and** server (authority) | Client for instant feedback; server because `accept=` is trivially bypassed — this is the FR-1 requirement |
| Size ≤ cap | Client (UX), server at signing, **bucket** at upload | Three layers; only the bucket check cannot be bypassed |
| MIME allowlist | Server at signing + **bucket** at upload | Declared MIME is a client claim; the bucket enforces independently |
| **Object actually exists** | Server at finalize | The false-success fix. Nothing else establishes that bytes landed |
| Magic bytes | Server at finalize, `Range: bytes=0-1023` | Real content validation without pulling 12 MB |
| All files `uploaded` | Server at submit | Makes "submitted with a broken attachment" unrepresentable |
| Byte-identical round trip | `checksum_sha256` | Turns FR-1's manual verify into an assertion |

---

## 5. Build order

### Dependency graph

```
                    ┌──────────────────────────────────┐
                    │ A. Supabase project + migrations │
                    │    quotes, line_items,           │
                    │    artwork_files, quote_events,  │
                    │    RLS deny-all, storage bucket  │
                    └───────────────┬──────────────────┘
                                    │  blocks everything
              ┌─────────────────────┼──────────────────────┐
              ▼                     ▼                      ▼
   ┌────────────────────┐ ┌──────────────────┐ ┌─────────────────────┐
   │ B. service-role    │ │ C. Clerk +       │ │ D. Zod schemas      │
   │    db client       │ │    middleware +  │ │    lib/schemas      │
   │    (server-only)   │ │    sign-in       │ │                     │
   └─────────┬──────────┘ └────────┬─────────┘ └──────────┬──────────┘
             │                     │                      │
             │                     │        ┌─────────────┴──────────┐
             │                     │        ▼                        ▼
             │                     │  ┌──────────────┐   ┌────────────────────┐
             │                     │  │ E. pricing   │   │ F. storage helpers │
             │                     │  │  rate-card   │   │  sign up/down,     │
             │                     │  │  calculate() │   │  verify-object     │
             │                     │  └──┬────────┬──┘   └─────────┬──────────┘
             │                     │     │        │                │
             │                     │     ▼        ▼                ▼
             │                     │ ┌─────────┐ ┌──────────────────────────┐
             │                     │ │G. /api/ │ │ H. artwork actions       │
             │                     │ │ estimate│ │   createUploadTarget     │
             │                     │ └─────────┘ │   finalizeUpload         │
             │                     │             └────────────┬─────────────┘
             ▼                     │                          │
   ┌────────────────────┐          │       ┌──────────────────┘
   │ I. audit log       │          │       │
   │    writer          │          │       │
   └─────────┬──────────┘          │       │
             │      ┌──────────────┼───────┘
             ▼      ▼              │
   ┌──────────────────────────┐    │
   │ J. submitQuote action    │    │   needs B,D,E,H,I
   │   TXN + claim + audit    │    │
   └────────────┬─────────────┘    │
                │                  │
                ▼                  ▼
   ┌──────────────────────────────────────┐
   │ K. quote form UI (public)            │   needs D,G,H,J
   └────────────┬─────────────────────────┘
                │  first real rows exist
                ▼
   ┌──────────────────────────────────────┐
   │ L. dashboard board + detail (RSC)    │   needs A,B,C + rows from J
   └────────────┬─────────────────────────┘
                │
        ┌───────┼───────────┬──────────────┐
        ▼       ▼           ▼              ▼
   ┌────────┐ ┌────────┐ ┌────────┐  ┌───────────────┐
   │M. stage│ │N. art  │ │O. leads│  │P. Resend      │
   │ moves +│ │ download│ │ view  │  │  notification │
   │ confirm│ │ (signed)│ │       │  │  (after commit│
   └────────┘ └────────┘ └────────┘  │   never blocks)│
        │                             └───────────────┘
        ▼
   ┌──────────────────────────────────┐
   │ Q. calendar, soft-delete UI,     │
   │    metrics                       │
   └──────────────────────────────────┘
```

### Hard blocks — stated explicitly

| This | Cannot start before | Reason |
|---|---|---|
| Anything touching data | **A** migrations | No tables |
| `/api/estimate` (G) | **E** pricing | It is a thin wrapper over `calculateQuote` |
| `submitQuote` (J) | **E** pricing | Must store the server-computed total (fixes §2.6) |
| `submitQuote` (J) | **H** artwork actions | It must refuse unverified attachments; that check is part of the action |
| Artwork **claim** (in J) | **H** | Claim needs `artwork_files` rows to claim |
| Artwork **retrieval** (N) | **L** detail view | Nowhere to put the download button |
| Board (L) | **J** | Nothing to render until real rows exist |
| Stage moves (M) | **I** audit | Every stage change must write an event; a stage change with no event is the bug |
| Leads view (O) | **J** | `needs_outreach` is set at insert |
| Email (P) | **J** | Fires after a confirmed commit |

### Two things that must be built early even though they ship late

These are the retrofit traps. Both are cheap now and expensive later.

**1. `quote_events` and the `quote.created` write belong in the first data phase, not the audit phase.**

`PRD` §8 puts the audit trail in Milestone 4. If the table and the creation write arrive then, every quote created during Milestones 2–3 has **no creation event**, so the detail view's history starts mid-story and the §5 lead-response-time metric has no `t=0`. Backfilling invents timestamps.

Split it: **table + write in Milestone 1–2; audit *UI* in Milestone 4.** Only the display is deferrable.

**2. `deleted_at` must exist in the first migration even though restore UI is Milestone 6.**

Every board, lead, calendar and metrics query needs `where deleted_at is null`. Add the column late and you must audit every query written before it — and the one you miss shows deleted jobs on the board. Ship the column in migration `0001`; ship the delete/restore UI whenever.

### Refinement to the PRD's milestone rationale

`PRD` §8 states: *"Milestone 2 precedes 3 because artwork attaches to a quote record that must exist first."*

Under the session-scoped design that premise is **false** — uploads happen before any quote row exists (that is what makes 12 MB work). The ordering still holds, for two different reasons:

1. The **claim** step lives inside `submitQuote`, so submission must exist to attach files.
2. Retrieval must be **provable**, and FR-1's exit criterion (*"a team member opens that job and downloads a byte-identical file"*) needs a dashboard detail view — which is Milestone 4.

Practical consequence: FR-1 is **not fully verifiable at the end of Milestone 3** as currently written. Either pull a minimal job-detail page forward into Milestone 3, or move FR-1's download half of the criterion into Milestone 4. Worth deciding at roadmap time rather than discovering at the checkpoint.

### Suggested phase mapping

| Phase | Contents | Verifiable exit |
|---|---|---|
| 1 Foundation | A, B, C, D + `deleted_at` + `quote_events` table + CI bundle-grep | Signed-in user reaches an empty dashboard; signed-out gets nothing; migrations apply to an empty DB |
| 2 Pricing + submit | E, G, I, J, K (form without artwork) | Quote submits with a server-computed total; injected price field is ignored; DB total == displayed total; failure shows an error and keeps the form data |
| 3 Artwork | F, H + submit-time gate | 12 MB PDF uploads; unsigned object URL is refused; a forced upload failure blocks submission with a visible error |
| 4 Board | L, M, N + audit UI | Jobs move both ways behind a confirmation, attributed to the right person; artwork downloads byte-identical |
| 5 Leads + email | O, P | 10 submissions → 10 outreach entries; one email per quote; email failure never blocks the customer |
| 6 Polish | Q | Calendar includes today; deletes restore; metrics visible |

---

## Data flow: the two critical paths

### Path 1 — quote submission (fixes §2.1, §2.4, §2.5, §2.6)

```
Customer fills form
  → per change: debounced POST /api/estimate (abortable)  → advisory total
  → per file:   createUploadTarget → direct PUT → finalizeUpload (server-verified)
  → Submit:
       submitQuote({ config, intakeSessionId })          [Server Action]
         1. Zod parse                     → fail: {ok:false}, form preserved   [FR-6]
         2. contains a price field?       → impossible; no such key exists      [FR-5]
         3. any artwork not 'uploaded'?   → fail: {ok:false}                    [FR-1]
         4. calculateQuote(input)         → authoritative total                 [FR-5]
         5. TRANSACTION
              insert quotes  (on conflict (intake_session_id) do nothing)       [dedupe]
              insert quote_line_items  × N   ← ONE quote, N lines               [F3 fix]
              update artwork_files set quote_id, line_item_id
              insert quote_events ('quote.created', actor_type='customer')
            commit
         6. commit failed?                → {ok:false, error}                   [FR-4]
         7. fire-and-forget Resend; failure logs an 'email.failed' event
            and NEVER affects the return value                                 [FR-14]
         8. return {ok:true, quoteNumber, totalCents}
  → success screen renders ONLY from ok:true, showing the RETURNED total
```

Every prototype failure has a named structural counter: unchecked writes → step 6; dual write + loop → step 5; placeholder price → step 4; discarded file → step 3.

### Path 2 — artwork retrieval

```
GET /dashboard/jobs/[id]
  → middleware: isProtected → auth.protect()          [gate 1, FR-2]
  → (dashboard)/layout.tsx: await auth.protect()      [gate 2, defence in depth]
  → RSC: db.quotes.getDetail(id)   -- service role, server-side only
  → RSC: db.artwork.listForQuote(id) where status='uploaded'
  → render metadata as React text nodes (auto-escaped; no dangerouslySetInnerHTML) [FR-8]
  → user clicks Download
       getArtworkDownloadUrl(fileId)                  [Server Action]
         auth.protect()                               [gate 3 — actions are public POSTs]
         createSignedUrl(path, 60, { download: filename })
         → 60s URL, Content-Disposition: attachment   [SVG-XSS closed]
       browser navigates → file downloads
```

---

## Anti-Patterns

### AP1 — Relying on middleware to protect Server Actions

**What people do:** match `/dashboard(.*)` in middleware and assume every action rendered on those pages is protected.
**Why it's wrong:** a Server Action is a POST endpoint addressed by action id, not by page. Middleware matches paths. Clerk's docs recommend explicit in-action checks rather than middleware alone.
**Instead:** `await auth.protect()` as the first statement of every dashboard action. Keep public and authenticated actions in separate directories so a reviewer can see the difference.

### AP2 — Routing file bytes through a Server Action

**What people do:** `FormData` with the `File` → Server Action → `storage.upload()`.
**Why it's wrong:** 1 MB Next.js default, 4.5 MB Vercel hard ceiling. FR-1's 12 MB PDF fails, and it fails as a `413` that is easy to mistake for a network blip — a *new* silent-failure mode.
**Instead:** signed upload URL, direct browser → Storage `PUT`.

### AP3 — Trusting the client's report that an upload succeeded

**What people do:** flip a record to `uploaded` because the client said the `PUT` returned 200.
**Why it's wrong:** it is §2.1 with extra steps — the app still cannot tell a stored file from a claimed one.
**Instead:** the server HEADs the object and writes `byte_size` from its own observation. The `uploaded_requires_server_evidence` CHECK makes the alternative unrepresentable.

### AP4 — Using Server Actions for high-frequency reads

**What people do:** the live estimate as a Server Action because "actions are the App Router way."
**Why it's wrong:** they dispatch one at a time per client and cannot be aborted; the docs point at Route Handlers for non-mutation requests.
**Instead:** Route Handler + debounce + `AbortController`. Server Actions for mutations.

### AP5 — Storing money as `numeric`/float and prices as client input

**What people do:** `estimated_total: totalCount * 11.50` — literally §2.6.
**Why it's wrong:** float drift, plus a client-writable field is a client-controlled field.
**Instead:** integer cents, written only by `calculateQuote`, with `total = subtotal + setup + rush` enforced as a CHECK.

### AP6 — Using the customer's filename as the storage path

**What people do:** `upload(\`${quoteId}/${file.name}\`, file)`.
**Why it's wrong:** traversal sequences, unicode collisions, and second-upload overwrites of the first.
**Instead:** server-generated `{session}/{uuid}.{ext}`; keep `original_filename` for display and for the download's `Content-Disposition`.

### AP7 — Joining the audit trail to a live users table

**What people do:** store `actor_user_id` and join Clerk at render time.
**Why it's wrong:** when someone leaves and is deleted, history reads "Unknown user" — precisely when attribution matters most. With no permission tiers, attribution *is* the accountability model.
**Instead:** snapshot `actor_name` and `actor_email` at write time. Enforce with `team_actions_are_attributed`.

### AP8 — Rendering customer artwork inline

**What people do:** `<img src={signedUrl}>` for a nicer board.
**Why it's wrong:** SVG is executable. Inline SVG from a storage origin re-creates §2.7 in a new location.
**Instead:** `download: true` on every signed URL. If previews are wanted later, server-side rasterise to PNG first.

---

## Scaling Considerations

Realistic ceiling: 3 users, tens of quotes a month. Nothing here needs to scale; the notes below are about what would break *first* if it did.

| Scale | Adjustment |
|---|---|
| 0–1k quotes (actual) | As designed. Supabase free/Pro, Vercel Hobby/Pro, Clerk free (50k MRU). No caching, no pagination. |
| 1k–10k quotes | Paginate the board (the partial indexes already cover the sort). Move `/api/estimate` to the Edge runtime — it is pure computation with no DB access. |
| 10k+ | Partition `quote_events` by month; add `pg_cron` for the orphan sweep; move metrics to a materialised view. |

**First bottleneck:** the board query fetching all non-deleted quotes with their line items. Fix by paginating per stage column. **Second:** storage cost from never-deleted artwork — which is the retention question already open in `PROJECT.md`. The `deleted_at` column on `artwork_files` is the hook for whatever policy is chosen.

---

## Integration Points

### External Services

| Service | Integration | Gotchas |
|---|---|---|
| Clerk | `clerkMiddleware` + `auth.protect()` in every dashboard action | Middleware does not cover Server Actions. `authMiddleware` is superseded. The Supabase JWT template (deprecated Apr 2025) is irrelevant here only because the browser never calls Supabase. |
| Supabase PG | Service-role client, server-only, never in a client component | Service role bypasses RLS by design — that is why deny-all policies are a backstop, not the gate |
| Supabase Storage | Signed upload URL out, signed download URL in | Upload tokens are fixed at **2h TTL** (not configurable). Set `upsert:false`. Set bucket `file_size_limit` + `allowed_mime_types` — these are enforced independently of app code. |
| Resend | Called after commit, awaited with its own try/catch | Must never affect `submitQuote`'s return value (FR-14). Log `email.failed` to `quote_events`. |
| Vercel | Env vars for all three secrets | 4.5 MB body limit is platform-level and shapes §4 |

### Internal Boundaries

| Boundary | Communication | Notes |
|---|---|---|
| client components ↔ pricing | Only via `/api/estimate` and `submitQuote` | Enforced by `server-only` + CI bundle grep |
| client components ↔ database | Never direct | No Supabase client exists in the browser |
| public actions ↔ dashboard actions | Separate directories | Public = unauthenticated by design; dashboard = `auth.protect()` first |
| `lib/schemas` ↔ everything | Shared import | Contains input shapes and enums only. Never a rate. |
| pricing ↔ persistence | `calculateQuote()` → the only writer of `*_cents` | Guarantees displayed total == stored total |

---

## Open items for roadmap

1. **Rush surcharge rate is missing.** The prototype shows a rush banner and charges nothing. `rush_fee_cents` and `rushMultiplierBps` exist but need a client-supplied number. Blocks Phase 2 exit if rush pricing is in scope.
2. **FR-1's download half is not verifiable in Milestone 3.** Pull a minimal detail view forward, or move that criterion to Milestone 4. Decide at roadmap time.
3. **Artwork retention** (already open in `PROJECT.md`) determines the orphan-sweep and `deleted_at` cleanup policy. Not blocking; the hooks exist.
4. **Rate limiting on public endpoints** (`/api/estimate`, `createUploadTarget`) is unspecified. Low risk at this scale; worth one line of Vercel or Upstash config in Phase 3.

---

## Confidence Assessment

| Claim | Confidence | Basis |
|---|---|---|
| Vercel 4.5 MB body limit blocks 12 MB via Server Action | HIGH | Vercel KB + Vercel Functions Limitations docs |
| Next.js Server Action default body limit is 1 MB | HIGH | Next.js `serverActions` config docs + framework source |
| Server Actions dispatch serially and are not abortable | HIGH | Next.js *Guides: Server Actions*, explicit; corroborated by `app-router-instance.ts` action queue |
| `server-only` fails the build on client import | HIGH | Next.js Data Security + Server/Client Components docs |
| Signed upload URLs are path-scoped, 2h TTL | HIGH | Supabase `createSignedUploadUrl` reference |
| Bucket `file_size_limit` / `allowed_mime_types` enforced server-side | HIGH | Supabase Storage buckets docs + storage-js PR #151 |
| Browser can PUT to a signed URL without a Supabase key | MEDIUM-HIGH | Supabase docs + community confirmation; verify with a spike in Phase 3 |
| Middleware does not protect Server Actions | HIGH | Clerk authorization-checks + Server Actions docs |
| Prototype creates one row per line item per table | HIGH | Read directly at `index.html:747-800` |
| Prototype never charges the rush surcharge | HIGH | Read directly at `index.html:692-707` |

---

## Sources

**Next.js**
- [Guides: Server Actions](https://nextjs.org/docs/app/guides/server-actions) — sequential dispatch; "use a Route Handler for non-mutation requests"
- [`serverActions` config](https://nextjs.org/docs/app/api-reference/next-config-js/serverActions) — 1 MB default `bodySizeLimit`
- [Data Security](https://nextjs.org/docs/app/guides/data-security) — `server-only`
- [Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components) — environment poisoning
- [Backend for Frontend](https://nextjs.org/docs/app/guides/backend-for-frontend) — Server Actions unsuited to data fetching
- [vercel/next.js discussion #50743](https://github.com/vercel/next.js/discussions/50743), [#84893](https://github.com/vercel/next.js/discussions/84893) — no parallelism, no abort

**Vercel**
- [Functions Limitations](https://vercel.com/docs/functions/limitations) — 4.5 MB payload
- [How to bypass the body size limit](https://vercel.com/kb/guide/how-to-bypass-vercel-body-size-limit-serverless-functions) — direct-to-storage

**Supabase**
- [`createSignedUploadUrl`](https://supabase.com/docs/reference/javascript/storage-from-createsigneduploadurl) — 2h TTL, path-scoped
- [`uploadToSignedUrl`](https://supabase.com/docs/reference/javascript/storage-from-uploadtosignedurl)
- [`createSignedUrl`](https://supabase.com/docs/reference/javascript/storage-from-createsignedurl) — `download` option
- [Storage Buckets fundamentals](https://supabase.com/docs/guides/storage/buckets/fundamentals) — private buckets
- [Creating buckets](https://supabase.com/docs/guides/storage/buckets/creating-buckets) — `fileSizeLimit`, `allowedMimeTypes`
- [Serving assets / downloads](https://supabase.com/docs/guides/storage/serving/downloads)
- [storage-js PR #151](https://github.com/supabase/storage-js/pull/151) — bucket-level limits are server-side

**Clerk**
- [`clerkMiddleware` / `createRouteMatcher`](https://clerk.com/docs/reference/nextjs/clerk-middleware)
- [Server Actions (App Router)](https://clerk.com/docs/reference/nextjs/app-router/server-actions)
- [Authorization checks](https://clerk.com/docs/guides/secure/authorization-checks) — enforce in the action, not only middleware

**Project**
- `ASSESSMENT.md` §2.1, §2.4, §2.5, §2.6, §2.7, §3.4 — failure modes and the pricing security model
- `PRD-hustle-print-hub.md` §4, §8 — acceptance criteria and proposed milestones
- `index.html` — domain model read at `:437`, `:471`, `:530`, `:653-657`, `:692-707`, `:747-800`, `:1056`, `:1146`

---
*Architecture research for: quoting + production tracking, 3-person print shop*
*Researched: 2026-07-30*
