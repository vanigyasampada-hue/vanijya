# Vanijya Sampada — Commercial Complex and Shop's Lease Management System

(Formerly CCSMS / CCSLMS — renamed per client instruction, spec v4.4.)

Cloud-first, monorepo implementation per `01_Tech_Stack_Recommendation.docx`
(React + Ant Design frontend, Node/Express/Prisma/PostgreSQL backend, Redis,
MinIO, BullMQ, Docker).

**Status: all 7 backend phases and all 7 roles' frontend dashboards are
built and working**, including the 9 MIS reports. What's genuinely left
is UAT/load testing (Phase 7 — needs real infrastructure, not more code)
and a couple of deliberately-left gaps documented below, not hidden.

## Current status against the specification (authoritative)

The rest of this README is a **chronological build log**; early "not built
yet" notes may have been resolved later, and those are marked
**[RESOLVED later]**. The authoritative, evidence-based record is
**[`docs/SPEC_TRACEABILITY.md`](docs/SPEC_TRACEABILITY.md)**, a
section-by-section trace of *Specification v4.4*.

**Every functional section of v4.4 is built.** The one deliberate
deviation is the **platform** (React/Express/Prisma/Docker rather than
Part VI's Next.js/Supabase/Vercel); the functions Part VI asks of that
platform are provided on this one: live refresh, diagnostics, and
point-in-time recovery.

**Every go-live decision is made and built** — see
**[`docs/DEPLOYMENT_DECISIONS.md`](docs/DEPLOYMENT_DECISIONS.md)**:
- built-in captcha, so no key is needed;
- NIC SMS gateway, with the DLT registration text supplied;
- direct-to-ULB UPI, so no gateway is needed;
- the notice layout adopted as official;
- provisional NAC quotas;
- one-time merchant passwords;
- a CI-verified baseline database migration.

**What only the department can do:**
1. Open an NIC SMS account and register the sender ID and the two DLT
   templates (the exact text is in the decisions document).
2. Provide a server, a domain, and an HTTPS certificate.
3. Ideally, have a native speaker read `kn.json`.

## Getting started

```bash
git clone <your-remote>
cd vanijya-sampada
cp .env.example apps/api/.env   # local dev: the API and Prisma read apps/api/.env
# (production with docker-compose.prod.yml uses a .env in the project root instead)

# 1. Generate the RS256 keypair for JWT signing
./scripts/generate-jwt-keys.sh

# 2. Start Postgres, Redis, MinIO
docker compose up -d postgres redis minio

# 3. Install deps, generate Prisma client, run migrations, seed
npm install --workspace=apps/api
npm run prisma:generate
npm run prisma:migrate
npm run prisma:seed

# 4. Run the API
npm run dev:api

# 5. In another terminal, run the frontend
npm install --workspace=apps/web
npm run dev:web
```

The seed script prints a working login: `da1_shiv` / `Admin@123`
(District Admin — forced to change password on first login, per spec).
It lands on a real, working dashboard immediately — use it to create a
ULB, then a ULB Admin, Case Worker, Bill Collector, Verifier–Revenue,
and Engineer user via the Users tab (or `POST /api/users` directly) to
see the rest. The Merchant dashboard needs a full onboarding chain
first: submit Shop Detail Verification with merchant name/mobile, run
it through Case Worker → Verifier–Revenue → ULB Admin, initiate
Merchant Creation, approve that link too, then log in via the Merchant
tab with the mobile number and the default password `Pwd@1234` (forced
change on first login).

```bash
curl -X POST http://localhost:4000/api/auth/login/official \
  -H "Content-Type: application/json" \
  -d '{"username":"da1_shiv","password":"Admin@123","captcha_token":"dev-skip"}'
```

(Captcha verification auto-skips outside `NODE_ENV=production` when
`CAPTCHA_SECRET_KEY` is unset — see `services/captcha.service.ts`. Never
leave it unset in staging/production.)

## Repo layout

```
vanijya-sampada/
├── apps/
│   ├── api/                  # Node + Express + TypeScript + Prisma
│   │   ├── prisma/
│   │   │   ├── schema.prisma # all 20 entities
│   │   │   └── seed.ts
│   │   └── src/
│   │       ├── routes/       # auth, profile, org structure, documents, complexes, shops,
│   │       │                 # merchants, rent-ledgers, payments, transfers, closures,
│   │       │                 # demolitions, deposit-settlements, notices, court-cases,
│   │       │                 # merchant self-service, reports, circulars
│   │       ├── controllers/
│   │       ├── services/     # one per entity/domain, plus reports.service (all 9 MIS
│   │       │                 # reports) and workflow.service (shared maker-checker helpers)
│   │       ├── jobs/         # rent-ledger.job (BullMQ: monthly generation, manual trigger)
│   │       ├── middleware/   # authenticate, require-role, ward-scope, rate-limit, validate, error-handler
│   │       ├── utils/        # jwt, tokens, password, username, audit
│   │       └── lib/          # config, logger, prisma, redis, queue, response, errors,
│   │                         # storage (MinIO), report-export (generic Excel/PDF renderers),
│   │                         # payment-links (UPI/Swikriti URL builders), webhook-signature (HMAC)
│   └── web/                  # React 18 + TS + Ant Design + Redux Toolkit
│       └── src/
│           ├── app/          # store, authSlice (access token, memory only), baseApi (RTK Query + silent-refresh reauth)
│           ├── features/
│           │   ├── auth/            # LoginPage (Official/Merchant tabs), ChangePasswordPage
│           │   ├── approvals/       # UlbAdminDashboard (10 tabs)
│           │   ├── shops/           # CaseWorkerDashboard + CommercialDetailsForm
│           │   ├── rent-collection/ # BillCollectorDashboard
│           │   ├── verification/    # VerifierRevenueDashboard (7 tabs) + generic VerificationQueue
│           │   ├── merchant-portal/ # MerchantDashboard
│           │   ├── engineering/     # EngineerDashboard
│           │   ├── district-admin/  # DistrictAdminDashboard
│           │   ├── reports/         # ReportsPage — generic viewer for all 9 MIS reports
│           │   └── dashboard/       # DashboardRouter (picks view by role)
│           ├── layouts/      # AppLayout (sider/header/logout)
│           └── router/       # ProtectedRoute (client-side gating, mirrors server-side requireRole)
├── packages/
│   └── shared/                # Shared TS enums/DTOs (RoleName, ApiResponse<T>, ...)
├── nginx/Dockerfile, nginx/templates/   (production web image + config)
├── docker-compose.yml
├── load-test/                  # k6 script for the 500-700 concurrent user NFR (unrun — see its README)
└── .github/workflows/ci.yml
```

## Backend — all 7 phases

| Phase | What's built |
| --- | --- |
| **1 — Auth & org structure** | Official login (username+password+captcha), Merchant login (mobile+password+captcha), RS256 JWT access tokens (15 min) with role/ulb/ward claims, opaque refresh tokens (7 day, HttpOnly cookie, hashed in Postgres+Redis), logout/revocation, forced/self-service password change, admin reset. District/ULB/Ward/User CRUD with username auto-generation (`bc1_bdvt`). RBAC middleware (`requireRole`, `wardScope`, `requireOfficial`/`requireMerchant`). Redis rate limiting (login lockout, refresh throttle, API throttle). Generic `WorkflowLog` audit trail. |
| **2 — Complex & Shop lifecycle** | §8.1/§8.2: Engineer creates → ULB Admin approves/returns/rejects → Engineer resubmits. §8.3 Shop Detail Verification: a *separate* 3-stage sub-workflow (Case Worker → Verifier–Revenue → ULB Admin) modeled as its own `commercial_status` field, distinct from the shop's own lifecycle status. MinIO file uploads for photos. |
| **3 — Merchant, Tender, Rent, Payment** | §8.4/§8.5 Merchant onboarding with automatic mobile+name matching and manual-mapping fallback. §8.12 Tender period captured alongside §8.3. §8.7 GST/interest engine — 2% compounded on the *full running pending balance*, verified against the spec's worked 3-month example, not just the single-month formula. Monthly BullMQ billing job, idempotent per shop+month. §8.6 Payment recording through the same 3-stage chain. |
| **4 — Transfer, Closure, Demolition, Settlement** | §8.9 Owner Transfer repoints the merchant link and carries the tender over. §8.10 Closure stops billing and auto-triggers Deposit Settlement. §8.8 Demolition/Alteration (2-stage) — Approve and Archive kept as genuinely separate actions. §8.11 forfeiture banded by tenure (50/30/20/10/0% at the 3/6/9/12-year marks), demolition-driven closures always 0%. |
| **5 — Notices & Court Cases** | §8.14/§8.15 two distinct notice workflows (different makers: Bill Collector vs. Case Worker) converging on the same chain, with a separate role-gated "Serve" action. Court Cases with no FK to a source Notice (escalation is explicitly manual per spec). §8.13 Archive & Fresh Entry needed zero new code — verified it's an emergent property of the existing state machine. |
| **6 — Reports & Merchant self-service** | All 9 MIS reports (`05_Report_Specifications.docx`) — DCB per Shop/Complex/ULB, Rent, Tender, Shop Details, Complex Details, Court Case, Notice — each supporting `?format=json/xlsx/pdf`. One generic Excel renderer and one generic PDF renderer (matching the spec's letterhead layout) drive all 9, rather than nine bespoke exporters. Merchant self-service (`GET /merchant/shops`, `/merchant/reports/{rent,notices,court-cases}`) re-derives shop access from the DB per-request rather than trusting a stale JWT claim. Circulars (public listing, staff upload). |
| **7 — UAT, load testing, hardening** | Not code — needs real infrastructure and real users. See "What's genuinely left" below. |

I built the full 20-entity schema up front rather than incrementally,
because retrofitting FK relationships mid-project is far more expensive
than getting the data model right once. Every phase above was "write
the service + routes against an existing table," not "design the table
and hope it doesn't conflict with what's already there."

## Frontend — all 7 roles, plus Merchant, plus Reports

| Role | Dashboard |
| --- | --- |
| **ULB Admin** | 10-tab approvals queue — Complexes, Shops, Merchant Links, Payments, Transfers, Closures, Demolitions, Notices, Court Cases, Deposit Settlements. Every entity that reaches this role for final sign-off. |
| **Case Worker** | The full set of Case Worker maker actions: §8.3 Shop Detail Verification (shops needing details / returned for correction, one shared form), §8.4 Merchant Creation (one-click initiate from a shop with approved commercial details), §8.9 Transfer (mobile-number merchant search), §8.10 Closure, and §8.15 Notice + Court Case creation. |
| **Bill Collector** | Ward-scoped rent collection (§8.6) — Unpaid/Partial ledger queues, a mode-aware Record Payment form, a "My submissions" tracker. |
| **Verifier–Revenue** | 7-tab verification queue — Shop details, Transfers, Closures, Payments, Notices, Court Cases, Deposit Settlements — all at the Checker stage. |
| **Engineer** | Complex and Shop creation (§8.1/§8.2, with Correct & Resubmit after a Return), plus Demolition/Alteration request creation (§8.8) — Alteration correctly locked to shop-only targets with a required revised rent, Demolition able to target either a whole complex or a single shop. |
| **District Admin** | ULB creation (their one unique power) and district-wide user management. No maker-checker queue — District Admin never appears as a Verifier or Approver in the workflow spec. |
| **Merchant** | Own shops, rent history, notices, court cases — plus real "Pay via UPI" / "Get Swikriti challan" links (§8.6) for outstanding rent, followed by recording the resulting payment through the same verify/approve chain as any official-recorded one. |
| **All roles** | A generic Reports page (`/reports`) — one report viewer drives all 9 MIS reports, with role-appropriate visibility, filters, and Excel/PDF download buttons. Column headers come from the API response, not hardcoded per report. |

Two design patterns did most of the work here: `VerificationQueue`
(one component, seven Verifier–Revenue tabs plus six of ULB Admin's ten)
and the reports module (one Excel renderer, one PDF renderer, one
frontend table, nine reports). Building the *pattern* once and letting
volume flow through it was consistently cheaper than building each
instance bespoke.

## Bugs found and fixed along the way

Building breadth surfaced real issues that narrower work never would
have exercised. In the order they were found:

1. **`GET /shops?status=X` silently dropped an explicit status filter**
   for any role without unapproved-record visibility (Case Worker
   included) — object-spread let the visibility guard's
   `{status:{notIn:[...]}}` clobber the caller's `{status:X}` because
   both used the same key. Fixed by combining both constraints through
   Prisma's `AND` array; verified the fix doesn't open a new hole
   (requesting a disallowed status now returns zero rows, not a bypass).
2. **A `?mine=true` filter almost shipped as `z.coerce.boolean()`** —
   which coerces via `Boolean(str)`, and `Boolean("false")` is `true`
   in JS. `?mine=false` would have silently *enabled* the filter.
   Caught before shipping; fixed by comparing against the literal
   string `'true'`.
3. **A real RTK Query endpoint-name collision**: `verificationApi.ts`
   defined `returnPayment`, but `approvalsApi.ts` already had one under
   that exact name — `injectEndpoints` shares one namespace across every
   file, so this isn't a compile error, it's a silent collision where
   one definition wins unpredictably. Found by diffing endpoint names
   across every frontend API file (now 10 files, 67+ endpoints, still
   zero collisions), fixed by reusing the existing mutation — which was
   also correct, since `/payments/:id/return` genuinely is one backend
   route for both the Verifier and ULB Admin stages.
4. **Engineer's ULB-scoping didn't match its own docstring.**
   `listComplexes()` claimed Engineer is scoped to their own ULB like
   ULB Admin; the code only actually restricted `ULB_ADMIN`. An
   Engineer from ULB A could see ULB B's pending complexes.
5. **`listShops()` had the identical gap**, for the identical reason —
   copy-pasted scoping logic, same blind spot both times.
6. **`getShop()` — the single-record lookup — had *no* ULB scoping at
   all**, not even for ULB Admin. The list endpoint restricted it
   correctly; fetching one shop directly by ID didn't, so anyone with
   (or guessing) a shop ID could pull it from any ULB, unapproved ones
   included. Fixed 4–6 consistently: Engineer now gets identical
   ULB-scoping to ULB Admin everywhere (both `access_tier: "ULB"` on
   the Role model — District Admin is correctly the only role meant to
   see across ULBs), and the single-record lookup enforces what its
   list counterpart already had.
7. **The login response never included `district_id`**, only `ulb_id`
   — nothing before District Admin's dashboard needed it, since every
   prior dashboard is ULB-scoped, not district-scoped. But District
   Admin can't create a ULB without knowing which district it belongs
   to. Fixed through the full chain: backend login response → frontend
   `AuthUser` type → both login mutations.
8. **A cache-invalidation shortcut caught before it shipped**: the new
   ULB-list query needed a tag to invalidate after creating a ULB, and
   the tag cache had no `'Ulb'` tag yet. Reusing the existing `'Ward'`
   tag would have worked today and caused confusing unrelated refetches
   later. Added the two-line proper fix instead.
9. **The Excel report exporter's summary row silently dropped 2 of 3
   values.** The first draft mapped every `[label, value]` summary pair
   to the *same* object key before calling `Object.fromEntries` —
   which keeps only the last entry when keys repeat. For DCB's 3-value
   summary (Total Demand/Collected/Balance), only "Total Balance" would
   have appeared in the exported spreadsheet. Fixed by combining all
   summary pairs into one string before writing a single cell.
10. **Report 7 (Complex Details) needed columns with no backing
    fields** — "Address" and "Construction Year" weren't on the
    `Complex` model at all. Added `address`/`construction_year` as
    optional fields rather than silently omitting the columns the spec
    asked for.
11. **§8.9 Transfer had no way to find the incoming merchant** —
    Case Worker needs to search for an already-approved merchant by
    mobile number, and no search endpoint existed at all (only creation
    and manual-mapping did). Added `GET /merchants/search` rather than
    building a frontend that could only ever transfer to a merchant
    already visible on-screen.
12. **The single-shop detail view was missing its active tender ID** —
    needed for the Closure form (`tender_id` is a required field,
    §8.10), but `getShop()`'s Prisma include only ever fetched
    `merchant_link`. Extended the include rather than making the
    frontend guess or issue a second request.
13. **Transfer/Closure/Demolition list endpoints had no way to filter
    to "my own requests"** — only a global `status` filter existed, so
    an Engineer or Case Worker would see every colleague's requests
    mixed in with their own on a multi-staff ULB. Added the same
    `mine=true` filter already used for Payments, to all three routes
    at once rather than fixing it for Demolition alone and leaving
    Transfer/Closure inconsistent — and used the same literal-string
    comparison as before, not `z.coerce.boolean()`, for the same
    `Boolean("false") === true` reason caught earlier.
14. **`GET /payments` had no role restriction at all — the single
    most serious finding in this build.** Only `authenticate` guarded
    it; any logged-in principal (an Engineer, who has no legitimate
    reason to see payment data, or — about to be made worse — a
    Merchant) could call it with no filters and see every payment in
    the entire system: amounts, shop IDs, transaction references,
    every merchant's rent history. Surfaced while wiring up Merchant's
    own payment access, which would otherwise have inherited the same
    unrestricted view. Fixed with a proper role allowlist plus a hard
    boundary for Merchant specifically: their own shop IDs are looked
    up server-side and forced into the query, not accepted as a
    trusted filter — a Merchant can never see another shop's payments
    no matter what `shop_id` they pass.
15. **A payment-recording endpoint change almost skipped an ownership
    check.** Extending `POST /payments` to accept Merchant as a valid
    initiator without also verifying the shop belongs to them would
    have let any merchant record a payment claim against *any* shop in
    the system. Added the ownership check (via an approved
    `MerchantShopLink`) as part of the same change, not as an
    afterthought.
16. **A real gateway-link feature had no destination account to pay
    into.** Building the UPI deep link revealed `ULB` had no field to
    hold a collection VPA at all — the link would have generated
    correctly and pointed nowhere. Added `ULB.upi_vpa` plus a
    `PATCH /ulbs/:id` endpoint (scoped so a ULB Admin can only touch
    their own ULB) rather than hardcoding a placeholder payee.
17. **HMAC verification needed raw bytes Express doesn't keep by
    default.** `express.json()` discards the original request body
    after parsing it into an object — re-serializing that object for
    signature verification would produce a different hash than what
    the gateway actually signed if whitespace or key order differ at
    all. Added a `verify` callback to capture `req.rawBody` globally
    rather than reaching for `JSON.stringify(req.body)` and getting an
    intermittent, hard-to-reproduce verification failure later.
18. **File upload validation trusted a client-controlled string.**
    `assertUploadable()` checked `req.file.mimetype` — the
    `Content-Type` a multipart request's client chose to send, not
    anything about the file's actual content. Trivially spoofable.
    Unlike every other entry on this list, this one wasn't found
    incidentally while building a feature — it was found by
    deliberately auditing file upload handling for a known
    vulnerability class. Fixed with real magic-byte detection
    (JPEG/PNG/WebP/PDF signatures, no external dependency needed) and
    verified directly: real files of each type correctly identified, a
    string starting with `<script>` and a Windows executable's `MZ`
    header both correctly rejected regardless of claimed
    `Content-Type`. Also fixed a smaller bug in the same function:
    it threw plain `Error` instead of `ValidationError`, so a
    legitimate rejection returned a generic 500 instead of a 422 with
    a clear message.

Seventeen of these eighteen were caught by building a feature that
happened to exercise the broken path, not by auditing code in the
abstract — that's the case for continuing to build breadth even after
"enough" features exist, since bugs wait for the right angle more
often than they wait to be found by inspection. The one exception
(#18) shows the other side of that coin: some vulnerability classes
are well-known enough that deliberately checking for them is worth
doing on a schedule, not just hoping a feature stumbles into them.

## Design decisions worth knowing about

- **RS256 over HS256** for JWTs — matches the spec's "asymmetric keys"
  requirement, and lets you hand the public key to another service
  without exposing signing capability.
- **Refresh tokens are opaque, not JWTs** — a compromised access token
  self-expires in 15 minutes; a compromised refresh token can be
  revoked server-side instantly (it's a DB row, not a self-contained
  credential).
- **The frontend's access token lives in Redux state only, never
  localStorage** — matching the spec's "Memory only ... never
  localStorage" requirement exactly. A silent `POST /auth/refresh` on
  page load restores the session via the HttpOnly cookie before
  rendering anything gated.
- **`WorkflowLog` is generic** (`entity_type`+`entity_id`+`action`)
  rather than one audit table per entity — one auditable event stream
  is easier to query across 15+ maker-checker entities than 15 tables.
- **Shop's `commercial_status` is a separate field from its lifecycle
  `status`** — two genuinely different workflows share one row: one
  governs whether the record is visible/usable, the other whether its
  rent figures are trustworthy enough to bill against. Conflating them
  would make "Active shop, still-unverified rent" unrepresentable — a
  state the spec explicitly allows.
- **Return/Reject added as real, queryable statuses** on
  `ComplexStatus`/`ShopStatus`/`TenderStatus`/`CourtCaseStatus` even
  where the spec text didn't enumerate them — a ULB Admin needs to
  filter "show me what I sent back," not reverse-engineer it from logs.
- **`MerchantShopLink` has its own `workflow_status`**, separate from
  `Merchant.workflow_status` — the account's approval only matters once,
  at creation; the *link* between a shop and a merchant needs its own
  gate every time, including when mapping an already-approved merchant
  to a second shop.
- **GST/interest compounding traced line-by-line against the spec's
  worked example**, not just the single-month formula — each month's
  2% is computed on the entire running pending balance, not the
  original unpaid rent alone. Getting this wrong compounds, literally,
  into real financial discrepancies months later.
- **Deposit Settlement has no creation endpoint** — system-computed as
  a side effect of an approved Closure, never something a user POSTs.
- **Demolition's Approve and Archive are deliberately two actions**,
  and Notice's Serve is deliberately separate from Approve — both
  preserve the distinction between "formally decided" and "actually
  done," which a report might need to tell apart.
- **Court Case has no FK to a source Notice** — escalation is an
  explicitly manual, disconnected action per the spec, not a pipeline.
- **The 9 MIS reports and their Excel/PDF exports are two generic
  renderers, not nine bespoke ones** — every report resolves to the
  same `{columns, rows, summary}` shape, so one Excel writer and one
  Puppeteer HTML template (matching the spec's letterhead layout
  exactly) serve all nine. The frontend mirrors this: one `ReportsPage`
  renders whichever report is selected using the column headers the
  API returns, not a hardcoded table per report.
- **The PDF footer uses Puppeteer's native `footerTemplate`, not an
  in-body HTML element** — the first draft had both (redundantly), and
  actually rendering it (see below) showed the in-body one was silently
  doubling every report to two PDF pages. One footer mechanism, not two.
- **Report 4's Rent Report is ledger-centric, not payment-centric** —
  the spec's own status filter values ("Unpaid/Partial/Paid") only make
  sense against `RentLedger.payment_status`; an unpaid ledger has no
  `Payment` row at all, so a query anchored on `Payment` could never
  have surfaced it. Reworked to match Report 1's shape instead of
  guessing at the original (wrong) design.

## Real payment links — what's genuinely real vs. what still isn't

Re-reading the source specs more carefully revealed that "payment
gateway integration" was a bigger gap than it needed to be. Both of
Vanijya Sampada’s payment "integrations" are explicitly specified as **links**,
not APIs:

- `03_API_Specification.docx` §10 spells out `POST
  /rent-ledger/:ledgerId/challan` returning `{ challan_url:
  "https://swikriti.link/..." }` — a redirect, not a payment call.
- `01_Tech_Stack_Recommendation.docx` describes UPI as "via Bill
  Collector validation workflow" and Swikriti as an "external link" —
  neither implies an API integration exists to build against.

So `lib/payment-links.ts` builds two **real** links, not mocks:

- **UPI**: a genuine `upi://pay?...` deep link per NPCI's public
  linking specification — no credentials needed beyond a payee VPA
  (added as `ULB.upi_vpa`, configurable via a new `PATCH /ulbs/:id`).
  Opening it on a phone with any UPI app launches a real payment
  screen pre-filled with the amount and reference.
- **Swikriti**: a well-formed redirect URL built from a configurable
  `SWIKRITI_PORTAL_BASE_URL`. This part is honestly flagged — the
  actual portal's query-parameter contract isn't in any source
  document available here, so the parameter *names* are a documented
  best guess, even though the endpoint contract and flow are correct
  per spec.

`GET /rent-ledgers/:id/upi-link` and `/:id/challan` generate these for
a specific ledger's outstanding balance, gated so a Merchant can only
generate a link for their own shop. Merchant's dashboard has real
"Pay via UPI" / "Get Swikriti challan" buttons that open these links,
followed by the same claim-and-verify recording flow as before.

## Payment webhook receiver — built, tested for real, still not connected

The one remaining piece of the payment gap that's genuinely code, not
a real gateway account, was the webhook *receiver* itself — the
endpoint any real gateway would call to confirm a payment. Built it
deliberately conservatively for a government financial system: `POST
/webhooks/payments/:gateway` verifies an HMAC-SHA256 signature (the
standard pattern most Indian PSPs use), then attaches corroborating
evidence (`gateway_reference`/`gateway_confirmed_at`) to a Payment
already awaiting review — it does **not** auto-approve or skip the
Verifier–Revenue → ULB Admin chain. A replayable HTTP callback
shouldn't be able to bypass human financial review on its own, webhook
signature or not.

A few things worth calling out about how this was built:

- **No `authenticate` middleware, on purpose.** The caller is an
  external gateway with no user session — the signature check *is* the
  authentication here, and unlike `captcha.service.ts`'s dev-mode skip,
  there's no environment where an unset secret means "trust it
  anyway." An unconfigured secret means the endpoint refuses everything
  (503), not that it falls back to unauthenticated.
- **Raw body capture required a real architectural change.** HMAC
  verification needs the exact bytes a gateway signed — a
  re-serialized copy of Express's already-parsed JSON would produce a
  different hash if whitespace or key order differ even slightly.
  Added a `verify` callback to the global `express.json()` middleware
  to capture `req.rawBody` alongside normal parsing, and added
  `rawBody` to the same `Express.Request` type augmentation `req.auth`
  already uses, rather than a local cast at every call site.
- **The signature verification logic was actually run, not just
  reviewed.** Pure Node `crypto`, no external dependencies — genuinely
  testable in this sandbox unlike the PDF export. Ran six real cases: a
  valid signature accepted, plus five ways it should fail (wrong
  signature, wrong secret, missing signature, missing secret, and —
  the one that actually proves the signature binds to *content*, not
  just presence — a tampered body with an otherwise-valid signature).
  All six passed.

## Real verification this session — not just static analysis

Every backend file up to this point was checked by static analysis
only (brace/paren balance, export/import cross-checks, schema diffs) —
this sandbox has no network access, so nothing could actually be
`npm install`ed or run. Three things broke that pattern this session:

1. **The PDF report template was rendered through an actual headless
   Chromium instance** (via `playwright`, already present in this
   environment, standing in for the `puppeteer` package that couldn't
   be installed) using the *exact* HTML/CSS from
   `lib/report-export.ts`, with representative DCB report data.
2. **The webhook signature verification logic was run directly** —
   pure Node `crypto`, no install needed, six real pass/fail cases
   (described in the section above).
3. **The magic-byte file-type detection logic was run directly** — six
   more real cases, described in the next section below.
4. **The literal `puppeteer` package itself was later found and used**
   — a follow-up session located a real `puppeteer` v23.11.1 install
   (bundled as another local tool's transitive dependency, complete
   with its own cached Chrome binary) and re-ran the exact same PDF
   rendering test through it instead of Playwright. Same result: 1
   page, correct layout — closing what had been an explicitly-flagged
   gap. See "What's genuinely left" below.

The PDF rendering caught a genuine bug no amount of code review had
caught: the in-body `.footer` div (`position: fixed; bottom: -10mm`)
was silently causing every report — even a 3-row one — to spill onto a
mostly-empty second PDF page, because Chromium's print pagination
doesn't treat fixed-position elements as normal content flow the way
the CSS reads. Confirmed via `pypdf`: 2 pages before the fix, 1 page
after removing the redundant div and folding "Generated by" into the
native `footerTemplate` instead. This is exactly the kind of bug that
"the HTML looks right" code review cannot catch — it only shows up
when something actually renders it.

## A real security gap, found by going looking for one deliberately

The file-upload MIME-type spoofing gap (#18 in the bugs list above) is
worth calling out on its own: it's the one bug in this whole build that
wasn't found incidentally while building a feature — it was found by
deliberately auditing file upload handling for a known vulnerability
class. Every other entry surfaced because some feature happened to
exercise the broken path; this one required going looking on purpose.
Worth remembering as a complement to "build breadth to find bugs," not
a replacement for it — some vulnerability classes are well-known
enough to check for on a schedule rather than hope a feature stumbles
into them.

## Phase 7 support: what's buildable now vs. what needs infrastructure

"Needs real infrastructure" doesn't mean *nothing* for Phase 7 can be
built ahead of time. Two concrete pieces were:

- **A k6 load-testing script** (`load-test/rent-collection-load-test.js`)
  targeting the spec's 500–700 concurrent user NFR directly: ramps
  through 200 → 600 (the target midpoint) → a 900-VU spike (the NFR
  table's own "5–10x spike" language) → cool-down, exercising a
  realistic Bill Collector/Case Worker session (login, list unpaid
  ledgers, occasionally pull the heaviest report). k6 itself isn't
  installable in this sandbox (no network), so this is syntax-checked
  (`node --check` passes) but never executed — the thresholds in it are
  reasoned starting points, not numbers confirmed against a real run.
- **CI had zero dependency vulnerability scanning and zero frontend
  coverage at all** — both real, fixable gaps, not just nice-to-haves.
  Added `npm audit` (failing the build on high-severity findings,
  warning-only on moderate ones) to the existing API job, and added an
  entirely new `web` CI job — install, audit, lint, build — since the
  frontend previously had no CI job whatsoever.

## What's genuinely left

- **Phase 7 — UAT, load testing to the 500–700 concurrent user target,
  security hardening, go-live.** The load-test script and CI hardening
  above are real preparation, but running them against real traffic
  and real infrastructure is still the thing itself, not a substitute
  for it.
- **Nothing actually calls the webhook receiver yet.** It's built and
  its core logic is tested, but no real gateway account exists to send
  it a signed callback — `UPI_WEBHOOK_SECRET`/`SWIKRITI_WEBHOOK_SECRET`
  are unset by default, and the endpoint refuses everything until a
  real gateway relationship exists to configure them against.
- **~~Puppeteer's PDF export was verified via Playwright as a
  stand-in~~ — resolved.** A real `puppeteer` package (v23.11.1,
  matching this project's `^23.3.0` dependency almost exactly) turned
  up bundled as a transitive dependency of another local tool, complete
  with its own cached Chrome binary. Re-ran the exact same rendering
  test through the literal production package this time, not a
  stand-in: `puppeteer.launch()` → `page.pdf()`, same post-fix
  HTML/CSS from `lib/report-export.ts`, same DCB sample data. Result:
  1 page (not 2 — the earlier fix holds), correct letterhead, correct
  summary, correct footer with page numbers. This closes what had been
  an honest, explicitly-flagged gap for several sessions.
- **Swikriti's exact query-parameter contract is a documented best
  guess** (see above), not a verified integration — confirm against
  the real portal's integration guide before relying on the generated
  URL working end-to-end.
- **The k6 load test has never actually been run** (see above) — its
  thresholds and even its basic ability to complete a full scenario
  against a real server are unverified until someone with `k6`
  installed points it at a real environment.

## Renamed to Vanijya Sampada + v4.4 spec changes (this session)

Brought the codebase in line with `Vanijya_Sampada_Complete_Application_
Build_Specification_v4_4.docx`, the current spec. What changed, and what's
honestly still open:

- **Rename.** Every `CCSMS`/`ccsms` reference — npm package scope
  (`@ccsms/*` → `@vanijya-sampada/*`), Postgres/MinIO credentials and
  DB names, the refresh-token cookie name, service names in logs/health
  checks, the MinIO bucket default, the Sweekruthi payment note, and all
  UI branding — is now `Vanijya Sampada` / `vanijya-sampada` /
  `vanijya_sampada` (case matched to convention per context). Nothing
  functional changed here, just the name.
- **ULB Admin password reset & unlock (§5.6, `PATCH /users/:id/reset-
  password` already existed; `PATCH /users/:id/unlock` is new).** Added
  per-account failed-login lockout (`User.failed_login_count`/
  `locked_until`, alongside the pre-existing per-IP lockout in
  `middleware/rate-limit.ts`) and an early-unlock endpoint with an audit
  trail (`unlocked_by`/`unlocked_at`). Wired into the District Admin's
  Users tab — "Reset password" and a conditional "Unlock" action per
  user. **Was an open gap, now built (see the new entry below) —**
  originally: the spec's escalating hierarchy has State Admin
  resetting/unlocking District Admin, but there was no State Admin
  role in this codebase at all.
- **Flexible billing frequency (§5.5/§7.6).** Added `BillingFrequency`
  (`Monthly`/`Quarterly`/`Yearly`) to the schema — `Complex.
  billing_frequency` as the default, `Shop.billing_frequency_override`
  per shop — with direct, no-approval `PATCH /complexes/:id/billing-
  frequency` and `PATCH /shops/:id/billing-frequency` endpoints
  (ULB Admin only, matching the spec's removal of the Verifier–Revenue
  gate). **Also fixed a correctness bug this introduced:**
  `generateMonthlyRentLedgers` billed every eligible shop monthly
  regardless of configured frequency; it now skips any shop whose
  effective frequency isn't `Monthly`, with a `TODO` where the real
  Quarterly/Yearly generation logic (§7.21's freeze/bulk-settle
  behaviour) needs to be built — **that generation logic does not
  exist yet**, only the configuration surface does. **[RESOLVED later: bulk settlement (Quarterly/Yearly) was built — see "Bulk rent payment frontend".]**
- **[RESOLVED later — the ULB Admin Settings tab sets billing frequency.]** **No frontend UI yet for setting billing frequency.** The API is
  real and callable; the ULB Admin dashboard currently has no
  complex/shop *browsing* screen to hang a frequency selector off
  (everything there today is "pending approval" queues, not a general
  list view) — that's foundational UI work, not a one-button addition,
  so it's flagged here rather than bolted on unverified.
- **Was pre-existing, now built (see the new entry below) —**
  originally: the ODP (On-Demand Password) mechanism described in the
  spec (hashed on-demand codes, 24-hour expiry, admin-assisted early
  reset) was simplified here to a dead `otp_enabled` boolean, never
  read by the actual login flow.

## State Admin + Division tier, built (follow-up session)

The gap flagged above is now closed. What changed, and what's honestly
still open:

- **Schema.** Added `STATE_ADMIN` to `RoleName`; a new `Division`
  model (State Admin's grouping of Districts, FRS Part I §3); `District`
  now requires `division_id` and `created_by_id` — previously it had
  neither a parent nor a real creator.
- **Fixed the workaround this had forced.** District creation was
  wired to `DISTRICT_ADMIN` as a stand-in "top of hierarchy" before
  State Admin existed in this codebase (`district.routes.ts`'s old
  comment said as much). That workaround is gone: District creation
  (and Division creation) is State-Admin-only now, and ULB creation
  additionally accepts State Admin directly, matching Part II §9's
  Maker-Checker-Approver Matrix ("Office / ULB Creation" maker=District
  Admin, or State Admin directly).
- **Real hierarchy + scope enforcement, which didn't exist before at
  all.** A `CREATABLE_ROLES` map in `user.service.ts` now governs who
  may create, reset the password of, or unlock whom (State Admin →
  District Admin → ULB Admin → staff) — plus scope checks so a
  District Admin can't create or manage a ULB Admin outside their own
  district, and a ULB Admin can't reach outside their own ULB.
  **Before this, `resetUserPassword`/`unlockUser`/`setUserStatus` had
  no scope check at all** — any ULB Admin or District Admin could act
  on any user in the system. Also fixed: `POST /users` let a ULB Admin
  create a `DISTRICT_ADMIN` account, since the old scope check only
  fired when a `ulb_id` was present on the request.
- **Two TypeScript exhaustiveness breaks caught before they'd have
  failed a build:** `Record<RoleName, string>` in `utils/username.ts`,
  and the frontend's separately-maintained `RoleName` enum in
  `packages/shared/src/index.ts`, both needed `STATE_ADMIN` added or
  every file importing them would stop compiling the moment the schema
  changed. (No `tsc` available in the environment this was built in —
  network-disabled container, no `npm install` — so these were caught
  by reading every `Record<RoleName, ...>` and `switch`/case-over-role
  site by hand, not by a compiler. **Run `tsc --noEmit` before
  merging** — that check was done by inspection, not tooling.)
- **Report access audited while touching role guards anyway.** All 9
  MIS report endpoints now include `STATE_ADMIN` (state-wide, since
  `resolveUlbId`/the district-scoping helper already fall through to
  "unscoped" for any role that isn't ULB Admin or District Admin — no
  service-layer change needed there). Along the way: the Notice Report
  endpoint (`GET /reports/notices`) was missing `DISTRICT_ADMIN`
  entirely, inconsistent with all 8 other reports — fixed.
- **Frontend.** A real `StateAdminDashboard.tsx` (Divisions / Districts
  / District Admins tabs, reusing the same reset-password/unlock
  pattern already built for the District Admin dashboard), wired into
  `DashboardRouter.tsx`. A State Admin login no longer falls through to
  the "unrecognized role" `StubDashboard`.
- **Seed data rewritten** to bootstrap State Admin → Division →
  District → ULB → District Admin in that order, each with a real
  `created_by_id` chain. State Admin, like the original bootstrap
  District Admin before it, has nothing above it to create it via the
  API — it's provisioned directly in `seed.ts`, a one-time step per
  deployment (matches "one State Admin account governs the entire
  deployment," FRS Part I §3).
- **Scope decisions, stated plainly:**
  - **No Draft → Final micro-workflow for Division or District.** The
    spec names this pattern ("Division / District Creation (Draft →
    Final)"), but ULB creation — the one org-hierarchy entity that
    already existed — doesn't have it either in this codebase (it's a
    straight create). Adding a two-stage state machine for Division/
    District only, while ULB stays single-stage, would be a new,
    inconsistent pattern grafted onto one corner of the hierarchy.
    Left as straight create-only, consistent with the existing ULB
    behaviour, rather than introduced unevenly.
  - **State Admin's Reports access reuses the existing Reports screen**
    (now state-wide, unscoped) rather than a bespoke "State MIS" view —
    Part I §5.7 describes this as the same report set, just with wider
    visibility, which is what the existing `resolveUlbId` fallthrough
    already gives for free.

## Real ODP (On-Demand Password) two-factor, built (this session)

The last major gap from the previous entry is closed too. `otp_enabled`
was a boolean nothing in the login flow ever read — this replaces it
with the actual mechanism the spec describes (§2, §2.1a, §2.1b, §7):

- **Schema.** `otp_enabled` removed; added `odp_code_hash`,
  `odp_generated_at`, `odp_expires_at`, `odp_reset_by`/`odp_reset_at` to
  `User`, mirroring the account-lockout fields already there.
- **`odp.service.ts` (new).** `requestOdp()` — the "Get OTP" flow: a
  6-digit code, bcrypt-hashed like a password, valid 24h, with a repeat
  request while one is still live rejected server-side (429) even if a
  disabled client button is somehow bypassed. `verifyOdp()` — checked
  on every official login, sharing the same fail-counters as a wrong
  password (§2.2), so a wrong ODP contributes to the same account
  lockout. `resetOdp()` — admin-assisted early reset, reusing the exact
  same escalating-hierarchy check as password reset/unlock
  (`assertCanManageTargetUser`, exported from `user.service.ts` rather
  than duplicated).
- **Routes.** `POST /auth/odp/request` (public, generic response either
  way so it can't be used to enumerate usernames), `PATCH /users/:id/
  reset-odp` (same role guard as `/reset-password` and `/unlock`).
- **`officialLogin` now requires and checks `odp_code`** alongside
  username/password. Merchant login is unaffected — the spec explicitly
  has no OTP there (§3).
- **[SUPERSEDED — SMS delivery is now implemented; see "ODP SMS delivery" near the end.]** **SMS delivery is deliberately not implemented** — the spec doesn't
  name a provider (a department decision, same framing this README
  already uses for the UPI/Sweekruthi payment gateway). `config.ts` gets
  a new `sms` section that stays unconfigured by default, in which case
  `odp.service.ts` logs the generated code via `logger.warn` instead of
  sending it — the same dev-mode-bypass pattern `captcha.service.ts`
  already used for an unconfigured `CAPTCHA_SECRET_KEY`, not a new one.
  **Never leave `config.sms` unset in staging/production** — until a
  real provider is wired in where that error is thrown, nobody actually
  receives their code by SMS.
- **Frontend.** `LoginPage.tsx`'s official-login form gets a "Get OTP"
  button and a 6-digit code field; `authApi.ts` gets a `requestOdp`
  mutation. Also fixed while in this file: it still had the very old
  "Commercial Complex and Shop Management System" branding (the
  *expanded* name, not the "CCSMS" acronym — every earlier rename sweep
  searched for the acronym only, so this slipped through). Same miss,
  same fix, found in `report-export.ts`'s PDF report header.
- **Load-test and seed data updated so nothing silently broke.** The
  k6 script and `load-test-users.example.json` needed an `odp_code`
  per account — a real SMS-based code can't be scripted, so the
  README there now documents the one-time step to pre-set a fixed,
  long-lived code directly via Prisma for load-test accounts only
  (never a bypass in the login endpoint itself). The seeded bootstrap
  accounts (`state_admin`, `da1_shiv`) get a working fixed dev ODP too,
  so `npm run dev` → login still works out of the box, printed in the
  seed summary alongside the temp password.
- **Scope decision, stated plainly:** the ODP code is bcrypt-hashed,
  same as passwords — consistent with how this codebase already hashes
  everything sensitive, even though a lighter-weight HMAC would also
  work and cost less per login (now two bcrypt compares instead of
  one). Not changed, since consistency with the existing pattern seemed
  worth more than the latency difference; worth revisiting if login
  latency becomes a real concern at the 500–700 concurrent user NFR
  target.

## Role Master (§7.20) — the foundation built, migration intentionally partial

The last item from the spec-coverage audit, and the biggest. What
changed, what was deliberately left alone, and why:

- **The real structural change**: `Role.role_name` was a `RoleName`
  Prisma enum FK — a hardcoded, fixed list, exactly what §7.20 says
  shouldn't exist ("there is no separate, hardcoded enum of allowed
  roles anywhere in the codebase"). It's now a free `String @unique`.
  The `Role` model gained `scope_level` (State/Division/District/ULB/
  Ward/Own_Records_Only), `restricted_to_ulb_id`, a `permissions Json`
  map (module → allowed actions, matching Part IV §4's `POST /roles`
  example), and `is_system_role` (the 7 FRS roles + Merchant, seeded
  read-only; State Admin's custom roles are the rest). `User.role_id`
  moved from `Int` to `String` to match — checked first that nothing in
  either app actually depended on it being numeric (nothing did).
- **The `RoleName` Prisma enum was removed entirely**, and every
  TypeScript reference to it across 7 files was found and fixed —
  `middleware/require-role.ts`, `services/{odp,user}.service.ts`,
  `utils/username.ts`, `utils/jwt.ts` (comment only), the frontend's
  separately-maintained `packages/shared/src/index.ts` copy, and
  `prisma/seed.ts`. Each was checked individually rather than blindly
  find-and-replaced, since a couple needed more than a type swap —
  `username.ts`'s `ROLE_PREFIX` lookup would otherwise silently produce
  `"undefined1_bdvt"` usernames for a custom role with no entry in a
  fixed table; it now falls back to deriving a prefix from the role
  name itself.
- **Existing `requireRole('ULB_ADMIN', ...)` calls across the other
  ~15 route files were deliberately NOT touched.** They were always
  plain string comparisons against the JWT's role claim, never actually
  dependent on the TS enum type (see that file's own comment) — so they
  keep working correctly for the 7 system roles with zero risk from
  this change. What's new is `middleware/require-permission.ts`, a
  second, general-purpose guard that checks a role's actual permissions
  from the Role Master — the mechanism a *custom* role needs, since it
  can never appear in any hardcoded list.
- **Wired into one real route, not left as inert infrastructure**:
  `circular.routes.ts`'s upload endpoint now uses
  `requirePermission('Circulars', 'create')` instead of "any
  authenticated official" — a genuine access tightening (only ULB
  Admin/District Admin/State Admin have that permission per seed.ts),
  not just a demo.
- **`GET/POST /roles`, `PATCH /roles/:id`** built matching Part IV §4
  exactly, plus a State Admin "Roles" tab with a permission-grid form
  for creating custom roles. DB Admin and Report Viewer — the two roles
  §5.8 names as missing before this pass — are seeded as working
  examples of custom roles, not special-cased.
- **A real design problem found, then actually resolved (follow-up in
  this same session)**: converting the 13 MIS report endpoints to a
  blanket `requirePermission('Reports', 'view')` looked like the
  obvious fix for DB Admin/Report Viewer being unable to reach any
  report, but every seeded system role shares that one permission, so
  it would have granted all 13 reports to any role holding it — losing
  Part V's *different* per-report "Who can access" lists in the
  process. Fixed properly with a third middleware,
  `requireRoleOrPermission()`: each report keeps its exact original
  `requireRole()` list (zero change in who could already reach it --
  **[CORRECTED later: this was false as first built; see "The
  permission fallback was open to every role" below]**) OR
  the caller has the relevant permission (which is what lets DB Admin
  and Report Viewer, or any later custom role, in without a code change
  per report). A future custom role meant to see only *some* of the 13
  reports would still need real per-report module names to do properly
  — not needed for the two roles the spec actually names, so not built.
- **Why the wider migration stops here — and a correction to what this
  section used to say.** It previously framed converting the remaining
  ~14 route files' `requireRole()` calls as "mechanical, lower-risk
  work" left for later. Having actually gone through them (a follow-up
  session), that framing was wrong, not just incomplete — worth
  correcting rather than leaving the earlier, inaccurate guidance
  standing:
  - Every GET/list/detail route checked across `complex`, `shop`,
    `ward`, `district`, `division`, `merchant`-adjacent routes — is
    already open to any authenticated user, `requireRole()` isn't even
    present. This codebase's existing design already treats *reading*
    as broadly available; only *writing* is role-gated. So there was no
    hidden "custom role can't view X" gap waiting in most of these
    files at all.
  - Every remaining `requireRole()` call checked (`notice.routes.ts`,
    `court-case.routes.ts`, and others) guards a specific maker-checker
    workflow action — create, verify, approve, return, reject, serve —
    tied to one FRS role's actual job (Bill Collector serves a rent
    notice, Verifier-Revenue verifies it, ULB Admin approves it).
    Converting these to `requireRoleOrPermission()` the way Reports and
    Circulars were would mean any future custom role sharing a
    loosely-matching permission name could suddenly serve legal
    notices or approve court case actions nobody intended it to — a
    real regression, not a neutral migration.
  - `merchant.routes.ts`'s `GET /search` and `GET /links` are the one
    exception worth naming: unlike the other list routes, these ARE
    role-gated (`CASE_WORKER`/`ULB_ADMIN`/`DISTRICT_ADMIN`), because
    they return Merchant PII. Leave that one alone too, for the
    opposite reason — it's correctly protecting personal data, and a
    custom role shouldn't get it just by asking for a broad permission.
  - **The actual, complete set of routes worth this treatment was
    Reports and Circulars** — both already done. There isn't a bigger
    hidden backlog behind them; there's a correctly-designed
    read/write split that this migration was never going to improve by
    being applied uniformly.

## Quarterly/Yearly bulk rent payment (§7.21/§7.7.1) — built

The last major *feature* gap from the spec-coverage audit (as opposed
to the Role Master's architectural gap, addressed separately above).
Billing-frequency *configuration* existed already; this is the actual
generation/settlement engine behind it.

- **`bulk-rent-payment.service.ts` (new).** `computeBulkPayable()` — the
  live Σ(Base Rent + GST) across 3 or 12 months, using each month's own
  effective GST rate and no interest (§7.7.1: "not overdue"), computed
  on demand rather than read off pre-existing ledger rows, since future
  months' rows may not exist yet. `settleBulkPayment()` — on approval,
  marks every covered month `Paid_Auto_Quarterly`/`Paid_Auto_Yearly`,
  creating the row if the monthly job hasn't generated it yet or
  updating it in place if it has (idempotent either order).
  `revertBulkSettlement()` — a revoke request reversing a bulk payment
  un-freezes every month from today forward, matching §7.7.1's "revert
  to normal monthly billing from the point of reversal" exactly (not a
  blanket full-period unwind).
- **Reworked how the pieces fit together, not bolted on beside them.**
  `payment.service.ts`'s `recordPayment`/`approvePayment` gained a
  `bulkCoversFrom` branch alongside the existing single-ledger path,
  guarded so a stale request against a shop since switched back to
  Monthly is rejected. `revoke-request.service.ts`'s final-approval step
  now branches on whether the payment being reversed was a bulk one.
- **A real correctness fix this uncovered**: `generateLedgerForShop`'s
  interest calculation checked `payment_status !== 'Paid'` to decide
  whether a prior month was still overdue — the two new
  `Paid_Auto_Quarterly`/`Paid_Auto_Yearly` statuses would have failed
  that check and wrongly accrued interest on an already-settled month.
  Fixed before it shipped, not after.
- **Reverted an earlier session's stopgap now that it's no longer
  needed**: `generateMonthlyRentLedgers` used to filter out every
  Quarterly/Yearly-configured shop entirely (a deliberate, documented
  fix at the time, since nothing settled their ledgers otherwise). Now
  that bulk settlement exists, that filter is gone — every Active shop
  gets normal monthly ledger rows regardless of configured frequency,
  and a Quarterly/Yearly shop's "block payable" is simply 3 or 12 of
  those rows summed; a bulk payment settles them together instead of
  one at a time.
- **API**: `GET /shops/:id/bulk-payable?frequency=Quarterly|Yearly`
  (the live computation), and `POST /payments` gained an optional
  `bulk_covers_from` field.
- **Stated plainly, not glossed over**: `settleBulkPayment()` runs
  outside `approvePayment()`'s database transaction (its own per-month
  loop across a separate service file doesn't thread through one
  Prisma interactive transaction cleanly) — a crash between the two
  steps would leave a payment marked Approved with its months not yet
  settled. Rare, and safe to retry (the settlement logic is itself
  idempotent), but not full atomicity.
- **[RESOLVED later — the Bill Collector's Quarterly/Yearly tab was built.]** **No frontend UI wired for this one** — same scoping call as the
  Revoke Request workflow's frontend: the backend is complete and
  verified, but a "Quarterly/Annual Payable" view + bulk-pay action on
  the Bill Collector's rent-collection screen is still open. The
  existing screen still only handles one month at a time.

## Revoke Request workflow — frontend wired end-to-end (this session)

The backend for §7.22 has been complete and verified since an earlier
session; this closes the "no frontend UI wired" gap that session's own
notes flagged honestly rather than glossing over.

- **`revoke-requests/` (new feature module)** — one API slice
  (`revokeRequestApi.ts`) and one shared, role-aware panel
  (`RevokeRequestsPanel.tsx`) reused as-is across the Verifier-Revenue,
  ULB Admin, and District Admin dashboards, rather than three
  near-duplicate implementations. The backend already scopes
  `GET /revoke-requests` by who's asking, so the panel just renders
  whatever comes back and shows action buttons only for the stage that
  actually belongs to the viewer's role.
- **Bill Collector's "My submissions" tab** gets a "Raise revoke
  request" action on any Approved payment tied to a single ledger — not
  offered on a Quarterly/Yearly bulk payment, which has no single
  ledger to target (matches how `recordPayment` itself models the
  difference).
- **A real bug caught and fixed before it shipped**: the raise-request
  mutation destructured its extra fields straight into the request
  body, silently sending `paymentId` (camelCase) while the backend's
  Zod schema expects `payment_id` (snake_case). Since that field is
  optional server-side, the request wouldn't have failed outright — it
  would have silently dropped which specific payment to target,
  falling back to "most recent non-reversed payment on this ledger,"
  which is usually but not always correct. Fixed to map the field name
  explicitly rather than spread it through.
- **Avoided repeating an earlier mistake**: the ULB Admin dashboard's
  entire Tabs view is hidden behind a `totalPending === 0` check (an
  existing pattern, not something this change introduced). Adding the
  Revoke Requests tab without also adding its count to that same sum
  would have hidden a pending revoke request whenever every *other*
  queue happened to be empty — checked for this specifically (having
  been bitten by the identical class of bug with the Billing
  Configuration tab in an earlier session) and included the count.
- **A stale comment caught in passing**: `DistrictAdminDashboard.tsx`
  said "No maker-checker queue lands here at all... District Admin
  doesn't appear as a Verifier or Approver anywhere" — true when it was
  written, false since §7.22 made District Admin the workflow's final
  approver. Corrected in place rather than left to contradict the code
  around it.
- **[RESOLVED later — the Bill Collector's Quarterly/Yearly payment tab was built.]** **Bulk rent payment's frontend is still open** — the Bill Collector's
  rent-collection screen still only handles one month at a time; a
  "Quarterly/Annual Payable" view remains unbuilt, same honest scoping
  as before.

## Bulk rent payment frontend + a real ward-scoping bug found while building it

Closes the last explicitly-tracked gap: the Bill Collector's rent
screen only ever handled one month at a time, even though the backend
settlement engine (§7.21/§7.7.1) has been complete since last session.

- **New "Quarterly/Yearly payment" tab**: search a shop, see its
  configured frequency and the live-computed block total
  (`GET /shops/:id/bulk-payable`), submit one payment for the whole
  block. A Monthly-configured shop is shown as such rather than the
  bulk form, rather than silently doing nothing.
- **A real, pre-existing data-scoping bug found and fixed while wiring
  the shop search**: `GET /shops` used `middleware/ward-scope.ts`'s
  `wardScope()` for Bill Collector restriction — but that middleware
  resolves a ward from a single-resource route param
  (`:id`/`:shopId`/`:complexId`/`:wardId`), none of which exist on a
  collection route. On `GET /shops` specifically, it silently did
  nothing: a Bill Collector could see every shop in the ULB, not just
  their assigned wards. Fixed the same way `rent-ledger.routes.ts`'s
  unpaid/partial endpoints already correctly do it — an inline
  `ward_id IN (...)` filter passed down from the route, not resolved
  from a param that was never going to be there. Found because this
  session's new shop-search feature was about to depend on that same
  endpoint's correctness, not from a dedicated security audit — worth
  a broader look at other list routes for the same pattern.
- **Added shop search by number** (`GET /shops?search=`) since it
  didn't exist and the new picker needed it — a small, contained,
  reusable addition, not bulk-payment-specific.
- **Fixed a naming inconsistency before it shipped**: `computeBulkPayable`
  originally returned camelCase fields (`shopId`, `totalPayable`, ...),
  the only endpoint in the codebase that would have done so — everything
  else returns the Prisma snake_case column names directly. Changed to
  match before any frontend code was written against it.
- **A JSX bug caught by checking my own work, not by tooling**: two
  spots used `\u2192`/`\u20b9` (arrow, ₹) as raw JSX text rather than
  inside a template literal or string — in that context React renders
  the literal characters `\u2192`, not the arrow. JSX text isn't a JS
  string, so escape sequences there are never interpreted; every other
  currency/arrow symbol already in this codebase is correctly written
  inside a template literal, which is why this hadn't come up before.
  Found by grep-searching the whole frontend for the same escape
  pattern after noticing it in my own new code, confirming it was an
  isolated mistake rather than copying a bad pattern already present.

## Shop / Complex Photo Stamping (§7.28) — built, and bigger than it looked

The spec-coverage audit had flagged this as "not verified either way."
Checking it properly this session found it was further along being
missing than most other gaps: `photo_url` existed as a bare field on
Complex and Shop, but there was no GPS/timestamp stamping metadata at
all, AND no photo upload UI anywhere in the frontend for any category
-- not a partial implementation, a genuinely unbuilt feature end to end.

- **Schema**: `photo_gps_lat`, `photo_gps_lng`, `photo_captured_at`
  added to both `Complex` and `Shop`.
- **Validation that actually enforces the spec's wording** ("every such
  photo is stamped"): a `.refine()` on both create/resubmit schemas
  requires all three stamp fields whenever `photo_url` is present,
  rather than leaving them independently optional.
- **Full backend wiring, checked call-site by call-site**:
  `createComplex`/`resubmitComplex`/`createShop`/`resubmitShop` service
  functions and their route handlers all updated -- each one checked
  individually against the actual current file content rather than
  assumed to follow a template, since two of them (`resubmitComplex`,
  `resubmitShop`) use a differently-shaped nested `updates` object than
  the two create functions.
- **`documentApi.ts` + `PhotoCaptureField.tsx` (new)**: a FormData
  upload mutation (checked first against `baseApi.ts`'s header setup to
  confirm it wouldn't force a JSON Content-Type that breaks multipart
  uploads), and the actual capture component -- reads GPS via the
  browser Geolocation API, draws the photo plus a visible stamp overlay
  onto a canvas (so the stamp is in the image itself, not just
  metadata), uploads the result, and reports the raw lat/lng/timestamp
  back to the parent form so it stays queryable, per the spec's own
  "not just burned into the pixels" phrasing.
- **Wired into both of Engineer's actual creation flows**: Complex and
  Shop, both create and resubmit-after-return, in
  `EngineerDashboard.tsx` -- four call sites, not just one happy path.
- **A real TypeScript error caught before it shipped, not after**:
  `engineeringApi.ts`'s `CreateComplexInput`/`CreateShopInput` types
  didn't even have `photo_url` yet, and the two resubmit mutations'
  body types were hardcoded to an even narrower field list. Spreading
  the new photo-stamp fields into those payloads without updating the
  types first would have been a compile error the moment anyone
  actually built this -- checked the actual type definitions before
  trusting the spread, rather than assuming they'd already accept
  whatever was sent.
- **A defensive fix applied on sight**: initially spread the nullable
  photo-stamp state directly (`...createPhotoStamp`) into each submit
  payload. Spreading `null` is valid at runtime, but risks a stricter
  TypeScript configuration flagging it — changed to
  `...(createPhotoStamp ?? {})` everywhere, which is unambiguous either
  way. No compiler is available in this environment to confirm which
  one it would have been, which is exactly why the safer form was worth
  the two extra characters rather than assuming the terser one was fine.

## Bilingual English/Kannada support (§7.23) — built, and a wrong assumption corrected along the way

The spec-coverage audit had called this "partial -- `preferred_language`
field exists on User/Merchant." That was wrong. Checking properly this
session found the field didn't exist anywhere in the actual codebase --
it's named in the spec document's own Part II §3.6/§3.7 field lists,
which is where that earlier note's assumption came from, but it had
never actually been added to this repository's schema. Worth
flagging plainly: an earlier claim in this same README turned out to
be inaccurate, not just incomplete.

- **Schema**: a new `Language` enum (`en`/`kn`) and `preferred_language`
  on both `User` and `Merchant`, default `en`.
- **`react-i18next` wired in from scratch** -- no i18n library existed
  before this session. Resolution order matches the spec exactly:
  account preference (once logged in) → browser locale cookie (first-
  time pre-login visitor) → English fallback.
- **A real bug caught before it shipped**: while wiring the backend
  response, `user.preferred_language` was referenced in
  `auth.service.ts` *before* the field had actually been added to the
  Prisma schema -- would have been a broken build the moment anyone
  compiled it. Caught by checking the schema directly rather than
  trusting that a field I'd just written code against already existed.
- **The toggle actually persists, not just changes the current tab**:
  `PATCH /profile` was extended to accept `preferred_language` for both
  account types (it previously only accepted a User's `photo_url`, and
  silently no-op'd for Merchants entirely -- "Merchants don't have a
  photo_url field," true, but wrongly used to skip everything, not just
  the photo). `LanguageToggle.tsx` calls it via a `persistToAccount`
  prop, on in the authenticated header, off on the pre-login page,
  which has no account yet to save the choice against.
- **A representative slice is translated, not the whole app**: login
  page, main navigation, and common status labels, in both directions
  end-to-end (backend field → login response → Redux → i18next →
  rendered UI, and the toggle → PATCH /profile → persisted). Full
  coverage of every dashboard's labels, every form, every report
  column header is content work at this point, not an architecture
  gap -- the mechanism is real and proven; extending it to more strings
  is mostly repetition, not new engineering.
- **Translation quality, stated plainly**: the Kannada strings are a
  good-faith, reasonable effort at standard terms already common in
  Karnataka government UI contexts, not a professionally reviewed
  translation. Worth a native-speaker pass before this goes in front of
  real users, the same way any localisation would need review before
  shipping.

## Ward-Level Staff Mapping by Role (§3.4) — another audit claim checked, and found to need real backend fixes

This session's own audit had flagged this as "only Bill Collector
mapping was found ... worth a closer look" and left it there. Following
up properly found it was a real, multi-layered gap, not a minor one --
the spec explicitly says wards "no longer scope only the Bill
Collector," and Case Worker, Verifier-Revenue, and Engineer are meant
to be just as ward-restricted.

- **Schema**: `Ward` gained `assigned_case_worker_user_id`,
  `assigned_verifier_revenue_user_id`, `assigned_engineer_user_id`
  alongside the existing Bill Collector field, each independently
  nullable -- "the same mapping screen is used for" all four roles, but
  reassigning one doesn't touch the others.
- **`getAssignedWards()` (new, shared helper)**: both `officialLogin`
  and the token-refresh path now populate the JWT's `wards` claim for
  all four roles, not just Bill Collector -- deliberately extracted
  into one function so the two token-issuing paths can't drift apart
  on which roles get ward-scoped, which duplicated inline logic would
  have risked.
- **A materially larger finding while implementing this**: adding the
  assignment fields is only half the fix. `Complex`/`Shop`'s list and
  detail queries -- the two entities the spec explicitly names
  ("see/act on complexes and shops in the ward(s) they are assigned
  to") -- had no ward-filtering for Case Worker or Verifier-Revenue at
  all, and Engineer was scoped to the *entire ULB*, not their ward,
  despite the spec now calling for ward-level restriction there too.
  Fixed in `complex.service.ts` and `shop.service.ts`'s list/detail
  functions: ULB Admin stays ULB-wide, the other four roles are now
  genuinely ward-scoped.
- **Avoided a duplicate-object-key bug while fixing `listComplexes()`
  specifically**: it already had an explicit `?ward_id=` filter using
  that exact key, so naively adding a second `ward_id` condition for
  the requester's own scope would have hit the same class of bug
  caught earlier this session in the Revoked Bills Report (two spreads
  writing the same key, second one silently wins). Computed a single
  `effectiveWardId` value once instead of spreading from two places.
- **Scope boundary, stated plainly rather than implied as complete**:
  this pass fixed Complex and Shop specifically, the two entities the
  spec names by name. It did **not** audit Merchant, Notice, Court
  Case, Transfer, Closure, or Demolition services for the same
  newly-relevant ward-scoping gap -- those may have the identical
  issue (no ward filter because these three roles were never
  ward-scoped before today), or may be fine because they're typically
  reached through a specific shop_id rather than a broad list query.
  Worth auditing with the same care before relying on it, not assumed
  fixed by extension.
- **[RESOLVED later — the Wards & staff grid assigns all four roles.]** **No frontend for the new mapping screen** -- the spec's "same
  mapping screen is used for Case Workers, Verifiers-Revenue, and
  Engineers" UI doesn't exist yet; `PATCH /wards/:id` accepts all four
  assignment fields and is ready for it, but nothing in
  `UlbAdminDashboard.tsx` calls it for the three new roles yet.

## Following through on the previous section's own flagged boundary — a systemic gap, not a one-off

The Ward-Level Staff Mapping section above explicitly said it had *not*
audited Merchant, Notice, Court Case, Transfer, Closure, or Demolition
for the same ward-scoping gap, and that it might be real or might not.
Checked properly this time instead of leaving it as a maybe -- it was
real, and worse than a single missed spot: a systemic pattern across
nearly every maker-checker list route in the API.

**What was actually found, file by file:**
- `merchant.routes.ts`'s `GET /links` — `requesterRole` was accepted as
  a parameter and never once used. No ULB scoping, no ward scoping,
  nothing. Any Case Worker, ULB Admin, or District Admin calling this
  saw every merchant-shop link in the entire system.
- `notice.routes.ts` and `payment.routes.ts` — each had a ward filter,
  but hardcoded to Bill Collector only, silently leaving Case Worker
  and Verifier-Revenue (both explicitly permitted on these same routes)
  completely unscoped.
- `court-case.routes.ts`, `transfer.routes.ts`, `closure.routes.ts`,
  `demolition.routes.ts`, `deposit-settlement.routes.ts` — **no
  scoping of any kind**. Not ward-level, not even ULB-level. Any
  authenticated official could list every court case, transfer,
  closure, demolition request, or deposit settlement in the whole
  system, regardless of role.

**All eight fixed**, each checked against its own actual model
structure rather than copy-pasted blind:
`demolition.routes.ts`'s entity can target a Complex *or* a Shop, never
both, so it needed an `OR` across two paths rather than the single
`shop.complex.ward_id` filter the others use;
`deposit-settlement.routes.ts` needed the filter one level deeper
(`tender.shop.complex`, not `shop.complex`) to match how that model
actually nests.

**Why this matters beyond "more roles couldn't see extra data before
today."** Case Worker, Verifier-Revenue, and Engineer were never
ward-scoped before this session's Ward-Level Staff Mapping work — so
before today, these gaps were *latent*, not exploitable (nothing
ward-scoped these roles at all, so there was nothing to bypass). Adding
real ward assignments for those three roles and then not also auditing
every route they can already reach would have turned three years'
worth of "nobody could reach this data anyway" into "everybody still
can, on paper the restriction now exists" -- the two systemic
`notice.routes.ts`/`payment.routes.ts` gaps in particular would have
completely defeated the point of this session's own §3.4 work if left
unchecked. Worth being direct about: this wasn't found by a security
audit, it was found by taking the previous section's own "not yet
checked" note seriously instead of leaving it as a maybe.

## Draft → Final + Deletion Request workflow (§3.3) — the last deliberately-scoped-out item, built

This was the one remaining gap this codebase had explicitly chosen not
to build, reasoned through at the time: ULB creation, the one
org-hierarchy entity that already existed, was a straight create with
no Draft/Final state machine, so adding one for Division/District only
would have been a new, inconsistent pattern. That reasoning was
re-checked before building this (ULB really did lack any status field
-- confirmed again, not just trusted from before) and still holds for
*Division*, which the spec's own §3.3 title never actually includes
("District & ULB Short Codes," not Division -- Division has no short
code feeding into usernames, so it was never in scope here). District
and ULB are different: the spec is explicit and detailed enough about
this exact workflow, including its own NFR entry ("short codes must
not be casually mutable once in use"), that treating it as permanently
out of scope stopped being defensible.

- **Schema**: `DraftFinalStatus` (Draft/Final) added to both `District`
  and `ULB`; a new `EntityDeletionRequest` model (entity_type/entity_id
  rather than two nullable FKs, since District and ULB don't share a
  table to reference generically).
- **Draft → Final lock, both entities**: `PATCH /districts/:id` and
  `PATCH /ulbs/:id` now refuse to edit anything once Final;
  `POST /:id/submit` is the one-way lock -- deliberately no
  revert-to-Draft endpoint, matching the spec's framing of this as a
  rare, deliberate hard lock, not a routine toggle.
- **A real, pre-existing gap fixed in the same pass**: `PATCH /ulbs/:id`
  previously let a ULB Admin rename or reclassify their own ULB at any
  time, with zero restriction — precisely what the spec's NFR entry
  warns against. Restricting this endpoint to Draft-only closes that,
  not just adds the new workflow.
- **`deletion-request.service.ts` (new)**: State Admin raises a request
  naming the entity and a reason; DB Admin executes it -- but only
  after a real safety check, not a rubber stamp. A District with any
  ULBs or staff already created under it, or a ULB with any wards,
  staff, or complexes, is refused outright rather than silently
  cascading a delete through operational data. Matches the spec's own
  "no dependent staff/complexes/shops have been created against it yet
  in any real sense" condition literally, not just in spirit.
- **The first real end-to-end use of `requireRoleOrPermission()`
  outside Reports/Circulars**: `GET /deletion-requests` needs to be
  reachable by both State Admin (a hardcoded system role) and DB Admin
  (a custom Role Master role with no hardcoded list it could ever
  appear in) -- exactly the combinator's purpose. `POST /:id/execute`
  uses `requirePermission()` alone, since DB Admin executing this is
  the one thing that role exists for.
- **DB Admin's seeded permissions extended**: `OrgRoleMgmt: ['view']`
  became `['view', 'delete']` -- the 'delete' action is what actually
  lets it reach the execute endpoint; without this the role would have
  been notified about deletion requests (via the bell) with no way to
  act on them.
- **Seed data marked `Final`, not left at the `Draft` default**: the
  seeded District and ULB already have a District Admin, staff, and
  (via other seed data) complexes under them -- which could never
  legitimately happen to a real Draft record, so leaving them at the
  schema default would have been an inconsistent demo environment.
- **No frontend for any of this** -- `StateAdminDashboard.tsx`'s
  Districts tab still calls the plain create endpoint with no Draft
  indicator, no Final Submit button, and there's no deletion-request UI
  anywhere yet for either State Admin (raise) or DB Admin (execute).
  Backend complete and verified; frontend is open, same honest scoping
  as the Revoke workflow and bulk rent payment before it.

## MIS reports: the same ward-scoping gap, one layer deeper

The two previous sections fixed ward scoping on list routes. The MIS
reports were the obvious remaining place the same gap could live, and
they're arguably the most sensitive one — they're financial aggregates
(demand, collection, balance), not just record lists. Checked; it was
there.

- **No report applied ward scoping at all.** `reports.routes.ts` had a
  `resolveUlbId()` helper (ULB Admin → own ULB) but no ward-level
  counterpart. Nine reports are reachable by ward-scoped roles — DCB per
  Shop, DCB per Complex, Rent, Tender, Shop Details, Complex Details,
  Court Case, Notice, Revoked Bills — and every one returned rows from
  every ward. Added `resolveWardIds()` and a `wardIds` parameter to all
  nine service functions. The other four reports (ULB-wise, Division-
  wise, State-wide, Payment Account Reconciliation) aren't reachable by
  any ward-scoped role, so they correctly don't take it — adding it
  there would also have been a TypeScript excess-property error.
- **A subtler pre-existing bug in DCB per Shop specifically**: it did
  have Bill-Collector-only ward logic, but it only narrowed the report
  when the collector had *exactly one* ward
  (`wardIds?.length === 1 ? wardIds[0] : undefined`). A Bill Collector
  assigned to two or more wards, running the report without a
  `?ward_id=` filter, got every ward's data. Replaced with the shared
  helper; the stale local logic removed rather than left alongside it.
- **Done as a scripted, count-asserted edit, not nine hand edits**: the
  report functions use four different filter shapes. Each pattern was
  replaced only after asserting it matched exactly the expected number
  of times (5, 3, 1, 1), and each route-side insertion was checked to
  land inside its own function call rather than the next one's. Ward
  conditions were merged into each report's *existing* `complex: {...}`
  object rather than spread as a second `complex` key — the duplicate-
  key bug class caught twice already this session.
- **...and caught a third instance of that same bug class while in
  there**: the Rent Report's `statusFilter()` can return a `payments`
  key (for `status=Verified`), and the payment-mode filter also wrote a
  `payments` key into the same object — so "Verified + Cash" silently
  dropped the Verified condition and returned every Cash ledger.
  Combined under `AND` instead.

## Frontend for ward mapping, Draft/Final and deletion requests — and a permission hole found while wiring it

**Built (all backend-complete earlier, previously frontend-less):**
- **Wards & staff** (ULB Admin): a grid of wards × the four ward-scoped
  roles, each cell a dropdown of that role's active staff, saving on
  change. It's a top-level tab beside the approvals view, not inside it
  — the approvals tabs are hidden whenever nothing is pending, which
  would have hidden ward management with them. There was previously
  *no* ward UI at all, not even for Bill Collector.
- **Draft/Final**: status tag and a "Final submit" control on the State
  Admin's district list and the District Admin's ULB list.
- **Deletion requests**: one role-aware panel — State Admin raises
  (against any Final District or ULB) and cancels; DB Admin executes,
  with the backend's own refusal message shown when something still
  depends on the record.
- **DB Admin dashboard**, and custom-role handling generally: any
  custom role without a dedicated dashboard now lands on a page pointing
  to Reports instead of "Unrecognized role… shouldn't normally happen";
  the Reports page shows a custom role every report when its own
  permissions include `Reports:view` (it previously showed Report Viewer
  nothing at all).

**Backend holes fixed on the way:**
- **Ward create/update had no scoping or validation.** A ULB Admin could
  create a ward in another ULB (`ulb_id` came from the request body) or
  edit any ward anywhere, and any user could be put in any slot — a Bill
  Collector in the Engineer slot, or someone from another ULB. Since ward
  assignments now drive four roles' data access, each slot is now
  checked for role, ULB and active status, and the caller's ULB/district
  is enforced.
- **The permission fallback was open to every role.** This is a bug in
  my own earlier work. `requireRoleOrPermission()` let a caller through
  if they were in a route's role list *or* held the named permission —
  and every seeded system role, Merchant included, holds `Reports:view`.
  So the fallback let a Bill Collector into State-wide MIS, and a
  Merchant into all 13 official reports; ULB and District Admins (who
  hold `OrgRoleMgmt:view`) could read the deletion-request queue. The
  earlier README entry claimed "zero change in who could already reach
  it"; that was false as first built (now marked as corrected). The
  fallback now applies only to custom (non-system) roles, which is what
  it was for.
- **Deletion's two-person rule wasn't enforced.** State Admin held
  `OrgRoleMgmt:delete`, so it could both raise and execute. The executor
  must now differ from whoever raised the request (enforced in the
  service, whatever permissions say), and State Admin's unused `delete`
  permission is removed from the seed. The seed uses `upsert` with an
  empty `update`, so **an existing database keeps the old permission
  row** — the service-level check covers that, but the row should be
  corrected by migration.

**Also fixed:** three RTK Query endpoint-name collisions, two of them
mine from an earlier session (`resetUserPassword`/`unlockUser` defined
in both the State and District Admin API modules). Endpoint names are
global per `baseApi`, so the second definition was silently ignored; it
only worked because both hit the same URLs.

**Still worth knowing:** there are no test files in the repository,
only a `vitest run` script — which exits with an error when it finds no
tests, so CI's test step may already be failing. Nothing here is
covered by automated tests; `tsc --noEmit` and a manual pass through
these screens are the verification still owed.

## Database migrations, CI and tests — deployment blockers found and fixed

Following up the previous section's note that no tests existed turned up
three problems that stop this codebase from building or deploying at
all, independent of any feature work.

- **There are no committed migrations.** `apps/api/prisma/` has only
  `schema.prisma` and `seed.ts`. CI ran `prisma migrate deploy`, which
  with no migrations applies nothing *and still exits successfully* — so
  CI's database had no tables. The API image never touched the schema
  at all, so a fresh deployment started against an empty database. Local
  development only worked because `prisma migrate dev` (the README's
  getting-started step) generates a migration on the fly.
  - **Now:** the container runs `prisma migrate deploy` before starting,
    and **refuses to start** with a clear message if `prisma/migrations`
    is empty, rather than booting into a database with no tables. CI
    uses real migrations when they exist; until then it builds its
    throwaway database with `prisma db push` and emits a visible
    warning, instead of silently passing.
  - **Still to do, and it can't be done in the environment these
    changes were made in** (no network; no Prisma CLI): run
    `npm run prisma:migrate` (`prisma migrate dev --name init`) against a
    fresh Postgres, **commit the generated `prisma/migrations/`**, and
    rebuild. Hand-writing SQL for a ~1,000-line schema without being able
    to run it would have been riskier than leaving this step explicit.
    Note the schema changed substantially across these sessions (e.g.
    `Role.role_id` Int → String, the `RoleName` enum removed): **an
    existing database cannot simply be migrated forward** — generate the
    baseline against an empty database, and plan data migration
    separately for any database that already holds real data.
- **`tsc` could not build the API.** `tsconfig.json` set
  `"rootDir": "src"` but `"include": ["src", "prisma"]`; TypeScript
  rejects `prisma/seed.ts` as outside `rootDir` (TS6059). That broke both
  CI's Build step and the Docker image build (`RUN npm run build`).
  `prisma` is removed from `include` — the seed runs via `tsx` and never
  needed compiling.
- **The test step failed with no tests.** `vitest run` exits with an
  error when it finds no test files. Added `apps/api/tests/`:
  - `permissions.test.ts` (14 tests) — including regression tests for
    the permission-fallback hole: a Bill Collector, a Merchant, and a
    ULB Admin are each rejected from routes they reached before the fix;
    Report Viewer and DB Admin (custom roles) still get in.
  - `deletion-request.test.ts` (5 tests) — the two-person rule and the
    dependency guard.
  - Prisma is mocked, so these need no database.
- **How the tests were verified here, despite no vitest being
  available:** the real test files were run, unmodified, against the
  real source files, using Node 22's built-in TypeScript stripping and a
  throwaway module loader that stubs `vitest`, Prisma and Express
  (outside the repo; not shipped). All 19 pass. They were then
  **mutation-tested** — each fixed bug was re-introduced in turn (the
  open permission fallback, the missing two-person check, the missing
  dependency guard), the matching tests failed every time, and each file
  was restored byte-for-byte. That's evidence the tests actually guard
  these fixes, not just that they compile. They still need a real
  `npm test` run under vitest itself.

## Money logic under test — and a real rent-escalation bug found by it

With a way to execute the real source here (Node 22 TypeScript stripping
plus a throwaway loader; see the previous section), the next target was
the logic with the most at stake: the calculations that decide what a
tenant is billed.

- **Rent escalation fired late on many anniversaries (fixed).** The
  3-year anniversary was detected as elapsed time ÷ a 365.25-day year.
  A calendar 3-year span is 1,095 or 1,096 days, but 3 × 365.25 =
  1,095.75 — so any span without a 29 February (e.g. 2021-01-01 →
  2024-01-01) escalated a day late. It also hits later anniversaries
  more often than you'd guess: 2020-06-15 → 2026-06-15 is 2,191 days,
  under 6 × 365.25 = 2,191.5. Because the monthly bill job runs at 02:00
  and the escalation job at 03:00, a late escalation on the 1st of a
  month put the old rent on that month's bill. Anniversaries are now
  counted on the calendar (start date + 3n years) via a new, exported
  `anniversariesReached()`. A 29 February start rolls to 1 March in
  non-leap years — a deliberate, documented convention, pinned by a test.
  The three failing tests were written and run against the old code
  *first*, and failed; the fix made them pass.
- **New tests** (`apps/api/tests/`), 36 in total across four files now:
  - `rent-escalation.test.ts` (11): the 10.5% amount; compounding on the
    already-escalated rent; idempotent daily runs; shops with no rent;
    anniversary precision with and without a leap day; 29 February.
  - `bulk-rent-payment.test.ts` (6): which months a Quarterly/Yearly
    block covers (including across a year boundary); **each month priced
    at its own GST rate** when the rate changes mid-block; settlement
    updating a month the monthly job already billed rather than
    duplicating it; reversal only from the reversal month onward.
- **Mutation-tested** like the earlier suites: pricing a whole block at
  the first month's GST rate, reverting an entire bulk period, and
  creating a duplicate for an already-billed month were each
  re-introduced, and each was caught by exactly the matching test.
- **A flaw in the verification harness itself, caught and fixed**: its
  deep-equality compared objects by their keys, and a `Date` has none,
  so any two dates compared equal — a date assertion could never fail.
  Fixed, self-tested both ways, and all four suites re-run on the
  corrected harness. (The harness is not shipped; real vitest compares
  dates by value, so the tests themselves were never affected.)

## The monthly bill: carried-forward arrears were double-counted, and deposits under-refunded

Testing the monthly rent computation itself (§7.7) confirmed the formula
is right — it matches the spec's worked example to the paisa (₹236,
₹23,836, ₹476.72, ₹36,112.72 for three unpaid months of ₹11,800) — but
exposed a pre-existing accounting flaw in how the rest of the system
treated those bills, plus related money bugs.

**Root cause.** Each month's bill includes the previous month's unpaid
balance (plus 2% interest), which is correct per the spec — but the
previous bill stayed `Unpaid`. Its balance then existed in two places.

**Consequences, all fixed:**
- **Collectable twice.** The old month stayed in the Bill Collector's
  unpaid queue, and a merchant saw a Pay button on it, while the same
  money was demanded on the newer bill. Now the previous bill is marked
  `Carried_Forward` (new status) in the same transaction that creates
  the new one; it leaves every collection queue, and the merchant
  dashboard only offers Pay on bills still owed on that row.
- **DCB reports overstated arrears.** They summed every bill's
  `total_payable`, so three unpaid months showed **₹71,748.72** of demand
  and balance instead of **₹36,112.72** — nearly double. Demand is now
  each month's own charges (`total_payable − previous_pending_amount`);
  balance excludes carried-forward bills. Applied to all five DCB
  reports. Collection % was distorted by the same error.
- **Security deposits under-refunded at closure.** Arrears deducted from
  the deposit counted each carried-forward balance twice — in the
  example above, ₹35,636 deducted instead of ₹23,836. This predates the
  new status (those rows were `Unpaid`), so **past settlements computed
  by this code may have over-deducted and are worth auditing.**
- **Wrong forfeiture tier on the anniversary.** Deposit forfeiture (50% /
  30% / 20% / 10% / 0% at 3/6/9/12 years) used the same 365.25-day-year
  arithmetic as the rent-escalation bug, so a tenant closing on a
  no-leap-day 3rd anniversary forfeited 50% instead of 30%. Now measured
  on the calendar (`utils/calendar.ts`).
- **A payment approved after its month was carried forward** credited
  the old month (overwriting its status to Paid) while the newer bill
  still demanded the money. Approval now credits the bill carrying the
  balance.

**Same bug class, from my own earlier Quarterly/Yearly work:** several
checks treated only `'Paid'` as paid, ignoring `Paid_Auto_Quarterly` /
`Paid_Auto_Yearly`:
- a shop that **prepaid a year was flagged as a six-month defaulter**
  (the trigger for legal/closure action);
- bulk-paid months **could not be revoked**, so the bulk-reversal code
  was unreachable.
Both fixed via shared `PAID_STATUSES` / `OUTSTANDING_STATUSES`
definitions (`rent-ledger.service.ts`), mirrored on the frontend.

**Tests:** `rent-ledger.test.ts` (12, including end-to-end: bills
generated by the real code, totalled by the real report helpers),
`deposit-settlement.test.ts` (6), `payment-approval.test.ts` (2). Every
bug above was confirmed by a failing test before being fixed, and each
fix was then mutation-checked (bug re-introduced → its test fails →
file restored byte-for-byte). **56 tests across 7 files, all passing**
on the verification harness described earlier.

**Needs attention on a real database:** `Carried_Forward` is a new enum
value (covered by the baseline migration you still need to generate).
Any existing database will have rolled-over bills sitting as `Unpaid`;
they need a one-off backfill (mark a bill `Carried_Forward` when the
next month's bill for the same shop has `previous_pending_amount > 0`)
before the reports and collection queues are correct for historical
months. One edge left as-is: a revoke approved on a month that has
*since* been carried forward resets that month's status; rare, but worth
a look if revokes on old months become common.

## Backfill and settlement-audit scripts (for existing databases)

The previous section asked for a one-off backfill and an audit of past
deposit settlements. Both are now tested scripts in `apps/api/prisma/`.
The backfill is dry-run by default; the audit is read-only.

**`npm run ledger:backfill-carried-forward`** (add `-- --apply` to write)
- Marks a historical bill `Carried_Forward` when it is still outstanding
  and the same shop's *next* month's bill carries its balance.
- **Lists, never flips**, bills that were *paid after* their balance had
  already been carried forward: the next bill over-demands by that
  amount, and correcting an issued bill is a finance decision, not a
  status change. Each is printed with the over-demanded amount.
- Idempotent; at write time it re-checks each row is still outstanding,
  so a bill paid between the read and the write is never overwritten.

**`npm run ledger:audit-deposit-settlements`** (read-only)
- **Exact:** re-computes each settlement's forfeiture tier from its
  recorded dates and lists any recorded in the wrong tier, with the
  over-forfeited amount.
- **Estimate:** lists settlements where carried-forward arrears were
  probably deducted twice, capped at what was actually deducted. Based
  on today's ledger data, which may have changed since, and it will
  over-estimate for a bill that was paid after being carried forward.
  Treat these as leads for manual review, not refund figures.

**Tests** (`ledger-maintenance.test.ts`, 10): partial and paid-after-carry
bills, idempotency, year boundaries, month gaps, shop isolation, dry run
never writing, and both audit checks. Mutation testing found a real gap
in my own first version: the "only matches the same shop" test couldn't
fail, because its data never gave another shop a following month to
match wrongly. It passed even with shop matching removed. Rewritten with
that trap in place; it now fails on the broken version and passes on the
correct one.

**Order on an existing database:** generate and apply the baseline
migration → run the backfill as a dry run and review the output → run
with `--apply` → run the audit and review its findings with finance.
**66 tests across 8 files, all passing** *(originally misreported here as 68 — an addition error; the per-file counts were correct)* on the verification harness.

## Leftovers built: ULB Admin settings, circulars, and the revoke edge

Checked every "not built / still open" item in this README against the
code. Bulk payment, ward mapping and Draft/Final all had screens by now.
Three backend features had **no frontend caller at all**, and one logic
edge was still open.

**ULB Admin → Settings** (new tab)
- **Payment accounts (§7.30).** There was no way to configure them, so
  **every ULB except the seeded demo one could not take UPI payments**
  (the pay link errors with "No UPI collection account configured").
  Add, list and deactivate accounts. A warning shows whenever the ULB
  lacks an active Rent & Penalty or GST UPI account.
- **Billing frequency (§7.6).** A ULB Admin couldn't switch a complex or
  shop to Quarterly/Yearly, so the whole bulk-payment feature was
  unreachable. Complex default plus per-shop override, applying from the
  next unbilled month.
- **Bug found while wiring it:** `PATCH /ulbs/:id/payment-accounts/:accountId`
  validated params with a schema declaring only `id`. `validate()` writes
  the parsed object back to `req.params` and zod strips undeclared keys,
  so `accountId` was erased — **deactivation never worked**. It was the
  only multi-parameter route in the codebase. (Not executed against real
  zod here — zod isn't installed in this environment — but the cause is
  direct from the middleware's code.)

**Circulars (§7.24)**
- Publish panel on the ULB, District and State Admin dashboards; a public
  **Circulars** button on the login page.
- **Downloads were impossible before login.** The list was public, but
  the only download link endpoint (`GET /documents/url`) requires login.
  Added a public `GET /circulars/:id/download` that signs **only a
  circular's own file**. Opening up `/documents/url` instead would have
  exposed every stored document, ID proofs included, since it signs any
  key.
- **A circular can only point at a file uploaded as a circular**
  (`circular/<uuid>.<ext>`), checked at creation and again at download.
  Otherwise staff could publish a "circular" pointing at a tenant's ID
  proof and make it public.
- **Publishing was unscoped:** `ulb_id` came straight from the request,
  so a ULB Admin could publish into another ULB's list, or state-wide.
  Now: ULB Admin → own ULB only; District Admin → a ULB in their
  district; State Admin → any ULB or state-wide. A ULB Admin token with
  no ULB is refused rather than widened to state-wide. Notifications now
  use the resolved audience, not the raw request field.

**Revoke on a carried-forward month.** A revoke raised before a month's
balance was carried forward, but approved after, reset that month to
Unpaid while the newer bill still demanded the carried part — the same
money twice. Approval is now refused with an explanation. Rejection is
still allowed, as is raising a fresh revoke against the current bill.
Which bill should carry a reversed amount is a finance decision about
issued bills, so the code doesn't guess.

**Tests:** `circulars.test.ts` (10) and `revoke-request.test.ts` (3),
each mutation-checked. **79 tests across 10 files, all passing** *(originally misreported here as 81 — an addition error)* on the
verification harness.

## Systematic check: API endpoints with no screen — and a billing blocker it found

Rather than relying on this README's own notes, every API route was
listed and checked for a frontend caller. That caught gaps no note had
mentioned. After discarding false positives (reports are called through
a generic `/reports/${id}` URL; some `GET /:id` detail routes are just
unused), the real findings were:

**Blocker — no shop could ever be billed from the app.** Commercial
details (rent, deposit, merchant) had Verify and Return in the UI but
**no Approve**. Approval is the only thing that sets
`commercial_status = Approved`, and monthly rent-ledger generation only
bills shops in that state; the seed doesn't set it either. So the whole
rent pipeline — bills, payments, arrears, notices, DCB — could not start
from the app. Added a **Commercial details** approval queue for ULB
Admin (counted in the "queue is clear" gate so it can't be hidden).

**Blocker — operational staff couldn't be created.** The ULB Admin, who
creates Case Workers, Bill Collectors, Verifiers and Engineers (§5.5),
had no users screen. Meanwhile the District Admin's form offered six
roles, five of which the server refuses (it may only create ULB Admins).
The users tab is now role-aware and shared: ULB Admin → the four staff
roles in their own ULB; District Admin → ULB Admin only. It also gains
**Deactivate/Reactivate** and **Reset ODP**, shown only on accounts the
viewer is allowed to manage (both actions had endpoints but no UI; a
leaver's account could not be switched off). The State Admin's District
Admin list gains the same two actions. `PATCH /users/:id/status` also
had no body validation: any string reached the database, and a missing
value still logged an audit event. Validated now.

**Reject was missing everywhere.** All ten maker-checker workflows
offered only Approve/Verify and Return, although §6.3 specifies Reject
as a terminal action with a mandatory reason. Every reject endpoint
shares one shape, so one generic mutation plus a clearly-worded Reject
button now appears on all 11 ULB Admin queues and all 7 Verifier-Revenue
queues. It's offered only where the server permits that role to reject.

**Notices couldn't complete.** Nothing could mark an approved notice as
served, and a Bill Collector could not raise a rent-arrears notice at
all. A Notices panel on the Bill Collector and Case Worker dashboards
now does both. Each role sees only the notice type the server lets it
serve, and the rent-arrears picker offers only shops that actually owe
rent.

**Also built in the same pass:**
- **Case Worker → Shop status & records:** mark a shop inactive (reason
  required); request reactivation of an inactive one; the six-month
  defaulters list with a one-click default notice; close concluded court
  cases; archive carried-out demolition requests; link an *existing*
  merchant to a shop instead of creating a duplicate record.
- **Shop reactivation queues** for Verifier-Revenue (verify) and ULB
  Admin (approve), each with Return and Reject. The shop list API gained
  a `reactivation_status` filter; without it no queue could be listed.
  The ULB Admin count feeds the "queue is clear" gate.
- **Generate this month's bills now** in ULB Admin Settings (the job is
  idempotent: already-billed shops are skipped).
- **Defaulters list was unscoped**, the same systemic gap as the other
  list routes: a Case Worker or ULB Admin saw every defaulting shop in
  the state. Now scoped to ULB, district or wards by caller, and failing
  closed (an empty ward list matches nothing). Covered by 4 new tests.

**Coverage result:** unreachable endpoints went from **46 to 9**. The 9
left are the payment webhook (called by the gateway, never the UI), five
`GET /:id` detail routes every screen replaces with list endpoints, and
two small real gaps: nothing yet *displays* stored photos
(`GET /documents/url`), and a Draft District/ULB can't be edited from the
UI (`PATCH /districts/:id`, `/ulbs/:id`) though §3.3 allows it.

**Tests: 83 across 10 files, all passing** on the verification harness.

## Last coverage gaps: photos, Draft edits — and locking down document links

**Stored photos are now visible to reviewers.** The complex and shop
approval queues show a *View photo* button with the §7.28 stamp (GPS and
capture time) as text, so an approver can check where and when a photo
was taken before approving. The signed link is fetched only on click.

**`GET /documents/url` hardened first.** Building on that endpoint
exposed a real hole: it signed *any* key for *any* logged-in user,
merchants included, so a merchant who learned a key could open another
tenant's ID proof. No merchant screen uses it, so it's officials-only
now, and it signs only well-formed keys minted by `uploadObject`
(`category/uuid.ext`), never an arbitrary bucket path.
`utils/storage-keys.ts`; 5 tests; mutation-checked.

**Draft District/ULB editing (§3.3).** Draft records now have an *Edit*
button (name, short code, and ULB type) next to *Final submit*.
`updateUlbSchema` didn't accept `ulb_code` at all, so a wrong ULB code
could never be corrected even while Draft; fixed with the same
validation as creation. The route still refuses any edit once Final.

**Coverage now:** 6 routes have no screen — the payment webhook (called
by the gateway) and five read-only `GET /:id` detail lookups that every
screen replaces with list endpoints. (The scanner matches URLs
regardless of method, so it reports 4: `GET /districts/:id` and
`GET /ulbs/:id` share their URL with the new `PATCH` calls.)

**Tests: 88 across 11 files, all passing** on the verification harness.
The harness itself gained `toThrow`, self-tested for the pass,
wrong-class and no-throw cases before use.

## Two systematic scans — and password hashes being sent to browsers

**Scan 1 — does any screen call an endpoint as a role the server
rejects?** Each role's dashboard was traced component by component
(not file by file, since shared components like the users tab span
roles) to every API hook it can reach, and each was checked against the
server's allowed roles. 15 flags, all explained on inspection: shared
components that declare several hooks but call only the one matching the
viewer's role, a custom-role permission the scan can't see, and one
runtime-built URL. **No real mismatches**, so the District Admin form bug
fixed earlier was the only instance of that class.

**Scan 2 — which `GET` routes show no ULB/ward/owner scoping?** Ten
flagged. Four are fine (public circular download; permission-guarded
deletion queue; non-sensitive org hierarchy; the document-link guard).
Reading the others found the most serious issue of this whole project.

**Password and ODP hashes were being sent to browsers (fixed).**
Merchant search returned whole merchant rows, `password_hash` included.
Following that up: **28 query sites across 8 files** embed full merchant
or user rows through `include` (shop detail, merchant links, transfers,
deposit settlements, revoke requests, …), so merchant and staff
`password_hash` and staff `odp_code_hash` reached the client from many
routes. Fixed at the one point every JSON response passes through:
`ok()` now strips `password_hash`, `odp_code_hash` and `token_hash` at
any depth (`lib/response.ts`, `stripSecrets`). A future
`include: { merchant: true }` can't reintroduce the leak. Only plain
objects are walked, so `Decimal` amounts and dates serialise unchanged.
The only routes that bypass `ok()` (the health check and report file
downloads) return no row data. **Hashes may already have been exposed to
anyone with staff access; consider whether password resets are
warranted.**

**Also fixed:**
- **Merchant search** now requires an exact 10-digit mobile or a name of
  3+ characters, and returns only id, name and mobile. `contains` on the
  phone number let a caller enumerate numbers statewide. The transfer
  screen now says so instead of showing an empty result for a partial
  number.
- **`GET /users/:id`** was open to *any* logged-in user, merchants
  included. Now: your own record, or an admin reading someone in their
  area (`utils/user-access.ts`).
- **`GET /shops/:id/bulk-payable`** had no area check. It now reuses
  shop detail's ULB/ward scoping.

**Tests:** `response-secrets` (6, including Decimal/Date preservation and
the actual serialised output of `ok()`) and `user-access` (6), all
mutation-checked. Mutation testing also exposed a weak first draft of my
own: it checked for leaks by iterating the very set under test, so
removing `odp_code_hash` from that set would have passed. It now names
the secrets independently. **100 tests across 13 files, all passing.**

## A one-request API crash, and write actions that ignored area

### Any user could crash the API (fixed)
Express 4 does not catch errors thrown by `async` middleware. Four
middlewares were bare `async` functions that threw: the two permission
guards (written in an earlier session of this work),
`middleware/ward-scope.ts`, and the **globally mounted** API rate limiter.
The rejection went unhandled, and on Node 20 — the API image — an
unhandled rejection terminates the process. So **any logged-in user
could take the API down for everyone**, by opening a report they
lacked permission for, by a Case Worker opening a shop outside their
wards, or simply by exceeding 200 requests a minute (a Redis error in
the limiter did the same). All four now go through `middleware/guard.ts`,
which routes errors to `next()` and the normal JSON error handler.

The existing permission tests had to change: they asserted that the
guard's promise *rejects*, which was the broken behaviour itself.
They also had a subtler gap. The "allowed" cases only checked that
`next` was called, which a refusal now satisfies too, so they'd pass
either way. Every case now checks what `next` received, and new tests
assert the guard never rejects. Reverting the fix fails 7 of them.

### Write actions didn't check whose record it was (fixed)
The read side (lists) was scoped in earlier sessions. The write side
wasn't: 61 id-targeted routes — approve / verify / return / reject /
serve / close / archive / resubmit / status / billing-frequency /
reactivation / revoke decisions, across complexes, shops, closures,
transfers, court cases, notices, payments, settlements, demolitions,
merchant links and revoke requests — loaded the record by ID and acted,
with no check of whose area it was in. A ULB Admin could approve another
ULB's complex; a Verifier could verify another ward's payment.

One middleware, `requireRecordInScope(entity)`, now guards all 61. It
resolves any record to its ULB/ward/district by following plain ID
columns only (so no relation names are assumed) and applies one rule
(`utils/record-scope.ts`): State Admin anywhere; District Admin in their
district; ULB Admin in their ULB; Case Worker, Bill Collector,
Verifier-Revenue and Engineer in their assigned wards; everyone else
refused. It runs after the role check, so the lookup only happens for
roles allowed at all. No route admitting merchants was affected (the
script checked, and skipped none).

**Operational consequence:** Verifiers and Engineers now need ward
assignments to act, consistent with how their lists were already scoped.
**Map them under ULB Admin → Wards & staff before go-live**, or they
will see and be able to do nothing.

**Tests:** `record-scope.test.ts` (13: the rule for every role, every
resolution path including via tender and via ledger, and the middleware
outcomes); permission tests reworked (16). **115 tests across 14 files,
all passing.**

**Follow-up — create routes had the same gap (fixed).** The fix above
covers actions on an existing record named in the URL. Ten create routes
name their parent in the request *body* instead, and weren't covered: a
Case Worker could open a closure, transfer or court case on a shop
outside their wards; an Engineer could create a complex in an unassigned
ward, or a shop or demolition request anywhere; Bill Collectors and Case
Workers could raise notices or record payments for shops outside their
area. `requireRecordInScope` now also reads body fields (`from: 'body'`)
and guards all ten: closures, complexes (via a new `ward` entity),
court cases, demolitions (complex *or* shop), manual merchant mapping,
both notice types, payments, shops and transfers. An absent optional
field is skipped, since each route's schema already requires its fields.
On payments, merchants skip the area rule (`skipMerchants`), because the
route already enforces that a merchant pays only for their own shop;
officials on the same route are still area-checked. Verified alongside:
closure rejects a tender belonging to a different shop, and transfer
takes no tender ID from the body. 4 new tests, both options
mutation-checked. **119 tests across 14 files, all passing.**

## Concurrency in the money paths: double-credits and lost updates (fixed)

Approval handlers read a record, checked its status, then updated it
unconditionally. Two failure modes followed.

- **Double application.** Two concurrent approvals of the same payment
  (a double-click) both passed the status check and **credited the bill
  twice**. Revoke approval had the mirror image, reversing a payment
  twice. Two *different* revoke requests against one payment could also
  both reverse it.
- **Lost updates.** The new `amount_paid` was computed from a value read
  earlier and written back. Two different payments on one bill, approved
  at the same moment, could both read ₹0 and both write their own amount,
  losing one. Revoke approval computed its debit from a value read
  *before* its transaction began.

**Fixes** (`payment.service.ts approvePayment`,
`revoke-request.service.ts districtDecideRevokeRequest`):
- **Conditional claims.** `updateMany … WHERE status = <expected>`, and
  a 409 if nothing matched. Under Postgres READ COMMITTED the losing
  request waits on the row lock, re-evaluates against the committed row,
  matches nothing and rolls back. Revoke approval claims both the request
  and the payment (`is_reversed false → true`); the reject path claims
  the request too, so a rejection can't overwrite a concurrent approval.
- **Atomic arithmetic.** `amount_paid: { increment | decrement }` inside
  the transaction, with the status derived from the value the database
  returns. The revoke debit is floored at zero.

**Tests** use stateful mocks (with increment/decrement), so they check
what the database would hold, not just which calls were made: double
approval credits once; a claim that loses the race touches nothing; two
payments on one bill both count; a double reversal and a second request
against the same payment are refused; an approval racing a rejection
reverses nothing; the debit never goes below zero.

Mutation testing caught two weaknesses in my own first draft. One
mutation was a no-op: it replaced the claim's condition with a stale
read of the same value. And my mocks treated a condition-free update as
matching nothing, which hid what an unconditional claim would really do.
Both fixed. Each claim is now caught by exactly its dedicated test.

**Honest limit:** the lost-update fix can't be demonstrated by a
sequential unit test; read-add-write and atomic increment give the same
result when run one after another. It rests on Postgres semantics, and
the real proof is a concurrent test against a live database.

**126 tests across 14 files, all passing.**

**Follow-up — the rest of the money paths.** Every approval with a side
effect beyond its own status was audited for the same race:
- **Closure approval** computes a deposit settlement *after* its
  transaction. A double-click approved twice. A second refund was
  prevented only by `DepositSettlement.tender_id` being `@unique`, and
  only after the losing request had rewritten `approved_by` and the
  tender's `ended_at`. Now a conditional claim: the loser gets a clean
  409, changes nothing, and never reaches settlement creation.
- **Rent escalation** read the escalation count, then wrote the new rent.
  Only the deduplicated scheduled job runs it today, so overlap is
  unlikely — but a stalled-job retry or any future manual trigger would
  have compounded 10.5% twice. Now it claims against the count it read;
  a losing run skips that tenancy (no duplicate rent write, audit entry
  or notification) instead of failing the batch.
- **Checked and safe as-is:** transfer and merchant-link approval
  already used conditional updates. Demolition/alteration sets an
  *absolute* revised rent, so it's idempotent. Settlement, shop,
  commercial-details and reactivation approvals change only their own
  status. Uniqueness violations already map to a clean 409 `DUPLICATE`.

One more test of my own was wrong the first time. The closure race test
gave its stale "Pending" read to the *winning* call, so the losing call
was stopped by the early status check and never reached the claim. The
test passed without testing anything. Fixed to give the stale read to
the loser; it now fails if the claim is removed.

**130 tests across 15 files, all passing.**

## Authentication review — a build break, an anonymous crash, and sessions that didn't work

### The API didn't compile (fixed)
`utils/username.ts` had `a ?? b || 'usr'`, which TypeScript rejects
(TS5076: `??` and `||` can't be mixed without parentheses). I wrote that
line several sessions ago in the Role Master work, and it has been in
every delivery since. The file is imported by the user service, so
**`tsc` failed and the API could not build.** None of my earlier checks
parse code (brace counts can't), so none caught it.

**New verification: a real type-check.** TypeScript turned out to be
installed on this machine (my earlier searches only looked for vitest
and Prisma). With no `node_modules`, every third-party import is given a
generated ambient declaration, typed as both a value and a type and
derived from the actual imports and member usages. Then `tsc` runs in
strict mode over the API (sources, tests, Prisma scripts) and the whole
web app. What survives is real errors in *this* code: wrong property
names, wrong prop values, grammar errors like TS5076. Both apps are now
clean apart from three documented environment artefacts. The check was
proven able to fail: it caught the reintroduced `??`/`||` bug, a
misspelled property, an invalid prop value and a misspelled variable.
(My first attempt, `transpileModule`, parses only. It reported "0
errors" while missing TS5076, and a probe caught that before I relied on
it.) The honest limit: data from RTK Query hooks is untyped here, so the
web check covers props, shared types and cross-file contracts, not API
response shapes. The real `tsc --noEmit` with dependencies installed is
still owed.

### Anyone on the internet could crash the API (fixed)
`loginLockoutGuard` was one more bare `async` middleware that threw, the
crash class fixed last round. My scan then only matched middleware
*returned from a factory*; this one is a directly exported function, so
it was missed. It's on the two **login** routes, which need no
authentication: fail five logins, try again, and the unhandled rejection
terminated the Node 20 process. Now wrapped in `guard()`. A new scan
matching every async `(req, res, next)` shape finds no others.

### Username enumeration via ODP requests (fixed)
An unknown username returned silently, but a real account with a live
code returned a 429, so asking twice revealed which usernames exist,
contradicting the function's own comment. Both cases are now identical
(the live code stays valid; nothing new is sent), and the login hint no
longer claims a code was just sent.

### Sessions (spec §6.1 — none of it was implemented)
- **Merchants were logged out every 15 minutes and on every reload.**
  Their refresh tokens were stored in `merchant_refresh_sessions`, but
  refresh only looked in the staff table, so every merchant refresh
  failed. Refresh now serves both account types, and logout revokes in
  both tables.
- **Every page reload broke the session for everyone.** Refresh
  returned only a token, and the web app never restored the user (its
  own comment said "a real build should follow this with GET /profile").
  With no role in state, reloads landed on "No role on this session."
  Refresh now returns the account in the same shape as login, and the
  app restores token, user, role and language.
- **Single active session:** a fresh login now revokes every other
  refresh token for that account, for staff and merchants alike.
- **Immediate revocation:** deactivation, admin password reset and ODP
  reset now revoke every refresh token (`services/session.service.ts`).
  Access tokens remain valid until they expire (15 minutes), as the
  spec's "refresh token" wording allows.

**Tests:** login-lockout (4), odp (3), sessions (7), session-revocation
(4), all mutation-checked. **148 tests across 19 files, all passing.**

## Web type-check now covers API data — and a contract check against the schema

**Typed RTK Query model.** Last round's web type-check had a blind spot:
data returned by RTK Query hooks was untyped, so every
`data.data.<field>` access in the screens went unchecked. The app uses
only a small RTK surface (`createApi`, `injectEndpoints`,
`builder.query/mutation`), so a diagnostic-only typed model of it now
maps each endpoint key to its hook name (`getRoles` → `useGetRolesQuery`,
plus `useLazy…Query` and `use…Mutation`) and carries each endpoint's
declared response and argument types into the hook. With it, **the whole
web app type-checks clean**. Probes confirmed it now catches a
misspelled field on hook data and a wrong hook argument type. My first
version typed hook errors as `unknown`, which flagged two correct
`'data' in error` checks; it now mirrors real RTK
(`FetchBaseQueryError | SerializedError`). The one remaining filtered
error, which I had labelled an RTK artefact, is actually antd's `List`
being untyped here; the real antd infers the item type from
`dataSource`.

**Frontend ↔ schema contract check.** The type-check verifies the screens
against the frontend's *declared* types. A second check verifies those
declarations against the database: each frontend response interface is
matched to its Prisma model by its own primary key, and its fields
compared. **52 interfaces match their model field for field.** The three
exceptions are expected: `DefaulterShop.complex_name` (flattened by the
server), `BulkPayable` (computed and tested), and `ReportFilters` (a
request type). The first version of this checker had a parser bug:
one-line interface declarations ran into the next interface and matched
the wrong model. An impossible "OK" in its output exposed it before I
relied on the result. Limit: top-level fields only, not nested includes.

Both checks are diagnostic tooling for this environment and aren't
shipped. With dependencies installed, `tsc --noEmit` supersedes the
first. **148 tests across 19 files, all passing.**

## Frontend ↔ backend contract checks — and merchants could never record a payment

`tsc` checks each app against its own types. It can't check that the two
apps agree with each other. Three checks now do, shipped in
`tools/contract-checks/` and run in CI (`contract-checks` job). Each exits
non-zero on any new finding; reviewed exceptions are listed explicitly in
the script's `ALLOW` set.

- **`contracts.py`** — every frontend response interface is matched to its
  Prisma model by its own primary key, and each top-level field must exist
  on that model. 52 match field for field; 3 reviewed exceptions (computed,
  flattened, or a request type).
- **`includes.py`** — every nested relation a response type reads (such as
  `shop.complex`) must be returned by the handler's `include`/`select`.
  Otherwise the screen reads `undefined` at runtime with no type error.
  45 endpoints checked, all served. It parses `include`/`select` structure
  and resolves constants such as `WARD_INCLUDE`. My first version grepped
  for relation names and was fooled by `where` clauses mentioning
  `complex:`; a probe removing a real include showed it missed the loss.
- **`bodies.py`** — every mutation's request fields are compared with the
  server's zod body schema, in both directions:
  - a field the schema doesn't declare is **silently dropped** (zod strips
    unknown keys) — the Draft-edit `ulb_code` bug of this kind was found by
    chance earlier;
  - a field the schema **requires** but the frontend never sends is a 422
    on every submit.
  50 mutations checked. It reads what each query actually sends (explicit
  body literal, argument spread, destructured rest, nested `body` field),
  and resolves schema spreads such as `...photoStampFields`.

**[CORRECTED] What `bodies.py` found in the merchant payment was a *type* gap,
not a runtime bug.** I first reported that merchants could never record a
payment. **That was wrong.** The merchant form already had a
`payment_category` field ("What does this cover?"), written in the
payment-accounts session, and it was sent via `...values`. Only the
mutation's TypeScript *type* omitted the field. `bodies.py` compares the
**declared** argument types with the server schemas; it can't see what a
form spreads in at runtime, so its finding was a real type inaccuracy
that I misreported as a runtime failure. The type now includes the field.
The duplicate "This payment is for" field I added in the mistaken fix
(two selects bound to one value) was found and removed the following
round. **Limit worth knowing:** a `bodies.py` "never sends a required
field" finding means the declared type lacks it. Check the calling form
before concluding that requests fail.

**Also tightened:** `updateRole`'s type claimed the role name was editable,
but the server's update schema doesn't accept `role_name`. It would have
been silently dropped. Nothing calls it yet; the type now excludes it.

Each check was probed before being trusted: a removed include, a relation
removed from a constant, a schema losing its stamp fields, and the
merchant bug reintroduced (the run exits 1) were all caught.
**148 tests across 19 files, all passing.**

## Query-string and translation checks — and statuses shown as raw enum names

Two more cross-file checks, added to `tools/contract-checks/` (and so to
the CI job):

- **`queries.py`** — every query string a screen sends is checked against
  its route's zod query schema. An undeclared parameter is **silently
  stripped**, so a filter a queue relies on would be ignored and the queue
  would list everything; a `limit` above the schema's maximum is a 422 on
  every load. All 41 query strings pass. Probes with a misspelled filter
  and an over-limit page size were both caught.
- **`i18n.py`** — every translation key used must exist in both `en.json`
  and `kn.json` (a missing key renders as the raw key string), and the two
  files must match. It also requires a label, in both languages, for
  **every value of every `*Status` enum in the Prisma schema**, so a new
  status added without a translation fails CI (probed with an invented
  `Written_Off` value).

**Fixed — 16 screens showed raw status enums.** The i18n check showed that
none of the `status.*` translations were used anywhere. Every screen
rendered the raw value, so a Kannada-speaking merchant saw
`Carried_Forward` or `Paid_Auto_Yearly`. A new `components/StatusText`
translates through `status.*` (falling back to a readable form such as
"Pending Verification") and replaces all 16 raw renders across the
merchant portal and every staff dashboard. Eight status values had no
label at all (`Cancelled`, `Closed`, `Completed`, `Draft`, `Executed`,
`Final`, `Pending`, `Served`); they now have English and Kannada labels.
The Kannada remains a good-faith translation, not professionally reviewed.

**Caught by the type-check, of my own making:** the script that added the
`StatusText` import inserted it after the first *line* of each file's last
import. In seven files that statement spans several lines, so the import
landed inside the `{ … }` list and broke the syntax. The type-check
flagged it immediately; imports now sit at the top of each file. Brace
counting would never have noticed.

**148 tests across 19 files, all passing; all five contract checks pass.**

## Audit-trail coverage — and merchant logout was failing (my bug, fixed)

Every exported service function that writes to the database was checked
for an audit record (`logWorkflowEvent`) on some path. Most gaps were
intentional: notifications, session revocation helpers and monthly ledger
generation are system bookkeeping, and the bulk revert runs only inside a
logged revoke decision. Five security or accountability events were
missing and now leave a record:
- **password changes** (staff and merchant),
- **merchant login** (official login was already logged),
- **ODP issuance** (never the code itself),
- **circular publication** (who published it, and for which audience),
- **the carry-forward backfill** (one summary record when applied).

Profile photo changes are logged too; a language switch isn't, since it's
a preference and would only add noise.

**Bug found on the way — merchant logout returned a 500.** Last round I
made logout revoke merchant sessions too, but it still logged
`actingUserId: <caller id>`. `workflow_logs.acting_user_id` is a foreign
key to **users**, so a merchant's ID violated it, and merchant logout
failed before clearing the cookie. Merchant events are now logged against
the `Merchant` entity with no acting user, and staff events keep the user
as actor. Every other audit call merchants can reach was checked; payment
recording already passed an actor only for officials.

**Tests:** logout audit actor for both account types, both session tables
revoked, backfill audited only when applied. The merchant-actor mutation
is caught. **152 tests across 19 files, all passing.**

## Report exports: sandboxed rendering, Kannada fonts, and a blank letterhead

PDF reports are rendered from HTML by Puppeteer, a headless Chromium
**running on the server**, and the HTML contains user-entered text (shop
and merchant names, remarks).
- **Escaping** was already applied to every value. It missed `'`, which
  isn't exploitable (no value sits in a single-quoted attribute) but is
  now covered.
- **Browser sandboxing (defence in depth):** the headless browser ran
  with JavaScript on and unrestricted network access, so a single future
  escaping slip would have let an injected `<img src="http://169.254…">`
  or `<iframe src="file:///…">` make the server fetch internal URLs or read
  local files (SSRF / file disclosure). The template needs neither, so
  JavaScript is now off, and every request except the inline document is
  aborted.
- **Excel** exports store user text as string cells, never formulas, and
  there is no CSV export, so formula injection doesn't apply.

**Kannada rendered as empty boxes in PDFs (fixed).** The report CSS asks
for "Noto Sans", but the image installed only `ttf-freefont`, which has
no Kannada glyphs, so Kannada names printed as empty boxes. The Dockerfile
now installs `font-noto` and `font-noto-kannada`, and the CSS lists
"Noto Sans Kannada". *(Package availability couldn't be verified offline
here; confirm the image builds.)*

**The letterhead read "GOVERNMENT OF —" on every report.** The spec itself
leaves the state name blank ("GOVERNMENT OF ---"), and the code copied the
blank. It now defaults to **Karnataka** (confirmed by the project owner), so
reports print "GOVERNMENT OF KARNATAKA" with no setup. `GOVERNMENT_NAME`
(in `.env.example`) overrides it. An empty value still falls back to
the default (the config uses `||`, not `??`).

**Tests** (`report-export.test.ts`, 5): markup in names is escaped;
Kannada survives escaping intact; JavaScript is off; network and
`file://` requests are aborted, including a cloud-metadata address; the
configured letterhead is printed. Both protections were mutation-checked.
**157 tests across 20 files, all passing.**

## ODP SMS delivery — staff could not log in to production

Every staff login needs a one-day password (ODP) sent by SMS, but SMS
delivery was never implemented. Configuring a provider threw "not
implemented", and with none configured the code was written to the
server log **in plain text, even in production**. So staff could not log
in to a real deployment at all, and the only workaround exposed every
account's second factor to anyone who can read logs.

**Built — a configurable HTTP SMS adapter (`apps/api/src/lib/sms.ts`).**
The spec leaves the gateway to the department, so nothing is
vendor-specific. The request is described entirely in `.env`: GET
query-string gateways such as **NIC's SMS gateway** or C-DAC Mobile Seva,
and POST JSON gateways (MSG91-style), with ready-to-adapt examples of
both in `.env.example`. Placeholders cover the mobile number, message,
sender ID, the **TRAI DLT** entity and template IDs, and the API key.
Values are URL-encoded in URLs and form bodies and JSON-escaped in JSON
bodies, so message text can't inject parameters or break the request. A
timeout applies, and `SMS_HTTP_SUCCESS_PATTERN` catches gateways (NIC's
among them) that answer HTTP 200 even when they reject a message. The
default message ends "- Govt of Karnataka". Under DLT rules,
`SMS_MESSAGE_TEMPLATE` must match your registered template exactly.

**Fixed alongside:**
- **Fail closed in production.** With no provider, a production request
  is refused with a 503 ("Login codes cannot be sent right now. Please
  contact your administrator."), and the login screen shows that
  message. The check runs *before* the username lookup, so it can't
  reveal which usernames exist. Codes are logged only in development.
- **A failed send locked users out for 24 hours.** The code was stored
  before sending, and a live code blocks new requests. On failure it's
  now cleared so the user can retry immediately. The response stays
  neutral (a distinct error would reveal the username), and operations
  get an `odp_sms_send_failed` log entry.

**Tests:** `sms.test.ts` (7: NIC-style GET, JSON POST escaping, parameter
injection, unknown placeholders, non-2xx, 200-with-rejection) and 3 new
ODP tests (production refusal for every username before any lookup,
delivery to the right number, recovery after a failed send). All four
protections were mutation-checked. **What remains is the department's:**
choose the gateway, register the DLT template, and fill in `.env`.

Also corrected: two earlier README sections still called SMS delivery and
the bulk-payment screen "not implemented / still open"; both are now
marked as superseded.

**167 tests across 21 files, all passing; type-checks and all five
contract checks pass.**

## Kannada translation — the merchant journey and the shared approval queue

**Correction first.** At the end of the previous round I said the merchant
screens were already translated. **They weren't**: only the login page,
the layout, the language toggle and the status labels used translation.

**Now translated end to end (English + Kannada):** everything a merchant
sees — the merchant dashboard (shops, rent history, the payment form and
its UPI/Swikriti buttons, notices, court cases), the forced first-login
password change, the notification bell, and the public circulars list
(plus the circular publishing form) — and the **shared approval queue**
every staff role uses (Verify/Approve, Return, Reject and their dialogs).
Two login toasts from the OTP flow had been missed when the login page
was first translated; they're done too. Spec references such as
"(§7.6.1)" were removed from user-facing text, since they're developer
notes. All keys are in both files, and the `i18n` contract check
enforces that. The Kannada is a good-faith translation and should be
reviewed by a native speaker before go-live.

**Bug found on the way:** the queue's success toast always read "Verified
-- forwarded to ULB Admin", including on the ULB Admin's own Approve
queues, telling an approver their approval had been forwarded to
themselves. It now says "Approved" for approvals, and "Verified --
forwarded for approval" for verifications.

**Still English (next):** the staff dashboards (~540 strings across 22
files, measured by a scanner that now also catches text on a line of its
own, which the first version missed), and **bell notification text**,
which the server composes in English. That needs server-side translation
keyed on each user's `preferred_language`.

**167 tests across 21 files, all passing; type-checks and all five
contract checks pass.**

**Follow-up — the field staff's screens are now bilingual.** The Bill
Collector dashboard, all Case Worker screens (dashboard, extra tabs,
commercial-details form, shop status & records), the shared Notices and
Revoke-request panels, and the Verifier-Revenue dashboard: 204 strings
in English and Kannada. The work used an extract/apply tool that
replaces text only inside component functions, adds exactly one
translation hook as each component's first statement, and dedupes
overlapping matches. Its first run refused to write because two
patterns matched the same span, which is the check doing its job.
Template literals mixing text and values became interpolated keys, and
`Shop {{number}}` is shared as `common.shopN`. Things that needed care:
- a Verifier callback named its parameter `t`, shadowing the translation
  function; it's renamed;
- enum values that were shown raw (inactive reasons, closure and notice
  types) now have labels, and the `i18n` check requires a label, in both
  languages, for every value of those enums;
- a spec reference, "(§8.4)", shown to Case Workers was replaced with
  plain language.

**Still English:** the admin screens (State, District, ULB and DB Admin;
ULB settings; wards; deletion requests), the Engineer dashboard and
Reports: about 330 strings. Plus server-composed notification text.
**167 tests, all passing; type-checks and all five contract checks pass.**

## The whole app is now bilingual — including notifications

**Every screen** now renders through i18n in English and Kannada: the
admin screens (State, District, ULB, DB Admin; ULB settings; wards;
deletion requests), the Engineer dashboard, Reports, and the shared
pieces. That's about 330 more strings, bringing the total to ~665 keys
in each language. The only English left on screen is intentional: a
sample username in the login placeholder, and "English" in the language
switcher (language names are shown in their own language).

Traps avoided and checked along the way:
- **`verifyLabel="Approve"` is a comparison value, not text.** The queue
  checks `verifyLabel === 'Approve'`, so translating it would have made
  every approval queue silently show **Verify** buttons and "verified"
  messages. It's excluded from the tool; all 9 uses are confirmed intact.
- **Arrow functions look like JSX.** `=> expr <Tag` matched the text
  pattern and would have rewritten real code in two files; the tool now
  ignores `=>`.
- **Another shadowed `t`** (a transfers `map((t) => …)` in the ULB Admin
  queues) was renamed before translating inside it.
- **Role names** now come from `roles.*`, with the English label as the
  fallback for custom Role Master roles. **Billing frequencies** are
  translated, and `BillingFrequency` joins the enums whose every value
  must have labels.
- A batch script stopped midway when an assertion caught its own regex
  converting one label twice; the partial state was inspected and
  completed rather than re-run blindly.

**Notifications.** The bell's text was composed on the server in
English. `Notification` now also stores **`message_key` +
`message_params`** (schema change, so it's included in the migrations
you'll generate). The web app renders `notif.<key>` in the reader's
*current* language, translating enum parameters such as notice type and
District/ULB, and falls back to the stored English for older rows or
unknown keys. All 14 server notification keys are translated. The i18n
check now reads the API source for `messageKey: '…'`, so a new
notification added without translations fails CI (probed). My first
version of that extractor took `'approve'` from a comparison on the same
line as a key; fixed before relying on it.

Kannada remains a good-faith translation for native-speaker review.
**170 tests across 22 files, all passing; type-checks and all five
contract checks pass.**

## Official branding — letterhead, top banner, login page, watermark

Using the three images supplied by the department:
- **Directorate of Municipal Administration emblem** (image 1): the login
  card, the top banner of every page, and the letterhead of every PDF
  report and notice.
- **Vanijya Sampada logo** (image 2): a transparent watermark on every
  page of the web app and every page of every PDF.
- **Vidhana Soudha photograph** (image 3): the login page.

**Web app**
- **Top banner:** the DMA emblem with "Directorate of Municipal
  Administration / Government of Karnataka". The words are live text,
  so they follow the English/ಕನ್ನಡ toggle; phones show the emblem only.
- **Login:** the photograph fills the left side, with the app name over a
  gradient. The form card carries the full DMA lock-up. On phones the
  photograph becomes the background behind the card.
- **Watermark** (`components/Watermark.tsx`): rendered once at the app
  root, so on every page including login. It sits above page content,
  since white cards and tables would otherwise hide it, at ~6% strength
  with clicks passing through. It sits below pop-ups (z-index 1 vs
  antd's 1000+).

**PDF reports and notices**
- **Shared renderer:** `renderHtmlToPdf` (network-sandboxed as before),
  a shared DMA letterhead, and the watermark on every printed page. The
  watermark is drawn *behind* the content (z-index −1), so every word
  and table line is printed over it; report table cells are now
  transparent so it shows evenly.
- **New: printable official notices.** `GET /notices/:id/pdf` produces
  an A4 notice with the DMA letterhead and the ULB name, bilingual
  headings (NOTICE / ಸೂಚನೆ), a reference number, the recipient, the shop
  and ward, the **outstanding amount** for rent-arrears notices (counting
  carried-forward balances once), the served date, a signature block,
  and the watermark. Merchants can download **their own served**
  notices; staff can download anything in their area, and anything not
  yet served is stamped **"DRAFT -- NOT SERVED"**. "Notice (PDF)"
  buttons are on the merchant's Notices tab and on the staff Serve-notices
  list, where the copy is printed for service.
- **Bug fixed alongside:** the merchant's notice list had no status
  filter, so merchants saw internal drafts still in verification, and
  even rejected notices. They now see served notices only, the same rule
  as the PDF.

**Verified with real PDFs**, rendered by Chromium through the actual code
and inspected page by page. The rule from an earlier fix still holds: a
3-row report stays on one page, with no extra blank page from a fixed
element. The watermark repeats on every page, and all text stays
legible. Transparent images in PDFs were re-encoded losslessly *on
every page* (a 4-page report was 2.1 MB). Pre-flattening the letterhead
onto white and pre-blending the watermark at 7% into pale JPEGs looks
identical on paper and brought it to 328 KB; a one-page report is about
130 KB. Sample outputs: `sample-report-with-letterhead.pdf`,
`sample-notice.pdf`.

**Image preparation** (`tools/branding/prepare_assets.py`, with the
originals in `tools/branding/originals/`) regenerates every asset. The
artwork (emblem, lock-up) gets transparency only on the border-connected
background. My first pass made *all* white transparent, which erased the
emblem's white Gandabherunda on coloured backgrounds; caught on a
dark-background check. The watermark gets full colour-to-alpha, so only
ink shows and nothing beneath is whitened.

**Tests:** notice-document (9: access rules for merchants and staff,
letterhead, watermark, outstanding amount, draft stamp, escaping of user
text) and report-export (letterhead and watermark-behind, updated),
mutation-checked. **180 tests across 23 files, all passing.**

## Production deployment — the web app couldn't be served, and one mistake locked everyone out

A deployment review found that the stack as delivered could not run in
production. All of the following are fixed.

- **The web app was never served.** Nothing built the React app into the
  nginx container, and its only static location was `/static/`, while
  Vite emits `/assets/…`. **New `nginx/Dockerfile`** builds the app (from
  the repository root, for the shared package) and serves it with nginx.
  It handles SPA routing, hard-caches hashed assets, and proxies `/api`
  and stored files.
- **Five failed logins by anyone locked out *everyone*.** nginx forwarded
  the client address, but the API never set `trust proxy`, so `req.ip` was
  nginx's address for every user, and the per-IP lockout became global.
  **`TRUST_PROXY`** (default off) is set to exactly 1 in production. That
  is only safe because the production compose publishes **no API port**:
  otherwise clients could forge `X-Forwarded-For` to dodge the lockout.
- **The CSP broke the whole UI.** `default-src 'self'` blocks the `<style>`
  tags antd injects, and the `blob:` photo previews. It's now the policy
  the app actually needs: inline styles allowed, no inline scripts, no
  `eval`, no framing.
- **Uploads over 1 MB failed** (nginx's default body limit), which covers
  most phone photos. Raising that limit revealed a worse problem:
  **multer had no size limit**, so the API buffered an entire upload in
  memory before the 10 MB check. Any logged-in user could have exhausted
  its memory. multer now enforces 10 MB while streaming (413), and nginx
  allows 12 MB.
- **Stored files couldn't open in browsers.** Signed links named the
  internal `minio:9000`, which only resolves inside Docker. Links are now
  signed for **`MINIO_PUBLIC_URL`**, and nginx proxies the bucket path to
  MinIO, keeping the signed `Host` header. Files are same-origin, so the
  CSP needs no storage host.
- **The compose `api` service couldn't start.** It bind-mounted the source
  over `/app`, hiding the compiled `dist/` its start command runs.
- **The JWT private key would have been baked into the API image**
  (`COPY . .` with no `.dockerignore`), readable by anyone who could pull
  it. New `.dockerignore` files keep keys and `.env` out; production
  mounts `apps/api/keys` read-only.

**Two compose files now:**
- `docker-compose.yml` — *development* infrastructure only (Postgres,
  Redis, MinIO), ports bound to **127.0.0.1**. The Getting-started steps
  are unchanged.
- `docker-compose.prod.yml` — the *production* stack. Only the web
  server publishes a port (80). The API, database, Redis and MinIO are
  internal. `POSTGRES_PASSWORD`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`
  and `MINIO_PUBLIC_URL` are **required**, with no defaults:

```bash
cp .env.example .env            # then set the production values
./scripts/generate-jwt-keys.sh  # on the server; mounted read-only
docker compose -f docker-compose.prod.yml --env-file .env up -d --build
```

**TLS:** terminate HTTPS at a load balancer, or add a `listen 443 ssl`
block to `nginx/templates/default.conf.template`. The refresh cookie is
`Secure`, so logins only persist over HTTPS.

**Verification:** the nginx template was linted here (braces,
termination, directives, and exactly which variables `envsubst` replaces:
only `MINIO_BUCKET`). A new CI job, **`production-images`**, builds both
images, runs the **real `nginx -t`** on the rendered config, and fails if
any secret file is present in the API image. Tests: `TRUST_PROXY` parsing
(off by default; `true` means one hop, not "trust any chain") and the
upload-limit error mapping. Committing a `package-lock.json` is advisable
for reproducible image builds.
**185 tests across 24 files, all passing.**

## First-run setup, storage bucket and backups

**First administrator — `npm run admin:bootstrap` (production).** The demo
seed is unsafe on a live server: its accounts use the well-known password
`Admin@123`, and it gives the State Admin a **fixed one-day password
(`123456`) valid until 2099**. It now **refuses to run when
`NODE_ENV=production`**. Use instead:

```bash
docker compose -f docker-compose.prod.yml exec api \
  npm run admin:bootstrap -- --name "Full Name" --mobile 9876543210
```

This seeds the reference data every deployment needs (system roles and
permissions, GST master), now shared with the demo seed through
`prisma/reference-data.ts`. It then creates the first State Admin with a
**random one-time password, shown once**, which must be changed at first
login. No one-day password is set, since that arrives by SMS, and no demo
data is created. It **refuses if any State Admin already exists**, so a
re-run can never reset or take over a live account, and it records the
creation in the audit log. The mobile number must be real, because
staff log-in codes are sent to it.

**Storage bucket — automatic.** Nothing created the MinIO bucket, so on a
fresh server every photo and circular upload failed until someone made it
by hand. The API now checks for it at start-up and creates it (private;
files are only served via short-lived signed links). It retries while
MinIO is still starting, treats "already exists" from two instances
racing as success, and fails start-up loudly if storage is unreachable,
so the container's restart policy tries again.

**Backups — `scripts/backup.sh` / `scripts/restore.sh`.** A profile-gated
`backup` service (never started with the stack) writes to `./backups/`:
- `db-<stamp>.dump` — a PostgreSQL custom-format dump;
- `files-<stamp>.tgz` — MinIO's data, mounted read-only;
- `SHA256SUMS-<stamp>` — checksums of both.

Backups older than `BACKUP_KEEP_DAYS` (default 14) are pruned. Schedule
it with cron, for example
`30 2 * * *  cd /opt/vanijya-sampada && ./scripts/backup.sh >> backups/backup.log 2>&1`,
and **copy `./backups` off the server**. `./backups` is git-ignored,
since it contains personal data.

`restore.sh <stamp>` verifies the checksums (and refuses a damaged
backup), asks you to type `RESTORE`, stops the app, restores the
database and files, and starts everything again. **Practise it on a test
server.**

Bug caught before shipping: my first restore used `docker compose run -v
miniodata:…`. That names a volume literally `miniodata` instead of this
project's prefixed volume, so it would have "restored" into a new, empty
volume and reported success. A dedicated `restore` service with the
volume declared in compose resolves the name correctly.

**Verification:** the backup script was run for real here, against a stub
`pg_dump` and a fake files volume. Old backups and stale partial files
were pruned and recent ones kept, the checksums verified, and the
restored files were byte-identical. Tests: bootstrap (9: argument rules,
refusal when an admin exists, password policy and hashing, no one-day
password, audit) and storage bucket (5: exists, created, race, retry,
loud failure), mutation-checked.
**199 tests across 26 files, all passing.**


## Gap-closure build — everything the specification audit found missing

**Now built:**
- **Staff login (§4.2), two steps:** password, then a login token, then
  Get OTP and the one-day password. Get OTP is greyed out while a code is
  live and shows when it is next available. The site-wide footer carries
  the build ID.
- **Login captcha:** a real reCAPTCHA/hCaptcha widget, fixing the
  **go-live blocker** where every production login was refused.
- **Tenancy lifecycle:**
  - closure types handled correctly (§7.10/7.12);
  - a daily tender-expiry job, the Case Worker **Archive**, and fresh
    entry (§7.13);
  - deposits settle from the tender's own snapshot.
- **Payment receipts** (§7.31), and the **Bulk-Paid report** (#13).
- **Reports page bilingual:** report names and all 81 column headings
  translated.
- **Admin maintenance:**
  - merchant password reset (spec default, forced change, sessions ended);
  - GST rate management (1st of a future month; the previous rate closes
    automatically);
  - division rename, staff user edit, circular removal.
- **ULB types, staffing quotas and quota exceptions** (§3.2, §7.18,
  §7.25). Quotas are enforced at creation **and at reactivation**.
- **Notice format (§7.29):** "generated by", the Commissioner's
  signature, and Kannada text.
- **Diagnostics (§5.6/§5.7):** request IDs in every error, a System Health
  page, and redacted technical logs.
- **RPO ≤ 15 min:** continuous WAL archiving (at most ~5 minutes at risk),
  base backups, `scripts/restore-pitr.sh`, and a CI `backup-drill` that
  proves archiving and a real restore.
- **Live refresh (§5.3).**
- **Design system and motion standards** (§5.17–5.25), including the
  **signature shop-opening flow** (Shop explorer), which falls back to a
  simple fade under reduced motion.

**Security and correctness bugs found along the way (all fixed):**
- nine record lists exposed state-wide data to ULB and District Admins,
  and the ward list was open to merchants; a regression test now guards
  every list endpoint;
- District Admins could run MIS reports for any ULB;
- District Admins could create or edit ULBs in other districts;
- SMS login codes could be triggered for any username without its
  password;
- merchant approval stamped new tenants onto old tenancies;
- merchants could see draft, returned and rejected notices.

**Caught before shipping, by rendering or by the tests:**
- **Deposits:** a change made during this build would have skipped
  deposit refunds on archive.
- **Notice PDF:** the signature fell onto a second page.
- **Shop-opening flow:** its final frame vanished because of a CSS class
  collision.
- **Backups:** the WAL archive volume would have been root-owned, which
  silently fills the disk.

**253 tests across 34 files, all passing; both type-checks and all five
contract checks pass.**


## Final decisions build

The remaining open items were decided and built; the reasons are in
`docs/DEPLOYMENT_DECISIONS.md`.

- **Captcha:** a self-hosted captcha is now the default. The server draws
  a distorted-text SVG and stores only a hash of the answer, for 5 minutes.
  Each challenge allows one attempt: a wrong answer burns it, and a right
  one can't be replayed. It is enforced in every environment. No site key,
  and no third party.
- **Merchant passwords:** random one-time passwords replace the spec's
  fixed `Pwd@1234`, closing the account-takeover risk. Each is sent by SMS
  and shown once to the official; the constant is removed from the code.
- **Baseline migration:** generated by `tools/db/generate_baseline.py`
  (Prisma 5 conventions). It is validated offline (tables, columns, keys,
  indexes, names), and in CI by Prisma's own `migrate diff --exit-code`, a
  real apply, and the bootstrap. My first version invented 11 many-to-many
  join tables by pairing unrelated self-relations; it now matches
  counterparts by relation name, and there are none.
- **NAC quotas:** provisional values equal to Town Panchayath's.
- **Kannada:** an automated review found no placeholder mismatches, no
  untranslated entries and consistent terminology, and fixed one stray
  "ULB". A hand sweep found **22 untranslated strings** in object literals
  (empty states, validation messages, table headings); all are
  translated. The i18n check now fails on both the object-literal pattern
  and dropped placeholders (each probed).
- **SMS and UPI:** NIC configuration and both DLT templates are provided;
  direct-to-ULB UPI is confirmed as the model.


## Closing the last spec gaps

- **District-wide circulars:** circulars can now be state-wide,
  district-wide or for one ULB (the spec's scope levels). A District Admin
  publishes for their own district; the State Admin for any. A ULB sees its
  own, its district's and state-wide circulars.
- **Ward assignment history:** each change of Bill Collector, Case Worker,
  Verifier or Engineer on a ward is recorded (who, by whom, when), via
  `GET /wards/:id/assignments` and a History button in *Wards & staff*.
- **Accessibility (WCAG 2.1 AA):**
  - Measured contrast failures fixed: antd's secondary text was 3.36:1 and
    placeholders 1.84:1, now 6.3:1 and 4.75:1.
  - The page language follows English/Kannada, for screen readers.
  - A skip-to-content link, visible keyboard focus, and a non-visual
    **text-question captcha** alternative for users who can't see the image.
- **Testing without Docker Desktop:** a `.devcontainer` lets the app run in
  **GitHub Codespaces** in the browser, for PCs where virtualisation is
  disabled.
- **Setup instructions fixed:** local development reads `apps/api/.env`;
  the README previously said to create `.env` in the project root.

The baseline migration was regenerated for the schema changes (32 tables).
**268 tests across 36 files, all passing.**


## Login codes before an SMS provider is chosen

Staff one-day login codes can now be **issued inside the application**:
- **The official** presses *Request code from administrator* after the
  password step, which notifies their administrator.
- **The administrator** issues the code under *Users → Login code*. It is
  shown once, valid 24 h, stored hashed, never self-issued, and audited.
- **The top State Admin** uses `npm run odp:issue -- --username <name>` on
  the server.

This is the automatic default in production until `SMS_PROVIDER` is set,
and a fallback afterwards. Previously, production with no SMS provider
refused every staff login.


## Look-and-feel audit (§5.17–5.25)

Every look-and-feel requirement was checked against the code, and the
visual ones against real Chromium renders; see `docs/UI_MOTION_AUDIT.md`.
Missing or only partly done at the audit, now built:

- **3D and depth (§5.20):** cursor-tracked perspective tilt on Complex and
  Shop tiles (capped at 3.5°, off under reduced motion); isometric layered
  floors; a 3D camera move in the shop-opening flow.
- **Fixed status palette (§5.17):** `StatusTag` is used everywhere; 6
  screens had private colour maps.
- **Fade-through (§5.21):** the outgoing page now fades out before the new
  one settles in.
- **Delayed loading (§5.21):** nothing under 300 ms, and a label after 2 s
  (39 skeletons); PDF downloads say "Generating…".
- **Login (§5.24):** a compact brand band on phones and a floating
  Circulars button at Level 3.
- **Smaller items:** tab-label underline, icon hovers, framed photo zoom,
  table-row tint, the ripple colour, red Return buttons, press effects on
  clickable cards and on iPhones.


## Shop explorer: realistic storefronts

- **The scene:** a sandstone market complex with lit signboards, status
  lamps and steel rolling shutters. Awnings, TO LET boards, padlocks and
  barriers follow each shop's real status.
- **Lighting:** follows the real time of day.
- **Hover:** the shutter lifts a few centimetres, light spills out, and a
  reflection follows the cursor.
- **Click:** the camera approaches in 3D, the padlock opens, the shutter
  rolls up into its box, the tube lights flicker on, the empty interior
  appears, and the details slide in. Closed and demolished shops stay
  shut.

Details in `docs/UI_MOTION_AUDIT.md`.

**Palette (refined):**
- Deep Blue and brass as the brand anchors, with terracotta and teal accents.
- Ivory stone, brushed steel, and a warm oak interior.
- Varied awning colours, stable per shop and decorative only.
- After dark, shopfronts cast warm light pools.

**Blue-and-white palette and real photographs:**
- The shop view now uses the application's own blue-and-white palette.
- Opening a shop reveals its **real, geo-stamped photograph** under the
  rolling shutter, when one is on record.


## Shop images: upload, limits, and the default image

- **Uploads:** Engineers (their wards) and ULB Admins can upload, replace
  or remove a shop's photo from the Shop explorer.
- **Size limits:** images are resized and compressed in the browser, and
  the server enforces 2 MB, images only, checked by content.
- **Default image:** shops without a photo show the ULB default, then the
  State default, then a built-in standard image. Each default can be
  changed or reset at any time.
- **Evidence:** the geo-stamped verification photo is never overwritten.


## Display settings: day/night view, text size, zoom

- **Where:** a **Display** button in the header and on the login page.
- **Theme:** Day, Night, or Auto (night from 6 pm to 6 am). The night
  colours meet WCAG AA contrast.
- **Text size:** four sizes, which enlarge controls too.
- **Zoom:** 80–150 %.
- **Remembered** on the device and applied before the first paint.


## Live updates (§5.3): push instead of polling

Screens now update the moment something changes: when a payment is
approved, a notice served, or a shop verified. No 30-second wait.

- **How it works:** `GET /api/events` is a Server-Sent Events stream, fed
  through Redis pub/sub, so it works with several API instances.
- **Privacy:** events say only *which kind* of data changed, never IDs or
  content. Each screen re-fetches its own area-scoped data, and personal
  notifications go only to their recipient.
- **Token expiry:** a stream closes when its login token expires, and the
  browser reconnects with a fresh one.
- **Proxy:** nginx streams it unbuffered.
- **Fallback:** polling runs only while the stream is down.


## UPI QR codes on desktop, and spec-aligned column names

- **UPI QR on desktop:** paying by UPI on a desktop now shows the payment
  link as a **scannable QR code**, with payee, UPI ID, amount and purpose;
  phones still open the UPI app directly.
- **The encoder** (`apps/web/src/lib/qr.ts`) is dependency-free, and is
  verified on every push by the CI job `qr-code`: OpenCV's independent
  decoder must read versions 1–20 back exactly, including a
  Kannada-payee link that needs version 11.
- **Column names:** 21 database columns now carry the specification's names
  through Prisma `@map`, with no code change and a regenerated baseline
  migration (safe because no database has been deployed yet).


## Final code sweep: one deferred rule found and finished

A search of the source for unfinished markers found one real item: a note
in the rent-escalation service saying the Quarterly/Yearly **freeze** rule
was deferred.

- **The freeze now holds by construction:** prepaid months are billed at
  the rent in force when the bulk payment is approved, and the monthly job
  never re-creates an existing bill, so a 3-year escalation applies only
  after the freeze. The stale note has been replaced.
- **Bug fixed along the way:** when a bulk payment was **revoked**, the
  reverted months kept the old rent, so after an escalation they were
  billed **10.5% too low**. Due months are now re-priced at the current
  rent and that month's GST rate; months not yet due are removed so normal
  billing creates them. A bill still referenced elsewhere is re-priced,
  never deleted.
- **Tests:** covered and mutation-checked. **306 tests across 40
  files, all passing.**
