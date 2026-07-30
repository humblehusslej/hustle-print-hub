# Project Research Summary

**Project:** Hustle Print Hub
**Domain:** Quoting + production-tracking web app for a 3-person screen-print/embroidery shop (Next.js App Router / Clerk / Supabase / Vercel / Resend)
**Researched:** 2026-07-30
**Confidence:** HIGH — an unusually convergent research cohort; see Confidence Assessment

---

## Executive Summary

Four independent research passes over the same locked stack (STACK, FEATURES, ARCHITECTURE, PITFALLS) converged on the same handful of load-bearing conclusions from different angles, which is the strongest signal this synthesis has to offer: **the PRD's own acceptance criterion for FR-1 — "a customer uploads a 12 MB PDF" — cannot be satisfied by the obvious implementation.** Vercel hard-caps function request bodies at 4.5 MB (not configurable, any plan) and Next.js caps Server Action bodies at 1 MB by default; three of the four researchers arrived at this independently and all three converged on the identical fix: a server-issued, single-path, time-limited Supabase signed upload URL, with the browser `PUT`-ing bytes directly to Storage. This does not weaken the locked "browser never holds a Supabase key" constraint — the token is a capability scoped to one path, not a credential. The same convergence pattern shows up on a second, related point: Server Actions are publicly reachable POST endpoints addressed by action ID, not by URL path, so Clerk's own middleware **cannot** protect them — three sources independently flag this and all recommend `auth.protect()` as the first statement in every resource/action rather than a middleware matcher.

Beyond infrastructure, FEATURES research found that the PRD's pricing requirement (FR-5) specifies **where** pricing must run but never **what** it computes — and the prototype's actual formula is a flat per-color rate with no quantity price-break tiers, no size upcharges despite already collecting the data, no distinction between a recurring per-screen setup fee and a one-time digitizing fee, no underbase/dark-garment handling, and a rush surcharge that is displayed in the UI and never charged. This is presented as the single highest-value fix in the entire build, alongside a genuine structural gap in the proposed production-board stage model: the PRD's 3-column board cannot represent the two states the tool's own stated purpose depends on — "waiting on the customer" and "waiting on the supplier" — both of which are real, multi-day stalls that look identical to active work under the current model.

The recommended path is architecturally coherent but front-loads more decision-making into Milestone 1 than the PRD's stated milestone breakdown assumes. Key risks are: (1) discovering the 4.5 MB wall mid-build rather than before it, which invalidates work already done on the quote form; (2) a live price-estimate feature naively built as a Server Action, which queues and cannot be cancelled (Route Handler required instead); (3) the classic "server returns `{data, error}`, doesn't throw" trap reproducing the prototype's worst bug (false success) in the new stack unless structurally prevented; and (4) a growing list of shop-specific numbers (rush multiplier, price tiers, size upcharges, setup/digitizing fees, whether the shop runs DTF) that block real pricing math regardless of how well the engine is built. Mitigation for all four is concrete and detailed below.

---

## Cross-Cutting Findings (Highest Confidence)

These are points where independent researchers, working from different angles (package source vs. official docs vs. domain-pricing literature vs. security advisories), reached the same conclusion. Convergence across independent research angles is the highest-confidence signal in this cohort — treat these as settled, not as options.

1. **The 12 MB artwork upload cannot go through a Server Action, on Vercel, at any configuration.** Confirmed independently by STACK (§5.2, reading the Next.js `serverActions` config reference and Vercel's Functions Limits page), ARCHITECTURE (finding F1, same sources), and PITFALLS (Pitfall 1, same sources plus the Vercel bypass-guide). Next.js's own default is 1 MB; Vercel's hard platform ceiling is 4.5 MB and is **not raisable on any plan**. All three independently prescribe the identical fix: a server-issued Supabase `createSignedUploadUrl` token (path-scoped, time-limited) that the browser `PUT`s to directly, bypassing Vercel's function body entirely. FEATURES does not address this (out of its scope) — it is a 3-of-4 convergence, not 4-of-4, and the strongest single finding in this research set.

2. **Server Actions are public, unauthenticated POST endpoints addressed by action ID — Clerk middleware cannot gate them.** STACK (§3.2, quoting Clerk's own migration-away-from-`createRouteMatcher` guidance), ARCHITECTURE (AP1 anti-pattern, plus the explicit statement "Middleware alone does NOT protect Server Actions"), and PITFALLS (Pitfall 2, plus two corroborating CVSS-9.1 CVEs: CVE-2025-29927 and GHSA-vqx2-fgx2-5wq9/CVE-2026-41248) all independently land on the same rule: **protect the resource, not the route.** Every Server Action and DAL function must open with `await auth.protect()`; `clerkMiddleware()` is kept only for redirect UX. All three treat this as the single most consequential conceptual gap in the stack, because the blast radius is the entire customer PII set (names, phones, emails — Tex. Bus. & Com. Code § 521 territory).

3. **`supabase-js` resolves to `{ data, error }` — it does not throw.** STACK's example code explicitly checks `error` after every mutation (§4.3), ARCHITECTURE's data model builds CHECK constraints specifically to make a false "uploaded" state unwritable, and PITFALLS (Pitfall 3) names this outright as "the single easiest way to rebuild the prototype's worst bug (§2.4) in the new stack" and proposes a structural fix (`must()` unwrapper, detailed under Automated Guards below). This is the same failure class as the prototype's core defect, one abstraction layer removed, and all three docs treat it as inevitable unless prevented structurally rather than by developer discipline.

4. **The rate card / price list must never reach the client bundle, and there must be exactly one implementation of the pricing function.** STACK (§4.4, CI grep for price constants), ARCHITECTURE (§3, rejecting "safe subset" client-side calculation as re-creating the prototype's exposed price list, and requiring the live estimate and the submission to call the *same* `calculateQuote()`), and PITFALLS (Pitfall 7, naming the two-implementations failure mode "§2.6 reproduced... harder to detect... because a drifted duplicate is plausibly wrong") converge on: one `server-only` pricing module, a bundle-grep CI check, and no price field ever accepted from the client.

5. **Verify-then-commit, never trust a client-reported upload success.** STACK (§5.2 step 3, §5.3), ARCHITECTURE (the `uploaded_requires_server_evidence` CHECK constraint, and naming AP3 "Trusting the client's report that an upload succeeded"), and PITFALLS (Pitfall 4, naming it "the new §2.1") all independently require the server to observe the stored object (via `storage.info()` / a HEAD request) before a row can claim `status='uploaded'`, with `byte_size` and `uploaded_at` written only from that server-side observation — never from the client's claim.

---

## Key Findings

### Recommended Stack

The stack is locked (Next.js App Router on Vercel, Clerk, Supabase Postgres + Storage, Server Actions with a secret-key admin client, Zod, Resend) — STACK.md's job was verifying *how* to implement it correctly as of July 2026, not choosing alternatives. All package versions were read live from the npm registry and package source, not training data.

**Core technologies and the version-specific traps that matter here:**
- `next@16.2.12` / `react@19.2.8` — Next 16 renamed `middleware.ts` -> `proxy.ts` (route protection guidance changed alongside it); async request APIs are fully Promise-only now; `next lint` is removed.
- `@clerk/nextjs@7.6.3` (Core 3) — `<SignedIn>`/`<SignedOut>`/`<Protect>` are gone, replaced by `<Show>`; `auth.protect()` now returns 401 (was 404 in v6); `auth()` is async.
- `@supabase/supabase-js@2.111.0` (floor, do not go below) — required because it auto-detects new-format `sb_secret_...` keys and routes them to the `apikey` header instead of `Authorization`. Legacy `anon`/`service_role` JWT keys are deprecated, scheduled for removal by end of 2026.
- `zod@4.4.3` — `z.email()` not `.string().email()`; `z.record(key, value)` now takes two arguments; `.strict()` is what turns a client-injected price field into a rejection rather than a silent strip.
- `resend@6.18.1` + `@react-email/components@1.0.12` — installing the render package is *mandatory*, not optional, if the `react:` prop is used; Resend returns `{data, error}` and does not throw.
- `typescript` — pin `npm:@typescript/typescript6@6.0.2`. Plain `typescript@latest` resolves to 7.0.2, which Next.js 16.2 cannot yet detect.
- Do **not** install `@supabase/ssr` — there is no browser Supabase client in this architecture at all.

### Expected Features

FEATURES.md's central finding: **FR-5 says where pricing runs but never what it computes, and the prototype's actual formula is wrong in ways that cost real revenue** — no quantity price-break tiers (the defining economic feature of screen printing; the prototype quotes a 12-piece order and a 500-piece order identically), no size upcharges despite already collecting the size data, a flat $30 "logo setup" standing in for three genuinely different fees (recurring per-screen setup, one-time digitizing, conditional art fee), no dark-garment/underbase handling, and a rush surcharge that renders in the UI copy and is never applied to the total.

**Must-have (table stakes), beyond what's already in the PRD:**
- Quantity price-break tiers, extended-size upcharges, per-screen setup fee (colours x locations), one-time digitizing fee distinct from setup, garment-colour/dark-garment flag, rush surcharge actually wired into the total
- Print locations with per-location colour counts (front/back integers can't express "left chest + sleeve")
- Multiple artwork files per line item, `.zip` acceptance, a realistic size cap (real AI/EPS files run 50-200 MB; a 5-10 MB default rejects normal customer art)
- Manual job entry (walk-ins/phone — most of a local shop's actual volume) and manual lead entry — both present in the prototype, both absent from the PRD
- Edit an existing job (nothing in FR-1...FR-16 permits changing a job after creation)
- A 5-stage board split (see below) instead of 3
- Per-method production checklist (screens burnt vs. DST proofed) carried forward from the prototype — real trade vocabulary, present nowhere in the PRD
- Sequential, collision-proof job numbers (the prototype's `Math.random()` scheme collides by roughly the 112th quote)

**Should-have (differentiators):** "who are we waiting on" as a first-class board read; a printable work order/pull sheet; time-in-stage / stall highlighting; a logo-on-file reorder registry; a customer confirmation email; a daily due/overdue digest; an MOQ soft-warning routing under-minimum requests to Needs Call instead of rejecting them.

**Deliberately not building:** drag-and-drop kanban (directly hostile to the stated "misclicks are a pain point" decision), real-time collaborative sync, exact/binding automated quotes, SanMar/blank-catalog integration, an online design studio, barcode job travelers, production scheduling/capacity planning, time tracking, BI dashboards, role separation, customer portals, marketing automation, names/numbers roster management, quote versioning.

### Architecture Approach

The system splits cleanly into a public, unauthenticated quote-intake surface and an authenticated dashboard, with **all** pricing and data access routed through `server-only` modules and a service-role Supabase client that the browser never touches. Three design decisions do the load-bearing work:

1. **Relational `quotes` -> `quote_line_items` (1:N), not JSONB arrays.** This is a correctness requirement, not a scale preference: the prototype's per-line-item `for` loop is what actually produced its 3-6x duplicate-row bug (finding F3), and only a real foreign key from `artwork_files` to a specific line item closes that class of bug permanently. Size breakdowns *within* a line item remain JSONB (closed, always-read-as-a-unit, shape genuinely varies by decoration type).
2. **Session-scoped artwork uploads (`intake_session_id`), not draft quote rows.** Artwork must exist before a quote row does (that's what makes the 12 MB path work), so `artwork_files.quote_id` is nullable until `submitQuote` claims it. This keeps "one submission = exactly one row" literally true with no `WHERE submitted_at IS NOT NULL` filter to forget anywhere in the app.
3. **The live price estimate must be a Route Handler (`/api/estimate`), not a Server Action** — this is ARCHITECTURE's own distinct finding (F2), not corroborated by the other three docs but load-bearing regardless: Next.js dispatches Server Actions serially per client with no abort signal, so a debounced high-frequency read on that transport would head-of-line-block the actual submission. Both the estimate and the submission call the *same* `calculateQuote()`, which is what makes "the total shown matches the row in the database" true by construction rather than by discipline.

**Major components:** `server/pricing` (rate card + `calculateQuote()`, the only function permitted to produce a dollar figure), `server/db` (service-role client + all queries, `server-only`), `server/storage` (signed upload/download issuance + existence verification), `server/audit` (append-only `quote_events` writer), `lib/schemas` (Zod shapes shared client/server — inputs only, never the rate card).

### Critical Pitfalls

1. **The 4.5 MB wall** (see Cross-Cutting #1) — required architecture: signed-URL direct-to-Storage upload.
2. **Server Actions unprotected by middleware** (see Cross-Cutting #2) — required pattern: `auth.protect()` in every resource.
3. **`supabase-js` doesn't throw** (see Cross-Cutting #3) — required pattern: a `must()` unwrapper, enforced by lint/CI.
4. **Path recorded, bytes absent** (Pitfall 4) — the decoupled upload/submit flow re-opens the prototype's original bug one layer up unless the server independently verifies the object exists before committing a reference to it.
5. **The secret key in the client bundle, and why rotating the env var doesn't fix it** (Pitfall 5) — `NEXT_PUBLIC_` variables are inlined at *build* time; every previously-deployed immutable Vercel URL keeps serving the leaked literal until the key is rotated (revoked) *and* rebuilt *and* redeployed. Changing the env var alone does nothing.

---

## Scope Changes

Findings that change **what** gets built, not when or how.

- **Pricing needs its own explicit FR**, not just FR-5's "where." Required inputs: quantity price-break tiers, size upcharges (2XL-5XL+), per-screen setup fee (colours x locations, not a flat number), a one-time digitizing fee kept structurally separate from setup, a dark-garment/underbase boolean, and the rush surcharge actually applied. FEATURES notes FR-5's current acceptance criteria all pass against the wrong formula — this is a scope gap, not a bug in what's already speced.
- **Print locations become a real field**, not two integers. Model as N locations (full front/back, left/right chest, sleeve, neck label, hem), each with its own colour count — required to count screens at all.
- **Garment colour / dark-garment flag restored to FR-7.** It's in the prototype (`input-color`), it drives underbase pricing, and it was dropped from the PRD's field enumeration.
- **Artwork cardinality decisions, currently unspecified in FR-1:** per line item (not per-quote), multiple files per line, `.zip` accepted (never expanded server-side), a realistic size cap (>=50 MB, not the 5-10 MB a naive default would pick), extension + magic-byte validation (MIME-only validation will reject legitimate `.ai`/`.eps` files that report inconsistent client MIME types).
- **Manual job entry — new FR.** Walk-ins/phone quoting is most of a local shop's volume; without it, the team will misuse the public customer form as a workaround and the audit trail will misattribute every such job to "customer."
- **Edit an existing job — new FR.** Nothing in FR-1...FR-16 permits post-creation edits; customers revise size counts and dates constantly.
- **Stage model split — amend FR-9.** Add "Awaiting Approval" (customer) and "Awaiting Blanks" (supplier) as distinct stages; fold pre-press into "In Production" behind the existing per-method checklist. See Contradictions below — this scope change is not yet reflected in ARCHITECTURE's schema.
- **Proof sent / approved date fields — new, minimum `proof_sent_at`, `approved_at`, `approved_by_name`.** No customer-facing approval link needed in v1 (that stays a deliberate, written-down v1 boundary).
- **In-hands date, separate from production date — new field.** "Due date" as a single field is ambiguous between what the customer needs and what the shop schedules to (with a deliberate buffer).
- **Sequential human-readable job number, not `Math.random()`.** Already addressed in ARCHITECTURE's DDL via a Postgres sequence — flagging here because it is a genuine PRD-level scope gap (FR-3 specifies the table but not the identifier scheme).
- **MOQ soft-warning — new, low cost.** Route sub-~12-piece screen-print requests to Needs Call rather than pricing them normally.
- **Payment-status booleans — needs an explicit FR or an explicit decision to drop.** PRD §2 already promises team members see "payment status"; the prototype has invoice-sent/paid toggles; no FR currently covers it. This is in scope (tracking, not processing) per `PROJECT.md`'s own boundary — it needs a decision, not a default.
- **Per-method production checklist — new FR**, formalizing what the prototype already does correctly (decoration-specific task labels; real trade knowledge that has no home in the current FR list).

## Sequence Changes

Findings that change **when** something must be built, independent of what it is or how it's implemented.

- **The artwork-upload architecture (bucket, size limits, signed-upload-URL action) must be decided in Milestone 1**, per PITFALLS — see Contradictions below for the tension with STACK's "prove in Milestone 3" framing.
- **Print locations and the dark-garment flag must land in the same milestone as pricing (Milestone 2), not be retrofitted later** — per FEATURES' dependency graph, they are pricing *inputs*; adding them after quotes exist means re-migrating and re-pricing existing rows.
- **Stage-enum changes belong in Milestone 1's schema, not Milestone 4.** Per FEATURES: adding enum values later is cheap; adding them after the board UI is built means rebuilding the board.
- **Artwork cardinality (per-line, multi-file, shop-side proof slot) must be decided before Milestone 3, not during it** — per FEATURES, both answers are schema decisions.
- **`quote_events` (the audit table) and its `quote.created` write must land in the first data phase (Milestone 1-2), not Milestone 4 as the PRD currently states.** Per ARCHITECTURE: if the table only arrives in Milestone 4, every quote created during Milestones 2-3 has no creation event, the detail view's history starts mid-story, and the lead-response-time metric has no `t=0`. Only the audit **UI** is safely deferrable to Milestone 4.
- **`deleted_at` must exist in the very first migration**, even though restore UI doesn't ship until Milestone 6. Per ARCHITECTURE and PITFALLS (independently): every board/lead/calendar/metrics query needs `WHERE deleted_at IS NULL`; adding the column late means auditing every query written before it, and the one you miss shows deleted jobs on the board.
- **`submission_id`/`intake_session_id` (the idempotency key) should exist from Milestone 1's schema**, enforced at Milestone 2. Per PITFALLS: mint it client-side at form-mount (not at submit-click), unique-constrain it, `upsert(... onConflict: 'submission_id', ignoreDuplicates: true)`.
- **Resend domain verification should start in Milestone 1, not Milestone 5.** Per PITFALLS: DNS propagation (SPF/DKIM) is wall-clock time that cannot be compressed later, and FR-14 stalls without it.
- **The PRD's stated rationale for its own milestone order is factually wrong, though the order itself still holds.** PRD §8 says "Milestone 2 precedes 3 because artwork attaches to a quote record that must exist first" — false under the session-scoped upload design (uploads happen *before* any quote row exists; that's what makes 12 MB work at all). ARCHITECTURE flags this explicitly: the real reasons the order still holds are (a) the *claim* step lives inside `submitQuote`, so submission must exist to attach files, and (b) FR-1's download-half acceptance criterion needs a dashboard detail view, which is Milestone 4. Practical consequence: **FR-1 is not fully verifiable at the end of Milestone 3 as currently scoped** — either pull a minimal job-detail page forward into Milestone 3, or move FR-1's download criterion to Milestone 4. This is a decision for roadmap time, not a discovery for the checkpoint.

## Implementation Specifics

Findings that change **how** something already in scope gets built.

- **Signed-upload-URL pattern, end to end:** Server Action validates (Zod: extension allowlist, declared size <= cap) -> server generates the storage path itself (never the client) -> `storage.from('artwork').createSignedUploadUrl(path)` (2-hour TTL, single path, `upsert:false`) -> browser `PUT`s raw bytes via plain `fetch` with **no Supabase client and no key in the browser at all** -> a second Server Action verifies via `storage.info(path)` (or a `Range: bytes=0-1023` HEAD/magic-byte sniff) before writing `status='uploaded'`.
- **Signed-download-URL pattern:** never store the URL, never mint at page-render time (the board is long-lived; a URL minted at load is dead by the time someone clicks it). Use an indirection route (`/artwork/[quoteId]`) that mints a short-TTL (60-300s) URL at click time and re-checks `auth.protect()` at that moment — this also means revoking a departed team member's Clerk account immediately kills their file access, which a page-render-time URL would not.
- **`download: <original filename>` (or the `Content-Disposition: attachment` equivalent) is mandatory on every artwork URL**, not optional UX polish — SVG is an executable document format; rendering it inline from the storage origin is a stored-XSS vector opened directly by FR-1's own accepted-format list.
- **Service-role client:** create Supabase **secret keys** (`sb_secret_...`) on day one, not legacy `anon`/`service_role` JWTs (deprecated, scheduled for removal end of 2026); lazy singleton factory (not a module-scope `const`, which bakes env vars in at build time); `persistSession:false`, `autoRefreshToken:false`, `detectSessionInUrl:false`.
- **Data model:** `quotes` (1) -> `quote_line_items` (N) as real tables, not JSONB arrays — this is what closes the F3 duplicate-row bug permanently, because a JSONB array element can't carry a foreign key for `artwork_files` to point at. Money as integer cents everywhere (the prototype's float math is finding §2.6's root cause). Stage as `text` + `CHECK`, not a native Postgres `ENUM` type, so stage values stay trivially alterable in a migration.
- **`proxy.ts` replaces `middleware.ts`** in Next.js 16 (the named export `middleware` is deprecated too); keep `clerkMiddleware()` bare with no `createRouteMatcher` route-gating — all real protection happens via `auth.protect()` in every resource/action instead.
- **Clerk v7/Core 3 syntax:** `<Show when="signed-in">` replaces `<SignedIn>`/`<SignedOut>`/`<Protect>`; `auth.protect()` now returns 401 (was 404); `auth()` is async and `.protect()` is a property on it, not a method on its result (`auth().protect()` is the wrong, deprecated v5 form).
- **Zod v4 syntax and discipline:** `z.email()` not `.string().email()`; `z.record(key, value)` needs two arguments; use `.strict()` everywhere on inputs so an injected `total` field is a loud, testable **rejection**, not a silent strip (plain `z.object()` strips unknown keys without complaint, satisfying FR-5's letter while giving zero signal that someone tried).
- **Email delivery uses `after()` from `next/server`**, not an inline awaited call — this lets the response return as soon as the DB commit is confirmed, with the Resend attempt (and its own independent error handling) happening afterward and never able to affect the customer-facing result.
- **Pin `npm:@typescript/typescript6@6.0.2`**, not `typescript@latest` (which resolves to 7.0.2 — a version Next.js 16.2's TypeScript detection does not yet support).

---

## Automated Guards (Verification Criteria)

Every researcher independently proposed **structural**, not disciplinary, protections — mechanisms that make the wrong outcome unrepresentable or un-shippable rather than relying on a developer remembering a rule. These should become literal verification/acceptance criteria for the phases that introduce them.

| Guard | What it prevents | Mechanism | Source(s) |
|---|---|---|---|
| **`must()` unwrapper for every `supabase-js` call** | The "returns `{data,error}`, doesn't throw" trap reproducing false-success (§2.4) | `export function must<T>(res): T { if (res.error) throw new DbError(...); return res.data }` — every DB call site goes through it | PITFALLS Pitfall 3 |
| **CI rule confining `db.from(`/`supabase.from(` to `lib/db/`/`server/db/`** | An unwrapped call slipping in anywhere else in the app | `grep`/lint rule failing the build if the pattern appears outside the DAL directory | PITFALLS Pitfall 3 |
| **DB CHECK constraints making false-success/unattributed-action unwritable** | Path recorded with no bytes; a "team" audit event with no actor | `uploaded_requires_server_evidence` (status can't be 'uploaded' without server-observed `byte_size`/`uploaded_at`); `team_actions_are_attributed`; `quotes_total_is_sum`; `line_total_is_consistent` | ARCHITECTURE §1 (DDL) |
| **Append-only trigger on `quote_events`** | Audit history being editable after the fact | `forbid_mutation()` trigger raising an exception on `UPDATE`/`DELETE` | ARCHITECTURE §1 |
| **Post-build grep of `.next/static` for price-list / secret sentinels** | Pricing constants or secret keys reaching the client bundle (FR-5's own literal acceptance criterion, automated) | A dedicated sentinel string (e.g. `hph_pricelist_v1_must_never_ship_to_browser`) grepped in CI post-build; separately, key-shape patterns (`sb_secret_`, `sk_live_`, `re_...`) grepped across bundle output | STACK §4.4, PITFALLS Pitfall 5 & 7 |
| **Verify-then-commit via `storage.info(path)` before any DB write claims artwork exists** | Path recorded, bytes never landed (Pitfall 4 — "the new §2.1") | Server independently HEADs/`.info()`s the object and writes `byte_size`/`uploaded_at` only from that observation, never the client's claim | STACK §5.2, ARCHITECTURE §4, PITFALLS Pitfall 1 & 4 |
| **Standing invariant SQL query, run nightly + in CI** | Silent drift between `quotes.artwork_path` and what's actually in Storage | `SELECT ... WHERE artwork_path NOT IN (SELECT name FROM storage.objects ...)` — must return zero rows, forever; should **alert**, not just log | PITFALLS Pitfall 4 & 11 |
| **CI check that every Server Action file references `auth`, with one documented exception** | An action shipping with no auth check at all | Loop over `'use server'` files, fail if `auth` is absent, except the one deliberately-public `submitQuote` | PITFALLS Pitfall 2 |
| **Unauthenticated-invocation integration tests asserting rejection *and* no side effect** | A test that checks "it rejected" without checking the row count/content is unchanged | Invoke every mutating action with no session cookie; assert both | PITFALLS Pitfall 2 |
| **Concurrency test for the idempotency key** | Double-click / double-submit producing two rows | Fire `submitQuoteAction` twice in parallel with the same `submission_id`; assert `count(*) = 1` | PITFALLS Pitfall 8 |
| **Quote/store equality fixture test** | Displayed estimate and stored total drifting apart | For N fixtures spanning every decoration type, rush/non-rush, and each size-tier boundary +/-1: assert `previewTotal(x) === createQuote(x).total === db row` | PITFALLS Pitfall 7 |
| **Byte-identity E2E on artwork round-trip** | "It opened" standing in for "it's correct" | Upload a fixture file, download it, assert SHA-256 equality — FR-1's own acceptance test, made automatable | PITFALLS Pitfall 4, ARCHITECTURE §4 |
| **`npm audit --audit-level=high` in CI** | Shipping a known-vulnerable Clerk/Next version (two CVSS-9.1 CVEs in 13 months) | Fail the build on high-severity advisories; pin version floors (`next >= 15.2.3`, `@clerk/nextjs >= 6.39.2`/`>=7.2.1`) | PITFALLS Pitfall 6 |
| **SVG/script-injection E2E + `dangerouslySetInnerHTML` grep** | Stored XSS via artwork or unescaped output (§2.7 reproduced) | Submit `<script>` in customer name, assert literal text; upload SVG containing `<script>`, assert forced download with no `image/svg+xml` content-type | PITFALLS Pitfall 9 |

---

## Client Input Required (By Milestone)

Concrete numbers and decisions only the shop owner can supply. These block completion (not necessarily *start*) of the milestones noted.

| Input needed | Why it blocks | Blocks |
|---|---|---|
| **Rush surcharge — multiplier or flat fee** | Rush is *detected* correctly (< 14 days) in the prototype and in the proposed schema (`is_rush`, `rush_fee_cents`, `rushMultiplierBps`), but no fee has ever been attached to it. The column/field exists; the number doesn't. | Milestone 2 (Pricing engine) |
| **Quantity price-break tiers and per-tier, per-colour rates** | The defining economic feature of screen-print pricing; industry bands are known (12/24/48/72/144/288) but the shop's actual dollar figures are not. | Milestone 2 |
| **Size upcharge amounts (2XL/3XL/4XL/5XL+, youth credit if any)** | Structure is well-established (~$2/$3/$4/$6+); exact figures are the shop's own. | Milestone 2 |
| **Per-screen setup fee** | Industry band is $15-30/screen; the prototype's flat $30 is wrong by 2-4x on real jobs and conflates three different fees. | Milestone 2 |
| **One-time digitizing fee** | Industry band $40-100; must be tracked as a distinct, non-recurring fee (different lifecycle from screen setup). | Milestone 2 |
| **Whether the shop runs DTF (direct-to-film transfers)** | If yes, it's a third decoration type with a materially simpler price model, and it's the natural home for below-MOQ requests. If no, record the answer so it isn't rediscovered mid-build. | Scope decision before Milestone 2 build begins (decoration-type enum) |
| **Payment-status tracking — build it or explicitly drop it** | PRD §2 already promises team members see "payment status"; no FR currently covers it. In scope per `PROJECT.md` (tracking != processing) but needs an explicit yes/no, not a default. | Scope decision before Milestone 6 (or earlier, if it affects schema) |
| **Artwork retention policy** | Already an open question in `PROJECT.md`. Non-blocking for build (the orphan-sweeper can ship with a configurable age threshold now), but the policy itself determines the eventual threshold and interacts with the logo-on-file reorder-registry feature (retention = what makes reorder-quoting speed possible). | Non-blocking; informs Milestone 3 (sweeper config) and Milestone 6 (policy applied) |
| **Confirm "Marketing Materials" stays a decoration type** | Present in the prototype (banners, business cards, signage — flat per-unit, no size grid); absent from the PRD's explicit decoration-type discussion. | Scope decision before Milestone 2 (decoration-type enum) |
| **Confirm old Supabase project data is unrecoverable** | Already assumed in `PROJECT.md`; stated there as needing client confirmation. | Non-blocking; confirm before Milestone 1 closes out (informs whether any migration/import work is needed at all) |

---

## Contradictions & Open Questions

Points where the four research documents disagree, or where one document's output isn't yet reconciled with another's.

### 1. When to prove the signed-upload-URL path works: Milestone 1 (spike) vs. Milestone 3 (full build)

**STACK.md** frames this as something to prove during the actual build: *"Prove the raw-`fetch` path works in Milestone 3 before committing to it; it is the only shape that satisfies both FR-1 and the 'no Supabase key in the browser' constraint."* (Confidence: MEDIUM-HIGH — the URL shape is verified from source, but the end-to-end browser behavior is not.)

**PITFALLS.md** argues this is too late: *"the upload architecture (bucket, size limit, signed-upload-URL action) must be decided and spiked in Milestone 1, because discovering the 4.5 MB wall during Milestone 3 invalidates the form built in Milestone 2."*

**These are not fully reconcilable as written, and PITFALLS' sequencing reasoning is stronger.** STACK's position only says *prove the raw-`fetch` path* — a narrow technical detail — before *committing* to it as the final shape; it doesn't argue for delaying the whole architecture decision. PITFALLS is making the larger claim: even the *decision* to use this architecture (as opposed to, say, TUS/resumable uploads, or discovering that a single-shot PUT is unreliable on shop-office broadband) needs to be de-risked before Milestone 2's quote form gets built on top of an assumption that hasn't been proven. **Recommendation for the roadmap: treat this as one item — a small, time-boxed upload spike in Milestone 1 (prove the signed-URL PUT works end-to-end against a deployed preview, not just `next dev`) — with STACK's "prove in Milestone 3" language understood as the full implementation-and-verification pass, not the first proof of feasibility.** ARCHITECTURE's own build order is silent on this specific timing question (it schedules the full artwork implementation in its "Phase 3," consistent with STACK, but doesn't argue against an earlier spike either).

### 2. The production-board stage model: FEATURES recommends 5 stages; ARCHITECTURE's schema has 4

FEATURES.md, after checking the prototype and PRD's 3-column model against the tool's own stated purpose ("what's stuck, who do we call"), recommends a 5-value stage enum that explicitly splits out **Awaiting Approval** (blocked on customer) and **Awaiting Blanks** (blocked on supplier) as distinct, visible stages — arguing the PRD's 3-column model can't represent either stall.

ARCHITECTURE.md's actual migration DDL, however, still encodes the **original** PRD model: `check (stage in ('pending','prepress','production','ready'))` — 4 values, with no Awaiting-Approval or Awaiting-Blanks state. ARCHITECTURE was not working from FEATURES' output when it wrote this constraint.

**This needs reconciliation before Milestone 1's migration is finalized.** FEATURES' own dependency analysis already states the consequence if it isn't caught early: *"Stage enum changes belong in Milestone 1's schema, not Milestone 4. Adding enum values later is cheap; adding them after the board UI is built means rebuilding the board."* Recommend the roadmap treat FEATURES' 5-stage model as authoritative (it's argued directly from the PRD's own stated purpose) and update the Milestone 1 migration's CHECK constraint accordingly — this is a small schema change if caught now, a board rebuild if caught in Milestone 4.

### 3. FR-1's acceptance criterion may not be fully verifiable at the end of Milestone 3 as currently scoped

ARCHITECTURE flags that FR-1's stated exit condition ("a team member opens that job and downloads a byte-identical file") requires a dashboard detail view to click "download" from — but the dashboard is Milestone 4's deliverable, not Milestone 3's. Two resolutions are offered, neither adopted yet: pull a minimal job-detail page forward into Milestone 3, or move the download half of FR-1's criterion to Milestone 4. **Undecided — flagged for roadmap time**, per ARCHITECTURE's own recommendation.

### 4. Lower-confidence items worth re-checking rather than trusting outright

- **TypeScript 7 vs. Next.js 16 compatibility** (STACK §7.6, confidence MEDIUM-HIGH) — sourced from a GitHub discussion thread, not an official compatibility statement; Next 16.3 was in preview at research time and may resolve this. Recheck at build start.
- **Vercel's Supabase Marketplace integration auto-injecting `NEXT_PUBLIC_`-prefixed variables** (STACK §2, confidence LOW) — not verified this pass. If that integration path is used, audit the injected variable list immediately and delete any `NEXT_PUBLIC_SUPABASE_*` entries.
- **Exact version floor for the Clerk GHSA-vqx2-fgx2-5wq9 fix** (PITFALLS, confidence MEDIUM-HIGH) — corroborated by three secondary sources but not the primary advisory directly in this pass; re-verify the exact patched version at install time.
- **Browser raw-`fetch` PUT to a signed upload URL, with no Supabase client at all** (STACK confidence MEDIUM-HIGH; ARCHITECTURE confidence MEDIUM-HIGH; PITFALLS confidence MEDIUM) — the URL shape is verified from package source, but the request headers/behavior needed for a reliable single-shot PUT (vs. needing TUS/resumable uploads above ~6 MB) is not yet proven end-to-end. This is exactly what the Milestone 1 spike (see Contradiction #1) should resolve.

---

## Implications for Roadmap

Based on combined research, the PRD's own 6-milestone structure (§8) is directionally sound but needs the amendments captured above folded in. Suggested structure:

### Phase 1: Foundation
**Rationale:** Everything else depends on the schema, auth pattern, and deploy pipeline existing first — and PITFALLS' analysis shows this phase carries substantially more weight than "scaffold + auth," because five of its named pitfalls (unprotected actions, unthrown DB errors, leaked secrets, middleware CVEs/matcher gaps, and preview-environment PII leakage) are all Milestone-1 preventions, not later fixes.
**Delivers:** Next.js scaffold with `proxy.ts` (bare `clerkMiddleware()`); Supabase project + migrations (`quotes`, `quote_line_items`, `artwork_files`, `quote_events` with the append-only trigger — reconciled to the **5-stage** model per Contradiction #2 — plus `deleted_at` and `intake_session_id`/`submission_id` from day one); RLS deny-all; storage bucket with mime/size limits; the DAL scaffolding (`must()` helper + the `auth.protect()`-first pattern); CI secret-grep and CI DAL-confinement rule; Vercel Deployment Protection + per-environment variable scoping (Preview never points at production Supabase); Resend domain verification **started** (DNS propagation is wall-clock time); and the artwork-upload architecture **spiked end-to-end against a deployed preview** — not just proven locally.
**Addresses:** FR-2, FR-3 (schema half).
**Avoids:** Pitfalls 1 (spike only, not full build), 2, 3, 5, 6, 13.

### Phase 2: Pricing engine + secure quote intake
**Rationale:** The pricing engine and the submission path are the actual product core, and per FEATURES' dependency graph, print locations and the dark-garment flag must land here — as pricing inputs — rather than be retrofitted onto existing rows later.
**Delivers:** The real pricing model (quantity tiers, size upcharges, per-screen setup, digitizing as a distinct one-time fee, dark-garment/underbase, rush surcharge actually applied — pending the client-input rate-card numbers listed above); print locations with per-location colours; garment colour restored to the intake form; `/api/estimate` as a Route Handler (debounced, `AbortController`-cancellable); `submitQuote` as an idempotent Server Action (`.strict()` Zod, `intake_session_id` uniqueness); the `quote.created` audit event written here (even though the audit **UI** waits for Phase 4).
**Addresses:** FR-4, FR-5 (fully specified), FR-6, FR-7 (amended), FR-8.
**Avoids:** Pitfalls 3, 7, 8 (schema half), 9 (escaping half), 12.

### Phase 3: Artwork pipeline
**Rationale:** Full implementation and verification of the architecture spiked in Phase 1.
**Delivers:** `createUploadTarget`/`finalizeUpload` actions; magic-byte sniffing; verify-then-commit; per-line-item + multi-file + `.zip` + realistic size cap; the orphan-sweep cron and the standing invariant query (written here even if its alerting/reporting surface ships later); a minimal job-detail view pulled forward if the roadmap resolves Contradiction #3 that way.
**Addresses:** FR-1 (fully).
**Avoids:** Pitfalls 1 (full build), 4, 9 (download-header half), 10, 11 (ordering half).

### Phase 4: Production board
**Rationale:** Nothing to render until Phase 2 produces real rows; stage semantics must already be correct from Phase 1's schema.
**Delivers:** The 5-stage board (New / Awaiting-Approval / Awaiting-Blanks / In-Production / Ready); bidirectional moves with confirmation modal; audit **UI** on top of the events already being written since Phase 2; job detail modal with artwork download (indirection-route pattern, short TTL, forced attachment); manual job entry; edit-an-existing-job; per-method production checklist carried forward from the prototype; proof-sent/approved-date fields.
**Addresses:** FR-9 (amended), FR-10, FR-11, FR-12, plus the manual-entry/edit/checklist/proof-tracking scope additions above.
**Avoids:** Pitfall 8 (idempotency verified against the leads view drift risk).

### Phase 5: Leads + notifications
**Rationale:** `needs_outreach` is a filter on `quotes`, not a table — this phase must not accidentally recreate the dual-write bug by giving leads their own table "because leads have different fields."
**Delivers:** Leads/outreach view; manual lead entry; MOQ soft-warning; new-quote email to the owner via `after()` + Resend (domain already verified since Phase 1); customer confirmation email.
**Addresses:** FR-13, FR-14.
**Avoids:** Pitfall 8 (the "drift back to two tables" failure mode specifically).

### Phase 6: Operational polish
**Rationale:** Everything here depends on data/attribution/board mechanics already being correct; nothing here is architecturally load-bearing for earlier phases.
**Delivers:** Calendar (in-hands vs. production date distinction applied); soft-delete/restore UI (the column has existed since Phase 1); payment-status booleans (pending the client scope decision above); time-in-stage/stall highlighting; metrics; artwork-retention policy applied to the sweeper's age threshold.
**Addresses:** FR-15, FR-16, success metrics from PRD §6.

### Phase Ordering Rationale

- Ordering follows the PRD's own stated 6-milestone shape, because ARCHITECTURE's build-order dependency graph and PITFALLS' pitfall-to-phase mapping both independently confirm the same skeleton is sound — the amendments are about **what's front-loaded within each phase**, not a different phase sequence.
- The single biggest sequencing risk is treating Phase 1 as "scaffold + auth" when PITFALLS' own analysis shows it must also carry the DAL discipline, the CI guards, the deployment protection, and the artwork-upload spike — skipping any of these to "get to the real features faster" is exactly how Milestone 3 becomes the point where the 4.5 MB wall is discovered instead of prevented.
- Two schema decisions (the 5-stage model, per Contradiction #2; and `deleted_at`/`intake_session_id` presence) must be locked in Phase 1's migration specifically because every later phase's queries assume them — this is stated independently by FEATURES and by ARCHITECTURE/PITFALLS respectively.

### Research Flags

**Needs deeper research/spike time during planning:**
- **Phase 1** — the artwork-upload spike specifically (exact `createSignedUploadUrl` -> browser `PUT` request shape, progress reporting, and whether a single-shot PUT at the chosen size cap is reliable enough or TUS/resumable uploads are needed). PITFALLS names this explicitly as the one area worth a short spike; everything else in Phase 1 is standard, well-documented Next.js/Clerk/Supabase pattern work.
- **Phase 3** — re-verification of the same upload flow under real artwork files (large layered `.ai`, `.eps` with embedded rasters) once the size cap and format list are finalized.

**Standard patterns (research-phase likely unnecessary):**
- **Phase 2** — Server Actions, Route Handlers, Zod v4, and the pricing-engine pattern are all well-documented, HIGH-confidence, directly-sourced patterns.
- **Phase 4** — board CRUD, confirmation modals, and audit-trail display are standard Next.js/RSC patterns with no open questions in any of the four research docs.
- **Phase 5** — Resend integration via `after()` is fully specified and HIGH-confidence.
- **Phase 6** — calendar/soft-delete/metrics are standard CRUD-adjacent work with no research gaps identified.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Every package version, peer constraint, and API signature was read directly from the npm registry, package source (`storage-js`, `supabase-js`, `resend`), or official upgrade guides — not training data. |
| Features | MEDIUM-HIGH | Pricing *structure* (tiers, upcharge shape, setup-vs-digitizing distinction) is corroborated across many independent shop-pricing sources — HIGH. Specific dollar figures are explicitly illustrative; the shop's real price list is a documented client input, not a research gap. Anti-features and stage-model reasoning are MEDIUM (reasoned from the project's own documented constraints, not external sources, but internally consistent). |
| Architecture | HIGH | All load-bearing platform claims (body-size limits, Server Action dispatch behavior, `server-only` build-time enforcement, signed-URL TTLs) verified against Next.js, Vercel, Clerk, and Supabase primary sources. One item (browser raw-`fetch` to a signed upload URL) is MEDIUM-HIGH and explicitly flagged for a Milestone 1 spike. |
| Pitfalls | HIGH | Official Vercel/Next.js/Clerk/Supabase docs, current as of June-July 2026, plus two independently corroborated CVEs (CVSS 9.1 each). A small number of items (Supabase MIME-validation spoofability, the exact Clerk CVE version floor, Resend's free-tier send caps) are MEDIUM or explicitly unverified and flagged as such. |

**Overall confidence: HIGH.** This is an unusually strong research cohort — the same conclusions were reached independently from source code, official docs, and security advisories across all four passes, and every meaningful contradiction between the docs (the M1-vs-M3 spike question, the stage-model mismatch) is a *scheduling or reconciliation* disagreement, not a factual one.

### Gaps to Address

- **Rate-card dollar figures** (rush multiplier, tier breakpoints/prices, size upcharges, per-screen fee, digitizing fee) — client input, blocks Phase 2 completion. See Client Input Required.
- **Whether the shop runs DTF** — undecided; affects the decoration-type enum's scope before Phase 2 build begins.
- **Payment-status FR** — PRD promises it, no FR covers it; needs an explicit build-or-drop decision.
- **Artwork retention policy** — open in `PROJECT.md`; non-blocking (hooks/config exist from Phase 3) but should be resolved before Phase 6.
- **Old Supabase project data** — assumed unrecoverable; needs client confirmation before Phase 1 closes out.
- **Stage-model reconciliation** (Contradiction #2) — FEATURES' 5-stage recommendation vs. ARCHITECTURE's 4-stage DDL must be resolved before the Phase 1 migration is finalized.
- **Upload-architecture spike timing** (Contradiction #1) — resolve by treating the Milestone-1 spike and the Milestone-3 full build as one continuous item, not competing recommendations.
- **FR-1's verifiability boundary at the end of Phase 3** (Contradiction #3) — decide whether to pull a minimal job-detail view forward or move the download criterion to Phase 4.
- **TypeScript 7 / Next.js 16.2 compatibility** — MEDIUM-HIGH confidence, sourced from a discussion thread; recheck at build start since a newer Next.js preview may have resolved it.
- **Vercel's Supabase Marketplace integration and `NEXT_PUBLIC_` injection** — LOW confidence, unverified this pass; audit immediately if that integration path is used.

---

## Sources

### Primary (HIGH confidence — official docs and source code, read directly)
- Next.js official docs: Upgrading to v16, `serverActions` config, `after()`, Data Security, Server/Client Components, Environment Variables (all doc version 16.2.12)
- Vercel official docs: Functions Limitations (updated 2026-07-01), Deployment Protection (updated 2026-06-26), Environment Variables (updated 2026-06-16), Sensitive Environment Variables (updated 2026-06-03)
- Clerk official docs (via `clerk/clerk-docs`): Core 3 upgrade guide, Migrate from `createRouteMatcher`, `clerk-middleware` reference, `auth()`/Server Actions reference, `<Show>` component reference
- Supabase official docs: Understanding API keys, Migrating to new API keys, Storage file limits, Standard/Resumable uploads, Serving/downloads, Creating buckets
- Zod v4 changelog; Resend Node SDK v5.0.0/v6.0.0 release notes
- Package source read directly: `@supabase/storage-js@2.111.0`, `@supabase/supabase-js@2.111.0`, `resend@6.18.1`
- npm registry (versions, dist-tags, peer/engine constraints), queried 2026-07-30

### Secondary (MEDIUM-MEDIUM-HIGH confidence)
- GHSA-vqx2-fgx2-5wq9 / CVE-2026-41248 (Clerk middleware bypass) and CVE-2025-29927 (Next.js middleware bypass) — corroborated across multiple independent security-vendor writeups
- Supabase issue trackers (`storage#576`, `supabase#27120`) on MIME-validation spoofability
- `vercel/next.js` discussion #95633 (TypeScript 7 support) and #50743/#84893 (Server Action dispatch behavior)
- Screen-print/embroidery pricing-structure sources (Printable Press, InkTracker, CraftsTrack, Branded Reno, MaggieFrames, Arklavo, Rise Digitizing) — structure corroborated across many independent shop guides; dollar figures explicitly illustrative
- Printavo's own published status-model guidance (production-stage recommendations)

### Tertiary (LOW confidence, explicitly flagged for follow-up)
- Vercel Supabase Marketplace integration auto-injecting `NEXT_PUBLIC_` variables — not verified this pass
- Resend free-tier daily/monthly send caps — not stated on the pages retrieved

### Project documents (read directly)
- `PRD-hustle-print-hub.md`, `.planning/PROJECT.md`, `ASSESSMENT.md`, `index.html` (the prototype being replaced)

---
*Research completed: 2026-07-30*
*Ready for roadmap: yes*
