# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Marina & Boat Storage Management · Created: 2026-05-25

## Philosophy

This model collapses the 18-table normalized schema into 7 core tables by embedding related data as JSONB documents within their parent entities. Marinas embed their slip map, staff roster, parts catalogue, fuel pump configuration, environmental logs, and integration settings. Vessels embed their owner (customer) data. Reservations and contracts embed line items and communication history. The principle: a marina operator loading a slip map or a reservation detail page should get everything in one query.

The relational anchors — `vessels`, `invoices`, and `payments` — remain as standalone tables because they have independent lifecycles: vessels persist across multiple reservations and service visits, invoices reconcile with QuickBooks/Xero independently, and payments reconcile with Stripe.

**Best for:** Early-stage projects prioritising development speed, single-query page loads for slip maps and reservation management, and schema flexibility as the product iterates. Ideal for single-marina operators or small marina groups where cross-facility analytics are secondary to operational speed.

**Trade-offs:**
- (+) 7 tables — minimal migration surface, fast to iterate
- (+) Single-row fetch loads a complete marina with all slips, staff, and configuration
- (+) JSONB flexibility accommodates per-marina utility billing rules and rate structures
- (+) GIN indexes on JSONB support containment queries for analytics
- (-) Cross-marina analytics (occupancy comparison, revenue per foot) require JSONB extraction
- (-) Slip map embedded in marina row — large marinas with 500+ slips push JSONB size
- (-) No foreign-key enforcement on embedded slip/staff/parts references
- (-) Utility reading time series embedded in slips limits historical query depth

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 8666 | `vessels.loa_cm`, `vessels.beam_cm`, `vessels.draft_cm` — relational columns for slip matching |
| USCG HIN | `vessels.hin` — standalone table with CHECK constraint |
| EPA Clean Marina | Environmental logs embedded in `marinas.environmental_logs_json[]` |
| ABYC E-13 | `vessels.has_lithium_batteries`; slip entries include `lithium_charging_approved` flag |
| PCI DSS v4.0 | No card data stored; `payments` references Stripe tokens only |
| QuickBooks/Xero | `invoices.qbo_invoice_id`, `payments.qbo_payment_id` |

---

## Core Tables

### marinas

```sql
CREATE TABLE marinas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    address_json    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"line1":"100 Harbour Way","city":"Key West","state":"FL","postal":"33040",
    --   "country":"US","lat":24.5551,"lng":-81.7800}
    phone           TEXT,
    email           TEXT,
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    settings_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"seasonal_rate_per_foot_cents":8500,"transient_rate_per_foot_cents":350,
    --   "fuel_markup_pct":15,"utility_billing":"metered","epa_clean_marina":true,
    --   "abyc_lithium_policy":"designated_slips_only","operating_hours":{"mon":"06:00-20:00"}}

    slips_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","slip_number":"A-12","slip_type":"wet_slip","dock":"A","section":"east",
    --   "max_loa_cm":1524,"max_beam_cm":457,"max_draft_cm":183,"has_power":true,"power_amps":50,
    --   "power_voltage":240,"has_water":true,"has_pumpout":false,"lithium_charging_approved":false,
    --   "status":"occupied","map_x":120.5,"map_y":340.2,"map_rotation":0,
    --   "seasonal_rate_cents":12750,"transient_rate_cents":500,
    --   "current_occupant":{"vessel_id":"uuid","vessel_name":"Sea Breeze","customer_name":"John Smith",
    --     "reservation_type":"seasonal","end_date":"2026-10-31"},
    --   "utility_readings":[{"type":"electric_kwh","date":"2026-05-01","reading":4521,"usage":187}]}]

    staff_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","email":"mike@marina.com","full_name":"Mike Johnson",
    --   "role":"dock_master","phone":"555-0201","hourly_rate_cents":2800,
    --   "certifications":["ABYC_certified"],"is_active":true}]

    parts_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","part_number":"MAR-OIL-5W30","description":"Marine Oil 5W30 Qt",
    --   "category":"fluids","brand":"Quicksilver","cost_cents":800,"retail_cents":1499,
    --   "qty_on_hand":24,"reorder_point":6,"bin_location":"S2-04"}]

    fuel_pumps_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"pump_id":"P1","fuel_type":"diesel","tank_capacity_litres":10000,
    --   "current_level_litres":7200,"unit_price_cents":175,"cost_cents":145,
    --   "fuelcloud_pump_id":"fc_p1"}]

    environmental_logs_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","log_type":"pumpout","quantity":50,"unit":"gallons",
    --   "reported_by":"uuid","logged_at":"2026-05-20T14:00:00Z"},
    --  {"id":"uuid","log_type":"fuel_spill","quantity":0.5,"unit":"gallons",
    --   "location":"Fuel dock B","incident_severity":"minor",
    --   "response_actions":"Absorbent pads deployed","epa_reported":false,
    --   "photos":["https://cdn/spill1.jpg"],"logged_at":"2026-05-18T09:00:00Z"}]

    integrations_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"provider":"quickbooks","status":"active","qbo_realm_id":"123"},
    --   {"provider":"fuelcloud","status":"active"},{"provider":"stripe","status":"active"}]

    stripe_account_id TEXT,
    qbo_realm_id    TEXT,
    xero_tenant_id  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_marinas_slug ON marinas(slug);
CREATE INDEX idx_marinas_slips ON marinas USING GIN (slips_json);
CREATE INDEX idx_marinas_staff ON marinas USING GIN (staff_json);
```

### vessels

Vessels remain relational — they accumulate service history across multiple reservations and seasons, and their HIN is a cross-system identifier.

```sql
CREATE TABLE vessels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    customer_json   JSONB NOT NULL,
    -- Example: {"id":"uuid","first_name":"John","last_name":"Smith","email":"john@example.com",
    --   "phone":"555-0199","address":{"line1":"456 Ocean Dr","city":"Miami","state":"FL"},
    --   "customer_type":"individual","membership_number":"YC-2024-042",
    --   "portal_enabled":true,"preferred_contact":"email","qbo_customer_id":"cust_123",
    --   "payment_methods":[{"stripe_pm_id":"pm_xxx","type":"card","last_four":"4242","brand":"visa"}]}
    name            TEXT,
    hin             TEXT CHECK (hin ~ '^[A-Z]{3}[A-Z0-9]{5}[A-Z0-9]{4}$'),
    cin             TEXT,
    registration_number TEXT,
    registration_state CHAR(2),
    registration_expiry DATE,
    insurance_policy TEXT,
    insurance_expiry DATE,
    vessel_type     TEXT,
    year            SMALLINT,
    make            TEXT,
    model           TEXT,
    loa_cm          INTEGER NOT NULL,
    beam_cm         INTEGER,
    draft_cm        INTEGER,
    displacement_kg INTEGER,
    design_category CHAR(1) CHECK (design_category IN ('A','B','C','D')),
    engine_type     TEXT,
    engine_count    SMALLINT DEFAULT 1,
    engine_hours    INTEGER,
    fuel_type       TEXT,
    fuel_capacity_litres INTEGER,
    has_lithium_batteries BOOLEAN NOT NULL DEFAULT false,
    has_holding_tank BOOLEAN NOT NULL DEFAULT false,
    photo_urls      TEXT[] DEFAULT '{}',
    ai_churn_risk   NUMERIC(4,3),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_vessels_hin ON vessels(marina_id, hin) WHERE hin IS NOT NULL;
CREATE INDEX idx_vessels_marina ON vessels(marina_id);
CREATE INDEX idx_vessels_size ON vessels(loa_cm, beam_cm);
CREATE INDEX idx_vessels_customer ON vessels USING GIN (customer_json);
```

### reservations

Reservations embed communication history and dynamic pricing details.

```sql
CREATE TABLE reservations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    slip_id         TEXT NOT NULL,
    vessel_id       UUID NOT NULL REFERENCES vessels(id),
    reservation_type TEXT NOT NULL CHECK (reservation_type IN ('seasonal','transient','monthly','daily')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','confirmed','checked_in','checked_out','cancelled','no_show')),
    hold_type       TEXT CHECK (hold_type IN ('hard','soft')),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    check_in_at     TIMESTAMPTZ,
    check_out_at    TIMESTAMPTZ,

    pricing_json    JSONB NOT NULL DEFAULT '{}',
    -- Example: {"rate_per_foot_cents":350,"flat_rate_cents":null,"total_cents":17850,
    --   "dynamic":{"base_rate":350,"demand_multiplier":1.25,"event_premium":0.10,
    --     "final_rate":473,"confidence":0.78}}

    messages_json   JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","direction":"outbound","channel":"email",
    --   "type":"reservation_confirmation","body":"Your slip A-12 is confirmed...",
    --   "at":"2026-05-15T10:00:00Z"},
    --  {"id":"uuid","direction":"outbound","channel":"sms","type":"arrival_briefing",
    --   "body":"Welcome! Fuel dock hours 7am-6pm, pump-out at dock C...",
    --   "at":"2026-05-19T08:00:00Z"}]

    special_requests TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_res_marina ON reservations(marina_id, start_date, end_date);
CREATE INDEX idx_res_vessel ON reservations(vessel_id);
CREATE INDEX idx_res_status ON reservations(marina_id, status);
```

### work_orders

Work orders embed line items, technician time, and parts used.

```sql
CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    wo_number       SERIAL,
    vessel_id       UUID NOT NULL REFERENCES vessels(id),
    assigned_to     UUID,
    wo_type         TEXT NOT NULL CHECK (wo_type IN ('repair','maintenance','haul_out','launch',
                                                     'bottom_paint','winterisation','commissioning',
                                                     'inspection','other')),
    status          TEXT NOT NULL DEFAULT 'requested'
                    CHECK (status IN ('requested','estimated','approved','in_progress',
                                      'waiting_parts','completed','invoiced','cancelled')),
    priority        TEXT NOT NULL DEFAULT 'normal' CHECK (priority IN ('low','normal','high','urgent')),
    description     TEXT NOT NULL,

    line_items_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","sort_order":1,"line_type":"labour","description":"Bottom clean and inspect",
    --   "technician_id":"uuid","hours_estimated":3.0,"hours_actual":2.5,
    --   "unit_price_cents":9500,"cost_cents":7000,"total_cents":28500,"photo_urls":["https://cdn/hull1.jpg"]},
    --  {"id":"uuid","sort_order":2,"line_type":"part","description":"Zincs x4",
    --   "part_number":"MAR-ZINC-04","quantity":4,"unit_price_cents":1200,"cost_cents":600,"total_cents":4800}]

    estimate_cents  BIGINT,
    estimate_approved_at TIMESTAMPTZ,
    scheduled_date  DATE,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    subtotal_cents  BIGINT NOT NULL DEFAULT 0,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL DEFAULT 0,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wo_marina ON work_orders(marina_id, status);
CREATE INDEX idx_wo_vessel ON work_orders(vessel_id);
CREATE INDEX idx_wo_lines ON work_orders USING GIN (line_items_json);
```

### invoices

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    invoice_number  SERIAL,
    vessel_id       UUID REFERENCES vessels(id),
    source_type     TEXT NOT NULL CHECK (source_type IN ('reservation','contract','work_order',
                                                         'fuel','utility','retail','other')),
    source_id       UUID,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','sent','viewed','paid','partial','overdue','void')),

    customer_json   JSONB NOT NULL,
    -- Snapshot: {"id":"uuid","name":"John Smith","email":"john@example.com","phone":"555-0199"}

    line_items_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"description":"Slip A-12 seasonal 2026","qty":1,"unit_price_cents":1275000,
    --   "total_cents":1275000,"taxable":false},
    --  {"description":"Electric May 2026 (187 kWh)","qty":187,"unit_price_cents":18,
    --   "total_cents":3366,"taxable":true}]

    subtotal_cents  BIGINT NOT NULL DEFAULT 0,
    tax_cents       BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL DEFAULT 0,
    due_date        DATE,
    sent_at         TIMESTAMPTZ,
    viewed_at       TIMESTAMPTZ,
    qbo_invoice_id  TEXT,
    xero_invoice_id TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoices_marina ON invoices(marina_id, status);
CREATE INDEX idx_invoices_vessel ON invoices(vessel_id);
```

### payments

```sql
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    amount_cents    BIGINT NOT NULL,
    method          TEXT NOT NULL CHECK (method IN ('card','cash','cheque','ach','terminal','payment_link')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','succeeded','failed','refunded','partial_refund')),
    stripe_payment_intent_id TEXT,
    refund_cents    BIGINT NOT NULL DEFAULT 0,
    qbo_payment_id  TEXT,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_invoice ON payments(invoice_id);
CREATE INDEX idx_payments_marina ON payments(marina_id, status);
```

### audit_log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('staff','customer','system','integration','ai')),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID,
    changes_json    JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_marina ON audit_log(marina_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Example Queries

### Load complete marina with slip map

```sql
SELECT m.*, 
       jsonb_array_length(m.slips_json) AS total_slips,
       (SELECT COUNT(*) FROM jsonb_array_elements(m.slips_json) s WHERE s->>'status' = 'occupied') AS occupied_slips
FROM marinas m
WHERE m.id = $1;
```

### Find available slips for a vessel

```sql
SELECT s->>'slip_number' AS slip_number, s->>'slip_type' AS slip_type,
       (s->>'max_loa_cm')::integer AS max_loa, (s->>'transient_rate_cents')::bigint AS rate
FROM marinas m, jsonb_array_elements(m.slips_json) AS s
WHERE m.id = $1
  AND s->>'status' = 'available'
  AND (s->>'max_loa_cm')::integer >= $2
  AND (s->>'max_beam_cm')::integer >= $3
ORDER BY (s->>'max_loa_cm')::integer ASC;
```

### Fuel margin by pump

```sql
SELECT p->>'pump_id' AS pump_id, p->>'fuel_type' AS fuel_type,
       (p->>'unit_price_cents')::bigint - (p->>'cost_cents')::bigint AS margin_cents_per_litre
FROM marinas m, jsonb_array_elements(m.fuel_pumps_json) AS p
WHERE m.id = $1;
```

### Unreported EPA incidents

```sql
SELECT e->>'log_type' AS log_type, e->>'incident_severity' AS severity,
       e->>'location' AS location, e->>'logged_at' AS logged_at,
       e->>'response_actions' AS response
FROM marinas m, jsonb_array_elements(m.environmental_logs_json) AS e
WHERE m.id = $1
  AND e->>'incident_severity' = 'reportable'
  AND (e->>'epa_reported')::boolean = false;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Marina (with slips, staff, parts, fuel, env logs, integrations) | 1 | marinas |
| Vessels (with customer) | 1 | vessels |
| Reservations (with messages, pricing) | 1 | reservations |
| Work Orders (with line items) | 1 | work_orders |
| Invoices (with line items) | 1 | invoices |
| Payments | 1 | payments |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **7** | |

---

## Key Design Decisions

1. **Slips embedded in marina** — the slip map is always rendered in marina context. Embedding slips with their map coordinates, amenities, occupancy status, and current occupant snapshot enables single-query map loads. The trade-off is that slip assignment updates require JSONB array manipulation.

2. **Customer embedded in vessel** — customers are always accessed through their vessels in a marina context. The customer document lives inside `vessels.customer_json`, and a snapshot is copied into `invoices.customer_json`. For customers with multiple vessels, the customer JSON appears across multiple vessel rows.

3. **Fuel pumps embedded in marina** — pump configuration and current tank levels are marina infrastructure. FuelCloud webhook events update the pump's level and price in-place within the JSONB array.

4. **Environmental logs embedded in marina** — pump-out logs, spill incidents, and waste tracking are low-volume, marina-scoped records. Embedding avoids a table that would hold a few dozen rows per year. EPA reporting status is tracked per incident within the array.

5. **Utility readings embedded in slips** — the most recent readings are kept inside each slip entry in `slips_json`. Historical readings beyond the current billing cycle are archived to `audit_log` events. This keeps the slip document current without unbounded growth.

6. **Vessels remain relational** — a vessel's service history spans multiple seasons, reservations, and work orders. Its HIN is a cross-system identifier used for insurance, registration, and recall checks. The vessel table is the cross-seasonal join point.

7. **Invoices remain relational** — they reconcile with QuickBooks/Xero and have an independent lifecycle (sent, viewed, paid, overdue). The polymorphic `source_type` links invoices to reservations, contracts, work orders, or fuel transactions.

8. **Contracts are handled as long-term reservations** — rather than a separate contracts table, seasonal and annual agreements are reservations with `reservation_type = 'seasonal'` or `'monthly'`. Contract-specific fields (auto-renew, signed status) are captured in the reservation's JSONB fields. This simplifies the model at the cost of some semantic clarity.

9. **Dynamic pricing on reservation** — `pricing_json` captures the AI yield management output inline with the reservation, preserving the pricing rationale at the time of booking.

10. **GIN indexes on slips and line items** — `slips_json` gets a GIN index to support slip availability queries using JSONB containment. `line_items_json` on work orders supports containment queries for parts usage analysis.
