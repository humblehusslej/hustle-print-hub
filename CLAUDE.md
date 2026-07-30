<!-- GSD:project-start source:PROJECT.md -->
## Project

**Hustle Print Hub**

A quoting and production-tracking tool for a three-person screen-print and embroidery shop. Customers use a public link to configure a job — garment, decoration, colors, sizes, due date — upload their artwork, and get an estimated price. The owner, his business partner, and his worker use a shared internal board to see what's in the queue, move jobs through production stages, and track leads who need a callback.

This is a rebuild. A working prototype exists as a single 1,184-line `index.html`; it is being replaced rather than extended.

**Core Value:** **A customer's quote — including their artwork — reliably reaches the team and appears on a board all three of them can trust.**

Everything else is secondary. The current tool fails this: artwork is never actually uploaded, and failed submissions are reported to customers as successes.

### Constraints

- **Tech stack**: Next.js App Router on Vercel — gives a real server boundary for pricing without operating a separate backend, and server-renders for future SEO.
- **Auth**: Clerk, individual accounts, flat permissions — fast to integrate, and the browser-never-touches-Supabase design removes the usual RLS/JWT coupling problem.
- **Data**: Supabase Postgres + Storage on a **new** project — the old one is gone and is being abandoned.
- **Security**: The browser never holds a Supabase key. All data access goes through Server Actions using the service role key. No client-supplied price is ever trusted.
- **Team size**: Three non-technical users — the tool must be obvious without training.
- **Commercial**: Barter engagement (development for company apparel). Scope boundary is explicit in `ASSESSMENT.md` §8 precisely because there is no invoice to anchor it.
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Scope of this document
| # | Finding | Section |
|---|---------|---------|
| 1 | `middleware.ts` is renamed `proxy.ts` in Next.js 16, and Clerk now says route protection does **not** belong in it | §3 |
| 2 | FR-1's 12 MB artwork upload **cannot** go through a Server Action — Vercel caps function request bodies at 4.5 MB | §5.2 |
| 3 | Clerk v7 removed `<SignedIn>` / `<SignedOut>` / `<Protect>`. Every blog post and most AI-generated code uses them | §3.5 |
## 1. Pinned versions
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
### Install
# Core runtime
# Dev
- `@supabase/ssr` — that package exists to manage Supabase *auth cookies* in the browser. This architecture has no browser Supabase client. Installing it invites someone to wire one up.
- `@supabase/supabase-js@next` (3.0.0-next.29) — prerelease, not `latest`.
- `zod@3` or the `zod/v4` subpath import — v4 is the default export of `zod@4.x`.
- `@clerk/clerk-react` — renamed to `@clerk/react` in Core 3 (and you don't need it directly; `@clerk/nextjs` depends on it).
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
### Vercel specifics (verified from Vercel docs, last updated 2026-06-16 / 2026-06-03)
- **Environments:** Production, Preview, Development, plus custom environments. Preview vars can be scoped to a specific Git branch; branch-specific values override general preview values.
- **Sensitive environment variables:** values become non-readable once created. Available **only for Production and Preview** — you cannot mark a Development-scoped variable sensitive. To convert an existing variable, you must delete and re-add it with the Sensitive toggle on. Vercel redacts sensitive values ≥32 chars from build logs.
- **Local development:** `vercel env pull` writes the **Development** environment into `.env`. Add `.env*` to `.gitignore`.
- **Size:** 64 KB total across all variables; no single variable over 64 KB. Not a constraint here.
- Recommendation: mark `SUPABASE_SECRET_KEY`, `CLERK_SECRET_KEY`, `CLERK_ENCRYPTION_KEY`, and `RESEND_API_KEY` **Sensitive** in Production and Preview. Use a separate Supabase project (or at minimum separate secret keys) for Preview so a preview deploy cannot write to production data.
## 3. Clerk + Next.js App Router — the current official setup
### 3.1 The file is `proxy.ts`, not `middleware.ts`
- The `proxy` runtime is **`nodejs` and cannot be configured**. The `edge` runtime is not supported in `proxy`. (If you ever needed edge, you'd have to stay on the deprecated `middleware` filename. You don't need edge here.)
- Config flags renamed too: `skipMiddlewareUrlNormalize` → `skipProxyUrlNormalize`.
### 3.2 Route protection does **not** go in the proxy
- **Server Functions bypass the proxy entirely.** They are dispatched by action ID, not by URL path. A Server Action defined under `/dashboard/` but invoked from a public page never passes through your `/dashboard` matcher.
- Regex path-matching mistakes silently leave resources open.
- Path-normalisation differences between Clerk and the framework have produced real bypasses (GHSA-vqx2-fgx2-5wq9), plus 2025 framework-level disclosures where requests skipped middleware altogether.
### 3.3 Reading the authenticated user inside a Server Action
- `auth()` is **async** — `await` it. It returns `{ isAuthenticated, userId, sessionId, sessionClaims, orgId, has(), getToken(), redirectToSignIn() }`.
- `auth.protect()` is a **property on `auth`**, not a method on its result. `auth().protect()` is the Clerk v5 form and is wrong. Clerk's own `app-router/auth.mdx` still contains one stale `await auth().protect()` snippet — **the docs are internally inconsistent here.** Use `await auth.protect()`, which is what the v6 upgrade guide, the Core 3 guide, the migration guide, and `protect-content.mdx` all show. **Confidence: HIGH** on `auth.protect()`; flagging the doc inconsistency so nobody "corrects" it back.
- `auth()` requires `clerkMiddleware()` to be configured — that's why you keep `proxy.ts` even though it does no protecting.
- **Core 3 status-code change:** `auth.protect()` now returns **401 Unauthorized** for unauthenticated Server Action requests. It returned **404** in v6. Any client-side `if (error.status === 404)` handling must become `401`.
### 3.4 `currentUser()` — for the audit trail (FR-11)
### 3.5 `<SignedIn>` / `<SignedOut>` / `<Protect>` are gone — use `<Show>`
| v6 and earlier | v7 (Core 3) |
|---|---|
| `<SignedIn>` | `<Show when="signed-in">` |
| `<SignedOut>` | `<Show when="signed-out">` |
| `<Protect role="admin">` | `<Show when={{ role: 'admin' }}>` |
### 3.6 `ClerkProvider` placement
## 4. Supabase server-side client with the service-role key
### 4.1 First: "service role key" now means a **secret key**
- Legacy JWT `anon` / `service_role` keys are **deprecated and scheduled for removal by the end of 2026** ([Supabase docs](https://supabase.com/docs/guides/getting-started/api-keys)).
- New keys are created in **Settings → API Keys**, under the name `default`.
- This is an implementation detail *within* the locked decision, not a change to it. The privilege level, the trust model, and the "browser never holds it" rule are all unchanged.
### 4.2 The admin client module
| Line | Why |
|---|---|
| `import 'server-only'` | Build-time error if this module is ever pulled into a Client Component's module graph. This is the same mechanism protecting `lib/pricing.ts` per ASSESSMENT §3.4. |
| Lazy `function`, not top-level `const` | A module-scope `createClient(...)` call evaluates during the build. Env vars read at that point get baked in, and a missing var throws at build time in confusing ways. A lazy getter reads `process.env` at request time. |
| Cached in `client` | Avoids constructing a new client per invocation while still being lazy. |
| `persistSession: false` | No `localStorage` on a server. Without this, supabase-js sets up session persistence machinery that is meaningless and, on a warm serverless instance, can leak state across requests. |
| `autoRefreshToken: false` | No user session to refresh; suppresses a background timer that keeps the function alive. |
| `detectSessionInUrl: false` | No browser URL to parse. |
### 4.3 Using it from a Server Action
### 4.4 Verifying the boundary held (FR-5 acceptance criterion)
# Fails the build if any Supabase key, price constant, or project URL reaches the browser.
## 5. Supabase Storage — private bucket, upload, signed download
### 5.1 Creating the private bucket
- `.ai` and `.eps` both report as `application/postscript` from most browsers, but **browser-reported MIME types are attacker-controlled**. FR-1 says type must be validated server-side. Validate the extension and the MIME type in the Server Action with Zod *before* issuing an upload token; treat `allowed_mime_types` on the bucket as the second line of defence, not the first.
- Deliberately **no** `storage.objects` policies. Zero policies + RLS on = nothing but the secret key gets in, which is exactly the FR-1 acceptance criterion ("a raw object URL without a valid signature is refused").
### 5.2 ⚠️ Uploading — the Server Action route does not work for 12 MB files
| Limit | Value | Source |
|---|---|---|
| Next.js Server Action request body, default | **1 MB** | [Next.js `serverActions` config reference](https://nextjs.org/docs/app/api-reference/config/next-config-js/serverActions), v16.2.12 |
| Vercel Function request body, **hard cap** | **4.5 MB** | [Vercel Functions Limits](https://vercel.com/docs/functions/limitations), updated 2026-07-01. Exceeding it returns `413 FUNCTION_PAYLOAD_TOO_LARGE`. |
### 5.3 Signed download URLs (FR-1, T5)
- **Generate on demand, never store.** A signed URL persisted in the DB or embedded in server-rendered HTML outlives its purpose and can be forwarded. Issue it from a Server Action triggered by the download click.
- **Short expiry.** 300 seconds is ample for a click-to-download. The PRD's threat model is "anyone with the URL," so keep the window small.
- `download: '<original filename>'` matters: object paths are UUIDs, so without it the customer's `logo-final-v3.ai` downloads as `9f2c….ai`.
- **Never use `transform`** on artwork. It is raster image processing; on a PDF or AI file it will either fail or silently mangle the file. ASSESSMENT §2.10 already flags that the trade sends vector formats.
- `createSignedUrls` (plural) returns **per-path errors inside the data array** — a partial failure is not surfaced on the top-level `error`. If you batch, check each element.
## 6. Resend from a Server Action (FR-14)
### 6.1 The pattern
### 6.2 ⚠️ React Email is a required install if you use `react:`
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
### 7.5 Resend
| Change | Impact |
|---|---|
| **v5.0.0:** `@react-email/render` became an *optional* peer dependency | §6.2 — you must install it (or `@react-email/components`) to use `react:`. Runtime error otherwise. |
| **v6.0.0:** attachment `inlineContentId` → `contentId` | Preview feature; irrelevant unless you use inline attachment images. |
| **v4.x:** `reply_to` → **`replyTo`** | The SDK is camelCase throughout. Snake_case field names in blog posts are pre-v4. |
### 7.6 ⚠️ TypeScript 7 vs Next.js 16
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
## 11. Sources
- npm registry — `next`, `react`, `react-dom`, `@clerk/nextjs`, `@supabase/supabase-js`, `zod`, `resend`, `@react-email/components`, `react-email`, `server-only`, `tailwindcss`, `@tailwindcss/postcss`, `typescript`, `@typescript/typescript6`, `@types/react`, `@types/react-dom`, `eslint` — versions, dist-tags, peers, engines, publish dates
- `@supabase/storage-js@2.111.0` source — `StorageFileApi.ts`, `StorageBucketApi.ts`, `lib/types.ts`
- `@supabase/supabase-js@2.111.0` source — `SupabaseClient.ts`, `lib/fetch.ts`
- `resend@6.18.1` — `dist/index.d.mts`, `dist/index.mjs`
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
- [vercel/next.js discussion #95633 — TypeScript 7 support](https://github.com/vercel/next.js/discussions/95633) — MEDIUM
- [How to bypass the 4.5 MB body size limit](https://vercel.com/kb/guide/how-to-bypass-vercel-body-size-limit-serverless-functions) — corroborating §5.2
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
