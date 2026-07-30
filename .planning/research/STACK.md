# Stack Research

**Domain:** Quoting + production-tracking web app for a 3-person screen-print shop
**Researched:** 2026-07-30
**Confidence:** HIGH (versions and API signatures read from npm registry and package source; patterns read from official docs)

---

## Scope of this document

**The stack is locked.** Next.js App Router on Vercel, Clerk, Supabase (Postgres + Storage), Server Actions with a service-role-privilege key, Zod, Resend. See `PRD-hustle-print-hub.md` §7 and `ASSESSMENT.md` §3 for the rationale. **No alternatives are proposed here.**

This document answers *how to implement the locked stack correctly as of July 2026*. Every version was checked against a live registry or official source — none is from training data.

**Read this first if you read nothing else:** three findings below will break the build if missed.

| # | Finding | Section |
|---|---------|---------|
| 1 | `middleware.ts` is renamed `proxy.ts` in Next.js 16, and Clerk now says route protection does **not** belong in it | §3 |
| 2 | FR-1's 12 MB artwork upload **cannot** go through a Server Action — Vercel caps function request bodies at 4.5 MB | §5.2 |
| 3 | Clerk v7 removed `<SignedIn>` / `<SignedOut>` / `<Protect>`. Every blog post and most AI-generated code uses them | §3.5 |

---

## 1. Pinned versions

All versions read from the npm registry via `npm view <pkg> version dist-tags` on **2026-07-30**. Registry `latest` dist-tag is the source unless noted.

### Runtime dependencies

| Package | Version | Purpose | Source / confidence |
|---------|---------|---------|---------------------|
| `next` | **16.2.12** | Framework + Server Actions boundary | npm `latest`, modified 2026-07-30 · HIGH |
| `react` | **19.2.8** | — | npm `latest` · HIGH |
| `react-dom` | **19.2.8** | — | npm `latest` · HIGH |
| `@clerk/nextjs` | **7.6.3** | Auth | npm `latest`, modified 2026-07-30 · HIGH |
| `@supabase/supabase-js` | **2.111.0** | Postgres + Storage client (server-side only) | npm `latest`, modified 2026-07-28 · HIGH |
| `zod` | **4.4.3** | Server-side validation | npm `latest` · HIGH |
| `resend` | **6.18.1** | Transactional email | npm `latest`, modified 2026-07-28 · HIGH |
| `@react-email/components` | **1.0.12** | **Required** if using Resend's `react:` prop — see §6.2 | npm `latest` · HIGH |
| `server-only` | **0.0.1** | Build-time guard on the price list and admin client | npm `latest` (stub package, unchanged since 2022 — this is correct, not stale) · HIGH |

### Dev dependencies

| Package | Version | Notes |
|---------|---------|-------|
| `typescript` | **`npm:@typescript/typescript6@6.0.2`** | See §7.6 — do **not** install plain `typescript@latest` (7.0.2) |
| `@types/react` | **19.2.17** | npm `latest` |
| `@types/react-dom` | **19.2.3** | npm `latest` |
| `tailwindcss` | **4.3.3** | Replaces the CDN script (ASSESSMENT §2.10 / FR P1 #15) |
| `@tailwindcss/postcss` | **4.3.3** | Tailwind v4's PostCSS plugin — separate package since v4 |
| `react-email` | **6.9.1** | Optional: local preview server for email templates |
| `eslint` | **10.8.0** | `next lint` was **removed** in Next 16 — run ESLint directly |

### Engine and peer constraints (verified from package manifests)

| Constraint | Value | Source |
|---|---|---|
| Node.js | **>= 20.9.0** | `next@16.2.12` engines AND `@clerk/nextjs@7.6.3` engines |
| `@clerk/nextjs@7.6.3` peer `next` | `^15.2.8 \|\| ... \|\| ^16.0.10 \|\| ^16.1.0-0` | 16.2.12 satisfies `^16.0.10` ✅ |
| `@clerk/nextjs@7.6.3` peer `react` | `^18.0.0 \|\| ~19.0.3 \|\| ~19.1.4 \|\| ~19.2.3 \|\| ~19.3.0-0` | 19.2.8 satisfies `~19.2.3` ✅ |
| `next@16.2.12` peer `react` | `^18.2.0 \|\| ^19.0.0` | ✅ |
| `resend@6.18.1` peer | `@react-email/render: "*"` marked **optional** | Optional in the manifest, but **mandatory at runtime** if you pass `react:` — see §6.2 |

> **Note on the React peer range.** Clerk pins React with a tilde (`~19.2.3` = `>=19.2.3 <19.3.0`). If React ships 19.3.0 before Clerk widens the range, `npm install react@latest` will produce a peer conflict. Pin React exactly and bump it deliberately.

### Install

```bash
# Core runtime
npm install next@16.2.12 react@19.2.8 react-dom@19.2.8 \
  @clerk/nextjs@7.6.3 \
  @supabase/supabase-js@2.111.0 \
  zod@4.4.3 \
  resend@6.18.1 @react-email/components@1.0.12 \
  server-only@0.0.1

# Dev
npm install -D typescript@npm:@typescript/typescript6@6.0.2 \
  @types/react@19.2.17 @types/react-dom@19.2.3 \
  tailwindcss@4.3.3 @tailwindcss/postcss@4.3.3 \
  eslint@10.8.0 react-email@6.9.1
```

**Do not install:**
- `@supabase/ssr` — that package exists to manage Supabase *auth cookies* in the browser. This architecture has no browser Supabase client. Installing it invites someone to wire one up.
- `@supabase/supabase-js@next` (3.0.0-next.29) — prerelease, not `latest`.
- `zod@3` or the `zod/v4` subpath import — v4 is the default export of `zod@4.x`.
- `@clerk/clerk-react` — renamed to `@clerk/react` in Core 3 (and you don't need it directly; `@clerk/nextjs` depends on it).

---

## 2. Environment variables

### Names and scoping

| Variable | Prefix | Where | Sensitive on Vercel |
|---|---|---|---|
| `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` | `NEXT_PUBLIC_` | Client + server | No (public by design) |
| `CLERK_SECRET_KEY` | none | Server only | **Yes** |
| `CLERK_ENCRYPTION_KEY` | none | Server only | **Yes** — required only if you pass `secretKey` to `clerkMiddleware()` programmatically (Core 3 change) |
| `SUPABASE_URL` | **none** | Server only | No, but keep unprefixed anyway |
| `SUPABASE_SECRET_KEY` | **none** | Server only | **Yes** |
| `RESEND_API_KEY` | none | Server only | **Yes** |
| `RESEND_FROM_ADDRESS` | none | Server only | No |
| `OWNER_NOTIFICATION_EMAIL` | none | Server only | No |
| `NEXT_PUBLIC_APP_URL` | `NEXT_PUBLIC_` | Client + server | No |

**The rule:** in Next.js, only `NEXT_PUBLIC_`-prefixed variables are inlined into the client bundle; everything else is replaced with an empty string in client code ([Next.js docs, "Preventing environment poisoning"](https://nextjs.org/docs/app/getting-started/server-and-client-components)). So the single most important discipline is: **no Supabase variable is ever `NEXT_PUBLIC_`.** Not the URL, not any key.

> ⚠️ **Do not use `NEXT_PUBLIC_SUPABASE_URL`.** It is the near-universal convention in Supabase tutorials and starter templates, and it is wrong for this architecture. Prefixing the URL is harmless on its own but it normalises the prefix, and the next person who adds a key follows the pattern. Confidence: HIGH (this is a design judgement flowing directly from the locked constraint, not a doc claim).

> ⚠️ **If you connect Supabase through the Vercel Marketplace integration**, it auto-injects a set of environment variables into the project, including `NEXT_PUBLIC_`-prefixed ones. Audit the injected list immediately after connecting and delete every `NEXT_PUBLIC_SUPABASE_*` variable. Confidence on the exact injected names: **LOW** (not verified in this pass) — but the action (audit and delete) is correct regardless.

### Vercel specifics (verified from Vercel docs, last updated 2026-06-16 / 2026-06-03)

- **Environments:** Production, Preview, Development, plus custom environments. Preview vars can be scoped to a specific Git branch; branch-specific values override general preview values.
- **Sensitive environment variables:** values become non-readable once created. Available **only for Production and Preview** — you cannot mark a Development-scoped variable sensitive. To convert an existing variable, you must delete and re-add it with the Sensitive toggle on. Vercel redacts sensitive values ≥32 chars from build logs.
- **Local development:** `vercel env pull` writes the **Development** environment into `.env`. Add `.env*` to `.gitignore`.
- **Size:** 64 KB total across all variables; no single variable over 64 KB. Not a constraint here.
- Recommendation: mark `SUPABASE_SECRET_KEY`, `CLERK_SECRET_KEY`, `CLERK_ENCRYPTION_KEY`, and `RESEND_API_KEY` **Sensitive** in Production and Preview. Use a separate Supabase project (or at minimum separate secret keys) for Preview so a preview deploy cannot write to production data.

---

## 3. Clerk + Next.js App Router — the current official setup

### 3.1 The file is `proxy.ts`, not `middleware.ts`

Next.js 16 deprecated the `middleware` filename and renamed the convention to `proxy` ([Next.js v16 upgrade guide](https://nextjs.org/docs/app/guides/upgrading/version-16)). Clerk's own reference now says: *"Create a `proxy.ts` file at your project root or in the `src/` directory."*

Two consequences worth knowing:
- The `proxy` runtime is **`nodejs` and cannot be configured**. The `edge` runtime is not supported in `proxy`. (If you ever needed edge, you'd have to stay on the deprecated `middleware` filename. You don't need edge here.)
- Config flags renamed too: `skipMiddlewareUrlNormalize` → `skipProxyUrlNormalize`.

```ts
// proxy.ts  (project root, or src/proxy.ts)
import { clerkMiddleware } from '@clerk/nextjs/server'

export default clerkMiddleware()

export const config = {
  matcher: [
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    '/(api|trpc)(.*)',
    '/__clerk/(.*)',
  ],
}
```

That matcher is verbatim from Clerk's current `clerk-middleware` reference. Note the third entry, `'/__clerk/(.*)'` — it is newer than most published examples and is required for Clerk's proxied frontend API routes. **Confidence: HIGH** (read from `clerk/clerk-docs` `docs/reference/nextjs/clerk-middleware.mdx`).

> The function name matters: Next 16 also deprecated the *named* export `middleware` in favour of `proxy`. With `export default clerkMiddleware()` you sidestep the naming issue entirely — this is the form Clerk documents.

### 3.2 Route protection does **not** go in the proxy

This is the biggest divergence between the official docs and essentially all published tutorials.

Clerk now ships a migration guide titled *"Migrate away from `createRouteMatcher`"*, and the `clerkMiddleware` reference states plainly:

> **"Middleware is not the best place to protect routes. Instead, protect access as close to the resource as possible, in the code that reads or mutates the data."**

Clerk's stated reasons:
- **Server Functions bypass the proxy entirely.** They are dispatched by action ID, not by URL path. A Server Action defined under `/dashboard/` but invoked from a public page never passes through your `/dashboard` matcher.
- Regex path-matching mistakes silently leave resources open.
- Path-normalisation differences between Clerk and the framework have produced real bypasses (GHSA-vqx2-fgx2-5wq9), plus 2025 framework-level disclosures where requests skipped middleware altogether.

**For this project this is not academic.** FR-5 requires that pricing is unforgeable and FR-2 requires the dashboard be closed. The Server Actions *are* the security boundary (ASSESSMENT §3.4). A middleware matcher would give the appearance of protection over exactly the layer it does not cover.

> ⚠️ **Do not write** `const isProtected = createRouteMatcher(['/dashboard(.*)']); export default clerkMiddleware(async (auth, req) => { if (isProtected(req)) await auth.protect() })`. It is in every tutorial, it is what an LLM will generate, and Clerk is actively migrating people off it. Keep `clerkMiddleware()` bare and put `auth.protect()` in each resource.

### 3.3 Reading the authenticated user inside a Server Action

Two forms, both current. Use `auth.protect()` when you want Clerk to reject; use `auth()` when you want to return a structured error to the UI.

```ts
// app/_actions/move-job.ts
'use server'

import { auth } from '@clerk/nextjs/server'

export async function moveJob(jobId: string, toStage: Stage) {
  // Throws → 401 for unauthenticated requests. No further code runs.
  const { userId } = await auth.protect()

  // ... mutation, attributed to userId
}
```

```ts
// Preferred here: FR-4 requires visible, actionable failure states,
// so return a result object rather than throwing.
'use server'

import { auth } from '@clerk/nextjs/server'

export async function moveJob(jobId: string, toStage: Stage) {
  const { isAuthenticated, userId } = await auth()

  if (!isAuthenticated) {
    return { ok: false as const, error: 'Your session expired. Please sign in again.' }
  }

  // ... mutation, attributed to userId
}
```

**API notes (all verified against `clerk/clerk-docs`):**

- `auth()` is **async** — `await` it. It returns `{ isAuthenticated, userId, sessionId, sessionClaims, orgId, has(), getToken(), redirectToSignIn() }`.
- `auth.protect()` is a **property on `auth`**, not a method on its result. `auth().protect()` is the Clerk v5 form and is wrong. Clerk's own `app-router/auth.mdx` still contains one stale `await auth().protect()` snippet — **the docs are internally inconsistent here.** Use `await auth.protect()`, which is what the v6 upgrade guide, the Core 3 guide, the migration guide, and `protect-content.mdx` all show. **Confidence: HIGH** on `auth.protect()`; flagging the doc inconsistency so nobody "corrects" it back.
- `auth()` requires `clerkMiddleware()` to be configured — that's why you keep `proxy.ts` even though it does no protecting.
- **Core 3 status-code change:** `auth.protect()` now returns **401 Unauthorized** for unauthenticated Server Action requests. It returned **404** in v6. Any client-side `if (error.status === 404)` handling must become `401`.

### 3.4 `currentUser()` — for the audit trail (FR-11)

```ts
'use server'

import { auth, currentUser } from '@clerk/nextjs/server'

export async function moveJob(jobId: string, toStage: Stage) {
  const { isAuthenticated, userId } = await auth()
  if (!isAuthenticated) return { ok: false as const, error: 'Not signed in.' }

  const user = await currentUser()   // Backend User object

  await db.from('audit_log').insert({
    job_id: jobId,
    actor_id: userId,
    actor_name: user?.fullName ?? user?.primaryEmailAddress?.emailAddress ?? userId,
    action: `stage → ${toStage}`,
  })
}
```

⚠️ Clerk's docs warn that **`currentUser()` counts against Backend API rate limits**. FR-11 needs a display name on every stage change. Two mitigations, in order of preference:

1. Add `name` / `email` to the session token via Clerk's custom session claims, then read `sessionClaims` from `auth()` — no Backend API call at all.
2. Store `actor_id` (from `auth()`, free) on the audit row and resolve display names once per dashboard render, not once per write.

**Confidence: HIGH** that the rate-limit warning is in the docs; **MEDIUM** on custom session claims being the best fix here (pattern is standard, not verified against a Clerk doc page in this pass).

### 3.5 `<SignedIn>` / `<SignedOut>` / `<Protect>` are gone — use `<Show>`

Clerk Core 3 (`@clerk/nextjs` v7, released 2026-03-03) replaced all three control components with a single `<Show>`.

```tsx
import { Show, SignInButton, UserButton } from '@clerk/nextjs'

export function Header() {
  return (
    <header>
      <Show when="signed-in">
        <UserButton />
      </Show>
      <Show when="signed-out">
        <SignInButton />
      </Show>
    </header>
  )
}
```

`<Show>` takes a `fallback` prop and works both client- and server-side:

```tsx
<Show when="signed-in" fallback={<p>Please sign in.</p>}>
  <Dashboard />
</Show>
```

| v6 and earlier | v7 (Core 3) |
|---|---|
| `<SignedIn>` | `<Show when="signed-in">` |
| `<SignedOut>` | `<Show when="signed-out">` |
| `<Protect role="admin">` | `<Show when={{ role: 'admin' }}>` |

> ⚠️ This project has flat permissions, so the `role`/`permission` forms are irrelevant. But `<SignedIn>` / `<SignedOut>` will be generated by any tool trained before March 2026. **Confidence: HIGH** — read from Clerk's Core 3 upgrade guide and current component reference.

**`<Show>` is a UI convenience, not a security boundary.** The dashboard is still gated by `auth.protect()` in the page/layout and in every Server Action.

### 3.6 `ClerkProvider` placement

Standard placement (no Cache Components):

```tsx
// app/layout.tsx
import { ClerkProvider } from '@clerk/nextjs'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body>{children}</body>
      </html>
    </ClerkProvider>
  )
}
```

⚠️ **If you ever enable `cacheComponents: true` in `next.config.ts`**, Core 3 requires `ClerkProvider` to move **inside `<body>`**, otherwise you get *"Uncached data was accessed outside of `<Suspense>`"*:

```tsx
<html lang="en">
  <body>
    <ClerkProvider>{children}</ClerkProvider>
  </body>
</html>
```

Recommendation: **do not enable `cacheComponents` for this project.** It is a per-request dashboard with three users; PPR buys nothing and adds a whole class of Suspense-boundary failures. **Confidence: HIGH** on the Clerk requirement; the recommendation is a judgement call.

---

## 4. Supabase server-side client with the service-role key

### 4.1 First: "service role key" now means a **secret key**

The locked decision (`PRD` §7, `ASSESSMENT` §3.4) says "service role key." As of 2026 the correct artifact for a new project is a Supabase **secret key** (`sb_secret_...`), which carries exactly the same privilege — it authenticates as the `service_role` Postgres role, which has `BYPASSRLS`.

- Legacy JWT `anon` / `service_role` keys are **deprecated and scheduled for removal by the end of 2026** ([Supabase docs](https://supabase.com/docs/guides/getting-started/api-keys)).
- New keys are created in **Settings → API Keys**, under the name `default`.
- This is an implementation detail *within* the locked decision, not a change to it. The privilege level, the trust model, and the "browser never holds it" rule are all unchanged.

**Do this on day one of the new project**: create the secret key, use it everywhere, and once nothing references them, deactivate the legacy `anon` and `service_role` keys in the dashboard. There is no reason for this project to ever have a working `anon` key — the browser has no Supabase client to use it.

⚠️ **Header caveat, and why it doesn't bite you:** the new keys are not JWTs and are **rejected in the `Authorization: Bearer` header** — they belong in the `apikey` header. You do not need to handle this manually: `@supabase/supabase-js` **2.111.0** detects new-format keys (`isNewApiKey()` in `src/lib/fetch.ts`) and sets headers accordingly. **Verified by reading the package source.** This is why the version floor matters — pin `>= 2.111.0`. If you ever hand-roll a `fetch` to the Storage or REST API, send `apikey`, not `Authorization`.

### 4.2 The admin client module

```ts
// lib/supabase/admin.ts
import 'server-only'

import { createClient, type SupabaseClient } from '@supabase/supabase-js'
import type { Database } from '@/lib/supabase/types'   // generated by `supabase gen types`

let client: SupabaseClient<Database> | undefined

/**
 * Service-role-privilege Supabase client. Bypasses RLS.
 * MUST only ever be called from a Server Action, Route Handler, or Server Component.
 * The `server-only` import above makes importing this file from a Client Component a build error.
 */
export function supabaseAdmin(): SupabaseClient<Database> {
  if (client) return client

  const url = process.env.SUPABASE_URL
  const key = process.env.SUPABASE_SECRET_KEY

  if (!url || !key) {
    // Fail loudly. A silently-undefined key produces confusing 401s at runtime.
    throw new Error('SUPABASE_URL and SUPABASE_SECRET_KEY must be set')
  }

  client = createClient<Database>(url, key, {
    auth: {
      persistSession: false,
      autoRefreshToken: false,
      detectSessionInUrl: false,
    },
  })

  return client
}
```

**Why each piece:**

| Line | Why |
|---|---|
| `import 'server-only'` | Build-time error if this module is ever pulled into a Client Component's module graph. This is the same mechanism protecting `lib/pricing.ts` per ASSESSMENT §3.4. |
| Lazy `function`, not top-level `const` | A module-scope `createClient(...)` call evaluates during the build. Env vars read at that point get baked in, and a missing var throws at build time in confusing ways. A lazy getter reads `process.env` at request time. |
| Cached in `client` | Avoids constructing a new client per invocation while still being lazy. |
| `persistSession: false` | No `localStorage` on a server. Without this, supabase-js sets up session persistence machinery that is meaningless and, on a warm serverless instance, can leak state across requests. |
| `autoRefreshToken: false` | No user session to refresh; suppresses a background timer that keeps the function alive. |
| `detectSessionInUrl: false` | No browser URL to parse. |

> **On `server-only`:** Next.js 16's docs state that installing the package is *optional* — Next handles `server-only` / `client-only` imports internally and ships its own type declarations. Install it anyway: it satisfies lint rules about extraneous dependencies, and `@clerk/nextjs` already depends on it, so it's in your tree regardless. **Confidence: HIGH.**

### 4.3 Using it from a Server Action

```ts
// app/_actions/submit-quote.ts
'use server'

import 'server-only'
import { z } from 'zod'
import { supabaseAdmin } from '@/lib/supabase/admin'
import { PRICE_LIST, calculateTotal } from '@/lib/pricing'   // also `server-only`

const QuoteInput = z.object({
  customerName: z.string().min(1).max(200),
  customerEmail: z.email(),                     // Zod 4: NOT z.string().email()
  decorationType: z.enum(['screen_print', 'embroidery', 'marketing']),
  brand: z.enum(BRANDS),
  frontColors: z.number().int().min(0).max(8),
  sizes: z.record(SizeKey, z.number().int().min(0).max(5000)),  // Zod 4: two args required
  dueDate: z.coerce.date().min(new Date()),
  // no price fields — by design (FR-5)
})

export async function submitQuote(raw: unknown) {
  const parsed = QuoteInput.safeParse(raw)
  if (!parsed.success) {
    return { ok: false as const, error: 'Some details are missing or invalid.' }
  }

  const total = calculateTotal(parsed.data, PRICE_LIST)   // server computes

  const { data, error } = await supabaseAdmin()
    .from('quotes')
    .insert({ ...parsed.data, estimated_total: total })
    .select('id')
    .single()

  if (error) {
    console.error('quote insert failed', error)
    return { ok: false as const, error: 'We could not save your quote. Please try again.' }
  }

  return { ok: true as const, quoteId: data.id, total }
}
```

Note `.select('id').single()` on the insert — you need the row id to attach artwork (§5) and to build the notification link (§6). A bare `.insert()` returns no data.

### 4.4 Verifying the boundary held (FR-5 acceptance criterion)

The PRD requires proving the price list is absent from the client bundle. Add this as a CI step:

```bash
# Fails the build if any Supabase key, price constant, or project URL reaches the browser.
npm run build
if grep -rE "sb_secret_|SUPABASE_SECRET|<a-known-price-constant>" .next/static/; then
  echo "SECRET LEAKED INTO CLIENT BUNDLE"; exit 1
fi
```

`server-only` catches the import at build time; this catches the case where someone inlined a value by hand. Belt and braces, and it is the literal acceptance criterion.

---

## 5. Supabase Storage — private bucket, upload, signed download

All signatures below were read from the **`@supabase/storage-js@2.111.0` package source** (`src/packages/StorageFileApi.ts`, `src/packages/StorageBucketApi.ts`, `src/lib/types.ts`), not from docs. **Confidence: HIGH.**

### 5.1 Creating the private bucket

`createBucket` signature:

```ts
createBucket(
  id: string,
  options: {
    public: boolean
    fileSizeLimit?: number | string | null
    allowedMimeTypes?: string[] | null
    type?: BucketType            // private beta; default 'STANDARD' — leave unset
  } = { public: false }
): Promise<{ data: { name: string }, error: null } | { data: null, error: StorageError }>
```

**Recommendation: create the bucket in a SQL migration, not at runtime.** FR-3 requires schema in version-controlled migrations, and the bucket is schema.

```sql
-- supabase/migrations/0002_artwork_bucket.sql
insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values (
  'artwork',
  'artwork',
  false,                              -- private: no public object URLs
  26214400,                           -- 25 MB, comfortably above FR-1's 12 MB case
  array[
    'application/pdf',
    'application/postscript',          -- .ai and .eps
    'image/svg+xml',
    'image/png',
    'image/jpeg'
  ]
)
on conflict (id) do nothing;

-- No policies on storage.objects for this bucket.
-- Default-deny. Only the secret key (service_role, BYPASSRLS) can read or write.
-- This is the RLS backstop from ASSESSMENT §3.4 point 4.
```

Two notes:
- `.ai` and `.eps` both report as `application/postscript` from most browsers, but **browser-reported MIME types are attacker-controlled**. FR-1 says type must be validated server-side. Validate the extension and the MIME type in the Server Action with Zod *before* issuing an upload token; treat `allowed_mime_types` on the bucket as the second line of defence, not the first.
- Deliberately **no** `storage.objects` policies. Zero policies + RLS on = nothing but the secret key gets in, which is exactly the FR-1 acceptance criterion ("a raw object URL without a valid signature is refused").

### 5.2 ⚠️ Uploading — the Server Action route does not work for 12 MB files

This is the most consequential finding in this document.

| Limit | Value | Source |
|---|---|---|
| Next.js Server Action request body, default | **1 MB** | [Next.js `serverActions` config reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/serverActions), v16.2.12 |
| Vercel Function request body, **hard cap** | **4.5 MB** | [Vercel Functions Limits](https://vercel.com/docs/functions/limitations), updated 2026-07-01. Exceeding it returns `413 FUNCTION_PAYLOAD_TOO_LARGE`. |

FR-1's acceptance criterion is literally *"a customer uploads a 12 MB PDF."* Sending that file to a Server Action as `FormData` **cannot work on Vercel at any configuration**. `serverActions.bodySizeLimit` can be raised, but it cannot exceed the platform's 4.5 MB — the request is rejected at the edge before your code runs. Worse, the Next.js failure mode here is known to be silent: the action can fail without reaching your `try/catch`, which would reproduce ASSESSMENT §2.4 (false success reported to the customer) in a new form.

**The correct pattern: server-issued signed upload URL, browser uploads direct to Supabase Storage.**

This does **not** violate the locked constraint. The browser receives a **single-use, path-scoped upload token**, not a Supabase API key. It cannot list the bucket, read other objects, write to a different path, or query Postgres. The key stays on the server; the server decides the path; the server records the row.

**Step 1 — Server Action issues the token.**

```ts
// app/_actions/create-artwork-upload.ts
'use server'

import 'server-only'
import { z } from 'zod'
import { randomUUID } from 'node:crypto'
import { supabaseAdmin } from '@/lib/supabase/admin'

const MAX_BYTES = 25 * 1024 * 1024

const ALLOWED = {
  'application/pdf': 'pdf',
  'application/postscript': 'eps',
  'image/svg+xml': 'svg',
  'image/png': 'png',
  'image/jpeg': 'jpg',
} as const

const Input = z.object({
  quoteId: z.uuid(),
  fileName: z.string().min(1).max(255),
  contentType: z.enum(Object.keys(ALLOWED) as [string, ...string[]]),
  byteSize: z.number().int().positive().max(MAX_BYTES),
})

export async function createArtworkUpload(raw: unknown) {
  const parsed = Input.safeParse(raw)
  if (!parsed.success) {
    return { ok: false as const, error: 'That file type or size is not supported.' }
  }
  const { quoteId, fileName, contentType } = parsed.data

  // Server chooses the path. The client never supplies it.
  const ext = ALLOWED[contentType as keyof typeof ALLOWED]
  const path = `${quoteId}/${randomUUID()}.${ext}`

  const { data, error } = await supabaseAdmin()
    .storage
    .from('artwork')
    .createSignedUploadUrl(path)

  if (error) {
    console.error('createSignedUploadUrl failed', error)
    return { ok: false as const, error: 'Could not start the upload. Please try again.' }
  }

  return {
    ok: true as const,
    path: data.path,
    token: data.token,
    originalName: fileName,
  }
}
```

`createSignedUploadUrl` signature (from source):

```ts
createSignedUploadUrl(
  path: string,
  options?: { upsert: boolean }
): Promise<
  | { data: { signedUrl: string; token: string; path: string }, error: null }
  | { data: null, error: StorageError }
>
```

**Step 2 — browser uploads with the token.**

The browser needs a Supabase client to call `uploadToSignedUrl`, and `createClient` requires a key argument. Rather than shipping any key, call the signed URL directly with `fetch` — no Supabase client in the browser at all:

```ts
// components/artwork-upload.tsx  ('use client')
// data.signedUrl is a fully-qualified URL that already carries ?token=...
const res = await fetch(signedUrl, {
  method: 'PUT',
  headers: { 'Content-Type': file.type },
  body: file,
})

if (!res.ok) {
  // FR-1: upload failure BLOCKS submission and tells the customer.
  setError('Your artwork did not upload. Please try again.')
  return
}
```

> **Confidence: MEDIUM-HIGH** on the raw-`fetch` variant. The `signedUrl` returned by `createSignedUploadUrl` is verified from source to be a complete URL with the token as a query parameter, and Storage accepts an authenticated `PUT` to it. If you hit friction, the documented alternative is `supabase.storage.from('artwork').uploadToSignedUrl(path, token, file)` — verified signature `(path: string, token: string, fileBody: FileBody, fileOptions?: FileOptions)`, and its doc comment confirms it requires **no** `objects` RLS permissions. That variant needs a browser Supabase client constructed with the *publishable* key, which the locked architecture forbids. **Prove the raw-`fetch` path works in Milestone 3 before committing to it**; it is the only shape that satisfies both FR-1 and the "no Supabase key in the browser" constraint.

**Step 3 — Server Action commits the record.** Only after the upload succeeds does a second Server Action write `artwork_path` and `artwork_original_name` onto the quote row, verifying via `storage.from('artwork').info(path)` that the object actually exists. This closes ASSESSMENT §2.1 and §2.4 together: no path is recorded for a file that isn't there, and no success screen renders without a confirmed commit.

**If the shop later decides files will never exceed ~3 MB**, the simpler `FormData` → Server Action path becomes viable:

```ts
// next.config.ts
const nextConfig: NextConfig = {
  experimental: {
    serverActions: { bodySizeLimit: '4mb' },   // still capped at 4.5MB by Vercel
  },
}
```

Note the docs' caveat: the limit applies to the **raw** HTTP body including multipart boundaries and part headers — budget 10–20 KB of overhead. Do not rely on this path for FR-1 as written.

### 5.3 Signed download URLs (FR-1, T5)

```ts
// app/_actions/get-artwork-url.ts
'use server'

import 'server-only'
import { auth } from '@clerk/nextjs/server'
import { supabaseAdmin } from '@/lib/supabase/admin'

export async function getArtworkDownloadUrl(quoteId: string) {
  // Protect the resource, not the route (§3.2)
  const { isAuthenticated } = await auth()
  if (!isAuthenticated) return { ok: false as const, error: 'Not signed in.' }

  const { data: quote, error: qErr } = await supabaseAdmin()
    .from('quotes')
    .select('artwork_path, artwork_original_name')
    .eq('id', quoteId)
    .single()

  if (qErr || !quote?.artwork_path) {
    return { ok: false as const, error: 'No artwork on file for this job.' }
  }

  const { data, error } = await supabaseAdmin()
    .storage
    .from('artwork')
    .createSignedUrl(
      quote.artwork_path,
      300,                                                  // 5 minutes
      { download: quote.artwork_original_name ?? true },    // force download, restore filename
    )

  if (error) {
    console.error('createSignedUrl failed', error)
    return { ok: false as const, error: 'Could not prepare the download.' }
  }

  return { ok: true as const, url: data.signedUrl }
}
```

Exact signatures (from source):

```ts
createSignedUrl(
  path: string,
  expiresIn: number,                       // SECONDS
  options?: {
    download?: string | boolean            // true → force download; string → force + set filename
    transform?: TransformOptions           // images only; do NOT use for PDF/AI/EPS
    cacheNonce?: string
  }
): Promise<{ data: { signedUrl: string }, error: null } | { data: null, error: StorageError }>

createSignedUrls(
  paths: string[],
  expiresIn: number,
  options?: { download?: string | boolean; cacheNonce?: string }
): Promise<{
  data: { error: string | null; path: string | null; signedUrl: string | null }[]
  error: null
} | { data: null, error: StorageError }>
```

Rules for this project:
- **Generate on demand, never store.** A signed URL persisted in the DB or embedded in server-rendered HTML outlives its purpose and can be forwarded. Issue it from a Server Action triggered by the download click.
- **Short expiry.** 300 seconds is ample for a click-to-download. The PRD's threat model is "anyone with the URL," so keep the window small.
- `download: '<original filename>'` matters: object paths are UUIDs, so without it the customer's `logo-final-v3.ai` downloads as `9f2c….ai`.
- **Never use `transform`** on artwork. It is raster image processing; on a PDF or AI file it will either fail or silently mangle the file. ASSESSMENT §2.10 already flags that the trade sends vector formats.
- `createSignedUrls` (plural) returns **per-path errors inside the data array** — a partial failure is not surfaced on the top-level `error`. If you batch, check each element.

---

## 6. Resend from a Server Action (FR-14)

### 6.1 The pattern

```ts
// lib/email/client.ts
import 'server-only'
import { Resend } from 'resend'

let resend: Resend | undefined

export function resendClient(): Resend {
  if (!resend) {
    const key = process.env.RESEND_API_KEY
    if (!key) throw new Error('RESEND_API_KEY must be set')
    resend = new Resend(key)
  }
  return resend
}
```

```ts
// app/_actions/submit-quote.ts (continued from §4.3)
'use server'

import { after } from 'next/server'
import { resendClient } from '@/lib/email/client'
import { NewQuoteEmail } from '@/emails/new-quote'

// ... inside submitQuote, after a confirmed insert:

  // FR-14: "Delivery failure is logged and never blocks the customer's submission."
  // `after` runs the callback once the response has been sent.
  after(async () => {
    const { data, error } = await resendClient().emails.send(
      {
        from: process.env.RESEND_FROM_ADDRESS!,     // must be on a verified domain
        to: process.env.OWNER_NOTIFICATION_EMAIL!,
        replyTo: parsed.data.customerEmail,          // camelCase — see §7.5
        subject: `New quote — ${parsed.data.customerName}`,
        react: NewQuoteEmail({
          customerName: parsed.data.customerName,
          total,
          dashboardUrl: `${process.env.NEXT_PUBLIC_APP_URL}/dashboard/jobs/${quoteId}`,
        }),
      },
      { idempotencyKey: `new-quote-${quoteId}` },    // second argument
    )

    if (error) console.error('resend send failed', { quoteId, error })
    else console.log('resend sent', { quoteId, emailId: data?.id })
  })

  return { ok: true as const, quoteId, total }
```

**Why `after()`:** FR-14 requires that email failure never blocks submission, and FR-4 requires the success screen render only after a confirmed DB commit. `after` gives you both: the response goes out as soon as the insert is confirmed; the email attempt happens afterwards without the customer waiting on Resend's latency. **Verified:** `after` has been stable since Next.js **15.1.0**, is explicitly supported in Server Functions, and on Vercel is backed by `waitUntil`. **Confidence: HIGH.**

⚠️ Do **not** use `after()` for the DB insert or the storage write — those must complete before you tell the customer it worked. `after()` is for the notification only.

**`resend.emails.send` shape** (verified from `resend@6.18.1` `dist/index.d.mts`):

```ts
send(payload: CreateEmailOptions, options?: CreateEmailRequestOptions): Promise<CreateEmailResponse>
```

`CreateEmailOptions` requires `from`, `to`, `subject`, and **at least one of** `react` | `html` | `text`. Also available: `cc`, `bcc`, `replyTo`, `headers`, `tags`, `attachments`, `scheduledAt`, `topicId`. `CreateEmailRequestOptions` carries `idempotencyKey`.

**Resend returns `{ data, error }` — it does not throw on API errors.** Ignoring `error` here would reproduce ASSESSMENT §2.4's failure mode in the email path. Check it.

### 6.2 ⚠️ React Email is a required install if you use `react:`

Resend **v5.0.0** made `@react-email/render` an *optional* peer dependency. Resend v6 dynamically imports it and throws this exact error when it's missing (verified in `dist/index.mjs`):

> `Failed to render React component. Make sure to install '@react-email/render' or '@react-email/components'.`

Install `@react-email/components@1.0.12` — it pulls in `@react-email/render` and gives you the email-safe primitives:

```tsx
// emails/new-quote.tsx
import { Body, Button, Container, Heading, Html, Text } from '@react-email/components'

export function NewQuoteEmail({
  customerName, total, dashboardUrl,
}: { customerName: string; total: number; dashboardUrl: string }) {
  return (
    <Html>
      <Body>
        <Container>
          <Heading>New quote from {customerName}</Heading>
          <Text>Estimated total: ${total.toFixed(2)}</Text>
          <Button href={dashboardUrl}>Open in the dashboard</Button>
        </Container>
      </Body>
    </Html>
  )
}
```

> Resend's v5 release note carries a warning worth heeding: if you upgraded from v4, `npm uninstall resend && npm install resend@latest` to be sure `@react-email/render` isn't lingering in the lockfile at the wrong version. On a greenfield install this doesn't apply.

**Alternative:** if you'd rather not add React Email at all, pass `html` and `text` strings instead of `react`. For a single internal notification email that is a defensible simplification — but you must then escape interpolated customer data yourself, and FR-8's spirit ("no unescaped user input") applies to email bodies too.

`react-email@6.9.1` (the dev CLI) is optional; it gives `npx email dev` for local preview. Nice for one template, not required.

---

## 7. Breaking changes and deprecations you will trip on

### 7.1 Next.js 16 (released 2025-10-22; 16.2.0 on 2026-03-18)

| Change | Impact |
|---|---|
| **`middleware.ts` → `proxy.ts`** | The named export `middleware` is deprecated too. `proxy` runs on **nodejs only** — no edge. `skipMiddlewareUrlNormalize` → `skipProxyUrlNormalize`. |
| **Async request APIs — sync access fully removed** | `cookies()`, `headers()`, `draftMode()`, `params`, `searchParams` are Promise-only. The Next 15 compat shim is gone. `params` in `page.tsx` must be `await`ed. |
| **`revalidateTag` requires a second argument** | `revalidateTag('quotes')` is a TypeScript error. Use `revalidateTag('quotes', 'max')`, or — better for this app — **`updateTag('quotes')`**, a Server-Actions-only API with read-your-writes semantics. When a team member moves a job, `updateTag` makes the board reflect it immediately instead of showing stale data. |
| **`refresh()` from `next/cache`** | New: refreshes the client router from a Server Action. Useful after stage changes (FR-10). |
| **Turbopack is the default** for `next dev` and `next build` | Remove `--turbopack` flags. If any plugin injects a `webpack` config, `next build` **fails**; opt out with `--webpack`. |
| **`next lint` removed** | `next build` no longer lints. Run `eslint` directly. `@next/eslint-plugin-next` now defaults to **flat config**. The `eslint` key in `next.config` is removed. |
| **`serverRuntimeConfig` / `publicRuntimeConfig` removed** | Use env vars. To read an env var at *runtime* rather than build time, `await connection()` from `next/server` first. |
| **Parallel routes need explicit `default.js`** | Builds fail without them. Relevant if FR-12's job detail modal uses an intercepting/parallel route. |
| **`experimental.ppr` / `dynamicIO` / `useCache` → `cacheComponents`** | Route-level `experimental_ppr` removed. Leave `cacheComponents` off (see §3.6). |
| **`next/image` defaults changed** | `minimumCacheTTL` 60s → 4h; `qualities` now `[75]` only; `16` removed from `imageSizes`; local images with query strings need `images.localPatterns.search`; `images.domains` deprecated in favour of `remotePatterns`. |
| **Node 20.9+, TypeScript 5.1+, Chrome/Edge/Firefox 111+, Safari 16.4+** | Node 18 unsupported. |

Codemod available: `npx @next/codemod@canary upgrade latest` — handles the `middleware` → `proxy` rename among others. Greenfield, so you'll mostly be starting correct rather than migrating.

### 7.2 Clerk v7 / Core 3 (released 2026-03-03)

| Change | Impact |
|---|---|
| **`<SignedIn>` / `<SignedOut>` / `<Protect>` removed → `<Show>`** | §3.5. The single most likely source of AI-generated broken code. |
| **`auth.protect()` returns 401, not 404**, for unauthenticated Server Action requests | Client-side error handling keyed on 404 must move to 401. |
| **Middleware-based protection actively deprecated** | §3.2. `createRouteMatcher` has its own migration guide. |
| **`ClerkProvider` inside `<body>`** when `cacheComponents: true` | Otherwise: *"Uncached data was accessed outside of `<Suspense>`"*. |
| **`@clerk/clerk-react` → `@clerk/react`**; `@clerk/clerk-expo` → `@clerk/expo`; `@clerk/types` → `@clerk/shared/types` | You import from `@clerk/nextjs`, so this is mostly informational — matters if you reach for types. |
| **`appearance.layout` → `appearance.options`**; `showOptionalFields` now defaults to `false` | Affects any `<SignIn>` styling. |
| **Redirect props removed:** `afterSignInUrl`, `afterSignUpUrl`, `redirectUrl` | Use `fallbackRedirectUrl` / `forceRedirectUrl` / `signUpFallbackRedirectUrl` / `signUpForceRedirectUrl`. |
| **`clerkJSUrl`, `clerkJSVersion`, `clerkUIUrl`, `clerkUIVersion` removed**; `clerkJSVariant="headless"` → `prefetchUI={false}` | |
| **`CLERK_ENCRYPTION_KEY` now required** if you pass `secretKey` to `clerkMiddleware()` | Not needed if you use the plain `clerkMiddleware()` from §3.1. |
| **Minimums:** Next 15.2.3+, Node 20.9+ | |
| Carried over from v6: **`auth()` and `clerkClient()` are async** | `auth().protect()` (v5 form) is wrong; use `await auth.protect()`. |

**Also still true, and still worth stating:** Clerk's older **Supabase JWT template was deprecated in April 2025** (ASSESSMENT §3.3). Do not wire one up. This architecture never needs Clerk and Supabase to know about each other — Next.js knows who the user is, Supabase only ever sees the secret key.

### 7.3 Supabase

| Change | Impact |
|---|---|
| **New API keys** (`sb_publishable_…` / `sb_secret_…`) replace `anon` / `service_role` | §4.1. Legacy JWT keys **deprecated by end of 2026**. Create secret keys on the new project immediately; deactivate legacy keys once nothing references them. |
| **New keys are rejected in `Authorization: Bearer`** | Must go in the `apikey` header. `supabase-js@2.111.0` handles this automatically (verified in source) — **pin `>= 2.111.0`**. Any hand-rolled `fetch` to Storage/REST must use `apikey`. |
| **`@supabase/supabase-js` v3 is in prerelease** (`3.0.0-next.29`) | Do not install the `next` tag. Stay on 2.x `latest`. |
| `createSignedUrl` gained a **`cacheNonce`** option | Not needed here; noting it because older examples don't have it. |
| Storage `type?: BucketType` on `createBucket` is **private beta** | Leave it unset; the default `STANDARD` is what you want. |

### 7.4 Zod 4 (v4.0.0 released 2025-07-09; 4.4.3 current)

| Zod 3 | Zod 4 |
|---|---|
| `z.string().email()` | **`z.email()`** — format validators are top-level now (tree-shakeable). The chained forms are deprecated. Also `z.uuid()`, `z.url()`. |
| `z.record(z.string())` | **`z.record(keySchema, valueSchema)`** — two arguments are now required. The ASSESSMENT §3.4 sketch already uses the two-arg form ✅ |
| `{ message: '…' }`, `invalid_type_error`, `required_error`, `errorMap` | **`{ error: '…' }`** or `{ error: (issue) => … }` — one unified parameter |
| `.default()` applied to the **input** type | `.default()` now applies to the **output** type and short-circuits parsing when input is `undefined`. Use **`.prefault()`** for the old behaviour. |
| `z.coerce.string()` input type was `string` | Now `unknown` |

`z.coerce.date().min(new Date())` from ASSESSMENT §3.4 works unchanged in v4.

### 7.5 Resend

| Change | Impact |
|---|---|
| **v5.0.0:** `@react-email/render` became an *optional* peer dependency | §6.2 — you must install it (or `@react-email/components`) to use `react:`. Runtime error otherwise. |
| **v6.0.0:** attachment `inlineContentId` → `contentId` | Preview feature; irrelevant unless you use inline attachment images. |
| **v4.x:** `reply_to` → **`replyTo`** | The SDK is camelCase throughout. Snake_case field names in blog posts are pre-v4. |

### 7.6 ⚠️ TypeScript 7 vs Next.js 16

`npm view typescript version` returns **7.0.2** (GA 2026-07-08) — the Go-native compiler. **Do not install it for this project yet.**

TypeScript 7.0 ships without the stable JavaScript Compiler API. Next.js uses that API to detect and drive TypeScript, so **Next.js reports `typescript` as not installed even when `typescript@7.0.2` is present.** Support landed via `vercel/next.js` PR #95639 behind an experimental flag, but was not in a stable Next release as of July 2026.

The stable choice is TypeScript **6.0.2**, published as `@typescript/typescript6` (npm `latest` = 6.0.2, verified). Note that plain `typescript`'s `6.0.0-beta` was never promoted — the `typescript` package jumped from 5.9.3 straight to 7.0.

```json
{
  "devDependencies": {
    "typescript": "npm:@typescript/typescript6@6.0.2"
  }
}
```

**Confidence: MEDIUM-HIGH.** Version numbers and package existence are HIGH (npm registry, verified). The Next.js incompatibility is sourced from the `vercel/next.js` discussion thread and secondary coverage — recheck at build start, since Next 16.3 was in preview (`16.3.0-preview.10`) at time of research and may resolve this.

---

## 8. What NOT to use — explicit list

| Avoid | Why | Do instead |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` / `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Normalises putting Supabase config in the client bundle; the anon key follows the URL. Violates PRD §7. | `SUPABASE_URL`, `SUPABASE_SECRET_KEY`, unprefixed, server-only |
| `@supabase/ssr`, `createBrowserClient`, `createServerClient` | Exists to manage Supabase auth cookies in a browser client. There is no browser Supabase client here. | `createClient` from `@supabase/supabase-js` in a `server-only` module |
| `createRouteMatcher` + `auth.protect()` inside `clerkMiddleware` | Clerk deprecated it; Server Actions bypass path-based matching entirely. | Bare `clerkMiddleware()`; `auth.protect()` / `auth()` in every page, layout, route handler, and Server Action |
| `<SignedIn>`, `<SignedOut>`, `<Protect>` | Removed in Clerk v7 | `<Show when="signed-in">` etc. |
| `auth().protect()` | Clerk v5 syntax. `auth()` is async and `protect` is a property on `auth`. | `await auth.protect()` |
| `FormData` file upload through a Server Action | Vercel hard-caps function request bodies at 4.5 MB; FR-1 requires 12 MB. Failure is silent. | Signed upload URL, browser PUTs direct to Storage (§5.2) |
| Storing a signed download URL in the database | Outlives its purpose; forwardable; defeats the private bucket | Generate per click from a Server Action, 300s expiry |
| `transform` option on artwork signed URLs | Raster image processing; breaks PDF/AI/EPS | Omit it |
| `z.string().email()` | Deprecated in Zod 4 | `z.email()` |
| `revalidateTag('x')` single-arg | TypeScript error in Next 16 | `updateTag('x')` in Server Actions, or `revalidateTag('x', 'max')` |
| `typescript@7.x` | Next.js 16.2 can't detect it | `npm:@typescript/typescript6@6.0.2` |
| Clerk's Supabase JWT template | Deprecated April 2025, and irrelevant — the browser never talks to Supabase | Nothing. There is no integration to configure. |
| `cacheComponents: true` | Adds Suspense-boundary failure modes and forces a `ClerkProvider` relocation, for zero benefit on a 3-user internal dashboard | Leave it off |
| Legacy `anon` key on the new project | Nothing in this architecture needs it; an existing anon key is an open door waiting for someone to use it | Create secret keys, deactivate legacy keys |

---

## 9. Version compatibility summary

| Package | Verified compatible with | Notes |
|---|---|---|
| `next@16.2.12` | `react@19.2.8`, `react-dom@19.2.8` | Peer range `^18.2.0 \|\| ^19.0.0` |
| `@clerk/nextjs@7.6.3` | `next@16.2.12` | Peer `^16.0.10` — satisfied |
| `@clerk/nextjs@7.6.3` | `react@19.2.8` | Peer `~19.2.3` — **tilde**; React 19.3 will conflict until Clerk widens |
| `@supabase/supabase-js@2.111.0` | `sb_secret_…` keys | Header handling verified in source (`isNewApiKey`). **Do not go below 2.111.0.** |
| `resend@6.18.1` | `@react-email/components@1.0.12` | `@react-email/render` resolved transitively |
| Everything | **Node >= 20.9.0** | Both `next` and `@clerk/nextjs` engines |
| `next@16.2.12` | `npm:@typescript/typescript6@6.0.2` | `typescript@7.x` not yet detected by Next |

---

## 10. Confidence assessment

| Claim | Confidence | Basis |
|---|---|---|
| All package versions in §1 | **HIGH** | npm registry `dist-tags.latest`, queried 2026-07-30 |
| Peer/engine constraints | **HIGH** | Read from published package manifests |
| Storage API signatures (§5) | **HIGH** | Read from `@supabase/storage-js@2.111.0` source |
| Resend `CreateEmailOptions` (§6) | **HIGH** | Read from `resend@6.18.1` `.d.mts` |
| supabase-js new-key header handling | **HIGH** | Read from `@supabase/supabase-js@2.111.0` source |
| Next.js 16 breaking changes (§7.1) | **HIGH** | Official upgrade guide, doc version 16.2.12 |
| Clerk Core 3 breaking changes (§7.2) | **HIGH** | Official `core-3.mdx` upgrade guide |
| `proxy.ts` + matcher (§3.1) | **HIGH** | Clerk `clerk-middleware.mdx` + Next.js v16 upgrade guide, corroborating |
| Middleware-protection deprecation (§3.2) | **HIGH** | Clerk `migrate-from-create-route-matcher.mdx` |
| Vercel 4.5 MB body limit (§5.2) | **HIGH** | Vercel Functions Limits, updated 2026-07-01 |
| Next.js 1 MB Server Action default | **HIGH** | Next.js `serverActions` config reference |
| `after()` stable + Vercel-supported | **HIGH** | Next.js `after` reference, version history table |
| Supabase new API keys / end-2026 deprecation | **HIGH** | Supabase official docs (api-keys, migrating-to-new-api-keys) |
| Zod 4 breaking changes | **MEDIUM-HIGH** | Official Zod v4 changelog |
| Browser raw-`fetch` to signed upload URL | **MEDIUM-HIGH** | URL shape verified in source; **prove in Milestone 3** |
| Vercel Supabase integration injects `NEXT_PUBLIC_*` | **LOW** | Not verified this pass. Audit the injected list after connecting. |
| TypeScript 7 / Next 16 incompatibility | **MEDIUM-HIGH** | `vercel/next.js` discussion #95633 + secondary coverage. Recheck at build start. |
| Clerk custom session claims as the `currentUser()` rate-limit fix | **MEDIUM** | Rate-limit warning is HIGH (in docs); the mitigation is a standard pattern, not doc-verified here |

---

## 11. Sources

**Registries and package source (read directly):**
- npm registry — `next`, `react`, `react-dom`, `@clerk/nextjs`, `@supabase/supabase-js`, `zod`, `resend`, `@react-email/components`, `react-email`, `server-only`, `tailwindcss`, `@tailwindcss/postcss`, `typescript`, `@typescript/typescript6`, `@types/react`, `@types/react-dom`, `eslint` — versions, dist-tags, peers, engines, publish dates
- `@supabase/storage-js@2.111.0` source — `StorageFileApi.ts`, `StorageBucketApi.ts`, `lib/types.ts`
- `@supabase/supabase-js@2.111.0` source — `SupabaseClient.ts`, `lib/fetch.ts`
- `resend@6.18.1` — `dist/index.d.mts`, `dist/index.mjs`

**Official documentation:**
- [Next.js — Upgrading to version 16](https://nextjs.org/docs/app/guides/upgrading/version-16) (doc version 16.2.12)
- [Next.js — `serverActions` config](https://nextjs.org/docs/app/api-reference/config/next-config-js/serverActions)
- [Next.js — `after`](https://nextjs.org/docs/app/api-reference/functions/after)
- [Next.js — Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components) (`server-only`, environment poisoning)
- Clerk docs via Context7 (`/clerk/clerk-docs`): `upgrade-guides/core-3.mdx`, `upgrade-guides/migrate-from-create-route-matcher.mdx`, `upgrade-guides/nextjs-v6.mdx`, `reference/nextjs/clerk-middleware.mdx`, `reference/nextjs/app-router/auth.mdx`, `reference/nextjs/app-router/server-actions.mdx`, `guides/secure/protect-content.mdx`, `guides/users/reading.mdx`, `reference/components/control/show.mdx`, `guides/development/clerk-environment-variables.mdx`
- [Supabase — Understanding API keys](https://supabase.com/docs/guides/getting-started/api-keys)
- [Supabase — Migrating to publishable and secret API keys](https://supabase.com/docs/guides/getting-started/migrating-to-new-api-keys)
- [Vercel — Functions Limits](https://vercel.com/docs/functions/limitations) (updated 2026-07-01)
- [Vercel — Environment variables](https://vercel.com/docs/environment-variables) (updated 2026-06-16)
- [Vercel — Sensitive environment variables](https://vercel.com/docs/environment-variables/sensitive-environment-variables) (updated 2026-06-03)
- [Zod — v4 changelog](https://zod.dev/v4/changelog)
- [Resend Node SDK — v5.0.0](https://github.com/resend/resend-node/releases/tag/v5.0.0), [v6.0.0](https://github.com/resend/resend-node/releases/tag/v6.0.0) release notes

**Lower-confidence / community:**
- [vercel/next.js discussion #95633 — TypeScript 7 support](https://github.com/vercel/next.js/discussions/95633) — MEDIUM
- [How to bypass the 4.5 MB body size limit](https://vercel.com/kb/guide/how-to-bypass-vercel-body-size-limit-serverless-functions) — corroborating §5.2

---
*Stack research for: print-shop quoting + production tracking (locked stack — implementation specifics only)*
*Researched: 2026-07-30*
