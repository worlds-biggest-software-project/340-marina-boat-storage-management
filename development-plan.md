# Marina & Boat Storage Management — Phased Development Plan

> Project: 340-marina-boat-storage-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model proposals. It adopts **data-model-suggestion-1 (Entity-Centric Normalized Relational, 18 tables)** as the canonical schema, because the platform targets multi-tenant marina groups and municipal authorities that need rich occupancy/revenue analytics, departmental P&L, and EPA/ABYC/USCG compliance reporting — all of which depend on referential integrity and standard SQL joins rather than JSONB extraction. JSONB is used surgically (settings, dynamic-pricing snapshots) as that model recommends.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | **TypeScript (Node 22 LTS)** | The product is API- and integration-heavy (Stripe, QuickBooks/Xero, FuelCloud, Twilio/SendGrid), shares a domain model with a web frontend, and benefits from one language end-to-end. AI features call hosted LLM APIs rather than running local models, so Python's ML stack is not required. |
| API framework | **Fastify 5 + `@fastify/swagger`** | High throughput for webhook ingestion (fuel/payment events), first-class JSON Schema validation that doubles as the OpenAPI 3.1 source of truth (standards.md requirement), and a plugin model that maps cleanly to marina modules. |
| Schema/validation | **TypeBox** | Generates JSON Schema (Draft 2020-12, per standards.md) and static TS types from one definition, feeding both Fastify validation and the published OpenAPI spec. |
| ORM / DB access | **Drizzle ORM** | Typed SQL close to the normalized schema, first-class Postgres support (UUID, JSONB, partial indexes, partitioning), and SQL-first migrations — a better fit than a heavy ORM for analytics-style joins. |
| Database | **PostgreSQL 16** | The chosen data model relies on UUID PKs, JSONB, `CHECK` constraints, partial indexes, `GIN` indexes, range-partitioned audit log, and row-level multi-tenancy. SQLite cannot serve these. |
| Multi-tenancy | **Shared schema + `marina_id` scoping + Postgres RLS** | All 18 tables are `marina_id`-scoped (per the data model). Row-Level Security enforces tenant isolation defensively at the DB layer. |
| Task queue | **BullMQ on Redis** | Async work: webhook processing, invoice batch cycles, SMS/email dispatch, LLM calls (pricing, briefings, compliance reports), reorder alerts. Redis also backs reservation soft/hard hold locks. |
| Cache / locks | **Redis 7** | Slip-hold locks, rate limiting, idempotency keys for webhooks, short-lived availability caches. |
| Frontend | **Next.js 15 (App Router) + React 19 + TypeScript** | Two surfaces: a staff operations console (slip map, reservations, work orders, dashboards) and a customer portal. Server Components reduce client weight; shared types with the backend via a monorepo package. |
| Slip map rendering | **`react-konva` (HTML5 Canvas)** | Drag-and-drop slip assignment over hundreds of slips needs canvas-level performance and hit-testing; `map_x/map_y/map_rotation` from the schema feed it directly. |
| UI components | **shadcn/ui + Tailwind CSS** | Fast, accessible, themeable component layer for dashboards, tables, and forms. |
| Mobile technician app | **Expo (React Native) + WatermelonDB** | Offline-first field workflows (photos, meter readings, job status) per features.md v1.1; WatermelonDB gives a local SQLite store with sync. Ships in a later phase. |
| Payments | **Stripe (PaymentIntents, Billing, Terminal, Connect)** | PCI DSS v4.0 burden offloaded (Level 1 provider), supports 3D Secure 2.0, payment links, ACH, and physical terminals — all required revenue flows. No PAN ever stored (`payment_methods` holds tokens only). |
| Accounting sync | **QuickBooks Online + Xero (OAuth 2.0)** | Both use OAuth 2.0 with short-lived tokens; sync invoices/payments/customers via background jobs (standards.md). |
| Fuel integration | **FuelCloud REST API + webhook ingestion** | Dispenser events → `fuel_transactions` reconciliation; feeds anomaly detection. |
| Comms | **Twilio (SMS) + SendGrid (email)** | Invoice/reservation notifications, marketing, AI-personalised briefings. |
| LLM provider | **Anthropic Claude API (server-side) via AI SDK** | Powers dynamic pricing rationale, arrival briefings, compliance report drafting, churn scoring. Prompt-cached, structured-output (tool) calls. |
| Object storage | **S3-compatible (AWS S3 / MinIO self-host)** | Work-order photos, signed contract PDFs, spill-incident images (`photo_urls` fields). |
| Calendar export | **`ical-generator`** | iCalendar RFC 5545 / RFC 9073 reservation export (standards.md). |
| Auth | **OAuth 2.0 + OpenID Connect via Auth.js (NextAuth)** | Multi-tenant SaaS staff + boater login; supports external IdPs (Google, Microsoft Entra) per standards.md. JWT sessions carry `marina_id` + role. |
| Testing | **Vitest (unit) + Supertest (API integration) + Playwright (E2E) + Testcontainers (real Postgres/Redis)** | Layered strategy; Testcontainers gives real-DB integration without external infra. |
| Lint / format / types | **ESLint (typescript-eslint) + Prettier + `tsc --noEmit`** | Standard TS quality gate. |
| Package manager / monorepo | **pnpm workspaces + Turborepo** | Shared `packages/types`, `packages/db`, `packages/sdk` across api/web/mobile. |
| Containerisation | **Docker + docker-compose** | Self-host story (api, web, postgres, redis, minio). Cloud-native multi-tenant deploy is the primary mode (README). |
| CI | **GitHub Actions** | Lint, typecheck, test (with service containers), build, OpenAPI diff check. |

### Project Structure

```
marina-boat-storage-management/
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── .github/workflows/ci.yml
├── packages/
│   ├── types/                      # Shared TypeBox schemas + inferred TS types
│   │   └── src/{vessel,slip,reservation,invoice,...}.ts
│   ├── db/                         # Drizzle schema, migrations, seed, RLS policies
│   │   ├── src/schema/{marinas,slips,vessels,reservations,contracts,
│   │   │                work-orders,parts,fuel,utilities,invoices,
│   │   │                payments,environmental,messages,integrations,audit}.ts
│   │   ├── migrations/
│   │   ├── seed/
│   │   └── rls/policies.sql
│   └── sdk/                        # Generated typed API client (from OpenAPI)
├── apps/
│   ├── api/                        # Fastify backend
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── plugins/            # auth, db, redis, swagger, error-handler, rls-context
│   │   │   ├── modules/
│   │   │   │   ├── marinas/        # routes + service + repo per module
│   │   │   │   ├── slips/
│   │   │   │   ├── vessels/
│   │   │   │   ├── customers/
│   │   │   │   ├── reservations/
│   │   │   │   ├── contracts/
│   │   │   │   ├── workorders/
│   │   │   │   ├── parts/
│   │   │   │   ├── fuel/
│   │   │   │   ├── utilities/
│   │   │   │   ├── billing/        # invoices + payments
│   │   │   │   ├── environmental/
│   │   │   │   ├── messaging/
│   │   │   │   ├── reporting/
│   │   │   │   └── ai/             # pricing, briefings, compliance, churn
│   │   │   ├── integrations/       # stripe, quickbooks, xero, fuelcloud, twilio, sendgrid
│   │   │   ├── jobs/               # BullMQ processors
│   │   │   ├── lib/                # money, hin, ical, llm-client, idempotency
│   │   │   └── webhooks/           # stripe, fuelcloud, accounting webhook routes
│   │   └── test/{unit,integration,fixtures}
│   ├── web/                        # Next.js staff console + customer portal
│   │   └── src/app/{(staff),(portal),api}
│   └── mobile/                     # Expo technician app (later phase)
└── docs/
    └── openapi.json                # Published spec artifact
```

The structure groups by **domain module**, not by phase. Each phase adds modules, routes, jobs, and UI without restructuring.

---

## Phase 1: Foundation — Monorepo, Database, Auth, Multi-Tenancy

### Purpose
Establish the skeleton every later phase depends on: the pnpm/Turborepo monorepo, the Fastify server with Swagger/OpenAPI, the full Postgres schema from data-model-suggestion-1 expressed as Drizzle migrations, Row-Level Security tenant isolation, and OIDC authentication carrying `marina_id` + role. After this phase the system boots, runs migrations, authenticates a staff user, and serves a healthcheck plus an empty but valid OpenAPI 3.1 document.

### Tasks

#### 1.1 — Monorepo & toolchain scaffold

**What**: Initialise pnpm workspaces, Turborepo, shared TS config, ESLint/Prettier, Vitest, and Docker Compose for local Postgres/Redis/MinIO.

**Design**:
- `pnpm-workspace.yaml` includes `packages/*` and `apps/*`.
- Root `tsconfig.base.json` with `"strict": true`, `"target": "ES2023"`, path aliases `@marina/types`, `@marina/db`, `@marina/sdk`.
- `turbo.json` pipeline: `build` → depends on `^build`; `test`, `lint`, `typecheck` as parallel tasks.
- `docker-compose.yml` services:
  - `postgres:16` (env `POSTGRES_DB=marina`, volume), exposes 5432
  - `redis:7`, exposes 6379
  - `minio` with console, exposes 9000/9001
- `.env.example` lists: `DATABASE_URL`, `REDIS_URL`, `S3_ENDPOINT/KEY/SECRET/BUCKET`, `JWT_SECRET`, `OIDC_*`, integration keys (left blank).

**Testing**:
- `Unit: tsconfig resolves @marina/types import → compiles with no error`
- `Integration: docker compose up → postgres accepts connection on 5432, redis PING returns PONG`
- `CI: pnpm install --frozen-lockfile succeeds on clean checkout`

#### 1.2 — Database schema & migrations (data-model-suggestion-1)

**What**: Implement all 18 tables, indexes, and `CHECK` constraints from data-model-suggestion-1 as Drizzle schema + a single baseline migration.

**Design**:
- One file per module in `packages/db/src/schema/`. Use `uuid().defaultRandom()`, `bigint` for `*_cents`, `jsonb` for `settings_json`/`dynamic_price_json`/`changes_json`.
- HIN constraint as raw SQL check: `hin ~ '^[A-Z]{3}[A-Z0-9]{5}[A-Z0-9]{4}$'`.
- `audit_log` declared `PARTITION BY RANGE (created_at)`; migration creates the current + next month partitions and a default partition.
- Money helper `packages/db` re-exports `lib/money` (see 1.5).
- All foreign keys exactly as specified (e.g. `work_order_lines.work_order_id … ON DELETE CASCADE`).
- Drizzle config points migrations at `packages/db/migrations`; `pnpm db:migrate` and `pnpm db:generate` scripts.

Reference DDL is authoritative in `data-model-suggestion-1.md` — implement it verbatim. Key entities (abbreviated):

```ts
// packages/db/src/schema/slips.ts
export const slips = pgTable('slips', {
  id: uuid('id').defaultRandom().primaryKey(),
  marinaId: uuid('marina_id').notNull().references(() => marinas.id),
  slipNumber: text('slip_number').notNull(),
  slipType: text('slip_type').notNull(), // CHECK in migration
  maxLoaCm: integer('max_loa_cm').notNull(),
  status: text('status').notNull().default('available'),
  mapX: numeric('map_x', { precision: 8, scale: 2 }),
  mapY: numeric('map_y', { precision: 8, scale: 2 }),
  seasonalRateCents: bigint('seasonal_rate_cents', { mode: 'bigint' }),
  // ...full column set per data-model-suggestion-1
}, (t) => ({ uniqSlip: unique().on(t.marinaId, t.slipNumber) }));
```

**Testing**:
- `Integration (Testcontainers): run baseline migration → 18 tables exist (query information_schema)`
- `Integration: insert vessel with hin='XYZ12345A101' → succeeds; hin='bad' → CHECK violation`
- `Integration: insert slip with slip_type='spaceship' → CHECK violation`
- `Integration: delete work_order → cascades to work_order_lines`
- `Integration: insert audit_log row dated this month → lands in current partition`

#### 1.3 — Row-Level Security & tenant context

**What**: Enable RLS on every `marina_id`-scoped table and a request-scoped DB session that sets `app.current_marina_id`.

**Design**:
- `packages/db/rls/policies.sql`: `ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;` plus policy `USING (marina_id = current_setting('app.current_marina_id')::uuid)` for each tenant table. A separate `app_superuser` role bypasses for migrations.
- Fastify `rls-context` plugin: per request, opens a transaction-bound connection and runs `SET LOCAL app.current_marina_id = $1` from the JWT before handlers execute.

**Testing**:
- `Integration: with marina_id=A set, SELECT * FROM slips returns only marina A rows`
- `Integration: attempt to UPDATE a marina B slip while context=A → 0 rows affected`
- `Integration: missing app.current_marina_id → query errors (fail closed)`

#### 1.4 — Fastify server, OpenAPI, error model

**What**: Boot Fastify with `@fastify/swagger` (OpenAPI 3.1), a global error handler, healthcheck, and the DB/Redis plugins.

**Design**:
- `GET /healthz` → `{ status: 'ok', db: 'up', redis: 'up' }` (checks both).
- Error handler emits RFC 9457 problem+json: `{ type, title, status, detail, instance }`. Validation errors (TypeBox) map to `400` with field paths.
- `GET /openapi.json` serves the generated spec; a build step writes `docs/openapi.json`.
- Standard pagination envelope: `{ data: [], page, pageSize, total }` with RFC 8288 `Link` headers (next/prev).

**Testing**:
- `Integration: GET /healthz with db+redis up → 200 {status:'ok'}`
- `Integration: GET /healthz with redis down → 503`
- `Unit: error handler given TypeBox ValidationError → 400 problem+json with field path`
- `Integration: GET /openapi.json → valid OpenAPI 3.1 (validated by openapi schema validator)`

#### 1.5 — Money & domain primitives

**What**: A `Money` helper enforcing integer-cents arithmetic and a `HIN` validator/normaliser.

**Design**:
```ts
// lib/money.ts — all monetary values are bigint cents
export const Money = {
  add: (a: bigint, b: bigint) => a + b,
  mulRate: (cents: bigint, rate: number) => BigInt(Math.round(Number(cents) * rate)),
  format: (cents: bigint, currency = 'USD') => /* Intl.NumberFormat */,
};
// lib/hin.ts
export function isValidHin(hin: string): boolean; // 12-char USCG MIC format
export function normaliseHin(hin: string): string; // uppercase, strip spaces
```

**Testing**:
- `Unit: Money.mulRate(35000n, 1.25) → 43750n`
- `Unit: isValidHin('XYZ12345A101') → true; isValidHin('xyz12') → false`
- `Unit: normaliseHin(' xyz 12345a101 ') → 'XYZ12345A101'`

#### 1.6 — Auth: OIDC + role-based access

**What**: Auth.js (OIDC) login issuing a JWT with `sub`, `marina_id`, `role`; Fastify auth plugin + `requireRole` guard.

**Design**:
- Roles from `staff.role` CHECK set: `owner | manager | dock_master | service_tech | fuel_attendant | office | accounting`, plus `customer` (portal) and `system`.
- JWT claims: `{ sub, marina_id, role, scope[] }`. `requireRole(...roles)` decorator returns `403` problem+json on mismatch.
- Customer portal tokens are scoped to a single `customer_id`.

**Testing**:
- `Integration: request with valid manager JWT to manager-only route → 200`
- `Integration: service_tech JWT to accounting route → 403`
- `Integration: expired/invalid JWT → 401`
- `Integration: customer token reading another customer's invoice → 403`

---

## Phase 2: Core Inventory — Marinas, Slips, Vessels, Customers

### Purpose
Build the foundational records the whole product revolves around. After this phase an operator can create a marina, lay out its slips/berths/dry-stack/moorings, register customers and their vessels (ISO 8666 dimensions, USCG HIN), and query slip availability by vessel size. This is the data substrate for reservations, billing, and the slip map.

### Tasks

#### 2.1 — Marina & staff CRUD

**What**: CRUD for `marinas` and `staff`, including the `settings_json` rate/policy config.

**Design**:
- `POST/GET/PATCH /marinas`, `GET /marinas/:id`. Owner/manager only for writes.
- `settings_json` validated against a TypeBox schema: `seasonal_rate_per_foot_cents`, `transient_rate_per_foot_cents`, `fuel_markup_pct`, `utility_billing ∈ {included,metered,flat_fee}`, `abyc_lithium_policy`, `epa_clean_marina_certified`, `operating_hours`.
- Staff CRUD scoped to marina; `is_active` soft-delete.

**Testing**:
- `Integration: create marina with valid settings_json → 201, slug unique`
- `Integration: duplicate slug → 409`
- `Unit: settings_json missing required rate → ValidationError naming field`
- `Integration: deactivate staff → is_active=false, excluded from active list`

#### 2.2 — Slip CRUD & layout

**What**: CRUD for `slips` across all `slip_type` values with map coordinates and amenities.

**Design**:
- `POST /marinas/:id/slips`, `PATCH /slips/:id`, `GET /marinas/:id/slips?type=&status=`.
- Bulk create endpoint `POST /marinas/:id/slips:bulk` for laying out a dock row.
- Map coordinates `map_x/map_y/map_rotation` accepted but not required (positioning happens in Phase 4 UI).

**Testing**:
- `Integration: create wet_slip with max_loa_cm=1524 → 201`
- `Integration: bulk create 30 slips → all persisted, unique (marina,slip_number) enforced`
- `Integration: duplicate slip_number in same marina → 409`

#### 2.3 — Customer & vessel CRUD

**What**: CRUD for `customers` and `vessels`, with HIN validation and compliance-expiry fields.

**Design**:
- `POST /customers`, `POST /customers/:id/vessels`. `customer_type ∈ {individual,club_member,commercial,government}`.
- Vessel write validates/normalises HIN (1.5), CIN optional, dimensions in cm (ISO 8666). `registration_expiry`, `insurance_expiry`, `has_lithium_batteries` captured.
- `GET /vessels/:id` returns vessel + owner summary.

**Testing**:
- `Integration: create vessel with valid HIN + loa_cm → 201`
- `Integration: duplicate HIN within marina → 409 (partial unique index)`
- `Integration: vessel with insurance_expiry in past → persists, flagged by compliance query`
- `Unit: vessel without loa_cm → ValidationError (NOT NULL)`

#### 2.4 — Slip availability engine

**What**: A query service returning slips that physically fit a vessel and are free for a date range.

**Design**:
```ts
findAvailableSlips(input: {
  marinaId: string; loaCm: number; beamCm?: number; draftCm?: number;
  startDate: string; endDate: string; slipType?: SlipType;
  requiresLithiumCharging?: boolean;
}): Promise<SlipMatch[]>
```
- Filter: `max_loa_cm >= loaCm` (and beam/draft when provided), `status != 'out_of_service'/'maintenance'`, no overlapping confirmed reservation/contract in `[startDate,endDate)`.
- If `requiresLithiumCharging`, require `lithium_charging_approved = true` (ABYC E-13).
- Returns matches sorted by tightest fit (`max_loa_cm ASC`) to minimise wasted linear footage.

**Testing**:
- `Unit (mocked repo): vessel loa=1500 → excludes slips with max_loa_cm=1400`
- `Integration: slip with overlapping confirmed reservation → excluded`
- `Integration: lithium vessel + only non-approved slips free → empty result`
- `Integration: returns tightest-fit slip first`

---

## Phase 3: Reservations, Contracts & Calendar

### Purpose
Deliver the booking heart of the product. After this phase staff can create transient and seasonal bookings with soft/hard holds, run the reservation lifecycle (pending → confirmed → checked_in → checked_out), manage long-term contracts with renewals, and export bookings as iCalendar. This unblocks billing (Phase 5) and the slip map (Phase 4).

### Tasks

#### 3.1 — Reservation lifecycle

**What**: Create/confirm/check-in/check-out/cancel transient and short-term reservations with hold semantics.

**Design**:
- State machine: `pending → confirmed → checked_in → checked_out`; `cancelled`/`no_show` terminal from earlier states. Illegal transitions → `409`.
- `hold_type`: `soft` (advisory, auto-expires via BullMQ delayed job) vs `hard` (blocks the slip). Holds use a Redis lock keyed `hold:{slipId}:{dateRange}` to prevent double-booking under concurrency.
- On create, re-validate slip fits vessel and is free (calls 2.4). `total_cents` computed from rate × LOA-feet × nights (dynamic pricing applied in Phase 8 fills `dynamic_price_json`).
- Endpoints: `POST /reservations`, `POST /reservations/:id/confirm|check-in|check-out|cancel`.

**Testing**:
- `Integration: two concurrent hard holds on same slip/dates → one 201, one 409`
- `Integration: check-out from pending (skipping states) → 409`
- `Integration: soft hold expires after TTL → slip becomes available`
- `Unit: total_cents = rate_per_foot * loaFeet * nights for transient`

#### 3.2 — Contracts & renewals

**What**: Seasonal/annual/dry-stack/mooring contracts with draft→signed→active→renewed lifecycle.

**Design**:
- `POST /contracts`, `POST /contracts/:id/send|sign|cancel|renew`. `auto_renew` flag drives a scheduled job (Phase 7 billing) that emits `renewal_offer_sent_at` and clones the contract on renewal.
- Signing stores `signature_url` (e-sign provider stub now; PDF assembly later) and `signed_at`.
- A contract reserves a slip for its term; availability engine treats active contracts as occupancy.

**Testing**:
- `Integration: sign draft contract → status=signed, signed_at set`
- `Integration: renew active contract → new contract row, original status=renewed`
- `Integration: active contract blocks slip in availability query`
- `Integration: sign already-cancelled contract → 409`

#### 3.3 — iCalendar export

**What**: Export reservations/contracts as RFC 5545 `.ics` with RFC 9073 structured data.

**Design**:
- `GET /marinas/:id/calendar.ics?from=&to=` → VEVENT per reservation: `DTSTART`/`DTEND` from dates, `SUMMARY` = vessel + slip, `STRUCTURED-DATA` JSON with `{ slip_number, vessel_name, reservation_type }`.
- Per-customer feed `GET /customers/:id/calendar.ics` (portal token).

**Testing**:
- `Unit: 2 reservations → ICS with 2 VEVENTs, valid per ical parser`
- `Unit: STRUCTURED-DATA contains slip_number + vessel_name`
- `Integration: date filter excludes out-of-range reservations`

---

## Phase 4: Slip Map & Staff Operations Console

### Purpose
Ship the signature UX that incumbents lead with: a visual, drag-and-drop slip map and a fast operations console. After this phase staff see real-time occupancy on a canvas map, drag a vessel onto a slip to assign it, and perform one-click berth operations (depart, invoice, swap, view customer) — matching Storable and Harbour Assist table stakes.

### Tasks

#### 4.1 — Next.js app shell, auth, and layout

**What**: Next.js App Router shell with Auth.js OIDC login, role-aware navigation, and the `@marina/sdk` client.

**Design**:
- Route groups `(staff)` and `(portal)`. Session middleware redirects unauthenticated users; `marina_id`/`role` from session.
- `@marina/sdk` generated from `docs/openapi.json`, used in Server Components and Server Actions.

**Testing**:
- `E2E (Playwright): unauthenticated visit to /staff → redirect to login`
- `E2E: manager logs in → sees Slips, Reservations, Work Orders, Reports nav`
- `E2E: fuel_attendant login → service/accounting nav hidden`

#### 4.2 — Visual slip map (canvas)

**What**: `react-konva` map rendering slips by `map_x/map_y/map_rotation`, colour-coded by status, with pan/zoom.

**Design**:
- Slip shapes sized to `max_loa_cm`; fill by `status` (available=green, occupied=blue, reserved=amber, maintenance=grey, out_of_service=red).
- Layout editor mode: drag a slip to update `map_x/map_y`, rotate handle updates `map_rotation`; saved via `PATCH /slips/:id`.
- Clicking an occupied slip opens an occupant panel (vessel, customer, reservation/contract).

**Testing**:
- `E2E: map renders N slips matching API count`
- `E2E: drag slip to new position → PATCH persists map_x/map_y`
- `E2E: occupied slip shows blue and opens occupant panel on click`

#### 4.3 — Drag-and-drop assignment & one-click berth ops

**What**: Drag an unassigned vessel onto a slip to create a reservation; one-click depart/invoice/swap/view-customer.

**Design**:
- Drag-drop fires `POST /reservations` (or assigns to active contract). Drop onto an ill-fitting slip is rejected client-side using the fit rule and surfaces a toast.
- "Depart" → `check-out`; "Invoice" → navigates to draft invoice (Phase 5); "Swap" → moves reservation to another free slip atomically.

**Testing**:
- `E2E: drag fitting vessel to free slip → reservation created, slip turns blue`
- `E2E: drag to undersized slip → rejected with message, no API call`
- `E2E: swap to occupied slip → blocked; swap to free slip → both maps update`

#### 4.4 — Reservations & customer/vessel management screens

**What**: List/detail/create screens for reservations, customers, and vessels.

**Design**:
- Reservations table with status filters and lifecycle action buttons (wired to 3.1).
- Customer detail aggregates vessels, reservations, contracts, invoices, message history.

**Testing**:
- `E2E: create transient reservation via form → appears in list as pending`
- `E2E: check-in action moves row to checked_in`
- `E2E: customer detail lists all owned vessels`

---

## Phase 5: Billing, Invoicing & Payments (Stripe)

### Purpose
Turn occupancy and service into revenue. After this phase the platform generates invoices from any source (reservation, contract, work order, fuel, utility, retail), collects payment via Stripe (cards, ACH, payment links, 3D Secure 2.0), records payments, and exposes a customer portal for self-service viewing and payment — the universal table-stakes capability.

### Tasks

#### 5.1 — Invoice generation (polymorphic source)

**What**: Create invoices with `source_type`/`source_id`, line items, tax, and totals.

**Design**:
- `POST /invoices` accepts source ref + line items; service can auto-build lines from a reservation/contract/work order/utility reading.
- Totals computed with `Money`; `status` lifecycle `draft → sent → viewed → paid|partial|overdue|void`.
- `GET /invoices/:id`, `POST /invoices/:id/send` (queues email via Phase 6 stub initially).

**Testing**:
- `Integration: invoice from reservation → line total = reservation.total_cents`
- `Unit: subtotal + tax = total`
- `Integration: void a paid invoice → 409`

#### 5.2 — Stripe payments & webhooks

**What**: PaymentIntents, payment links, ACH, refunds; idempotent Stripe webhook ingestion → `payments`.

**Design**:
- `POST /invoices/:id/pay` creates a PaymentIntent (3D Secure 2.0 enabled); `payment_link` flow for no-login collection.
- Webhook `POST /webhooks/stripe` verifies signature, deduped by Redis idempotency key on `event.id`; on `payment_intent.succeeded` upserts `payments` (status=succeeded), marks invoice paid/partial.
- No PAN stored — only `stripe_payment_intent_id` and tokenised method refs (PCI DSS v4.0).

**Testing**:
- `Integration (mocked Stripe): valid webhook signature → payment recorded, invoice paid`
- `Integration: invalid signature → 400, no DB write`
- `Integration: duplicate event.id → processed once (idempotent)`
- `Integration: refund event → payment.refund_cents updated, status=refunded`

#### 5.3 — Bulk invoice & contract billing cycles

**What**: Batch-generate annual mooring/harbour-dues invoices for all active contracts (Harbour Assist parity).

**Design**:
- `POST /marinas/:id/billing-runs` enqueues a BullMQ job iterating active contracts, creating invoices, and dispatching them. Returns a run id; `GET /billing-runs/:id` reports progress/results.

**Testing**:
- `Integration: billing run over 50 active contracts → 50 invoices created`
- `Integration: contract already invoiced this cycle → skipped (no duplicate)`
- `Integration: run with one failing contract → others succeed, failure recorded`

#### 5.4 — Customer portal

**What**: Portal pages for invoice viewing, payment, booking requests, and calendar.

**Design**:
- `(portal)` routes behind customer-scoped token: invoice list/detail with Stripe payment element, booking-request form, `.ics` subscribe link (3.3).
- `viewed_at` set on first invoice view.

**Testing**:
- `E2E: customer logs in, pays invoice via Stripe test card → invoice shows paid`
- `E2E: customer submits booking request → creates pending soft-hold reservation`
- `E2E: customer cannot see another customer's invoices (403 enforced)`

---

## Phase 6: Service, Work Orders, Parts & Communications

### Purpose
Add the service department and the messaging backbone. After this phase staff manage work orders end-to-end (estimate → approve → in-progress → completed → invoiced) with labour/parts line items, track parts inventory with reorder alerts, and the platform sends email/SMS for invoices, confirmations, and reminders.

### Tasks

#### 6.1 — Work orders & line items

**What**: Work-order CRUD with `work_order_lines` (labour, part, sublet, fee, haul/launch) and status lifecycle.

**Design**:
- `POST /work-orders`, line endpoints, `POST /work-orders/:id/transition`. Lifecycle: `requested → estimated → approved → in_progress → (waiting_parts) → completed → invoiced`.
- Line totals roll up to WO `subtotal/tax/total`. Part lines decrement `parts.qty_on_hand` on completion. `hours_actual` vs `hours_estimated` captured for later technician analytics.

**Testing**:
- `Integration: add labour + part lines → WO total = sum of line totals`
- `Integration: complete WO with part line → parts.qty_on_hand decremented`
- `Integration: approve before estimate → 409`
- `Integration: completed WO → can create invoice (source_type=work_order)`

#### 6.2 — Parts inventory & reorder

**What**: Parts CRUD, stock adjustments, and automated reorder-point alerts.

**Design**:
- `qty_on_hand` adjusted by WO consumption and manual receipts. A job checks `qty_on_hand <= reorder_point` and enqueues a reorder alert message to managers.

**Testing**:
- `Integration: consumption drops qty to reorder_point → alert enqueued once`
- `Integration: manual receipt raises qty above point → no alert`

#### 6.3 — Messaging (email/SMS) & templates

**What**: Outbound email (SendGrid) and SMS (Twilio) with templated message types, persisted to `messages`.

**Design**:
- `MessageType` matches schema enum (`invoice`, `reservation_confirmation`, `arrival_briefing`, `weather_alert`, `renewal_offer`, `review_request`, `maintenance_reminder`, `general`).
- Dispatch via BullMQ; provider webhooks update `delivered_at`/`read_at`. Channel chosen from `customers.preferred_contact`.

**Testing**:
- `Integration (mocked SendGrid): send invoice email → messages row, provider called once`
- `Integration: delivery webhook → delivered_at set`
- `Integration: customer preferred_contact=sms → Twilio path chosen`

---

## Phase 7: Fuel, Utilities, Environmental Compliance & Accounting Sync

### Purpose
Integrate the operational hardware/financial surfaces that distinguish marinas from generic rental businesses. After this phase fuel dispenser events flow in from FuelCloud, utility meter readings drive metered billing, environmental incidents are logged against EPA Clean Marina categories, and invoices/payments sync to QuickBooks/Xero.

### Tasks

#### 7.1 — Fuel transactions & FuelCloud ingestion

**What**: Record `fuel_transactions` and ingest FuelCloud dispenser webhooks.

**Design**:
- `POST /webhooks/fuelcloud` (signed) maps a dispense event → `fuel_transactions` with `quantity_litres`, price, `cost_cents`, optional vessel/customer match; idempotent on `fuelcloud_transaction_id`.
- Manual `POST /fuel-transactions` for attendant-entered sales. Fuel sale → invoice (source_type=fuel).

**Testing**:
- `Integration (mocked FuelCloud): dispense webhook → fuel_transaction row, margin = total-cost`
- `Integration: duplicate fuelcloud_transaction_id → ignored`
- `Integration: fuel sale → invoiceable`

#### 7.2 — Utility metering & metered billing

**What**: `utility_readings` capture and conversion of unbilled usage into invoice lines.

**Design**:
- `POST /slips/:id/utility-readings` computes `usage = current - previous`. Billing run gathers `billed=false` readings into utility invoices, then marks `billed=true` + `invoice_id`.

**Testing**:
- `Unit: usage = current_reading - previous_reading`
- `Integration: billing run invoices unbilled readings, sets billed=true`
- `Integration: already-billed reading not re-invoiced`

#### 7.3 — Environmental compliance logging

**What**: `environmental_logs` capture and an EPA Clean Marina report generator.

**Design**:
- `POST /environmental-logs` with `log_type` (pumpout/waste_oil/fuel_spill/…), severity, photos. Reportable+unreported incidents surface on a compliance dashboard via `idx_envlog_unreported`.
- `GET /marinas/:id/compliance-report?year=` aggregates logs into EPA-checklist-aligned categories (LLM-drafted narrative added in Phase 8).

**Testing**:
- `Integration: log fuel_spill severity=reportable → appears in unreported list`
- `Integration: mark epa_reported=true → leaves unreported list`
- `Integration: report aggregates pumpout gallons for the year`

#### 7.4 — Accounting sync (QuickBooks & Xero)

**What**: OAuth 2.0 connect + background sync of customers, invoices, payments.

**Design**:
- OAuth connect flow stores tokens in `integrations.credentials_json` (encrypted). A sync job pushes new/changed invoices/payments, storing `qbo_invoice_id`/`xero_invoice_id`/`qbo_payment_id`. Token refresh handled (QBO 1h/Xero 30m expiry).

**Testing**:
- `Integration (mocked QBO): sync invoice → qbo_invoice_id stored`
- `Integration: expired access token → refresh attempted, then retry`
- `Integration: sync failure → integrations.status='error', error_message set`

---

## Phase 8: AI-Native Features

### Purpose
Deliver the differentiators that no incumbent offers as open source: dynamic transient pricing, predictive maintenance, fuel anomaly detection, personalised guest communications, churn scoring, and AI-drafted compliance reports — all via server-side LLM/forecasting calls, writing results into existing schema fields.

### Tasks

#### 8.1 — Dynamic transient pricing

**What**: Compute a demand-adjusted transient rate and store the rationale in `reservations.dynamic_price_json`.

**Design**:
- `POST /pricing/quote` input `{ marinaId, slipId, startDate, endDate, vesselLoaCm }`. Service blends: base rate, occupancy-based demand multiplier (current vs capacity for the dates), seasonality, and local-event premium (event calendar). An LLM produces a short human rationale + confidence; output matches the documented `dynamic_price_json` shape `{ base_rate_cents, demand_multiplier, event_premium, final_rate_cents, model_confidence }`.

**Testing**:
- `Unit: 95% occupancy → demand_multiplier > 1.0`
- `Unit: low season + low occupancy → multiplier <= 1.0`
- `Integration (mocked LLM): quote returns final_rate_cents and rationale`
- `Integration: accepted quote persists into reservation.dynamic_price_json`

#### 8.2 — Predictive maintenance

**What**: Recommend maintenance per vessel from engine hours, service history, and OEM-style schedules.

**Design**:
- Job scans vessels; rule baseline (e.g. engine hours since last service > threshold) plus LLM reasoning over `work_orders` history → recommended WO drafts (`status=requested`) and a `maintenance_reminder` message.

**Testing**:
- `Integration: vessel over engine-hour threshold → recommendation generated`
- `Integration (mocked LLM): recommendation creates draft work order`
- `Integration: recently serviced vessel → no recommendation`

#### 8.3 — Fuel anomaly detection

**What**: Flag leaks/theft from `fuel_transactions` patterns.

**Design**:
- Time-series check (rolling mean/stddev per pump) flags outlier dispenses or reconciliation gaps; flagged events create a manager alert and a potential `environmental_logs` draft.

**Testing**:
- `Unit: dispense 5σ above pump mean → flagged`
- `Integration: flagged anomaly → manager alert message enqueued`
- `Unit: normal-range dispense → not flagged`

#### 8.4 — Guest experience automation

**What**: AI-personalised arrival briefings, weather alerts, and post-visit surveys.

**Design**:
- On reservation confirmation, generate an arrival briefing tailored to vessel type/stay length (marina amenities, fuel/pumpout hours, weather); send via preferred channel; set `arrival_briefing_sent=true`. Post-checkout enqueues a `review_request`.

**Testing**:
- `Integration (mocked LLM+comms): confirm reservation → briefing sent, flag set`
- `Integration: checkout → review_request enqueued`
- `Unit: sailboat vs jet_ski → different briefing content tokens`

#### 8.5 — Churn scoring & AI compliance narrative

**What**: Populate `customers.ai_churn_risk` and draft the EPA compliance-report narrative (extends 7.3).

**Design**:
- Scheduled job scores churn from reservation recency/frequency/payment behaviour → `ai_churn_risk` (0–1); high scorers feed a retention list. Compliance endpoint adds an LLM-written narrative over the aggregated logs.

**Testing**:
- `Integration: lapsed customer → ai_churn_risk high; recent active → low`
- `Integration (mocked LLM): compliance report includes narrative + aggregates`

---

## Phase 9: Mobile Technician App (Offline-First)

### Purpose
Provide the offline-capable field experience (Storable/DockMaster parity). Technicians view assigned work orders, capture photos and meter readings, and update job status from the dock with poor connectivity, syncing when back online.

### Tasks

#### 9.1 — Expo app shell, auth, sync engine

**What**: Expo app with OIDC login and WatermelonDB local store syncing to the API.

**Design**:
- Local tables mirror work orders, lines, parts, utility readings. Sync endpoint `POST /sync` does last-write-wins with `updated_at`; conflicts logged to `audit_log`.

**Testing**:
- `E2E (Detox): login offline with cached creds → assigned WOs visible`
- `Integration: offline status change → syncs on reconnect`
- `Integration: conflicting edits → last-write-wins, audit entry written`

#### 9.2 — Field capture (photos, meter readings, status)

**What**: Photo upload to S3, utility/engine-hour readings, and WO status transitions from mobile.

**Design**:
- Photos queued locally, uploaded to S3 on connectivity, URLs appended to `work_order_lines.photo_urls`. Meter readings post to 7.2 endpoints.

**Testing**:
- `E2E: capture photo offline → uploads on reconnect, URL on line`
- `E2E: enter engine hours → vessel.engine_hours updated`
- `E2E: mark line complete offline → WO progresses after sync`

---

## Phase 10: Reporting, Multi-Location, Public API & Hardening

### Purpose
Make the platform sellable to marina groups and integrators: cross-location dashboards, departmental P&L, the published OpenAPI as a de-facto open marina schema (standards.md opportunity), OAuth 2.0 for third parties, and security/compliance hardening.

### Tasks

#### 10.1 — Reporting & dashboards

**What**: Occupancy, revenue-per-foot, fuel margin, service absorption, and departmental P&L.

**Design**:
- SQL-backed report endpoints (joins across the normalized schema). Staff console dashboard widgets with date-range filters. Occupancy = occupied slip-nights / available slip-nights.

**Testing**:
- `Integration: occupancy report matches seeded reservations`
- `Integration: revenue-per-foot = invoiced dockage / total linear feet`
- `Integration: fuel margin = sum(total-cost) by pump`

#### 10.2 — Multi-location groups

**What**: Group several marinas under an org with centralised reporting and group-level roles.

**Design**:
- Org concept above marinas; group `owner`/`manager` JWT scope spans member marina_ids. RLS context accepts a set of marina_ids for group reads.

**Testing**:
- `Integration: group manager sees aggregated occupancy across 3 marinas`
- `Integration: single-marina staff cannot see sibling marina data`

#### 10.3 — Public API (OAuth 2.0) & SDK

**What**: OAuth 2.0 client-credentials for partners; finalise and publish `docs/openapi.json` and `@marina/sdk`.

**Design**:
- Scoped tokens (`reservations:read`, `slips:read`, …) per RFC 6749. Rate limiting and OWASP API Top 10 controls (object-level auth checks via RLS, resource-consumption limits).

**Testing**:
- `Integration: client-credentials token with reservations:read → can read, cannot write`
- `Integration: BOLA attempt cross-tenant → 403/empty (RLS)`
- `CI: OpenAPI diff check fails build on undocumented route`

#### 10.4 — Security & compliance hardening

**What**: Audit logging coverage, secrets encryption, PCI/GDPR posture, rate limiting, partition maintenance.

**Design**:
- All mutations write `audit_log`. `integrations.credentials_json` encrypted at rest. GDPR erasure endpoint anonymises customer PII while preserving financial records. Monthly job creates next `audit_log` partition.

**Testing**:
- `Integration: any entity mutation → audit_log row with actor + changes`
- `Integration: GDPR erase → PII nulled, invoices retained`
- `Integration: rate limit exceeded → 429`
- `Integration: partition job creates next-month partition`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, RLS, auth, OpenAPI)   ─── required by everything
    │
Phase 2: Core Inventory (marinas, slips, vessels, custs) ─── requires P1
    │
Phase 3: Reservations & Contracts                        ─── requires P2
    ├── Phase 4: Slip Map & Staff Console   ── requires P3 (can parallel with P5)
    └── Phase 5: Billing & Payments         ── requires P3 (can parallel with P4)
         │
Phase 6: Service, Work Orders, Parts, Comms ── requires P5 (comms used by P5 send)
    │
Phase 7: Fuel / Utilities / Env / Accounting ── requires P5, P6
    │
Phase 8: AI-Native Features                  ── requires P3 (pricing), P6/P7 (maint/fuel/compliance)
    ├── Phase 9: Mobile Technician App        ── requires P6 (can parallel with P8/P10)
    └── Phase 10: Reporting / Multi-Loc / API ── requires P5–P7 (can parallel with P8/P9)
```

Parallelism:
- **Phases 4 and 5** can be built concurrently once Phase 3 lands (UI vs billing).
- **Phases 8, 9, and 10** can all proceed concurrently once Phases 6–7 are complete.

---

## Definition of Done (per phase)

1. All tasks implemented and merged.
2. All unit and integration tests pass (Vitest + Supertest); E2E pass for UI phases (Playwright/Detox).
3. ESLint and Prettier pass with zero warnings.
4. `tsc --noEmit` passes across all workspaces.
5. New/changed Drizzle migrations created and apply cleanly on a fresh DB (verified via Testcontainers).
6. RLS policies cover any new `marina_id`-scoped table.
7. New API endpoints appear in the generated `docs/openapi.json`, and the CI OpenAPI diff check passes.
8. New config/env vars documented in `.env.example`.
9. `docker compose up` boots the affected services and the feature works end-to-end.
10. New mutations write to `audit_log`; no card PAN or raw secrets stored.
11. Feature demonstrated against the seed dataset.
