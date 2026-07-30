# Phase 1 — Foundation & Access Control Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Three people sign in to a trustworthy empty shell — nobody else gets in, the database holds the whole story from day one, and the one remaining upload unknown is proven.

**Architecture:** Next.js App Router on Vercel. Clerk owns identity; a bare `clerkMiddleware()` in `proxy.ts` exists only so `auth()` works — it protects nothing, because Server Actions are dispatched by action ID and bypass path matchers. Every dashboard resource guards itself with `await auth.protect()`. All database access flows through a `server-only` data-access layer using the Supabase secret key; the browser holds no durable Supabase credential. Three npm guard scripts run in `prebuild`, so a violation fails the Vercel build and ships nothing.

**Tech Stack:** Next.js 16.2.12 · React 19.2.8 · `@clerk/nextjs` 7.6.3 · `@supabase/supabase-js` 2.111.0 · Zod 4.4.3 · Tailwind 4.3.3 · Vitest · TypeScript 6 (see Global Constraints)

**Spec:** `docs/superpowers/specs/2026-07-30-phase-1-foundation-design.md`
**Requirements:** FOUND-01…06, AUTH-01…05 in `.planning/REQUIREMENTS.md`

## Global Constraints

Every task's requirements implicitly include this section.

- **Branch:** all work on `develop`. Never commit to `main` — it serves the legacy `index.html` the shop still uses.
- **TypeScript must be installed as `"typescript": "npm:@typescript/typescript6@6.0.2"`.** `typescript@latest` is 7.0.2 and Next.js 16.2 cannot detect it. Do not "fix" this to `typescript@latest`.
- **The file is `proxy.ts`, not `middleware.ts`.** Next.js 16 renamed the convention; `proxy` runs on the Node.js runtime and cannot be configured to edge.
- **Use `<Show when="signed-in">`, never `<SignedIn>` / `<SignedOut>` / `<Protect>`.** Those were removed in Clerk v7 (Core 3). Any generated code using them is wrong.
- **`await auth.protect()`**, not `auth().protect()`. `auth()` is async and `protect` is a property on `auth`. Clerk's own docs contain one stale snippet of the wrong form.
- **Never create a `NEXT_PUBLIC_SUPABASE_*` variable.** The browser holds no durable Supabase credential. `guard:bundle` fails the build if one appears.
- **The Supabase client must be a lazy function, never a module-scope `const`.** A module-scope `createClient(...)` evaluates at build time and bakes in env vars.
- **Money is always integer cents.** Never floats.
- **`supabase-js` returns `{data, error}` and does not throw.** Every call goes through `must()`.
- **Node >= 20.9.0** (required by both `next` and `@clerk/nextjs`).
- Package manager: **npm** (repo has no lockfile yet; npm is the default `create-next-app` uses).

## File Structure

| Path | Responsibility |
|---|---|
| `proxy.ts` | Bare `clerkMiddleware()`. Protects nothing by design. |
| `app/layout.tsx` | Root layout, `<ClerkProvider>` inside `<body>` |
| `app/page.tsx` | Public landing placeholder |
| `app/dashboard/layout.tsx` | Server-side auth gate for every dashboard route |
| `app/dashboard/page.tsx` | Empty board shell |
| `lib/env.ts` | Zod-validated, fail-fast server env access |
| `lib/db/must.ts` | Unwraps `{data, error}`, throws on error |
| `lib/db/client.ts` | Lazy `server-only` Supabase admin client |
| `lib/auth/role.ts` | `getRole()` — the only place role is resolved |
| `lib/pricing/sentinel.ts` | `server-only` sentinel proving the price list never ships |
| `actions/dashboard/` | Server Actions requiring `auth.protect()` |
| `actions/public/` | Server Actions reachable without auth |
| `scripts/guard-*.mjs` | Build-time guards |
| `supabase/migrations/0001_init.sql` | Complete schema |
| `legacy/index.html` | The prototype, kept as domain reference until cutover |

---

### Task 1: Scaffold Next.js and preserve the legacy prototype

**Files:**
- Create: `package.json`, `tsconfig.json`, `next.config.ts`, `app/layout.tsx`, `app/page.tsx`, `app/globals.css`, `vitest.config.ts`
- Move: `index.html` → `legacy/index.html`; delete four 2-byte junk files
- Test: `tests/smoke.test.ts`

**Interfaces:**
- Consumes: nothing (first task)
- Produces: a working Next.js app with `npm run build`, `npm test`, and the exact dependency versions later tasks assume

- [ ] **Step 1: Move legacy files aside**

`create-next-app` refuses to scaffold into a directory containing unrecognized files. Clear them first. The prototype is kept — its domain modelling (stage names, size breakdowns, stitch counts, rush logic) came from the shop owner and is the reference for later phases.

```bash
mkdir -p legacy
git mv index.html legacy/index.html
git rm -q "hussle" "hussle ready 2" "index.html. (2).txt" "index.html..txt"
git commit -m "chore: move prototype to legacy/, drop empty junk files"
```

- [ ] **Step 2: Scaffold into a temp directory and hoist**

Scaffolding directly into the repo root still trips on `.planning/`, `docs/`, and the markdown files. Scaffold into a temp subdirectory, then move everything up.

```bash
npx create-next-app@latest .scaffold-tmp \
  --typescript --tailwind --eslint --app --src-dir=false \
  --import-alias "@/*" --use-npm --no-turbopack --yes

# hoist, including dotfiles, without clobbering our .gitignore or env files
mv .scaffold-tmp/app .scaffold-tmp/public . 2>/dev/null || true
mv .scaffold-tmp/package.json .scaffold-tmp/tsconfig.json .scaffold-tmp/next.config.ts .
mv .scaffold-tmp/postcss.config.mjs .scaffold-tmp/eslint.config.mjs .
rm -f .scaffold-tmp/.gitignore
rm -rf .scaffold-tmp
```

- [ ] **Step 3: Pin the exact dependency set**

The scaffold's defaults are wrong in two places: TypeScript 7 is undetectable by Next 16.2, and we need the server-only guard package.

```bash
npm pkg set dependencies.next="16.2.12"
npm pkg set dependencies.react="19.2.8"
npm pkg set dependencies.react-dom="19.2.8"
npm pkg set dependencies."@clerk/nextjs"="7.6.3"
npm pkg set dependencies."@supabase/supabase-js"="2.111.0"
npm pkg set dependencies.zod="4.4.3"
npm pkg set dependencies.resend="6.18.1"
npm pkg set dependencies."server-only"="0.0.1"

npm pkg set devDependencies.typescript="npm:@typescript/typescript6@6.0.2"
npm pkg set devDependencies."@types/react"="19.2.17"
npm pkg set devDependencies."@types/node"="26.1.2"
npm pkg set devDependencies.vitest="latest"
npm pkg set devDependencies.dotenv="latest"

npm install
```

- [ ] **Step 4: Add scripts and the Vitest config**

`prebuild` is the enforcement point — it runs before every `next build`, including on Vercel, so a guard failure means nothing deploys. The guards are added in Task 5; the script entries are stubbed to `true` now so the build works until then.

```bash
npm pkg set scripts.test="vitest run"
npm pkg set scripts.typecheck="tsc --noEmit"
```

`prebuild` / `postbuild` are deliberately **not** set here. Wiring them to scripts that do not exist yet would break `npm run build` for every task until Task 5. Task 5 adds both, along with the guards themselves.

Create `vitest.config.ts`:

```typescript
import { defineConfig } from 'vitest/config'
import path from 'node:path'

export default defineConfig({
  test: {
    environment: 'node',
    include: ['tests/**/*.test.ts'],
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, '.') },
  },
})
```

- [ ] **Step 5: Write the failing smoke test**

Create `tests/smoke.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import pkg from '../package.json'

describe('scaffold', () => {
  it('pins next to the version the plan assumes', () => {
    expect(pkg.dependencies.next).toBe('16.2.12')
  })

  it('uses the TypeScript 6 alias, not typescript@7 which Next 16.2 cannot detect', () => {
    expect(pkg.devDependencies.typescript).toBe('npm:@typescript/typescript6@6.0.2')
  })

  it('runs guards before build so a violation cannot deploy', () => {
    expect(pkg.scripts.prebuild).toContain('guard:')
  })
})
```

- [ ] **Step 6: Run the test**

Run: `npm test`
Expected: PASS — first and third assertions. The `prebuild` assertion FAILS, because Step 4 deliberately did not set it. Change that assertion to `expect(pkg.scripts.prebuild).toBeUndefined()` for now; Task 5 Step 8 flips it back once the guards exist. This is the test correctly tracking reality rather than aspiration.

- [ ] **Step 7: Verify the build works**

Run: `npm run build`
Expected: build succeeds cleanly. No guards run yet — that is expected at this stage.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: scaffold Next.js 16 app with pinned dependency set"
```

---

### Task 2: Fail-fast environment validation

**Files:**
- Create: `lib/env.ts`
- Test: `tests/env.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces: `serverEnv(): ServerEnv` — throws on missing/invalid vars. `ServerEnv` has `SUPABASE_URL: string`, `SUPABASE_SECRET_KEY: string`, `MAX_ARTWORK_BYTES: number`, `CLERK_SECRET_KEY: string`, `RESEND_API_KEY: string | undefined`, `QUOTE_NOTIFICATION_TO: string | undefined`. Tasks 3, 7, and 8 call `serverEnv()`.

- [ ] **Step 1: Write the failing test**

Create `tests/env.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest'

const VALID = {
  SUPABASE_URL: 'https://example.supabase.co',
  SUPABASE_SECRET_KEY: 'sb_secret_abc',
  CLERK_SECRET_KEY: 'sk_test_abc',
  MAX_ARTWORK_BYTES: '52428800',
}

let saved: NodeJS.ProcessEnv

beforeEach(() => {
  saved = { ...process.env }
  for (const k of Object.keys(VALID)) delete process.env[k]
})
afterEach(() => {
  process.env = saved
})

describe('serverEnv', () => {
  it('returns parsed values when everything is present', async () => {
    Object.assign(process.env, VALID)
    const { serverEnv } = await import('@/lib/env')
    const env = serverEnv()
    expect(env.SUPABASE_URL).toBe('https://example.supabase.co')
    expect(env.MAX_ARTWORK_BYTES).toBe(52428800)
  })

  it('throws naming the missing variable rather than failing later at a call site', async () => {
    Object.assign(process.env, VALID)
    delete process.env.SUPABASE_SECRET_KEY
    const { serverEnv } = await import('@/lib/env')
    expect(() => serverEnv()).toThrow(/SUPABASE_SECRET_KEY/)
  })

  it('rejects a non-numeric size cap instead of coercing it to NaN', async () => {
    Object.assign(process.env, VALID, { MAX_ARTWORK_BYTES: 'fifty megabytes' })
    const { serverEnv } = await import('@/lib/env')
    expect(() => serverEnv()).toThrow(/MAX_ARTWORK_BYTES/)
  })
})
```

- [ ] **Step 2: Run it to confirm it fails**

Run: `npm test -- tests/env.test.ts`
Expected: FAIL — cannot resolve `@/lib/env`.

- [ ] **Step 3: Implement**

Create `lib/env.ts`:

```typescript
import 'server-only'
import { z } from 'zod'

// Zod 4: format validators are top-level (z.url(), not z.string().url()),
// and the error option is `error`, not `message`.
const schema = z.object({
  SUPABASE_URL: z.url({ error: 'SUPABASE_URL must be a valid URL' }),
  SUPABASE_SECRET_KEY: z.string().min(1),
  CLERK_SECRET_KEY: z.string().min(1),
  MAX_ARTWORK_BYTES: z.coerce.number().int().positive(),
  RESEND_API_KEY: z.string().min(1).optional(),
  QUOTE_NOTIFICATION_TO: z.string().min(1).optional(),
})

export type ServerEnv = z.infer<typeof schema>

// Not cached: tests mutate process.env between cases, and this runs once per
// request path at most. Caching would also defeat runtime env reads on Vercel.
export function serverEnv(): ServerEnv {
  const parsed = schema.safeParse(process.env)
  if (!parsed.success) {
    const detail = parsed.error.issues
      .map((i) => `${i.path.join('.')}: ${i.message}`)
      .join('; ')
    throw new Error(`Invalid server environment — ${detail}`)
  }
  return parsed.data
}
```

- [ ] **Step 4: Run the tests**

Run: `npm test -- tests/env.test.ts`
Expected: PASS, all three.

- [ ] **Step 5: Commit**

```bash
git add lib/env.ts tests/env.test.ts
git commit -m "feat: fail-fast environment validation (FOUND-04)"
```

---

### Task 3: Data access layer — `must()` and the lazy admin client

**Files:**
- Create: `lib/db/must.ts`, `lib/db/client.ts`
- Test: `tests/must.test.ts`

**Interfaces:**
- Consumes: `serverEnv()` from Task 2
- Produces: `must<T>(result: { data: T | null; error: { message: string } | null }, context: string): T` — throws on error or null data, returns data otherwise. `db()` returns a cached `SupabaseClient`. Every later task that touches Postgres calls `must(await db().from(...)...)`.

**Why this exists:** `supabase-js` returns `{data, error}` and does **not** throw. `ASSESSMENT.md` §2.4 — failed writes rendering a success screen to the customer — is one missing destructure away from recurring. `must()` converts silence into a thrown error; `guard:dal` (Task 5) enforces that nothing bypasses it.

- [ ] **Step 1: Write the failing test**

Create `tests/must.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { must } from '@/lib/db/must'

describe('must', () => {
  it('returns data when the call succeeded', () => {
    expect(must({ data: [{ id: 1 }], error: null }, 'quotes.select')).toEqual([{ id: 1 }])
  })

  it('throws when supabase reports an error instead of returning silently', () => {
    expect(() =>
      must({ data: null, error: { message: 'permission denied' } }, 'quotes.insert')
    ).toThrow(/quotes\.insert.*permission denied/)
  })

  it('throws when data is null without an error — the silent-failure case', () => {
    expect(() => must({ data: null, error: null }, 'quotes.selectOne')).toThrow(
      /quotes\.selectOne/
    )
  })

  it('allows an empty array, which is a legitimate result not a failure', () => {
    expect(must({ data: [], error: null }, 'quotes.select')).toEqual([])
  })
})
```

- [ ] **Step 2: Run it to confirm it fails**

Run: `npm test -- tests/must.test.ts`
Expected: FAIL — cannot resolve `@/lib/db/must`.

- [ ] **Step 3: Implement `must`**

Create `lib/db/must.ts`:

```typescript
export type SupabaseResult<T> = {
  data: T | null
  error: { message: string } | null
}

/**
 * Unwraps a supabase-js result, throwing on failure.
 *
 * supabase-js returns {data, error} and never throws. Calling code that forgets
 * to check `error` proceeds as if the write succeeded — which is exactly how the
 * prototype told customers their quote was saved when it was not
 * (ASSESSMENT.md 2.4). Routing every call through here makes that impossible.
 */
export function must<T>(result: SupabaseResult<T>, context: string): T {
  if (result.error) {
    throw new Error(`Database call failed [${context}]: ${result.error.message}`)
  }
  if (result.data === null || result.data === undefined) {
    throw new Error(`Database call returned no data [${context}]`)
  }
  return result.data
}
```

- [ ] **Step 4: Run the tests**

Run: `npm test -- tests/must.test.ts`
Expected: PASS, all four.

- [ ] **Step 5: Implement the lazy admin client**

Create `lib/db/client.ts`:

```typescript
import 'server-only'
import { createClient, type SupabaseClient } from '@supabase/supabase-js'
import { serverEnv } from '@/lib/env'

let cached: SupabaseClient | null = null

/**
 * Supabase client authenticated with the secret key. Server-only.
 *
 * Deliberately a lazy function, never a module-scope const: a top-level
 * createClient() evaluates during `next build`, baking in whatever env vars
 * exist at build time and failing confusingly when one is absent.
 *
 * The three auth flags are off because there is no browser session here.
 * Without them supabase-js starts session-persistence machinery that is
 * meaningless on a server and can leak state across requests on a warm
 * serverless instance.
 */
export function db(): SupabaseClient {
  if (cached) return cached
  const env = serverEnv()
  cached = createClient(env.SUPABASE_URL, env.SUPABASE_SECRET_KEY, {
    auth: {
      persistSession: false,
      autoRefreshToken: false,
      detectSessionInUrl: false,
    },
  })
  return cached
}
```

- [ ] **Step 6: Verify it typechecks**

Run: `npm run typecheck`
Expected: no errors.

- [ ] **Step 7: Commit**

```bash
git add lib/db tests/must.test.ts
git commit -m "feat: data access layer with must() unwrapper (FOUND-02)"
```

---

### Task 4: Complete database schema in one migration

**Files:**
- Create: `supabase/migrations/0001_init.sql`
- Test: applied against the live project and verified by query

**Interfaces:**
- Consumes: nothing
- Produces: tables `quotes`, `quote_line_items`, `artwork_files`, `quote_events`. Later phases read these column names directly.

**Why everything ships now:** `quote_events`, `deleted_at`, and `actor_id` are load-bearing from the first query written, even though their UIs land in Phases 5 and 7. A quote created before `quote_events` exists has no creation event, so its history starts mid-story and time-in-stage has no `t=0`.

- [ ] **Step 1: Write the migration**

Create `supabase/migrations/0001_init.sql`:

```sql
-- Phase 1 schema. Complete on purpose: audit and soft-delete columns are
-- load-bearing from the first query even though their UI lands much later.

create extension if not exists pgcrypto;

-- Human-readable job number. The prototype used "HPC-" + random(1000..9999),
-- which collides roughly 50% of the time by the 112th quote; two jobs sharing a
-- number means the wrong job gets pulled in the shop.
create sequence if not exists quote_number_seq start 1001;

create table public.quotes (
  id                uuid primary key default gen_random_uuid(),
  quote_number      integer not null unique default nextval('quote_number_seq'),

  customer_name     text not null,
  customer_phone    text not null,
  customer_email    text not null,

  due_date          date,
  is_rush           boolean not null default false,
  notes             text,

  -- text + CHECK rather than a PG enum, so altering the stage set stays a
  -- one-line migration. Three stages: the shop's existing board.
  stage             text not null default 'pending'
                      check (stage in ('pending','production','ready')),
  stage_changed_at  timestamptz not null default now(),

  -- Outreach is a flag on a quote, never a second table. The prototype's
  -- duplicate-lead bug came from splitting app_orders / app_manual_leads.
  needs_outreach    boolean not null default true,
  approved_at       timestamptz,
  approved_by       text,
  contacted_at      timestamptz,
  contacted_by      text,

  -- Money is integer cents, always. The prototype used floats.
  -- Written only by the pricing engine (Phase 4).
  subtotal_cents    integer not null default 0 check (subtotal_cents  >= 0),
  setup_fee_cents   integer not null default 0 check (setup_fee_cents >= 0),
  rush_fee_cents    integer not null default 0 check (rush_fee_cents  >= 0),
  total_cents       integer not null default 0 check (total_cents     >= 0),
  invoice_sent      boolean not null default false,
  is_paid           boolean not null default false,

  source            text not null default 'customer'
                      check (source in ('customer','manual')),
  created_by        text,

  created_at        timestamptz not null default now(),
  updated_at        timestamptz not null default now(),
  deleted_at        timestamptz
);

-- Relational, not JSONB. Two reasons: artwork needs a real FK to a specific
-- line (you cannot FK to an element inside a JSONB array), and the prototype's
-- loop at index.html:739 wrapped both inserts so a 3-garment quote produced
-- 6 rows. Only a parent/child relationship fixes that.
create table public.quote_line_items (
  id               uuid primary key default gen_random_uuid(),
  quote_id         uuid not null references public.quotes(id) on delete cascade,
  position         integer not null default 1,

  decoration_type  text not null
                     check (decoration_type in ('screen_print','embroidery','marketing')),
  brand            text,
  garment_color    text,
  front_colors     integer not null default 0 check (front_colors between 0 and 8),
  back_colors      integer not null default 0 check (back_colors  between 0 and 8),
  print_locations  jsonb   not null default '[]'::jsonb,
  stitch_count     integer check (stitch_count >= 0),
  logo_on_file     boolean not null default false,

  -- JSONB is right here: ~12 sparse keys, always read and written as a unit,
  -- never queried individually, legitimately shaped differently for garments
  -- vs hats vs flat goods.
  sizes            jsonb   not null default '{}'::jsonb,
  total_quantity   integer not null default 0 check (total_quantity >= 0),

  line_total_cents integer not null default 0 check (line_total_cents >= 0),

  created_at       timestamptz not null default now(),
  updated_at       timestamptz not null default now()
);

create table public.artwork_files (
  id             uuid primary key default gen_random_uuid(),
  line_item_id   uuid not null references public.quote_line_items(id) on delete cascade,
  storage_path   text not null unique,
  original_name  text not null,
  mime_type      text not null,
  byte_size      bigint,
  status         text not null default 'pending'
                   check (status in ('pending','uploaded','failed')),
  uploaded_at    timestamptz,
  created_at     timestamptz not null default now(),

  -- A row cannot claim 'uploaded' without server-verified evidence. byte_size is
  -- written from a server-side check of the stored object, never the client's
  -- word. This is the structural version of "no false success".
  constraint artwork_uploaded_requires_evidence
    check (status <> 'uploaded' or (uploaded_at is not null and byte_size is not null))
);

create table public.quote_events (
  id          bigserial primary key,
  quote_id    uuid not null references public.quotes(id) on delete cascade,
  action      text not null check (action in (
                'quote.created','quote.updated','quote.deleted','quote.restored',
                'stage.changed','line_item.updated','lead.approved',
                'lead.contacted','payment.updated','artwork.attached')),
  actor_id    text,
  -- Snapshotted at write time so rendering history needs no Clerk API call,
  -- which counts against Backend API rate limits.
  actor_name  text,
  before      jsonb,
  after       jsonb,
  created_at  timestamptz not null default now()
);

-- Partial indexes: every board query filters deleted_at is null.
create index quotes_stage_idx     on public.quotes(stage)          where deleted_at is null;
create index quotes_due_date_idx  on public.quotes(due_date)       where deleted_at is null;
create index quotes_outreach_idx  on public.quotes(needs_outreach) where deleted_at is null;
create index line_items_quote_idx on public.quote_line_items(quote_id);
create index artwork_line_idx     on public.artwork_files(line_item_id);
create index events_quote_idx     on public.quote_events(quote_id, created_at desc);

-- RLS default-deny, enabled EXPLICITLY here rather than by a hidden event
-- trigger. The prior project had an `ensure_rls` DDL hook doing this invisibly;
-- it was removed during teardown. No policies are created — nothing but the
-- secret key reads or writes these tables.
alter table public.quotes           enable row level security;
alter table public.quote_line_items enable row level security;
alter table public.artwork_files    enable row level security;
alter table public.quote_events     enable row level security;
```

- [ ] **Step 2: Apply the migration**

Apply via the `printhub` Supabase MCP `apply_migration` tool with name `init`, or paste into the SQL editor. The project is empty — zero tables, zero migrations — so this applies to a clean database.

- [ ] **Step 3: Verify the schema landed correctly**

Run this query (MCP `execute_sql`):

```sql
select
  (select count(*) from information_schema.tables
     where table_schema='public'
       and table_name in ('quotes','quote_line_items','artwork_files','quote_events')) as tables,
  (select count(*) from pg_tables
     where schemaname='public' and rowsecurity = true)                                  as rls_enabled,
  (select count(*) from pg_policies where schemaname='public')                          as policies;
```

Expected: `tables=4`, `rls_enabled=4`, `policies=0`. Four tables, RLS on every one, and zero policies — default-deny.

- [ ] **Step 4: Verify the false-success constraint actually fires**

```sql
-- Must FAIL with a check-constraint violation.
insert into public.quotes (customer_name, customer_phone, customer_email)
  values ('Constraint Test','555','t@example.com');
insert into public.quote_line_items (quote_id, decoration_type)
  select id, 'screen_print' from public.quotes where customer_name='Constraint Test';
insert into public.artwork_files (line_item_id, storage_path, original_name, mime_type, status)
  select li.id, 'probe/x.ai', 'x.ai', 'application/postscript', 'uploaded'
  from public.quote_line_items li
  join public.quotes q on q.id = li.quote_id
  where q.customer_name = 'Constraint Test';
```

Expected: the third insert FAILS on `artwork_uploaded_requires_evidence`. A guard that has never been seen firing is indistinguishable from a broken one.

- [ ] **Step 5: Clean up the test rows**

```sql
delete from public.quotes where customer_name = 'Constraint Test';
select count(*) as should_be_zero from public.quotes;
```

- [ ] **Step 6: Commit**

```bash
git add supabase/migrations/0001_init.sql
git commit -m "feat: complete schema in first migration (FOUND-01)"
```

---

### Task 5: Build guards that fail the build

**Files:**
- Create: `scripts/guard-dal.mjs`, `scripts/guard-actions.mjs`, `scripts/guard-bundle.mjs`, `lib/pricing/sentinel.ts`
- Test: `tests/guards.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces: three executables that exit non-zero on violation. `HPH_PRICELIST_SENTINEL` is exported from `lib/pricing/sentinel.ts` and imported by the Phase 4 price list.

**Why a sentinel and not price numbers:** grepping a minified bundle for values like `12.75` false-positives constantly — those digits occur naturally in minified output. A sentinel string travels with the module, which catches the case `server-only` misses: a shared barrel file re-exporting the price list into a client component's import graph.

- [ ] **Step 1: Create the sentinel**

Create `lib/pricing/sentinel.ts`:

```typescript
import 'server-only'

/**
 * If this string ever appears in the client bundle, the price list reached the
 * browser and FR-5's tamper-proofing is void. guard:bundle fails the build on it.
 *
 * Phase 4's rate card must import this so the two travel together.
 */
export const HPH_PRICELIST_SENTINEL = 'hph_pricelist_v1_must_never_ship_to_browser'
```

- [ ] **Step 2: Write the failing guard tests**

Create `tests/guards.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { execFileSync } from 'node:child_process'
import { writeFileSync, rmSync, mkdirSync } from 'node:fs'

function runGuard(script: string): { code: number; out: string } {
  try {
    const out = execFileSync('node', [`scripts/${script}`], { encoding: 'utf8' })
    return { code: 0, out }
  } catch (e: any) {
    return { code: e.status ?? 1, out: `${e.stdout ?? ''}${e.stderr ?? ''}` }
  }
}

describe('guard:dal', () => {
  it('passes on a clean tree', () => {
    expect(runGuard('guard-dal.mjs').code).toBe(0)
  })

  it('fails when db.from( is used outside lib/db/', () => {
    mkdirSync('actions/public', { recursive: true })
    writeFileSync('actions/public/__violation.ts', `const x = db().from('quotes')\n`)
    try {
      const r = runGuard('guard-dal.mjs')
      expect(r.code).not.toBe(0)
      expect(r.out).toMatch(/__violation/)
    } finally {
      rmSync('actions/public/__violation.ts', { force: true })
    }
  })
})

describe('guard:actions', () => {
  it('passes on a clean tree', () => {
    expect(runGuard('guard-actions.mjs').code).toBe(0)
  })

  it('fails when a dashboard action omits auth.protect()', () => {
    mkdirSync('actions/dashboard', { recursive: true })
    writeFileSync(
      'actions/dashboard/__violation.ts',
      `'use server'\nexport async function unguarded() { return 1 }\n`
    )
    try {
      const r = runGuard('guard-actions.mjs')
      expect(r.code).not.toBe(0)
      expect(r.out).toMatch(/__violation/)
    } finally {
      rmSync('actions/dashboard/__violation.ts', { force: true })
    }
  })
})
```

- [ ] **Step 3: Run to confirm they fail**

Run: `npm test -- tests/guards.test.ts`
Expected: FAIL — guard scripts do not exist.

- [ ] **Step 4: Implement `guard:dal`**

Create `scripts/guard-dal.mjs`:

```javascript
#!/usr/bin/env node
// Every database call must route through lib/db/ so it passes through must().
// supabase-js returns {data, error} and never throws; a raw db().from() outside
// the DAL is one missing destructure away from reviving ASSESSMENT.md 2.4.

import { readdirSync, readFileSync, statSync } from 'node:fs'
import { join } from 'node:path'

const ROOTS = ['app', 'actions', 'lib', 'components']
const ALLOWED_PREFIX = join('lib', 'db')
// Covers Postgres (.from) AND Storage (.storage). Storage calls return the same
// {data, error} shape and carry the identical silent-failure risk, so they route
// through lib/db/ too. A guard that covers one and not the other is theatre.
const PATTERN = /\bdb\(\)\s*\.\s*from\s*\(|\bsupabase\s*\.\s*from\s*\(|\bdb\(\)\s*\.\s*storage\b/

const violations = []

function walk(dir) {
  let entries
  try {
    entries = readdirSync(dir)
  } catch {
    return
  }
  for (const entry of entries) {
    const full = join(dir, entry)
    if (statSync(full).isDirectory()) {
      if (entry === 'node_modules' || entry === '.next') continue
      walk(full)
    } else if (/\.(ts|tsx|js|jsx|mjs)$/.test(entry)) {
      if (full.startsWith(ALLOWED_PREFIX)) continue
      const src = readFileSync(full, 'utf8')
      if (PATTERN.test(src)) violations.push(full)
    }
  }
}

for (const root of ROOTS) walk(root)

if (violations.length) {
  console.error('guard:dal FAILED — database access outside lib/db/:\n')
  for (const v of violations) console.error(`  ${v}`)
  console.error('\nRoute the call through lib/db/ so it passes through must().')
  process.exit(1)
}
console.log('guard:dal OK')
```

- [ ] **Step 5: Implement `guard:actions`**

Create `scripts/guard-actions.mjs`:

```javascript
#!/usr/bin/env node
// Clerk middleware CANNOT protect Server Actions: actions are POST endpoints
// dispatched by action ID, and middleware matches paths. Every dashboard action
// must guard itself. Clerk ships a migration guide away from createRouteMatcher
// for exactly this reason.

import { readdirSync, readFileSync, statSync } from 'node:fs'
import { join } from 'node:path'

const DIR = join('actions', 'dashboard')
const violations = []

function walk(dir) {
  let entries
  try {
    entries = readdirSync(dir)
  } catch {
    return
  }
  for (const entry of entries) {
    const full = join(dir, entry)
    if (statSync(full).isDirectory()) {
      walk(full)
    } else if (/\.(ts|tsx)$/.test(entry)) {
      const src = readFileSync(full, 'utf8')
      if (!src.includes('auth.protect()')) violations.push(full)
      if (/\bauth\(\)\s*\.\s*protect\s*\(/.test(src)) {
        violations.push(`${full} (uses auth().protect() — v5 form; use await auth.protect())`)
      }
    }
  }
}

walk(DIR)

if (violations.length) {
  console.error('guard:actions FAILED — unguarded dashboard Server Actions:\n')
  for (const v of violations) console.error(`  ${v}`)
  console.error('\nAdd `await auth.protect()` as the first statement.')
  process.exit(1)
}
console.log('guard:actions OK')
```

- [ ] **Step 6: Implement `guard:bundle`**

Create `scripts/guard-bundle.mjs`:

```javascript
#!/usr/bin/env node
// Post-build: nothing secret may reach the browser bundle.
// server-only catches direct imports at build time; this catches indirect ones,
// e.g. a shared barrel re-exporting the price list into a client component.

import { readdirSync, readFileSync, statSync, existsSync } from 'node:fs'
import { join } from 'node:path'

const BUNDLE = join('.next', 'static')

const FORBIDDEN = [
  ['price-list sentinel', 'hph_pricelist_v1_must_never_ship_to_browser'],
  ['Supabase secret key', 'sb_secret_'],
  ['Supabase legacy service JWT', 'eyJhbGciOiJIUzI1NiIs'],
  ['Clerk secret key (live)', 'sk_live_'],
  ['Clerk secret key (test)', 'sk_test_'],
  ['Resend API key', 're_'],
]

if (!existsSync(BUNDLE)) {
  console.error(`guard:bundle FAILED — ${BUNDLE} not found. Run after next build.`)
  process.exit(1)
}

const hits = []

function walk(dir) {
  for (const entry of readdirSync(dir)) {
    const full = join(dir, entry)
    if (statSync(full).isDirectory()) walk(full)
    else if (/\.(js|mjs|json|css|map)$/.test(entry)) {
      const src = readFileSync(full, 'utf8')
      for (const [label, needle] of FORBIDDEN) {
        // `re_` is short enough to false-positive; require key-ish length.
        const re = needle === 're_' ? /\bre_[A-Za-z0-9]{16,}/ : null
        const found = re ? re.test(src) : src.includes(needle)
        if (found) hits.push(`${label} in ${full}`)
      }
    }
  }
}

walk(BUNDLE)

if (hits.length) {
  console.error('guard:bundle FAILED — secrets or price data in the client bundle:\n')
  for (const h of [...new Set(hits)]) console.error(`  ${h}`)
  process.exit(1)
}
console.log('guard:bundle OK')
```

- [ ] **Step 7: Add the storage helper the guard now requires**

Because `guard:dal` covers `.storage` as well, storage access lives in the DAL too. Create `lib/db/storage.ts`:

```typescript
import 'server-only'
import { db } from './client'
import { must } from './must'

/** Mints a signed upload token for one server-chosen path. */
export async function createSignedUpload(bucket: string, path: string) {
  return must(
    await db().storage.from(bucket).createSignedUploadUrl(path),
    'storage.createSignedUploadUrl'
  )
}
```

Only this one function. Phase 3 will need a server-side object-stat helper for verify-then-commit, but Phase 1 does not — the spike verifies by SQL query. Adding it now would be dead code.

- [ ] **Step 8: Add a sentinel-drift test**

`guard-bundle.mjs` is plain Node and cannot import the TypeScript sentinel, so the string exists in two places. If they drift apart the guard silently stops protecting anything — the worst possible failure for a security check. Append to `tests/guards.test.ts`:

```typescript
import { readFileSync } from 'node:fs'
import { HPH_PRICELIST_SENTINEL } from '@/lib/pricing/sentinel'

describe('sentinel drift', () => {
  it('guard-bundle.mjs greps for the exact string the sentinel module exports', () => {
    const guardSrc = readFileSync('scripts/guard-bundle.mjs', 'utf8')
    expect(guardSrc).toContain(HPH_PRICELIST_SENTINEL)
  })
})
```

- [ ] **Step 9: Wire the guards into the build**

```bash
npm pkg set scripts."guard:dal"="node scripts/guard-dal.mjs"
npm pkg set scripts."guard:actions"="node scripts/guard-actions.mjs"
npm pkg set scripts."guard:bundle"="node scripts/guard-bundle.mjs"
npm pkg set scripts.prebuild="npm run guard:dal && npm run guard:actions"
npm pkg set scripts.postbuild="npm run guard:bundle"
```

Now restore the smoke-test assertion deferred in Task 1 Step 6:

```typescript
it('runs guards before build so a violation cannot deploy', () => {
  expect(pkg.scripts.prebuild).toContain('guard:')
})
```

- [ ] **Step 10: Run the full test suite**

Run: `npm test`
Expected: PASS. Both violation cases must show the guard exiting non-zero and naming the offending file — a guard nobody has watched fail is indistinguishable from a broken one.

- [ ] **Step 11: Verify the full build path end to end**

Run: `npm run build`
Expected: `guard:dal OK`, `guard:actions OK`, build succeeds, `guard:bundle OK`.

- [ ] **Step 12: Commit**

```bash
git add scripts lib/pricing lib/db/storage.ts tests/guards.test.ts tests/smoke.test.ts package.json
git commit -m "feat: build guards for DAL, action auth, and bundle secrets (FOUND-02, FOUND-03)"
```

---

### Task 6: Clerk authentication

**Files:**
- Create: `proxy.ts`, `app/dashboard/layout.tsx`, `app/dashboard/page.tsx`
- Modify: `app/layout.tsx`

**Requirements:** AUTH-01 (individual accounts), AUTH-02 (dashboard requires auth, returns no data signed out), AUTH-03 (`auth.protect()` at the resource, never a middleware matcher), AUTH-04 (`?admin=true` and triple-click absent).

**Interfaces:**
- Consumes: nothing
- Produces: a protected `/dashboard` route tree. Task 7's `getRole()` relies on `auth()` working, which requires `clerkMiddleware()` to be configured.

**Prerequisite:** `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` and `CLERK_SECRET_KEY` must be populated in `.env.local`. Create the application at `dashboard.clerk.com` first — nothing in this task runs without them.

- [ ] **Step 1: Create `proxy.ts`**

Not `middleware.ts` — Next.js 16 renamed the convention. This deliberately protects nothing; it exists so `auth()` works. Route protection lives on each resource.

Create `proxy.ts` at the repo root:

```typescript
import { clerkMiddleware } from '@clerk/nextjs/server'

// Bare on purpose. No createRouteMatcher, no path-based protection.
// Server Actions are dispatched by action ID and never pass through a path
// matcher, so a matcher here would give the appearance of protection over
// exactly the layer it does not cover. Clerk now ships a migration guide away
// from createRouteMatcher for this reason.
export default clerkMiddleware()

export const config = {
  matcher: [
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    '/__clerk/:path*',
    '/(api|trpc)(.*)',
  ],
}
```

- [ ] **Step 2: Add `ClerkProvider` to the root layout**

Modify `app/layout.tsx`. Keep whatever `className` values the scaffold put on `<html>` and `<body>` — fonts and `antialiased` come from there.

```tsx
import type { Metadata } from 'next'
import { ClerkProvider, SignInButton, Show, UserButton } from '@clerk/nextjs'
import './globals.css'

export const metadata: Metadata = {
  title: 'Hustle Print Hub',
  description: 'Quoting and production tracking for Humble Hussle Print Shop',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <ClerkProvider>
          <header className="flex justify-end gap-3 p-4">
            {/* <Show>, not <SignedIn>/<SignedOut> — those were removed in Clerk v7. */}
            <Show when="signed-out">
              <SignInButton />
            </Show>
            <Show when="signed-in">
              <UserButton />
            </Show>
          </header>
          {children}
        </ClerkProvider>
      </body>
    </html>
  )
}
```

- [ ] **Step 3: Protect the dashboard route tree**

Create `app/dashboard/layout.tsx`:

```tsx
import { auth } from '@clerk/nextjs/server'

export default async function DashboardLayout({
  children,
}: {
  children: React.ReactNode
}) {
  // auth.protect() — property on auth, not a method on its result.
  // `auth().protect()` is the Clerk v5 form and is wrong; one stale snippet in
  // Clerk's own docs still shows it.
  await auth.protect()
  return <section className="p-6">{children}</section>
}
```

- [ ] **Step 4: Create the empty board shell**

Create `app/dashboard/page.tsx`:

```tsx
export default function DashboardPage() {
  return (
    <div>
      <h1 className="text-2xl font-bold">Production Board</h1>
      <p className="mt-2 text-sm text-zinc-500">
        No jobs yet. The quote form lands in Phase 2.
      </p>
    </div>
  )
}
```

- [ ] **Step 5: Verify signed-out access is refused**

Run `npm run dev`, then in a private browser window request `http://localhost:3000/dashboard`.
Expected: redirected to sign-in. **No dashboard content and no data in the HTML source.** View source to confirm — a redirect that still ships the payload is not protection.

- [ ] **Step 6: Verify `?admin=true` grants nothing**

Request `http://localhost:3000/dashboard?admin=true` while signed out.
Expected: identical refusal. Then confirm the prototype's bypasses exist nowhere in application code:

```bash
grep -rn "admin=true\|clickCountTracker\|handleSecretShopClick" app lib actions components 2>/dev/null \
  && echo "VIOLATION — legacy bypass present" || echo "OK — no legacy bypass"
```

Expected: `OK`. (`legacy/index.html` still contains them; that is the archived prototype, not application code.)

- [ ] **Step 7: Verify a signed-in user reaches the dashboard**

Sign up through the header button, then visit `/dashboard`.
Expected: the board shell renders and `<UserButton />` appears.

- [ ] **Step 8: Commit**

```bash
git add proxy.ts app
git commit -m "feat: Clerk auth with resource-level protection (AUTH-01..04)"
```

---

### Task 7: Role model

**Files:**
- Create: `lib/auth/role.ts`
- Test: `tests/role.test.ts`

**Interfaces:**
- Consumes: `auth()` from Clerk (needs Task 6)
- Produces:
  - `type Role = 'owner' | 'staff'`
  - `getRole(): Promise<Role>` — the only place role is resolved
  - `requireOwner(): Promise<void>` — throws unless owner
  - `stripMoneyFields<T>(role: Role, row: T): Partial<T>` — the only place money fields are removed

Phase 4's `serializeQuoteFor` and Phase 5's board both build on these.

**Required Clerk configuration:** `publicMetadata` does **not** appear in `sessionClaims` by default. In the Clerk dashboard, configure a custom session token containing:

```json
{ "metadata": "{{user.public_metadata}}" }
```

Without it `getRole()` always returns `staff` and pricing hides from everyone, which will present as a mysterious bug rather than a missing setting. Metadata changes propagate to the token on roughly a 60-second delay.

- [ ] **Step 1: Write the failing test**

Create `tests/role.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { stripMoneyFields, MONEY_FIELDS } from '@/lib/auth/role'

const row = {
  id: 'q1',
  customer_name: 'Acme',
  stage: 'pending',
  total_cents: 125000,
  subtotal_cents: 100000,
  setup_fee_cents: 15000,
  rush_fee_cents: 10000,
  invoice_sent: true,
  is_paid: false,
}

describe('stripMoneyFields', () => {
  it('gives the owner everything', () => {
    expect(stripMoneyFields('owner', row)).toEqual(row)
  })

  it('removes every money field for staff — omitted, not blanked', () => {
    const out = stripMoneyFields('staff', row) as Record<string, unknown>
    for (const f of MONEY_FIELDS) {
      expect(Object.hasOwn(out, f)).toBe(false)
    }
  })

  it('leaves non-money fields intact for staff', () => {
    const out = stripMoneyFields('staff', row) as Record<string, unknown>
    expect(out.customer_name).toBe('Acme')
    expect(out.stage).toBe('pending')
  })

  it('does not mutate the input row', () => {
    const copy = { ...row }
    stripMoneyFields('staff', row)
    expect(row).toEqual(copy)
  })
})
```

- [ ] **Step 2: Run to confirm it fails**

Run: `npm test -- tests/role.test.ts`
Expected: FAIL — cannot resolve `@/lib/auth/role`.

- [ ] **Step 3: Implement**

Create `lib/auth/role.ts`:

```typescript
import 'server-only'
import { auth } from '@clerk/nextjs/server'

export type Role = 'owner' | 'staff'

/**
 * Fields the staff role must never receive. Widening or narrowing pricing
 * visibility is a one-line change here — that is the entire point of routing
 * every decision through this module.
 */
export const MONEY_FIELDS = [
  'total_cents',
  'subtotal_cents',
  'setup_fee_cents',
  'rush_fee_cents',
  'line_total_cents',
  'invoice_sent',
  'is_paid',
] as const

/**
 * The ONLY place a role is resolved. Reads the custom session claim rather than
 * calling currentUser(), which counts against Clerk's Backend API rate limits.
 *
 * Defaults to 'staff'. Failing closed means a misconfiguration hides pricing
 * rather than exposing it.
 */
export async function getRole(): Promise<Role> {
  const { sessionClaims } = await auth()
  const metadata = (sessionClaims?.metadata ?? {}) as { role?: string }
  return metadata.role === 'owner' ? 'owner' : 'staff'
}

export async function requireOwner(): Promise<void> {
  if ((await getRole()) !== 'owner') {
    throw new Error('Forbidden: owner role required')
  }
}

/**
 * The ONLY place money fields are removed from a payload.
 *
 * Fields are OMITTED, never blanked or hidden in the UI. A conditionally
 * rendered value is still readable in the browser network tab — that is
 * `?admin=true` in a new costume (ASSESSMENT.md 2.2).
 */
export function stripMoneyFields<T extends Record<string, unknown>>(
  role: Role,
  row: T
): Partial<T> {
  if (role === 'owner') return row
  const out: Record<string, unknown> = {}
  for (const [k, v] of Object.entries(row)) {
    if (!(MONEY_FIELDS as readonly string[]).includes(k)) out[k] = v
  }
  return out as Partial<T>
}
```

- [ ] **Step 4: Run the tests**

Run: `npm test -- tests/role.test.ts`
Expected: PASS, all four.

- [ ] **Step 5: Set the owner role in Clerk**

After Josh signs in once, in the Clerk dashboard set his user's `publicMetadata` to:

```json
{ "role": "owner" }
```

Until this is set, nobody resolves as owner and pricing is hidden from all three users.

- [ ] **Step 6: Commit**

```bash
git add lib/auth tests/role.test.ts
git commit -m "feat: owner/staff role model with server-side money stripping (AUTH-05)"
```

---

### Task 8: Prove the upload path from a real browser

**Files:**
- Create: `actions/public/upload-token.ts`, `app/spike/page.tsx`
- Test: manual, against a deployed preview

**Interfaces:**
- Consumes: `db()` from Task 3, `serverEnv()` from Task 2
- Produces: `createUploadToken(filename: string): Promise<{ path: string; token: string; signedUrl: string }>` — Phase 3 reuses this shape.

**What is already known.** The server side was measured before this plan was written: a signed upload token alone — no `apikey`, no `Authorization`, no Supabase session — uploads successfully. 1 MB in 1.6 s, 8 MB in 8.8 s, 25 MB in 37.6 s. 100 MB returns `413`; the project ceiling is 50 MB and raising it needs a paid plan. TUS was tested and rejected: it authenticates but fails at RLS, and would require an `anon` insert policy the standard path does not need.

**The one unknown is CORS.** Node's `fetch` enforces no CORS, so none of the above proves a real browser origin can do it. That is all this task tests.

- [ ] **Step 1: Create the artwork bucket**

Via MCP or the Supabase dashboard, create a **private** bucket named `artwork`. Do not add any storage policies — zero policies plus RLS means only the secret key gets in, which is the point.

- [ ] **Step 2: Write the token-minting action**

Create `actions/public/upload-token.ts`. This lives under `actions/public/` because customers are not authenticated — `guard:actions` therefore does not require `auth.protect()` here, which is correct and deliberate.

```typescript
'use server'

import { createSignedUpload } from '@/lib/db/storage'
import { serverEnv } from '@/lib/env'
import { randomUUID } from 'node:crypto'

const BUCKET = 'artwork'

export async function createUploadToken(filename: string) {
  const ext = filename.includes('.') ? filename.split('.').pop()!.toLowerCase() : 'bin'
  // Object paths are server-chosen UUIDs. The client never picks its own path,
  // so a token cannot be redirected at someone else's object.
  const path = `${new Date().toISOString().slice(0, 10)}/${randomUUID()}.${ext}`

  // Storage access goes through lib/db/ like every other Supabase call —
  // guard:dal enforces it, because storage returns the same {data, error}
  // shape and carries the same silent-failure risk.
  const data = await createSignedUpload(BUCKET, path)

  return { path: data.path, token: data.token, signedUrl: data.signedUrl }
}

export async function maxArtworkBytes(): Promise<number> {
  return serverEnv().MAX_ARTWORK_BYTES
}
```

- [ ] **Step 3: Build the throwaway spike page**

Create `app/spike/page.tsx`. Deliberately minimal — no styling, no validation, no database rows. Delete or absorb it in Phase 3.

```tsx
'use client'

import { useState } from 'react'
import { createUploadToken, maxArtworkBytes } from '@/actions/public/upload-token'

export default function SpikePage() {
  const [log, setLog] = useState<string[]>([])
  const say = (m: string) => setLog((l) => [...l, m])

  async function onFile(e: React.ChangeEvent<HTMLInputElement>) {
    const file = e.target.files?.[0]
    if (!file) return
    setLog([])

    const cap = await maxArtworkBytes()
    say(`file: ${file.name} — ${file.size} bytes (cap ${cap})`)
    if (file.size > cap) return say('REJECTED: over cap')

    const { path, signedUrl } = await createUploadToken(file.name)
    say(`token minted for ${path}`)

    const started = Date.now()
    // No apikey. No Authorization. The signed URL carries its own credential.
    const res = await fetch(signedUrl, {
      method: 'PUT',
      headers: { 'Content-Type': file.type || 'application/octet-stream' },
      body: file,
    })
    const secs = ((Date.now() - started) / 1000).toFixed(1)
    say(res.ok ? `UPLOAD OK in ${secs}s` : `UPLOAD FAILED ${res.status} after ${secs}s`)
  }

  return (
    <main style={{ padding: 24, fontFamily: 'monospace' }}>
      <h1>FOUND-05 upload spike</h1>
      <input type="file" onChange={onFile} />
      <pre>{log.join('\n')}</pre>
    </main>
  )
}
```

- [ ] **Step 4: Deploy a preview**

```bash
git add actions app/spike
git commit -m "spike: browser upload CORS probe (FOUND-05)"
git push
```

Wait for the Vercel preview deploy on `develop`. **`next dev` cannot answer this** — a same-origin local request exercises no CORS at all.

- [ ] **Step 5: Test from the deployed preview**

Open `<preview-url>/spike` in a real browser. Upload a file near the cap (40–50 MB).

Expected: `UPLOAD OK` with an elapsed time. If it fails with a CORS error, that is the finding — record the exact console message and stop; the artwork design needs revisiting before Phase 3.

- [ ] **Step 6: Verify the bytes actually landed**

Query via MCP:

```sql
select name, (metadata->>'size')::bigint as bytes, created_at
from storage.objects
where bucket_id = 'artwork'
order by created_at desc
limit 5;
```

Expected: the object exists and `bytes` matches the file size exactly. The client reporting success is not evidence.

- [ ] **Step 7: Record the timing**

Write the measured elapsed time into the spec's §8 success bar. At the 50 MB cap this is the number Josh needs before deciding on a paid plan — a customer watching a progress bar for over a minute, with no resume if their connection drops, is a product constraint rather than a technical one.

- [ ] **Step 8: Clean up the probe objects**

```sql
delete from storage.objects where bucket_id = 'artwork';
```

- [ ] **Step 9: Commit the finding**

```bash
git add docs/superpowers/specs/2026-07-30-phase-1-foundation-design.md
git commit -m "docs: record measured browser upload result (FOUND-05)"
```

---

### Task 9: Operational setup

**Files:**
- Modify: `.env.example` if any variable was missed
- No application code

**Interfaces:**
- Consumes: nothing
- Produces: a deployed, protected preview environment; Resend domain verification in flight

These steps are configuration in third-party dashboards, not code. They are last because nothing else blocks on them — except Resend, which should be **started first** since DNS propagation is wall-clock time nobody can compress.

- [ ] **Step 1: Start Resend domain verification (do this first)**

In Resend, add the shop's sending domain and publish the DNS records. Verification can take hours. FOUND-06 requires it **submitted**, not complete — Phase 6 needs it finished.

- [ ] **Step 2: Scope environment variables per Vercel environment**

In the Vercel project (`hustle-print-hub-twdk`, which serves `hussle-printshop.vercel.app`), set for **Preview** and **Production** separately:

`SUPABASE_URL`, `SUPABASE_SECRET_KEY`, `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `MAX_ARTWORK_BYTES`, `RESEND_API_KEY`, `QUOTE_NOTIFICATION_TO`

Mark `SUPABASE_SECRET_KEY`, `CLERK_SECRET_KEY`, and `RESEND_API_KEY` **Sensitive**. Never create a `NEXT_PUBLIC_SUPABASE_*` variable — `guard:bundle` fails the build if one appears.

> **Blocked on Josh** granting Vercel team access. Until then the preview deploy uses whatever is already configured, and Task 8 may fail on missing variables rather than on CORS. Distinguish those two failures carefully — they look similar and mean opposite things.

- [ ] **Step 3: Enable deployment protection**

Turn on Vercel deployment protection for preview deployments, so preview URLs are not publicly reachable. Previews will point at the same Supabase project as production during Phase 1 — an unprotected preview is therefore a second unauthenticated door to real data, which is the exact class of problem this phase exists to close.

- [ ] **Step 4: Verify no secret reaches the client bundle**

```bash
npm run build
```

Expected: `guard:bundle OK`. This is FOUND-03's acceptance criterion, run against a real production build.

- [ ] **Step 5: Confirm the Phase 1 definition of done**

- [ ] Team members sign in with individual accounts; signed out, `/dashboard` returns no data
- [ ] `?admin=true` grants nothing and appears nowhere in application code
- [ ] A file at the cap uploads from a browser on a deployed preview
- [ ] Resend domain verification submitted
- [ ] Planting a rate-card constant in a client component fails the build
- [ ] Migrations apply cleanly to an empty database
- [ ] Deployment protection on; env vars scoped per environment

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "chore: Phase 1 operational setup complete (FOUND-04, FOUND-06)"
```
