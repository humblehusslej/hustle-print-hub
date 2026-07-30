# Pitfalls Research

**Domain:** Quoting + production-tracking SaaS for a 3-person print shop
**Stack:** Next.js App Router (Vercel) · Clerk · Supabase Postgres + Storage · Server Actions w/ secret key · Zod · Resend
**Researched:** 2026-07-30
**Confidence:** HIGH (Vercel/Next.js/Clerk/Supabase official docs, current as of June–July 2026)

---

## Read this first

Two findings change the design before a line of code is written:

1. **The artwork upload cannot go through a Server Action.** Vercel caps request bodies at **4.5 MB**; Next.js caps Server Action bodies at **1 MB** by default. The PRD's own acceptance test (FR-1: *"a customer uploads a 12 MB PDF"*) **fails in production** with the obvious implementation — and **passes locally**, because `next dev` enforces neither limit. See Pitfall 1.
2. **Server Actions are unauthenticated public POST endpoints, and Clerk middleware cannot protect them.** Clerk's own docs now state that Server Functions "are called *by ID*, not by path," so a route matcher never sees them. Clerk is deprecating `createRouteMatcher()` for exactly this reason. See Pitfall 2.

Everything else is downstream of these two.

---

## Critical Pitfalls

### Pitfall 1: The 4.5 MB wall — artwork uploaded through a Server Action

**What goes wrong:**
The natural implementation is `<input type="file">` → `FormData` → Server Action → `supabase.storage.upload()`. Two stacked limits kill it:

| Limit | Value | Enforced by | Error |
|---|---|---|---|
| Server Action body | **1 MB default** | Next.js | `413` + "Request body exceeded limit" |
| Function request body | **4.5 MB hard cap** | Vercel | `413 FUNCTION_PAYLOAD_TOO_LARGE` |

Raising `experimental.serverActions.bodySizeLimit` to `'25mb'` **does not help**. Vercel's 4.5 MB is a platform limit on the function invocation and is not configurable on any plan. You can only ever lower the ceiling, never raise it.

Print artwork is routinely 5–50 MB (layered AI, high-res PDF, EPS with embedded rasters). A 4.5 MB cap makes the feature useless for the trade this shop serves.

**Why it happens:**
`next dev` enforces neither limit. A developer uploads a 12 MB PDF locally, it works, they ship. The first real customer file 413s in production. Because the prototype's failure mode was *silent* (§2.1), the team's instinct will be to trust a green checkmark — which is exactly how this becomes §2.1 again.

**How to avoid — the required architecture (client-direct upload):**

Bytes must never traverse Vercel. Browser → Supabase Storage, directly, authorized by a short-lived server-issued token.

```
1. Client → Server Action  requestArtworkUpload({ filename, declaredSize, declaredMime })
     · Zod-validate extension allowlist + declaredSize <= cap
     · path = `artwork/${crypto.randomUUID()}/${sanitizedFilename}`
     · storage.from('artwork').createSignedUploadUrl(path)   // uses sb_secret_ key, server-side
     · return { path, signedUrl }        ← a scoped, 2-hour, single-path token. NOT a Supabase key.

2. Browser → Supabase Storage   PUT signedUrl, body = File
     · Plain fetch(). No supabase-js client, no anon key, no publishable key in the browser.
     · Bytes never touch Vercel → 4.5 MB limit does not apply.

3. Client → Server Action  submitQuote({ ...config, artworkPath })

4. Server: VERIFY THEN COMMIT (see Pitfall 4)
     · Zod .strict() parse
     · storage.from('artwork').info(path) → object exists? size > 0? size <= cap?
     · compute price server-side
     · insert row, CHECK THE ERROR
```

This preserves the locked constraint in `PROJECT.md` ("the browser never holds a Supabase key"). A signed upload URL is a capability scoped to one path for 2 hours, not a database credential. It grants no read access, no other path, and no Postgres access.

**Size cap — enforce in three places, because only one of them is trustworthy:**

| Where | What it does | Trustworthy? |
|---|---|---|
| Zod on `declaredSize` in step 1 | Fast rejection, good UX | **No** — client-supplied |
| Bucket `fileSizeLimit` (Supabase) | Rejects the actual PUT | **Yes** — this is the boundary |
| Post-upload `.info(path)` size check in step 4 | Confirms what actually landed | **Yes** — this is the verification |

Supabase's **global** project file-size limit caps everything above it: **50 MB on Free**, 500 GB on Pro. A per-bucket limit cannot exceed the global limit. Recommend a **25 MB** bucket cap — comfortably under the Free-plan 50 MB ceiling, generous enough for real print artwork, and a number the UI can state honestly up front.

Supabase recommends standard (single-shot) uploads only up to **~6 MB**; above that use resumable/TUS (`chunkSize` must be exactly 6 MB). For a 25 MB cap on shop-office broadband a single PUT is acceptable; if uploads prove flaky, TUS is the escape hatch and it also runs browser→Supabase directly.

**Warning signs:**
- `FormData` containing a `File` is passed to any `'use server'` function.
- `experimental.serverActions.bodySizeLimit` appears in `next.config.ts` with a value above `'4mb'` — a sure sign someone hit the wall and tried to configure their way out.
- Upload works in `next dev` but has never been tested against a deployed preview URL.
- A `@supabase/supabase-js` client is constructed in a `'use client'` file.

**Automated check:**
- CI: `grep -rn "bodySizeLimit" next.config.*` → fail if the value exceeds `4mb`.
- CI: fail the build if `@supabase/supabase-js` is imported from any file containing `'use client'`.
- E2E against a **deployed preview**, not localhost: upload a 12 MB PDF; assert HTTP 200 and byte-identical SHA-256 on download.

**Phase to address:** Architecture decided in **Milestone 1 (Foundation)** — the bucket, its size limit, and the signed-upload-URL action must exist before the form is built. Implemented and verified in **Milestone 3 (Artwork pipeline)**.

---

### Pitfall 2: Server Actions are public HTTP endpoints — and middleware cannot protect them

**What goes wrong:**
Every exported `'use server'` function that is referenced anywhere in the app is compiled into a reachable POST endpoint. Next.js's own docs are unambiguous:

> "By default, when a Server Action is created and exported, it is reachable via a direct POST request, not just through your application's UI. This means, even if a Server Action or utility function is not imported elsewhere in your code, it can still be called externally."
> — *Next.js, "How to think about data security in Next.js" (v16.2.12, updated 2026-06-23)*

And on page-level auth:

> "A page-level authentication check does not extend to the Server Actions defined within it. Always re-verify inside the action."

Clerk says the same thing more sharply, and it is the reason they are deprecating `createRouteMatcher()`:

> "Server Functions are called *by ID*, not by path. A protected Server Function in `/protected/server-functions.ts` could be invoked while visiting `/public/`, allowing middleware to incorrectly permit access."
> — *Clerk, "Migrate from createRouteMatcher"*

So a `matcher` covering `/dashboard(.*)` protects the *page*. It does not protect `advanceStage()`, `deleteQuote()`, or `restoreQuote()` — those are POSTs to a path the matcher may never have heard of, carrying an action ID in a header. An attacker who has ever loaded the sign-in page can scrape action IDs from the RSC payload and replay them.

Next.js's built-in mitigations (encrypted non-deterministic action IDs, dead-code elimination of unused actions, Origin/Host comparison for CSRF) are real but explicitly described as risk-reduction, not authorization. Action IDs are cached up to **14 days**, so an ID harvested today is likely valid for a fortnight.

**Why it happens:**
The mental model "the dashboard is behind auth, therefore everything in it is behind auth" is correct for a traditional SPA + REST API where every endpoint is a visible path. It is false for Server Actions. This is the highest-consequence conceptual gap in the whole stack.

For this project the blast radius is the entire PII set — every customer name, phone, and email — which is the exact exposure `ASSESSMENT.md` §7 flags under Tex. Bus. & Com. Code § 521. An unprotected `listQuotes()` action is `?admin=true` with extra steps.

**How to avoid:**
Protect at the **resource**, not the route. Every Server Action and every DAL function starts with an auth check.

```ts
// lib/db/quotes.ts
import 'server-only'
import { auth } from '@clerk/nextjs/server'
import { db } from './client'

export async function advanceStage(quoteId: string, to: Stage) {
  const { userId } = await auth.protect()   // throws/redirects if unauthenticated
  // ... every read/write in here is now attributable to userId (also feeds FR-11 audit trail)
}
```

```ts
// app/actions/board.ts
'use server'
import 'server-only'
import { advanceStage } from '@/lib/db/quotes'

export async function advanceStageAction(quoteId: string, to: Stage) {
  return advanceStage(quoteId, to)   // thin wrapper; auth happens in the DAL
}
```

Keep `clerkMiddleware()` for the *redirect UX* (signed-out user lands on sign-in instead of a broken page). Treat it as convenience, never as the boundary. Clerk's own words: middleware protection "can lead to a false sense of security."

Note the one asymmetry this project has: `submitQuote` is **deliberately public** (customers have no account). That makes it the single action where the guard is Zod + rate limiting + no-price-in-payload rather than `auth.protect()`. Every *other* action must be authenticated. Write that rule down; it is the kind of exception that quietly becomes the default.

**Warning signs:**
- Any `'use server'` file with no `auth` import.
- `createRouteMatcher` used as the *only* gate.
- Auth logic that lives exclusively in `proxy.ts` / `middleware.ts`.
- `<SignedIn>` / `useUser()` used to decide whether data is *shown* rather than whether it is *fetched* (see Pitfall 9).

**Automated check:**
```bash
# CI: every server action file must reference auth, with one documented exception
for f in $(grep -rl "'use server'" app/actions); do
  case "$f" in *submit-quote*) continue;; esac
  grep -q "auth" "$f" || { echo "UNPROTECTED ACTION: $f"; exit 1; }
done
```
Plus an integration test: invoke each mutating action with no session cookie; assert it rejects **and** that the row count / row contents are unchanged. "Rejects" alone is not enough — assert the absence of the side effect.

**Phase to address:** **Milestone 1 (Foundation)** — establish the DAL + `auth.protect()` pattern before any action exists. Re-verified in **Milestone 4** when mutating board actions arrive.

---

### Pitfall 3: `supabase-js` returns errors instead of throwing — §2.4 reproduced verbatim

**What goes wrong:**
This is the single easiest way to rebuild the prototype's worst bug in the new stack.

```ts
// LOOKS FINE. IS THE PROTOTYPE BUG.
await db.from('quotes').insert(row)
return { ok: true }                        // ← success screen on a failed write
```

`supabase-js` **does not throw** on database errors. It resolves to `{ data, error }`. A missing `await`, an unchecked `error`, or a `try/catch` that never fires because nothing was thrown, all produce the same outcome as `index.html:385-396`: the customer sees *"logged successfully into our production queue"* and no row exists.

Two secondary traps in the same family:
- `.insert()` **without** `.select()` returns `data: null` even on complete success. Code that checks `if (!data) return { ok: false }` reports false failures.
- A broad `catch (e) { return { ok: false } }` wrapped around a `redirect()` will swallow Next.js's `NEXT_REDIRECT` control-flow error and break navigation. Keep `redirect()` **outside** the try block.

**Why it happens:**
Developers coming from Prisma/TypeORM/`fetch`-with-`res.ok` expect a throw. PostgREST clients return. The ergonomic difference is one line of code and the failure is invisible in the happy path — it only surfaces when the database is down, which is precisely when you need it.

**How to avoid:**
Make it structurally impossible to ignore an error. One helper, one lint rule.

```ts
// lib/db/client.ts
import 'server-only'
import { createClient } from '@supabase/supabase-js'

export class DbError extends Error {}

export const db = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SECRET_KEY!,          // sb_secret_… — server only, never NEXT_PUBLIC_
  { auth: { persistSession: false } }
)

/** Unwraps a PostgREST result or throws. Every call site goes through this. */
export function must<T>(res: { data: T | null; error: unknown }): T {
  if (res.error) throw new DbError(JSON.stringify(res.error))
  return res.data as T
}
```

Then: **ban `db.from(` outside `lib/db/`.** Inside `lib/db/`, every call is `must(await db.from(...)...)`. `supabase-js` also offers `.throwOnError()`; using it in addition is cheap belt-and-braces.

The customer-facing contract that satisfies FR-4:

```ts
'use server'
export async function submitQuoteAction(prev: State, form: FormData): Promise<State> {
  const parsed = QuoteInput.strict().safeParse(fromForm(form))
  if (!parsed.success) return { ok: false, fieldErrors: parsed.error.flatten().fieldErrors, values: raw }

  try {
    const { id, total } = await createQuote(parsed.data)   // throws on any DB error
    return { ok: true, id, total }
  } catch (e) {
    captureException(e)
    return { ok: false, message: 'We could not save your quote. Nothing was submitted — please try again.', values: raw }
  }
}
// redirect() — if used at all — goes here, OUTSIDE the try.
```

Note `values: raw` on every failure path: FR-4 requires the form retains its input on error. A `useActionState` form that returns only a message will blank the customer's 20 minutes of size-grid entry, and they will not re-enter it.

**Warning signs:**
- Any `await db.from(` / `await supabase.from(` whose result is discarded.
- A success view rendered in the same expression as the write.
- `console.error` as the only handler (this is `index.html:381` — "a broken backend and a quiet week look identical").
- Zero code paths in the UI that render a submission failure.

**Automated check:**
- ESLint `no-restricted-imports` / a simple CI grep: fail if `db.from(` or `supabase.from(` appears outside `lib/db/**`.
- Integration test with the database credential deliberately invalidated: assert the action returns `ok: false`, the success screen does **not** render, and the form's values survive.

**Phase to address:** **Milestone 1 (Foundation)** — the `must()` helper and the lint rule ship with the DAL. Verified in **Milestone 2 (Secure quote intake)** against FR-4.

---

### Pitfall 4: The path exists, the bytes don't — §2.1 reproduced in new form

**What goes wrong:**
The client-direct upload architecture from Pitfall 1 *decouples the file transfer from the submission*. That decoupling is exactly what re-opens §2.1:

```ts
// The new §2.1.
const { path } = await requestArtworkUpload({ filename: file.name, ... })
fetch(signedUrl, { method: 'PUT', body: file })   // ← not awaited / error not checked
await submitQuoteAction({ ...config, artworkPath: path })
// Row saved. Path recorded. Object never created.
```

The customer sees confirmation. The team sees a job card with an artwork link. The link 404s — possibly weeks later, on the shop floor, at the moment the job needs printing. **This is a strictly worse outcome than §2.1**, because §2.1 at least failed immediately and consistently; this fails at retrieval time, intermittently, after the customer has moved on.

The same failure arrives via: an upload that 400s on the bucket size limit, a signed upload token that expired (2-hour validity) because the customer left the tab open, a network drop mid-PUT, or a client-supplied `artworkPath` that was simply fabricated.

**Why it happens:**
Trusting a client-reported success. The prototype trusted `file.name`; the rebuild will trust `artworkPath`. Same class of error, one abstraction layer up. The path string in the payload is **client-controlled input** and deserves the same suspicion as a price field.

**How to avoid — verify-then-commit, always:**

```ts
// lib/db/quotes.ts
import 'server-only'

export async function createQuote(input: QuoteInput) {
  if (input.artworkPath) {
    // 1. The path must match the shape we issue. Never accept arbitrary paths.
    if (!/^artwork\/[0-9a-f-]{36}\/[\w.\- ]{1,120}$/.test(input.artworkPath)) {
      throw new ValidationError('Invalid artwork reference.')
    }
    // 2. The object must actually exist, and be non-empty, and be within the cap.
    const info = must(await db.storage.from('artwork').info(input.artworkPath))
    if (!info || info.size === 0)          throw new ArtworkMissingError()
    if (info.size > MAX_ARTWORK_BYTES)     throw new ArtworkTooLargeError()
    // 3. Sniff magic bytes — Content-Type is client-supplied (see Pitfall 10).
    await assertRealFileType(input.artworkPath, ALLOWED_TYPES)
  }

  const total = calculateTotal(input, PRICE_LIST)              // server-computed
  return must(await db.from('quotes').insert({ ...input, total }).select('id, total').single())
}
```

`ArtworkMissingError` must surface to the customer as a *failure*, with the form intact and the file re-selectable. Never as a success with a warning.

**Warning signs:**
- `artworkPath` written to the database without any storage lookup preceding it.
- The upload `fetch`/`PUT` result is not checked.
- The submit button is enabled while the upload is still in flight.
- No test exists in which the upload fails.

**Automated check:**
1. **Byte-identity E2E** (this is FR-1's acceptance test, made automatable): upload a fixture PDF, sign in as a team member, download, assert SHA-256 equality.
2. **Negative E2E:** stub the storage PUT to return 500. Assert (a) an error is shown, (b) `SELECT count(*) FROM quotes` is unchanged, (c) the file input still holds a selection.
3. **Standing invariant query** — run nightly against production and in CI against the seeded staging DB:
   ```sql
   -- must return zero rows, forever
   SELECT id, artwork_path FROM quotes
   WHERE artwork_path IS NOT NULL AND deleted_at IS NULL
     AND artwork_path NOT IN (SELECT name FROM storage.objects WHERE bucket_id = 'artwork');
   ```
   This single query is the direct, permanent, automated answer to §2.1. It should alert, loudly.

**Phase to address:** **Milestone 3 (Artwork pipeline)**. The invariant query is a **Milestone 6** metrics deliverable but should be written in Milestone 3.

---

### Pitfall 5: The secret key in the client bundle — and why rotating the env var won't fix it

**What goes wrong:**
`NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY`. It happens constantly, because half the Supabase tutorials on the internet are written for the browser-queries-Supabase model, and copy-paste carries the prefix along.

The mechanics make it worse than it first looks. `NEXT_PUBLIC_` variables are **inlined at build time** — `process.env.NEXT_PUBLIC_X` is textually replaced with the literal value in the emitted JavaScript. Next.js docs:

> "After being built, your app will no longer respond to changes to these environment variables… all `NEXT_PUBLIC_` variables will be frozen with the value evaluated at build time."

Consequences:
- Changing the Vercel env var **does not** remove the key from already-deployed bundles. Every prior deployment on its immutable `*.vercel.app` URL still serves it.
- Remediation is: rotate the key in Supabase (invalidating it), fix the prefix, rebuild, **and** redeploy — in that order.
- A Supabase secret/`service_role` key bypasses RLS entirely. Leaking it is total database compromise: read, write, and delete on every table plus every storage object. RLS default-deny (FR-3) provides **zero** protection against it, because the key is *defined* as the thing that bypasses RLS.

Related trap: the mirror-image mistake. Naming a genuinely public value *without* the prefix (`CLERK_PUBLISHABLE_KEY` instead of `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`) yields `undefined` in the browser and a confusing runtime break — annoying, not dangerous, but it trains people to add `NEXT_PUBLIC_` reflexively until something works. That reflex is how the dangerous version happens.

**Why it happens:**
The prefix is a single token with a catastrophic, invisible, irreversible effect. There is no compile error, no warning, no runtime signal.

**How to avoid:**
- **Naming rule:** `SUPABASE_SECRET_KEY`, `CLERK_SECRET_KEY`, `RESEND_API_KEY`. No `NEXT_PUBLIC_` on this project except `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, which is the only public value the app needs.
- Read `process.env` **only** inside `lib/db/` and `lib/pricing/`, both marked `import 'server-only'`. Nowhere else. This is the Data Access Layer discipline Next.js recommends and it makes the audit surface one directory.
- Mark the secrets as **Sensitive** in Vercel so their values cannot be read back from the dashboard.
- Optionally enable `experimental.taint` and `experimental_taintUniqueValue` on the key — a genuine second layer, though Next.js correctly notes it is defence in depth, not a substitute for the DAL.

**Automated check** — the highest value-per-line check in this document:

```bash
# CI, post-build. Fails the build if any secret shape reaches the browser bundle.
set -e
BUNDLE=$(find .next/static -name '*.js')
grep -l -E 'sb_secret_|eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9|sk_live_|sk_test_|re_[A-Za-z0-9]{20}' $BUNDLE \
  && { echo "FATAL: secret material in client bundle"; exit 1; }
grep -rn -E 'NEXT_PUBLIC_[A-Z_]*(SECRET|SERVICE_ROLE|PRIVATE|_SK_)' . --include='*.ts' --include='*.tsx' --include='.env*' \
  && { echo "FATAL: secret named with NEXT_PUBLIC_ prefix"; exit 1; }
exit 0
```

Key shapes to match: Supabase new-format secret `sb_secret_<22>_<8>`, Supabase legacy `service_role` JWT (`eyJhbGciOiJIUzI1NiIs…`), Clerk `sk_live_` / `sk_test_`, Resend `re_`.

**Warning signs:**
- `process.env` referenced in any `'use client'` file.
- `.env.example` containing a `NEXT_PUBLIC_` name next to anything called key, token, or secret.
- `@supabase/supabase-js` in the client dependency graph at all.

**Phase to address:** **Milestone 1 (Foundation)** — the CI grep is part of the deploy pipeline, which is Milestone 1's exit criterion.

---

### Pitfall 6: Clerk middleware matcher gaps, and two real CVEs that bypass middleware entirely

**What goes wrong:**
Three distinct failure modes, all producing "the route looked protected and wasn't."

**(a) The matcher silently doesn't run.** The historically common matcher excludes anything containing a dot:
```js
matcher: ['/((?!.+\\.[\\w]+$|_next).*)']   // legacy pattern — excludes paths with an extension
```
Any route whose final segment contains a `.` is never seen by middleware. A job detail route like `/dashboard/job/quote.2026-07.pdf`, or any dynamic segment where a customer-derived slug contains a dot, falls straight through. The failure is silent: no error, no log, just an unprotected page. The negation-lookahead syntax is also easy to get subtly wrong in ways that fail open rather than closed.

Clerk's current recommended matcher, for reference:
```js
export const config = {
  matcher: [
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    '/(api|trpc)(.*)',
    '/__clerk/(.*)',
  ],
}
```

**(b) `createRouteMatcher()` itself is bypassable.** **GHSA-vqx2-fgx2-5wq9 / CVE-2026-41248** (published 2026-04-15, **CVSS 9.1**): Clerk normalizes a path one way, the framework router normalizes it another — a CWE-436 interpretation conflict, exploitable via partial percent-encoding. Fixed in `@clerk/nextjs` **5.7.6 / 6.39.2 / 7.2.1** and `@clerk/shared` **2.22.1 / 3.47.4 / 4.8.1**. Sessions are not compromised; only the middleware gating decision is bypassed — which is the entire protection if middleware is your only gate.

**(c) The framework's middleware is bypassable.** **CVE-2025-29927** (CVSS 9.1): a spoofed `x-middleware-subrequest` header causes Next.js to skip middleware entirely. Patched in **12.3.5 / 13.5.9 / 14.2.25 / 15.2.3**.

Two independent CVSS-9.1 middleware bypasses in thirteen months is the argument. **Middleware is not an authorization boundary.**

**Why it happens:**
Middleware feels like the right place — it is one file, it runs first, it looks like Express. Every tutorial does it. And it *appears* to work: you test signed-out, you get redirected, you move on. The gaps are in paths nobody manually tests.

**How to avoid:**
- Implement Pitfall 2's resource-level `auth.protect()`. That makes all three of the above non-events rather than breaches — a matcher gap becomes a cosmetic UX bug instead of a PII disclosure.
- **Pin version floors** in `package.json`: `next` ≥ 15.2.3 (any current 16.x satisfies this), `@clerk/nextjs` ≥ 6.39.2 (or ≥ 7.2.1 on v7). Enable Dependabot/Renovate on these two packages specifically.
- Note the **Next.js 16 rename**: `middleware.ts` → `proxy.ts`, exported function `middleware` → `proxy`, `skipMiddlewareUrlNormalize` → `skipProxyUrlNormalize`. Codemod: `npx @next/codemod@canary middleware-to-proxy .`. Nearly every Clerk tutorial online still says `middleware.ts`; scaffolding from one produces a file Next.js 16 silently ignores — **a middleware file that does not run is indistinguishable from one with a broken matcher.** Confirm the filename against the installed Next.js major version on day one.

**Warning signs:**
- `middleware.ts` present in a Next.js 16 project (or `proxy.ts` in a 15 project) — either way, nothing runs.
- `npm audit` flags `@clerk/nextjs` or `next`.
- The matcher regex has been hand-edited.
- No test exercises a signed-out request to a protected route.

**Automated check:**
- `npm audit --audit-level=high` in CI, failing the build.
- E2E: for each dashboard route in a hardcoded list, request signed-out and assert a redirect to sign-in **and** that the response body contains no customer PII fixture string. Include one path with a dot in the final segment.
- Assert middleware actually executes: have `proxy.ts` set a header (`x-mw: 1`) and assert its presence in an E2E response.

**Phase to address:** **Milestone 1 (Foundation)** — this *is* the milestone's exit criterion ("signed-out users are refused").

---

### Pitfall 7: The price list duplicated to the client for live preview — §2.6 reproduced

**What goes wrong:**
`ASSESSMENT.md` §3.4 has the pricing engine right. The bug arrives through a UX requirement, not a security decision.

The customer wants the estimate to update as they adjust quantities. A server round-trip per keystroke feels slow. So a developer — reasonably, locally, in an afternoon — reimplements `calculateTotal` in a client component for instant feedback, "just for the preview; the server is still authoritative."

Now there are two implementations. They agree on the day they are written. Then the rush surcharge changes, or a size-break threshold moves, and only one is updated. The customer is quoted $487 and the database stores $511. **That is §2.6, arrived at from the opposite direction** — and it is harder to detect than the prototype's version, because the placeholder in §2.6 was obviously wrong while a drifted duplicate is plausibly wrong.

Second route to the same place: the client sends its computed total "for reference/logging" and someone later reads that field instead of recomputing. `ASSESSMENT.md` §3.4's rule ("the submission payload contains no prices") exists to close this. Enforce it with `.strict()` — plain Zod objects *strip* unknown keys silently, which satisfies FR-5's "rejected or ignored" but gives you no signal that someone is trying. `.strict()` turns an injected `total` field into a loud, testable rejection.

**Why it happens:**
Perceived latency. It is a real UX concern with a real solution that isn't duplication.

**How to avoid:**
**One implementation. Zero exceptions.** The live preview calls the same server function.

```ts
// lib/pricing/index.ts
import 'server-only'
export const PRICE_LIST_FINGERPRINT = 'hph_pricelist_v1_must_never_ship_to_browser'
export const PRICE_LIST = { /* … */ }
export function calculateTotal(input: QuoteInput): number { /* … */ }
```

```ts
// app/actions/pricing.ts
'use server'
import 'server-only'
import { calculateTotal } from '@/lib/pricing'
// Public by design (no auth) — quoting is the public feature.
// Zod-validated; rate-limited; returns a number and nothing else.
export async function previewTotal(raw: unknown) {
  const input = QuoteInput.strict().parse(raw)
  return { total: calculateTotal(input) }
}
```

Client: debounce ~300 ms, `useTransition` for a pending state. On a Vercel function this is a sub-100 ms round trip — fast enough that nobody will ask for a client-side copy. `import 'server-only'` makes the alternative a **build failure**, which is the real guarantee: the module resolves to a throwing stub under the non-`react-server` export condition.

**Warning signs:**
- Any price constant, rate, multiplier, or size-break threshold in a file reachable from `'use client'`.
- A `total`, `price`, `estimate`, or `subtotal` field in any Zod input schema.
- `lib/pricing` missing `import 'server-only'`.

**Automated check:**
1. **Bundle fingerprint grep** — this is FR-5's acceptance criterion, automated:
   ```bash
   grep -r "hph_pricelist_v1_must_never_ship_to_browser" .next/static/ && exit 1 || exit 0
   ```
   A sentinel constant is far more reliable than grepping for numbers, which appear everywhere.
2. **Quote/store equality test** (FR-5, third criterion): for N generated input fixtures, assert `previewTotal(x).total === (await createQuote(x)).total` and that both equal the value in the DB row. Run over a fixture matrix covering every decoration type, rush/non-rush, and each size-break boundary ±1.
3. **Injection test:** POST `{...validInput, total: 1}` and assert rejection, and that the stored total is the computed one.

**Phase to address:** **Milestone 2 (Secure quote intake)** — the pricing engine and the bundle grep land together.

---

## High-Severity Pitfalls

### Pitfall 8: Duplicate quotes return via double-submit, not dual-write — §2.5 reproduced

**What goes wrong:**
FR-3's single `quotes` table removes the *structural* cause of §2.5. Three new causes replace it:

1. **Double-click.** The customer clicks Submit twice on a slow connection. Two POSTs, two rows, two outreach entries. Indistinguishable, to the owner, from the prototype's bug.
2. **Retry-on-error.** A submission that times out client-side but committed server-side, retried by the user, yields two rows.
3. **Drift back to two tables.** Milestone 5 builds the leads view. If it gets its own table "because leads have different fields," §2.5 is rebuilt from scratch — and `ASSESSMENT.md` §2.5's closing observation ("two tables hold overlapping copies… They will drift") applies again.

**How to avoid:**
- **Idempotency key.** Client mints a `crypto.randomUUID()` when the form mounts (not at submit — that would generate a fresh one per click). Column `submission_id uuid UNIQUE`. Server does `.upsert(row, { onConflict: 'submission_id', ignoreDuplicates: true })` and returns the existing row's result as success. Double-submit becomes provably impossible rather than merely unlikely.
- **Disable during pending:** `const [state, action, pending] = useActionState(...)`; `<button disabled={pending}>`. Necessary, not sufficient — it does not survive a page refresh mid-submit.
- **`needs_outreach` is a boolean column on `quotes`.** The leads view is `SELECT … FROM quotes WHERE needs_outreach AND NOT contacted`. It is a *filter*, not a table. Write this in the migration comment so Milestone 5 cannot forget.

**Automated check:**
- Concurrency test: fire `submitQuoteAction` twice in parallel with the same `submission_id`; assert `count(*) = 1` and that both calls returned success.
- Schema test: assert the migration set contains exactly one table storing submissions — e.g. fail if any table other than `quotes` has both a `customer_email` and a `due_date` column.
- FR-13's own criterion, automated: seed 10 submissions, assert the outreach query returns exactly 10.

**Phase to address:** Idempotency in **Milestone 2**; the single-table assertion in **Milestone 1** (schema) and re-verified in **Milestone 5**.

---

### Pitfall 9: Stored XSS returns through the artwork, not the customer name — §2.7 reproduced

**What goes wrong:**
React escapes text by default, so §2.7's literal reproduction is unlikely. Four live vectors remain, and the first is created *by FR-1*:

1. **SVG artwork.** FR-1 explicitly accepts **SVG**. SVG is an XML document that may contain `<script>` and `on*` handlers. If a team member opens the artwork link and it renders inline, that script executes. Served from `*.supabase.co` it is cross-origin from the dashboard, so it cannot read the Clerk session — real but bounded (phishing, storage-origin access). **If anyone later builds `/api/artwork/[id]` that streams bytes with the original content type from the app's own domain, it becomes same-origin stored XSS in the owner's authenticated session — precisely §2.7's outcome, via a door FR-1 opened.**
2. `dangerouslySetInnerHTML` anywhere. FR-8 says don't; nothing enforces it.
3. **`javascript:` URLs.** If the form has a website/link field and it is rendered as `<a href={value}>`, React does **not** sanitize the protocol.
4. Filenames rendered as-is — lower risk in React, but they also flow into the Resend email (HTML, often not escaped by templating) and into `Content-Disposition` headers.

**How to avoid:**
- **Never render artwork inline. Ever.** `createSignedUrl(path, ttl, { download: filename })` forces `Content-Disposition: attachment`. FR-1 already says "download rather than inline preview" — treat that as a security requirement, not a UX preference.
- If artwork is ever proxied through the app's own domain, respond with `Content-Type: application/octet-stream`, `Content-Disposition: attachment`, and `X-Content-Type-Options: nosniff`. All three.
- ESLint `react/no-danger` at **error**.
- Zod every URL field: `z.string().url().refine(u => ['http:','https:'].includes(new URL(u).protocol))`.
- Sanitize filenames at upload-request time: strip everything outside `[A-Za-z0-9._ -]`, cap at 120 chars, and never let the client choose the directory component (Pitfall 4's path regex enforces this).
- Add a CSP header. Even a modest `default-src 'self'` closes vector 2 and 3's impact.

**Automated check:**
- E2E: submit a quote with customer name `<script>alert(1)</script>` and a second with `"><img src=x onerror=alert(1)>`; open the dashboard; assert both render as literal text and no dialog fires. This is FR-8's acceptance criterion verbatim.
- CI: `grep -rn "dangerouslySetInnerHTML" app/ components/ && exit 1`.
- E2E: upload an SVG containing `<script>`; assert the download response carries `content-disposition: attachment` and does not carry `content-type: image/svg+xml`.

**Phase to address:** **Milestone 2** (escaping, URL validation, lint rule); **Milestone 3** (artwork download headers).

---

### Pitfall 10: Signed URL expiry and MIME validation you cannot trust

**What goes wrong:**

**Expiry.** Signed download URLs are time-limited bearer tokens. Two mistakes:
- **Storing the signed URL in the database** alongside the quote. It expires; every historical job's artwork link dies. Store the **path**; mint URLs on demand.
- **Minting at render time with a short TTL.** The board is a long-lived page. A URL generated with a 60-second TTL when the page loaded is dead by the time someone clicks it ten minutes later. Raising the TTL to 7 days to "fix" it turns every link into a long-lived credential that leaks through `Referer`, browser history, screenshots, and shared Slack messages.

**MIME.** Supabase's bucket `allowedMimeTypes` validates against the **client-supplied** content type / filename extension (see `supabase/storage#576`, `supabase/supabase#27120`). It is a UX guardrail, not a security control. `evil.html` renamed `logo.pdf`, or any file uploaded with `Content-Type: application/pdf`, passes.

**How to avoid:**

**Expiry — use an indirection route.** The DOM holds a stable app URL; the signed URL is minted at click time:

```ts
// app/artwork/[quoteId]/route.ts
import { auth } from '@clerk/nextjs/server'
export async function GET(_req: Request, { params }: { params: Promise<{ quoteId: string }> }) {
  await auth.protect()                                   // re-checked at click time, not page-load time
  const { quoteId } = await params
  const q = await getQuoteArtworkPath(quoteId)           // DAL, throws if absent
  const { signedUrl } = must(
    await db.storage.from('artwork').createSignedUrl(q.path, 60, { download: q.filename })
  )
  return Response.redirect(signedUrl, 302)
}
```
Board card renders `<a href={`/artwork/${id}`} download>`. The link never expires, the signed URL lives 60 seconds, authorization is enforced at the moment of access, and revoking a team member's Clerk account immediately kills their access to every file. A signed URL minted at page render would not.

**MIME — validate content, not claims.** Read the first bytes server-side and check magic numbers before committing the row (Pitfall 4, step 3): `%PDF-` (PDF), `\x89PNG`, `\xFF\xD8\xFF` (JPEG), `%!PS` (EPS), and for AI files note that modern `.ai` is PDF-wrapped (`%PDF-`). SVG has no magic number — parse it, reject if it contains `<script`, `on*=`, or `<foreignObject`, or simply accept it and rely on forced-download (Pitfall 9). Keep `allowedMimeTypes` on the bucket anyway as the cheap first filter.

**Automated check:**
- Assert no column named `%url%` exists on `quotes` (paths only).
- E2E: load the board, wait 90 seconds, click artwork, assert 200.
- E2E: sign out, GET `/artwork/{id}` directly, assert redirect to sign-in and no bytes.
- Upload `payload.html` renamed to `logo.pdf`; assert rejection at the verify-then-commit step.

**Phase to address:** **Milestone 3 (Artwork pipeline)**.

---

### Pitfall 11: Orphaned files and orphaned references — ordering and cleanup

**What goes wrong:**
Storage and Postgres are two systems with no shared transaction. Any ordering leaves a window:

| Ordering | Failure window | Result | Severity |
|---|---|---|---|
| Upload → insert | Insert fails after upload succeeds | Orphaned **bytes** | Low — costs pennies, sweepable |
| Insert → upload | Upload fails after insert succeeds | Orphaned **reference** — row claims artwork that doesn't exist | **Critical — this is §2.1** |

**Recommended ordering: upload first, verify, then commit. Never the reverse.**

The asymmetry is the whole argument. An orphaned file is invisible to users and reclaimable by a scheduled job. An orphaned reference is a job card that lies to the shop floor, and the customer's file is gone — unrecoverable, discovered late. Optimise for never having a row that lies, and accept some garbage.

**Do not "fix" a failed insert with a synchronous delete as the only cleanup.** That compensating delete can itself fail (which is why you were in the error path to begin with). Attempt it opportunistically, but the *guarantee* must come from a sweeper.

**Cleanup strategy:**
- Upload straight to the final path — `artwork/{uuid}/{filename}` — with no `pending/` prefix and **no move step**. A move between prefixes is one more operation that can half-succeed. Reconciliation is by *reference*, not by location.
- **Vercel Cron, daily:** delete objects in the `artwork` bucket older than 24 hours with no matching `quotes.artwork_path`. The 24-hour grace window prevents a race with an in-flight submission.
- **The reverse sweep is the important one:** the invariant query from Pitfall 4 — rows referencing objects that don't exist — should alert immediately, not just log. Zero tolerance. It is the automated form of "artwork uploads never happen."
- Both sweeps report counts into the FR-metrics surface as **artwork upload success rate** (`ASSESSMENT.md` §5, "System health"), so a regression is visible rather than silent.

**Note on the open question (`PROJECT.md`, artwork retention):** the sweeper is where the retention policy will live. Build it with a configurable age threshold now, even though the policy is undecided, so adding "delete artwork for jobs completed >N months ago" later is a config change rather than new infrastructure.

**Phase to address:** Ordering in **Milestone 3**; cron sweeper and invariant alerting in **Milestone 6 (Operational polish)**, but write the query in Milestone 3.

---

### Pitfall 12: Zod schemas that validate less than they appear to

**What goes wrong:**
Four specific traps, all present in or adjacent to `ASSESSMENT.md` §3.4's example schema:

1. **`new Date()` evaluated at module load.** `dueDate: z.coerce.date().min(new Date())` — the schema object is constructed once when the module is first imported. On a warm serverless instance that persists for hours, `new Date()` is the *cold-start* time, not request time. The "no past dates" rule drifts. Use `.refine(d => d >= startOfToday(), 'Due date must be today or later')`.
2. **Timezone.** `z.coerce.date()` on `"2026-08-15"` parses as UTC midnight. For a Texas shop, a job due "today" at 19:00 CDT is already tomorrow UTC and is rejected — or the reverse. Normalise to the shop's timezone explicitly at the boundary.
3. **Non-strict objects strip silently.** Plain `z.object()` discards unknown keys without complaint. Use `.strict()` so an injected `total` is a *rejection* you can test for (Pitfall 7).
4. **`.parse()` throws; `safeParse()` returns.** If `.parse()` throws inside an action wrapped in a broad `catch` that returns a generic message, the customer gets "something went wrong" instead of "due date must be in the future," and FR-6's requirement that errors be actionable fails. Use `safeParse` and return `error.flatten().fieldErrors`.

Also: validate the **totals**, not just the fields. `sizes: z.record(SizeKey, z.number().int().min(0).max(5000))` permits every size at 5000 — a 60,000-piece order that will produce an absurd quote. Add a `.refine()` on the sum, and a `.refine()` requiring the sum > 0 (a quote for zero garments should not be submittable).

**Automated check:** a fixture table of malformed payloads — `quantity: -5` (FR-6's stated criterion), past due date, unknown decoration enum, `total` injected, all-zero sizes, 60,000-piece sum, `artworkPath: '../../other-bucket/x'` — each asserting rejection **and** `count(*)` unchanged.

**Phase to address:** **Milestone 2 (Secure quote intake)**.

---

### Pitfall 13: Preview deployments leak real data under weaker auth

**What goes wrong:**
Vercel builds a preview for every branch. Two default behaviours combine badly:

- **Env vars scoped to "all environments"** mean previews point at the **production Supabase project**. Real customer PII on an ephemeral URL. Worse, a preview branch running a destructive migration hits production data.
- **Clerk dev keys on previews.** Clerk cannot use production keys with `*.vercel.app` domains, so previews must use `pk_test_`/`sk_test_`. Clerk **development instances** have relaxed settings and separate user pools — meaning the three real team accounts don't exist there, and whatever sign-up restrictions exist in production may not apply.

Result: a publicly-reachable URL, weaker auth, production PII.

**How to avoid:**
- **Enable Vercel Deployment Protection → Standard Protection with Vercel Authentication.** It is **available on all plans including Hobby**, protects all preview and generated deployment URLs, leaves the production domain public, and costs nothing. Do this on day one of Milestone 1.
- **Scope env vars explicitly.** `SUPABASE_URL` / `SUPABASE_SECRET_KEY` set for **Production only**; a separate Supabase project (or Supabase branch) wired to **Preview**. Never tick all three environments for a database credential. Vercel supports branch-specific preview overrides if a branch needs its own target.
- Clerk `sk_test_`/`pk_test_` on Preview + Development; `sk_live_`/`pk_live_` on Production only.
- Use `VERCEL_ENV` to render an unmissable banner on non-production builds. The three users are non-technical; they must never wonder which board they are looking at.
- Resend on preview: point at a test address or disable, so preview submissions don't page the owner at 2 a.m.

**Warning signs:** one Supabase project URL in the env list with all environments ticked; a preview URL that loads the dashboard without a Vercel auth wall.

**Phase to address:** **Milestone 1 (Foundation)** — part of "deploy pipeline."

---

## Technical Debt Patterns

| Shortcut | Immediate benefit | Long-term cost | When acceptable |
|---|---|---|---|
| Upload artwork through a Server Action | Simplest possible code; works locally | Hard 4.5 MB ceiling; the feature fails for real print files; rearchitecting later touches the form, the action, the schema, and the tests | **Never** — the cap is below the median real artwork size |
| Client-side price calculation "just for the preview" | Instant UI feedback | Two implementations that drift → §2.6 with a plausible-looking wrong number | **Never** — debounced server call is fast enough |
| `catch (e) { return { ok: false } }` with no logging | Ships fast | Failures become invisible; you cannot answer "why did this customer's quote vanish" | Only if `captureException` is in the same block |
| Skipping the idempotency key, relying on `disabled={pending}` | One less column | Duplicates reappear on refresh/retry; looks exactly like §2.5 to the owner | Only if a unique constraint on `(email, created_at::date, due_date)` exists as a fallback |
| Signed URLs stored in the database | No route handler needed | Every historical artwork link dies; revoking a user doesn't revoke their links | **Never** |
| No audit trail until P1 | Ships the board sooner | `ASSESSMENT.md` §5: "cheap to add now, awkward to retrofit" — backfilling attribution for existing rows is impossible | Acceptable *only* if `actor_id` and `changed_at` columns exist from Milestone 1 even while unused |
| Middleware-only route protection | One file, matches every tutorial | Two CVSS-9.1 bypasses in 13 months; Server Actions unprotected regardless | **Never** as the sole boundary; fine as redirect UX |
| Hard delete for MVP, soft delete "later" | Simpler schema | Retrofitting `deleted_at` means auditing every query in the app for the filter | Only if `deleted_at timestamptz` ships in the initial migration, even if unused until Milestone 6 |

---

## Integration Gotchas

| Integration | Common mistake | Correct approach |
|---|---|---|
| **Supabase (Postgres)** | Assuming `.insert()` throws on failure | It returns `{data, error}`. Route every call through a `must()` unwrapper; ban `db.from(` outside `lib/db/` |
| **Supabase (Postgres)** | Believing RLS default-deny protects the app | The secret/`service_role` key **bypasses RLS entirely**. RLS is a backstop against *key leakage only* — it is worth zero against a logic bug in a Server Action. `ASSESSMENT.md` §3.4 states this correctly; make sure nobody downgrades their vigilance because "RLS is on" |
| **Supabase (Postgres)** | Worrying about connection pooling in serverless | Non-issue here: `supabase-js` speaks HTTP to PostgREST, not raw Postgres. Don't spend Milestone 1 on PgBouncer |
| **Supabase Storage** | Trusting `allowedMimeTypes` | Validated from client-supplied extension/header; spoofable. Sniff magic bytes server-side |
| **Supabase Storage** | Signed **upload** URL treated as long-lived | Valid **2 hours**, single path. A customer who leaves the tab open overnight gets a failed upload — surface that as a retryable error, not a success |
| **Supabase Storage** | Bucket left public, or public because "signed URLs are complicated" | Bucket `public: false`; FR-1 requires a raw object URL without a signature to be refused. Test it with `curl` |
| **Clerk** | `createRouteMatcher()` as the authorization boundary | Deprecated by Clerk for this purpose. `await auth.protect()` in every resource |
| **Clerk** | Using the deprecated Supabase JWT template | Deprecated **April 2025** (`ASSESSMENT.md` §3.3). Not needed — the browser never touches Supabase |
| **Clerk** | Production keys on `*.vercel.app` previews | Not possible; use dev keys on Preview and protect previews with Vercel Authentication |
| **Resend** | Assuming you can send from `onboarding@resend.dev` | Resend requires **at least one verified domain**. This blocks FR-14. DNS records (SPF/DKIM) take propagation time — **start domain verification in Milestone 1, not Milestone 5**, or FR-14 stalls on DNS |
| **Resend** | `await resend.emails.send()` inline in the submit path | FR-14: "Delivery failure is logged and never blocks the customer's submission." Use `after()` from `next/server` so it runs post-response; wrap in its own try/catch that only logs. Rate limit is **10 req/s per team** — irrelevant at this volume, but the failure must never propagate |
| **Resend** | Interpolating customer name into an HTML email body unescaped | Email templates are the one place React's escaping doesn't cover by default. Escape, or use `react-email` |
| **Vercel** | Assuming `maxDuration` needs tuning | With Fluid compute (default on new projects) Hobby gets **300s default and maximum**; Pro 300s default / 800s max. Nothing here approaches that. **Duration is not a risk on this project — body size is** |

---

## Performance Traps

At three internal users and (realistically) tens of quotes per week, almost nothing here is a scale problem. Listed for completeness with honest thresholds; **do not pre-optimise any of these.**

| Trap | Symptoms | Prevention | When it actually breaks |
|---|---|---|---|
| Board fetches every quote, forever | Board slows month over month | Default filter to non-archived + a date window; index `(stage, due_date)` | ~5,000 rows — likely year 2+ |
| Signed URL minted for every card on board render | Slow board render; N storage API calls per page | Indirection route (Pitfall 10) — mints on click, not on render | ~30 cards on screen |
| Unindexed `needs_outreach` filter | Leads view lags | `CREATE INDEX ... WHERE needs_outreach AND NOT contacted` (partial index) | ~10,000 rows |
| Debounced price preview with no debounce | Action call per keystroke; 30+ invocations per field | 300 ms debounce + `useTransition` | Immediately noticeable, no scale needed |
| Audit trail as unbounded append with no index | Detail modal slow to open | Index `(quote_id, created_at DESC)` | ~50,000 events |

---

## Security Mistakes

Beyond the OWASP basics, specific to this stack and this data.

| Mistake | Risk | Prevention |
|---|---|---|
| Server Action without its own auth check | Full customer PII disclosure (Tex. Bus. & Com. Code § 521) | `auth.protect()` in every action/DAL function; CI grep; unauthenticated-invocation test |
| Secret key inlined via `NEXT_PUBLIC_` | Total database compromise; **not fixed by changing the env var** — requires key rotation + rebuild + redeploy | Naming convention; `process.env` only in `server-only` modules; post-build bundle grep |
| Public `submitQuote` action with no rate limit | Quote spam floods the board and the owner's inbox; storage cost via signed-upload-URL abuse | Rate limit by IP on `submitQuote` **and** `requestArtworkUpload`. The upload-URL action is the one that mints storage write capabilities — it needs a limit more than the submit does |
| Client-supplied `artworkPath` accepted verbatim | Path traversal / referencing another customer's object | Strict regex on the path shape + `.info()` existence check; server generates the UUID, client never chooses the directory |
| SVG served inline from the app's own origin | Stored XSS in the owner's authenticated session — §2.7 | Forced download; `nosniff`; never proxy with the original content type |
| `total` accepted in the payload | §2.6 — customer-controlled pricing | `.strict()` Zod; no price field in any schema; injection test |
| Preview deployment on production data with dev auth keys | PII exposure on an unlisted-but-public URL | Vercel Standard Protection (free on Hobby) + per-environment env var scoping |
| Deleting a Clerk user without revoking outstanding signed URLs | A departed worker retains file access for the URL's TTL | 60-second TTL via the indirection route makes this a non-issue; a 7-day TTL makes it a real one |
| Error messages echoing raw Postgres errors to the browser | Schema disclosure | `must()` throws `DbError`; the action returns a generic message and logs the detail server-side |

---

## UX Pitfalls

Specific to three non-technical users and customers who will not come back if the form fails them.

| Pitfall | User impact | Better approach |
|---|---|---|
| Error message that discards the form | Customer loses 15 minutes of size-grid entry and leaves — worse than the prototype's false success, which at least felt fine | FR-4 requires it: return `values` on every failure path in `useActionState` |
| "Upload successful" before the bytes land | Rebuilds §2.1's exact emotional contract: confident confirmation, no file | Progress indicator tied to real upload progress; Submit disabled until the PUT resolves; confirmation only after the row commits |
| File too large discovered after a 4-minute upload | Customer waits, then fails | Check `file.size` client-side **before** requesting the upload URL; state the cap in the UI up front ("PDF, AI, EPS, SVG, PNG, JPG — up to 25 MB") |
| Rejecting a print file because the picker only offered images | §2.10 — the current bug. `accept="image/*"` excludes the formats print customers actually send | `accept=".pdf,.ai,.eps,.svg,.png,.jpg,.jpeg"` **and** server-side validation. The `accept` attribute is a hint, never a control |
| Confirmation modal on every stage change becomes noise | Three people click through it reflexively; the guard stops guarding | Name the job **and** the destination stage in the modal text (FR-10 requires this). Specific text is read; generic "Are you sure?" is not |
| No visible distinction between preview and production | Team moves a real job on a preview deployment, or vice versa | `VERCEL_ENV`-driven banner |
| Silent email failure | Owner assumes no quotes arrived; lead response time (a tracked metric) degrades invisibly | Log delivery failures and surface an unsent-notification indicator on the board — `ASSESSMENT.md` §5's "email delivery success" metric needs somewhere to show |

---

## "Looks Done But Isn't" Checklist

- [ ] **Artwork upload:** works locally — has it been tested with a **>5 MB file on a deployed URL**? Local `next dev` enforces neither the 1 MB nor the 4.5 MB limit.
- [ ] **Artwork upload:** the row commits only after the object is confirmed to exist. Test with the upload deliberately failing.
- [ ] **Artwork download:** byte-identical? Compare SHA-256, not "it opened."
- [ ] **Private bucket:** a raw object URL with no signature returns 4xx, and the bucket is not listable. `curl` it (FR-1's second criterion).
- [ ] **Auth:** every Server Action tested by direct invocation **with no session**, asserting both rejection *and* no side effect.
- [ ] **Auth:** `?admin=true` and the triple-click handler are absent from the codebase — `grep -rn "admin=true\|tripleClick\|detail === 3"` returns nothing (FR-2).
- [ ] **Auth:** the middleware/proxy file is named correctly for the installed Next.js major, and is proven to execute.
- [ ] **Pricing:** the built client bundle grepped for the price-list sentinel — nothing found (FR-5).
- [ ] **Pricing:** quoted total === stored total, over a fixture matrix, not one happy-path case (FR-5).
- [ ] **Write errors:** submission tested with the database unreachable — error shown, form values retained, no success screen (FR-4).
- [ ] **Validation:** `quantity: -5` rejected **and** nothing written (FR-6).
- [ ] **Duplicates:** ten submissions → ten outreach rows, not twenty (FR-13); concurrent double-submit → one row.
- [ ] **XSS:** `<script>` in the customer name renders as text (FR-8); an SVG with `<script>` downloads rather than renders.
- [ ] **Secrets:** post-build bundle grep for `sb_secret_`, `eyJhbGciOiJIUzI1NiIs`, `sk_live_`, `re_` — clean.
- [ ] **Env scoping:** the Preview environment does **not** hold the production Supabase credential.
- [ ] **Preview protection:** a preview URL in a logged-out incognito window shows the Vercel auth wall, not the board.
- [ ] **Resend:** domain verified and DNS propagated — checked before Milestone 5 starts, not during it.
- [ ] **Audit trail:** two different users each move a job; both entries attributed correctly (FR-11) — not just "an entry was written."
- [ ] **Calendar:** the visible window contains *today* on a machine whose clock is not the developer's (§2.9's bug was a hardcoded date; a timezone-naive rebuild reproduces it for half the day).
- [ ] **Soft delete:** every board/leads/calendar/metrics query filters `deleted_at IS NULL`. One missed query and deleted jobs reappear somewhere.

---

## Recovery Strategies

| Pitfall | Recovery cost | Recovery steps |
|---|---|---|
| Secret key shipped in a client bundle | **HIGH** | Rotate the key in Supabase immediately (this is what actually revokes it) → fix the env var name → rebuild → redeploy → audit `storage.objects` and table history for unauthorized access. Changing the env var alone fixes nothing; old immutable deployments still serve the literal |
| Artwork rows referencing missing objects | **HIGH** — often unrecoverable | The bytes are gone. Query the invariant, contact each affected customer, ask them to re-send. This is why the invariant must *alert*, not log: recovery cost scales with how long it went unnoticed |
| Unprotected Server Action discovered post-launch | **MEDIUM–HIGH** | Add `auth.protect()`; redeploy (new build → new action IDs, invalidating harvested ones); review Vercel function logs for anomalous POSTs; assess §521 notification obligations if PII was read |
| Orphaned storage objects | **LOW** | Run the sweeper. Costs storage, nothing else |
| Duplicate quote rows | **LOW–MEDIUM** | Dedupe by `(email, due_date, created_at` window`)`; add the unique constraint; backfill `submission_id` |
| Price drift between quoted and stored | **MEDIUM** | Recompute all stored totals from inputs with the current price list — possible **only** because inputs are stored separately from the total. Preserve that property in the schema; it is what makes this recoverable at all |
| Middleware CVE disclosed | **LOW** | Bump the package. Cheap *if* resource-level auth was in place; a breach investigation if it wasn't |
| Vercel 4.5 MB discovered after the form is built | **MEDIUM** | Rearchitect to client-direct upload: new action, changed client flow, changed schema (path instead of blob), rewritten tests. Roughly a Milestone-3-sized chunk of rework — which is why it belongs in Milestone 1's design |

---

## Pitfall-to-Phase Mapping

| # | Pitfall | Prevention phase | Verification |
|---|---|---|---|
| 1 | 4.5 MB Server Action body wall | **M1** design / **M3** build | 12 MB PDF uploaded and downloaded byte-identical **on a deployed preview** |
| 2 | Server Actions publicly callable | **M1** | Unauthenticated invocation of every action → rejected, no side effect |
| 3 | `supabase-js` errors not thrown | **M1** | DB-unreachable submission → error state, values retained (FR-4) |
| 4 | Path recorded, bytes absent | **M3** | Failed-upload E2E → no row; nightly invariant query returns zero rows |
| 5 | Secret key in client bundle | **M1** | Post-build grep for key shapes fails the build |
| 6 | Clerk matcher gaps / middleware CVEs | **M1** | Signed-out request to every dashboard route (incl. one with a dot) → redirect, no PII; `npm audit` clean |
| 7 | Price list duplicated client-side | **M2** | Bundle sentinel grep clean; quoted === stored across fixture matrix (FR-5) |
| 8 | Duplicate quotes | **M2** schema / **M5** leads view | Concurrent double-submit → 1 row; 10 submissions → 10 outreach entries (FR-13) |
| 9 | Stored XSS via artwork / `innerHTML` | **M2** escaping / **M3** download headers | `<script>` name renders literally (FR-8); SVG downloads with `nosniff` |
| 10 | Signed URL expiry; spoofable MIME | **M3** | Board open 90s → artwork click works; `.html` renamed `.pdf` rejected |
| 11 | Orphaned files / references | **M3** ordering / **M6** sweeper | Both invariant queries return zero |
| 12 | Weak Zod schemas | **M2** | Malformed-payload fixture table all rejected, zero writes (FR-6) |
| 13 | Preview leaks production data | **M1** | Incognito preview URL → Vercel auth wall; Preview env vars point at a non-production database |

**Ordering consequences for the roadmap:**

- **The `ASSESSMENT.md` §9 order of work is right, with one amendment.** §9 puts artwork at step 4 and the PRD puts it in Milestone 3, both after quote intake. Correct — artwork attaches to a quote record. But the *upload architecture* (bucket, size limit, signed-upload-URL action) must be decided and spiked in **Milestone 1**, because discovering the 4.5 MB wall during Milestone 3 invalidates the form built in Milestone 2.
- **Milestone 1 carries more weight than "scaffold + auth."** Pitfalls 2, 3, 5, 6, and 13 are all Milestone 1 preventions. Its exit criteria should include the DAL + `must()` helper, the CI secret-grep, deployment protection, and per-environment variable scoping — not only "a signed-in user reaches an empty dashboard."
- **Start Resend domain verification in Milestone 1.** DNS propagation is wall-clock time that cannot be compressed in Milestone 5.
- **Ship `deleted_at`, `actor_id`, and `submission_id` columns in the first migration**, even though soft delete (M6), audit trail (M4) and idempotency (M2) land later. Adding columns is free now and awkward once there are rows.
- **Flag for deeper phase research:** Milestone 3 only. The client-direct upload flow (exact `createSignedUploadUrl` → browser `PUT` request shape, progress reporting, and whether a 25 MB single-shot PUT is reliable enough or TUS is needed) is the one area where a short spike will pay for itself. Milestones 2, 4, 5, 6 are standard patterns.

---

## Confidence Notes

| Claim | Confidence | Basis |
|---|---|---|
| Vercel request/response body cap **4.5 MB**, error `FUNCTION_PAYLOAD_TOO_LARGE`, applies to Server Actions | **HIGH** | `vercel.com/docs/functions/limitations` (updated 2026-07-01) + Vercel KB bypass guide |
| Next.js Server Action `bodySizeLimit` default **1 MB** | **HIGH** | Next.js `serverActions` config docs + framework source (`app-page.ts`) |
| Vercel duration: Hobby 300s default/max, Pro 300s/800s (1800s beta) with Fluid compute | **HIGH** | Same page |
| Vercel Deployment Protection: Vercel Authentication + Standard Protection available on **all plans incl. Hobby** | **HIGH** | `vercel.com/docs/deployment-protection` (updated 2026-06-26) |
| `NEXT_PUBLIC_` inlined at build, frozen in deployed bundles | **HIGH** | Next.js environment-variables guide (v16.2.12) |
| Server Actions reachable by direct POST; page-level auth does not extend to them; IDs cached ≤14 days | **HIGH** | Next.js data-security guide (v16.2.12, updated 2026-06-23) |
| Clerk deprecating `createRouteMatcher()`; Server Functions called by ID not path; protect at the resource | **HIGH** | Clerk "Migrate from createRouteMatcher" upgrade guide + `clerk-middleware` reference |
| CVE-2025-29927 (CVSS 9.1), patched 12.3.5 / 13.5.9 / 14.2.25 / 15.2.3 | **HIGH** | Multiple independent security vendors, consistent |
| GHSA-vqx2-fgx2-5wq9 / CVE-2026-41248 (CVSS 9.1, 2026-04-15), fixed `@clerk/nextjs` 5.7.6 / 6.39.2 / 7.2.1 | **MEDIUM-HIGH** | Official GitHub advisory via search summary + 3 corroborating sources. **Re-verify the exact version floor against the advisory at install time.** |
| Next.js 16: `middleware.ts` → `proxy.ts`, codemod `middleware-to-proxy` | **HIGH** | Official `nextjs.org/docs/messages/middleware-to-proxy` + Next.js docs referencing `proxy.ts` |
| Supabase global file-size limit: Free **50 MB**, Pro/Team **500 GB**; bucket limit cannot exceed global | **HIGH** | Supabase storage file-limits guide |
| Standard upload recommended ≤ **6 MB**; TUS `chunkSize` must be exactly 6 MB | **HIGH** | Supabase uploads + resumable-uploads guides |
| `createSignedUploadUrl` valid **2 hours**, scoped to one path | **HIGH** | Supabase API reference |
| `createSignedUrl(..., { download })` forces attachment; `?download` on the URL | **HIGH** | Supabase serving/downloads guide |
| Supabase MIME validation is extension/header based and spoofable | **MEDIUM** | `supabase/storage#576`, `supabase/supabase#27120` — open issue reports, behaviour consistently reported. Treat as spoofable regardless; the mitigation costs nothing |
| Browser `PUT` to `signedUrl` with no Supabase client at all | **MEDIUM** | Follows from the signed-URL design; exact request shape (headers, `x-upsert`) **should be spiked in Milestone 3** |
| Resend requires ≥1 verified domain to send | **HIGH** | Resend domains docs |
| Resend rate limit **10 req/s per team** | **HIGH** | Resend API reference |
| Resend free-tier daily/monthly send caps | **NOT VERIFIED** | Not stated on the pages retrieved. Confirm before relying on FR-14 volume |

---

## Sources

**Official documentation**
- Vercel — Functions Limits: https://vercel.com/docs/functions/limitations
- Vercel — Bypassing the 4.5 MB body limit: https://vercel.com/kb/guide/how-to-bypass-vercel-body-size-limit-serverless-functions
- Vercel — Deployment Protection: https://vercel.com/docs/deployment-protection
- Vercel — Environment Variables: https://vercel.com/docs/environment-variables
- Next.js — How to think about data security: https://nextjs.org/docs/app/guides/data-security
- Next.js — Environment Variables: https://nextjs.org/docs/app/guides/environment-variables
- Next.js — `serverActions` config (`bodySizeLimit`, `allowedOrigins`): https://nextjs.org/docs/app/api-reference/config/next-config-js/serverActions
- Next.js — Renaming Middleware to Proxy: https://nextjs.org/docs/messages/middleware-to-proxy
- Next.js — Security in Server Components & Actions: https://nextjs.org/blog/security-nextjs-server-components-actions
- Clerk — Migrate from `createRouteMatcher`: https://clerk.com/docs/guides/development/upgrading/upgrade-guides/migrate-from-create-route-matcher
- Clerk — `clerkMiddleware()` reference: https://clerk.com/docs/reference/nextjs/clerk-middleware
- Clerk — Managing environments / preview deployments: https://clerk.com/docs/guides/development/managing-environments
- Supabase — Storage file limits: https://supabase.com/docs/guides/storage/uploads/file-limits
- Supabase — Standard uploads: https://supabase.com/docs/guides/storage/uploads/standard-uploads
- Supabase — Resumable uploads: https://supabase.com/docs/guides/storage/uploads/resumable-uploads
- Supabase — Serving / downloads: https://supabase.com/docs/guides/storage/serving/downloads
- Supabase — Migrating to new API keys (`sb_publishable_` / `sb_secret_`): https://supabase.com/docs/guides/getting-started/migrating-to-new-api-keys
- Resend — Domains: https://resend.com/docs/dashboard/domains/introduction
- Resend — API reference (rate limits): https://resend.com/docs/api-reference/introduction

**Security advisories**
- GHSA-vqx2-fgx2-5wq9 — Clerk middleware route-protection bypass (CVE-2026-41248): https://github.com/clerk/javascript/security/advisories/GHSA-vqx2-fgx2-5wq9
- CVE-2025-29927 — Next.js middleware authorization bypass: https://snyk.io/blog/cve-2025-29927-authorization-bypass-in-next-js-middleware/ · https://securitylabs.datadoghq.com/articles/nextjs-middleware-auth-bypass/ · https://projectdiscovery.io/blog/nextjs-middleware-authorization-bypass

**Issue trackers**
- Supabase Storage — Improper MIME type validation based on file extensions: https://github.com/supabase/storage/issues/576
- Supabase — MIME type does not check uploaded files, only the filename: https://github.com/supabase/supabase/issues/27120
- Next.js — Size limitation for Server Actions: https://github.com/vercel/next.js/discussions/57973

**Project documents**
- `ASSESSMENT.md` §2 (verified prototype failures), §3 (architecture), §5 (metrics), §9 (order of work)
- `PRD-hustle-print-hub.md` §4 (functional requirements + acceptance criteria), §8 (proposed milestones)
- `.planning/PROJECT.md` (locked constraints)

---
*Pitfalls research for: quoting + production-tracking tool, Next.js/Clerk/Supabase/Vercel*
*Researched: 2026-07-30*
