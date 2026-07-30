# Phase 1 — Foundation & Access Control

**Date:** 2026-07-30
**Status:** Approved, ready for implementation planning
**Branch:** `develop`

**Upstream:** `.planning/ROADMAP.md` (Phase 1), `.planning/REQUIREMENTS.md` (FOUND-01…06, AUTH-01…05), `.planning/PROJECT.md` (locked decisions), `ASSESSMENT.md` (verified prototype failures), `.planning/research/ARCHITECTURE.md` (full DDL)

---

## 1. Goal

Three people sign in to a trustworthy empty shell. Nobody else gets in, the database has room for the whole story from day one, and the riskiest unknown in the build is already proven.

Phase 1 carries more than "scaffold + auth" deliberately. Five prototype-class failures are prevented here rather than fixed later: unprotected Server Actions, unthrown database errors, leaked secrets, client-side-only auth gating, and preview environments pointed at production data.

## 2. Requirements covered

| ID | Requirement |
|---|---|
| FOUND-01 | Complete schema in the first migration, including `quote_events`, `deleted_at`, `actor_id` |
| FOUND-02 | All database access through a DAL with an error-unwrapping helper that throws |
| FOUND-03 | Production build grepped for rate-card values; any hit fails the build |
| FOUND-04 | Vercel deployment protection enabled, env vars scoped per environment |
| FOUND-05 | Direct browser-to-Storage upload proven against a **deployed preview** |
| FOUND-06 | Resend sending domain verification submitted |
| AUTH-01 | Each team member signs in with their own individual account |
| AUTH-02 | Every dashboard route requires auth and returns no data when signed out |
| AUTH-03 | Every dashboard Server Action calls `auth.protect()` as its first statement |
| AUTH-04 | `?admin=true` and the triple-click handler absent from the codebase |
| AUTH-05 | Accounts carry an `owner` or `staff` role, resolved server-side |

## 3. Decisions made during design

| Decision | Choice | Rationale |
|---|---|---|
| Role storage | **Clerk `publicMetadata`** | Clerk already owns identity. A separate `staff` table creates a sync problem — delete a Clerk user and an orphan row keeps granting pricing access. Josh can grant access himself from the Clerk dashboard without a deploy. Session claims avoid the Clerk API rate limit flagged for per-request user lookups. |
| Artwork size cap | **~100 MB**, provisional | Covers most real designer files including layered `.ai`. Confirm with the shop owner before Phase 3 closes; the spike produces a real number to bring him. |
| Guard runner | **npm scripts invoked from `prebuild`** | GitHub Actions is blocked pending owner access on Josh's repo. Scripts run identically from a Vercel build, a local shell, or Actions later — zero rework when access lands. Guard logic in workflow YAML would leave Phase 1 with no enforcement at all. |
| Guard scope | **Three custom scripts only** | CodeRabbit and Semgrep already cover secret scanning and SAST locally, on demand. Gitleaks, Trivy, OSV-Scanner, and pgTAP are disproportionate for a three-person internal tool. |
| Branch strategy | **Work on `develop`** | `main` stays at Josh's live `index.html`. Local `main` was reset to `origin/main` so an accidental push cannot carry rebuild work into the live site. Branch protection deferred — it needs owner access. |

## 4. Data model

All tables ship in the **first migration**. Audit and soft-delete columns are load-bearing from the first query written even though their interfaces land in Phases 5 and 7 — a quote created before `quote_events` exists has no creation event, so its history starts mid-story and time-in-stage has no `t=0`.

### Tables

- **`quotes`** — one row per submission. Sequence-backed `quote_number`, replacing the prototype's `"HPC-" + random(1000..9999)`, which collides roughly 50% of the time by the 112th quote. Two jobs sharing a number means the wrong job gets pulled.
- **`quote_line_items`** — N per quote, relational.
- **`artwork_files`** — foreign key to a **line item**, because artwork is per-garment.
- **`quote_events`** — audit log: actor, action, timestamp, before/after.

### Why line items are relational, not JSONB

The prototype's duplicate-record bug has **two independent causes**:

1. A dual write across `app_orders` and `app_manual_leads`
2. A loop at `index.html:739` enclosing *both* inserts — so a 3-garment quote produced **6 rows**

Consolidating to one table fixes the first. Only a real parent/child relationship fixes the second. Additionally, `artwork_files` needs a foreign key to a specific line, and you cannot place a foreign key on an element inside a JSONB array — storing an array index or client-minted id in a blob with no constraint reintroduces exactly the orphaned-reference class of bug this rebuild exists to eliminate.

### Shape decisions

| Aspect | Choice | Reason |
|---|---|---|
| Money | **Integer cents** | The prototype used floats (`totalCount * 11.50`). Cents removes rounding drift. |
| Size breakdown | **JSONB inside the line item** | ~12 sparse keys, always read and written as a unit, never queried individually, and legitimately shaped differently for garments vs hats vs flat goods. |
| `stage` | **`text` + `CHECK`**, seeded `pending` / `production` / `ready` | The shop's existing three stages. `text` + `CHECK` keeps changing the set a one-line migration if that ever changes. |
| Soft delete | `deleted_at` from migration 1 | Every board query needs `where deleted_at is null` from the first query written. |
| Attribution | `actor_id` from migration 1, plus `actor_name` snapshotted into events at write time | The snapshot means audit display needs no Clerk API call per row. |
| RLS | **Default-deny on every table, enabled explicitly in the migration** | Not via a hidden event trigger. The prior project had an `ensure_rls` event trigger silently mutating DDL behavior; it was removed during teardown. Invisible DDL hooks produce baffling bugs. |

### Lead handling

"Needs a callback" is a **flag on a quote**, not a separate table. The prototype's duplicate-outreach bug came from splitting `app_orders` and `app_manual_leads` and writing to both. Giving leads their own table "because leads have different fields" is exactly how that returns.

## 5. Role model

Two functions, and only two:

```ts
getRole(userId)                  // reads Clerk publicMetadata — the ONLY place role is resolved
serializeQuoteFor(role, quote)   // the ONLY place money fields are stripped from a payload
```

This indirection is the design. It makes later changes one-line edits rather than hunts:

- **Add another person who sees pricing** → toggle their Clerk metadata, no deploy
- **Ungate pricing entirely** → `serializeQuoteFor` strips nothing
- **Gate only some fields** → edit the field list in one place
- **Replace the storage backend** → rewrite the body of `getRole`; call sites don't move

The failure mode being avoided is `if (role === 'owner')` scattered across thirty components, where every change becomes a search.

### Enforcement

Money fields are **omitted from the server payload** for `staff` accounts. They are never sent and conditionally hidden — conditionally-rendered values remain readable in the browser's network tab, which is `?admin=true` in a new costume.

### Server Action authorization

`await auth.protect()` is the **first statement of every dashboard Server Action**. Clerk middleware cannot protect Server Actions: actions are POST endpoints dispatched by action ID, and middleware matches paths. Clerk now ships a migration guide away from `createRouteMatcher()` for this reason. Public actions live in a separate directory from dashboard actions so the distinction is reviewable at a glance.

## 6. Guards

Three scripts wired to `prebuild`. A failure fails the Vercel build, and a failed build ships nothing.

| Script | Prevents | Requirement |
|---|---|---|
| `guard:bundle` | Rate card or key shapes reaching `.next/static` | FOUND-03, PRICE-10 |
| `guard:actions` | A dashboard Server Action shipping without `auth.protect()` | AUTH-03 |
| `guard:dal` | `db.from(` outside `lib/db/` — the unchecked-error bug returning | FOUND-02 |

### What `guard:bundle` greps for

**Not price numbers.** Grepping `.next/static` for values like `12.75` false-positives endlessly — those digits occur naturally in minified output. Instead:

1. **A sentinel constant** exported from the price-list module, e.g. `HPH_PRICELIST_SENTINEL = 'hph_pricelist_v1_must_never_ship_to_browser'`. Its presence anywhere in the client bundle is a hard failure. Because it travels with the module, it catches the case `server-only` misses: a shared barrel file re-exporting the price list into a client component's graph.
2. **Key shape patterns:** `sb_secret_`, `sk_live_`, `sk_test_`, `re_`, and the legacy Supabase JWT prefix `eyJhbGciOiJIUzI1NiIs`.

`server-only` catches direct imports at build time; the sentinel catches indirect ones. Both are needed.

Plus `tsc --noEmit`, `next lint`, and husky + lint-staged for formatting. The husky/lint-staged split mirrors the `4-kingdom` repo: fast formatting locally, everything slower at the gate.

**Deliberately excluded:** Gitleaks, Trivy, OSV-Scanner, Semgrep in CI, pgTAP isolation suites, GitHub Actions. CodeRabbit and Semgrep run locally on demand. pgTAP cross-tenant isolation exists to prove multi-tenant separation; this app has one tenant.

## 7. Error handling

`supabase-js` returns `{data, error}` and **does not throw**. That is `ASSESSMENT.md` §2.4 — failed writes rendering a success screen — one missing destructure away from returning in the new stack.

Every database call goes through a `must()` unwrapper that throws on error. `guard:dal` enforces that nothing bypasses the DAL. The protection is structural, not a promise to remember.

The Supabase client is a **lazy getter, not a module-scope `const`**. A module-scope `createClient(...)` evaluates during the build, baking in env vars and failing confusingly when one is missing. The `4-kingdom` repo documents hitting exactly this with `new Resend(...)` at import time, requiring placeholder env vars in CI purely to let the build finish.

## 8. The upload spike (FOUND-05)

### What it proves

A Server Action mints a signed upload URL; the browser PUTs bytes to it with a plain `fetch` — no Supabase client, no key, only a capability token scoped to one server-chosen path. At ~100 MB. On a **deployed preview URL**.

Never `next dev`. Neither the Vercel 4.5 MB function body cap nor the Next.js 1 MB Server Action cap is enforced locally, so a passing local test proves nothing and the failure appears first in production.

### What it deliberately does not build

No UI beyond a bare file input, no database rows, no validation, no cleanup logic. Throwaway proof living on the branch, absorbed or deleted in Phase 3.

### Success bar

1. A ~100 MB file uploads from a browser on a deployed preview
2. The server independently confirms the object exists with matching byte size — never trusting the client's report of success
3. A signed download returns byte-identical content
4. **Three consecutive successful runs**, not one
5. Wall-clock time recorded on a normal connection

Criterion 5 is a product finding, not a technical one. If 100 MB succeeds but takes eight minutes on the shop's upload speed, customers abandon the form long before it completes, and the owner needs to hear that.

### The fork it produces

The spike cannot fail in a way that blocks Phase 1. It produces information:

- **Single-shot PUT is reliable at 100 MB** → Phase 3 is one `fetch`
- **It is not** → Phase 3 needs a resumable (TUS) client with chunking, progress, and resume state

Either outcome is known before the quote form is built on top of it. That is why this moved out of Phase 3 — discovering the 4.5 MB wall mid-Phase-3 would invalidate the form built in Phase 2.

## 9. Testing

Phase 1 is mostly wiring, so there is little business logic to unit test. The tests that earn their keep are on the guards:

**Plant a violation, confirm the build fails, remove it, confirm it passes.**

- Add a rate-card constant to a client component → `guard:bundle` fails
- Add a dashboard action without `auth.protect()` → `guard:actions` fails
- Add `db.from(` outside `lib/db/` → `guard:dal` fails

A guard nobody has watched fire is indistinguishable from a broken one. That is how `?admin=true` felt safe for months.

Migrations are verified by applying them to an empty database and confirming they succeed from scratch.

## 10. Definition of done

- [ ] Three team members sign in with individual accounts; signed out, every dashboard route returns no customer data
- [ ] `?admin=true` grants nothing, and neither it nor the triple-click handler exists anywhere in the codebase
- [ ] A ~100 MB file uploads from a browser directly to Storage on a deployed preview, three times consecutively
- [ ] Resend sending domain verification is **submitted** (in flight is sufficient; DNS propagation is wall-clock time)
- [ ] Planting a rate-card constant in client code fails the build
- [ ] All migrations apply cleanly to an empty database
- [ ] Vercel deployment protection enabled, env vars scoped separately for preview and production

## 11. Dependencies

External, with owners. None block design; several block execution.

| Needed | From | Blocks |
|---|---|---|
| Clerk application + publishable/secret keys | Chris — create at `dashboard.clerk.com` | First `npm run dev` |
| Vercel env var access, preview scope | Josh — team access | FOUND-04, the spike's deployed run |
| Resend account + API key | Chris | FOUND-06 |
| GoDaddy access for the subdomain | Josh | Custom domain, not Phase 1 |
| GitHub owner access | Josh | GitHub Actions, branch protection — deferred by design |
| Confirm ~100 MB cap | Josh | Phase 3 sign-off, not Phase 1 |

## 12. Out of scope for Phase 1

Quote form, pricing engine, artwork feature beyond the spike, production board, leads view, notifications beyond domain verification, calendar, soft-delete UI, payment status. All are later phases with their own requirements.
