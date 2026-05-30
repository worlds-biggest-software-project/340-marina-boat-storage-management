# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Marina & Boat Storage Management · Created: 2026-05-25

## Philosophy

This model assigns a dedicated table to every first-class concept in the marina domain — marinas, slips, vessels, customers, reservations, contracts, work orders, parts, fuel transactions, utility readings, and environmental compliance records each live in their own table with strict foreign-key relationships. The design separates the physical infrastructure (marinas, slips, fuel docks, dry-stack bays) from the operational entities (reservations, contracts, work orders) and financial entities (invoices, payments), enabling rich cross-dimensional queries: occupancy by slip size, revenue per linear foot, fuel margin by pump, technician utilisation, and environmental compliance status.

The schema aligns with marine industry standards wherever possible: vessel dimensions follow ISO 8666, Hull Identification Numbers follow USCG HIN format, environmental tracking aligns with EPA Clean Marina audit checklists, and financial fields use BIGINT cents. The slip/berth model accommodates wet slips, dry-stack bays, moorings, and rack storage as different `slip_type` values within a unified table, reflecting the multi-facility operating models seen in platforms like Harbour Assist.

**Best for:** Multi-facility marina groups, boatyards, and municipal harbour authorities that need rich occupancy analytics, departmental P&L (dockage, service, fuel, retail), environmental compliance reporting, and integration with accounting systems, fuel dispensers, and utility meters.

**Trade-offs:**
- (+) Full referential integrity — no orphaned reservations, orphaned vessel records, or untracked fuel sales
- (+) Every compliance surface (EPA, ABYC, USCG) has its own queryable table
- (+) Standard SQL joins for any KPI (occupancy rate, revenue per foot, fuel margin, service absorption)
- (+) Natural fit for QuickBooks/Xero, Stripe, and FuelCloud webhook ingestion
- (-) 21 tables is significant migration surface for schema changes
- (-) Slip map rendering requires joins across marinas, slips, and current reservations
- (-) Flexible per-facility configurations (seasonal rate tiers, utility billing rules) require additional settings or application logic

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 8666 (Small Craft Principal Data) | `vessels.loa_cm`, `vessels.beam_cm`, `vessels.draft_cm` — LOA drives slip sizing and tariff calculation |
| USCG HIN Standard | `vessels.hin` — 12-character Hull Identification Number with format validation |
| ISO 10087 (CIN) | `vessels.cin` — Craft Identification Number for international vessels |
| EPA Clean Marina | `environmental_logs` table aligns with Clean Marina audit checklist categories |
| ABYC E-13 (Lithium Battery) | `vessels.has_lithium_batteries` flag; charging protocol enforcement in slip assignment |
| NMEA 0183/2000 | Informs vessel telematics integration; engine hours captured in `vessels.engine_hours` |
| PCI DSS v4.0 | No card data stored; `payment_methods` holds Stripe/EMV token references only |
| EMV / 3D Secure 2.0 | POS terminal and online payment compliance via Stripe integration |
| iCalendar RFC 5545 | Reservation export uses iCalendar format; `reservations` fields map to VEVENT properties |
| QuickBooks/Xero API | `invoices.qbo_sync_id`, `payments.qbo_payment_id` for accounting sync |

---

## Marina & Infrastructure

```sql
CREATE TABLE marinas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_code      CHAR(2),
    postal_code     TEXT,
    country_code    CHAR(2) DEFAULT 'US',
    latitude        NUMERIC(9,6),
    longitude       NUMERIC(9,6),
    phone           TEXT,
    email           TEXT,
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    total_slips     INTEGER NOT NULL DEFAULT 0,
    settings_json   JSONB NOT NULL DEFAULT '{}',
    -- Example: {"seasonal_rate_per_foot_cents":8500,"transient_rate_per_foot_cents":350,
    --   "fuel_markup_pct":15,"utility_billing":"metered","operating_hours":{"mon":"06:00-20:00"},
    --   "epa_clean_marina_certified":true,"abyc_lithium_policy":"designated_slips_only"}
    qbo_realm_id    TEXT,
    xero_tenant_id  TEXT,
    stripe_account_id TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE slips (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    slip_number     TEXT NOT NULL,
    slip_type       TEXT NOT NULL CHECK (slip_type IN ('wet_slip','dry_stack','mooring','rack','end_tie','face_dock')),
    dock            TEXT,
    section         TEXT,
    max_loa_cm      INTEGER NOT NULL,
    max_beam_cm     INTEGER,
    max_draft_cm    INTEGER,
    max_displacement_kg INTEGER,
    has_power       BOOLEAN NOT NULL DEFAULT true,
    power_amps      INTEGER,
    power_voltage   INTEGER,
    has_water       BOOLEAN NOT NULL DEFAULT true,
    has_pumpout     BOOLEAN NOT NULL DEFAULT false,
    has_wifi        BOOLEAN NOT NULL DEFAULT false,
    lithium_charging_approved BOOLEAN NOT NULL DEFAULT false,
    status          TEXT NOT NULL DEFAULT 'available'
                    CHECK (status IN ('available','occupied','reserved','maintenance','out_of_service')),
    map_x           NUMERIC(8,2),
    map_y           NUMERIC(8,2),
    map_rotation    NUMERIC(5,2),
    seasonal_rate_cents BIGINT,
    transient_rate_cents BIGINT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (marina_id, slip_number)
);

CREATE INDEX idx_slips_marina ON slips(marina_id, slip_type, status);
CREATE INDEX idx_slips_size ON slips(marina_id, max_loa_cm);
```

## Staff

```sql
CREATE TABLE staff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('owner','manager','dock_master','service_tech',
                                                  'fuel_attendant','office','accounting')),
    phone           TEXT,
    hourly_rate_cents BIGINT,
    certifications  TEXT[] DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (marina_id, email)
);

CREATE INDEX idx_staff_marina ON staff(marina_id) WHERE is_active;
```

## Customer & Vessel

```sql
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    address_line1   TEXT,
    city            TEXT,
    state_code      CHAR(2),
    postal_code     TEXT,
    country_code    CHAR(2) DEFAULT 'US',
    customer_type   TEXT NOT NULL DEFAULT 'individual'
                    CHECK (customer_type IN ('individual','club_member','commercial','government')),
    membership_number TEXT,
    portal_enabled  BOOLEAN NOT NULL DEFAULT false,
    preferred_contact TEXT NOT NULL DEFAULT 'email' CHECK (preferred_contact IN ('email','sms','phone')),
    ai_churn_risk   NUMERIC(4,3),
    qbo_customer_id TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_customers_marina ON customers(marina_id);
CREATE INDEX idx_customers_email ON customers(marina_id, email);

CREATE TABLE vessels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    name            TEXT,
    hin             TEXT CHECK (hin ~ '^[A-Z]{3}[A-Z0-9]{5}[A-Z0-9]{4}$'),
    cin             TEXT,
    registration_number TEXT,
    registration_state CHAR(2),
    registration_expiry DATE,
    insurance_policy TEXT,
    insurance_expiry DATE,
    vessel_type     TEXT CHECK (vessel_type IN ('sailboat','powerboat','pontoon','catamaran',
                        'trawler','center_console','rib','jet_ski','houseboat','other')),
    year            SMALLINT,
    make            TEXT,
    model           TEXT,
    loa_cm          INTEGER NOT NULL,
    beam_cm         INTEGER,
    draft_cm        INTEGER,
    displacement_kg INTEGER,
    design_category CHAR(1) CHECK (design_category IN ('A','B','C','D')),
    engine_type     TEXT CHECK (engine_type IN ('inboard','outboard','sterndrive','jet','sail_only','electric')),
    engine_count    SMALLINT DEFAULT 1,
    engine_hours    INTEGER,
    fuel_type       TEXT CHECK (fuel_type IN ('gasoline','diesel','electric','none')),
    fuel_capacity_litres INTEGER,
    has_lithium_batteries BOOLEAN NOT NULL DEFAULT false,
    has_holding_tank BOOLEAN NOT NULL DEFAULT false,
    smartcar_vehicle_id TEXT,
    photo_urls      TEXT[] DEFAULT '{}',
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_vessels_hin ON vessels(marina_id, hin) WHERE hin IS NOT NULL;
CREATE INDEX idx_vessels_customer ON vessels(customer_id);
CREATE INDEX idx_vessels_marina ON vessels(marina_id);
CREATE INDEX idx_vessels_size ON vessels(loa_cm, beam_cm);
```

## Reservations & Contracts

```sql
CREATE TABLE reservations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    slip_id         UUID NOT NULL REFERENCES slips(id),
    vessel_id       UUID NOT NULL REFERENCES vessels(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    reservation_type TEXT NOT NULL CHECK (reservation_type IN ('seasonal','transient','monthly','daily')),
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','confirmed','checked_in','checked_out','cancelled','no_show')),
    hold_type       TEXT CHECK (hold_type IN ('hard','soft')),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    check_in_at     TIMESTAMPTZ,
    check_out_at    TIMESTAMPTZ,
    rate_per_foot_cents BIGINT,
    flat_rate_cents BIGINT,
    total_cents     BIGINT NOT NULL DEFAULT 0,
    dynamic_price_json JSONB,
    -- Example: {"base_rate_cents":350,"demand_multiplier":1.25,"event_premium":0.10,
    --   "final_rate_cents":473,"model_confidence":0.78}
    special_requests TEXT,
    arrival_briefing_sent BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_res_marina_dates ON reservations(marina_id, start_date, end_date);
CREATE INDEX idx_res_slip ON reservations(slip_id, start_date, end_date);
CREATE INDEX idx_res_customer ON reservations(customer_id);
CREATE INDEX idx_res_status ON reservations(marina_id, status);

CREATE TABLE contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    vessel_id       UUID NOT NULL REFERENCES vessels(id),
    slip_id         UUID NOT NULL REFERENCES slips(id),
    contract_type   TEXT NOT NULL CHECK (contract_type IN ('seasonal','annual','monthly','dry_stack','mooring')),
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','sent','signed','active','expired','cancelled','renewed')),
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    annual_rate_cents BIGINT NOT NULL,
    utility_plan    TEXT CHECK (utility_plan IN ('included','metered','flat_fee')),
    utility_flat_fee_cents BIGINT,
    auto_renew      BOOLEAN NOT NULL DEFAULT false,
    signed_at       TIMESTAMPTZ,
    signature_url   TEXT,
    renewal_offer_sent_at TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contracts_marina ON contracts(marina_id, status);
CREATE INDEX idx_contracts_customer ON contracts(customer_id);
CREATE INDEX idx_contracts_slip ON contracts(slip_id);
CREATE INDEX idx_contracts_expiry ON contracts(end_date) WHERE status = 'active';
```

## Service & Parts

```sql
CREATE TABLE work_orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    wo_number       SERIAL,
    vessel_id       UUID NOT NULL REFERENCES vessels(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    assigned_to     UUID REFERENCES staff(id),
    wo_type         TEXT NOT NULL CHECK (wo_type IN ('repair','maintenance','haul_out','launch',
                                                     'bottom_paint','winterisation','commissioning',
                                                     'inspection','other')),
    status          TEXT NOT NULL DEFAULT 'requested'
                    CHECK (status IN ('requested','estimated','approved','in_progress',
                                      'waiting_parts','completed','invoiced','cancelled')),
    priority        TEXT NOT NULL DEFAULT 'normal' CHECK (priority IN ('low','normal','high','urgent')),
    description     TEXT NOT NULL,
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

CREATE TABLE work_order_lines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    work_order_id   UUID NOT NULL REFERENCES work_orders(id) ON DELETE CASCADE,
    sort_order      SMALLINT NOT NULL DEFAULT 0,
    line_type       TEXT NOT NULL CHECK (line_type IN ('labour','part','sublet','fee','haul_launch')),
    description     TEXT NOT NULL,
    technician_id   UUID REFERENCES staff(id),
    part_id         UUID REFERENCES parts(id),
    quantity        NUMERIC(10,2) NOT NULL DEFAULT 1,
    unit_price_cents BIGINT NOT NULL DEFAULT 0,
    cost_cents      BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL DEFAULT 0,
    hours_estimated NUMERIC(5,2),
    hours_actual    NUMERIC(5,2),
    photo_urls      TEXT[] DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wol_wo ON work_order_lines(work_order_id);

CREATE TABLE parts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    part_number     TEXT NOT NULL,
    description     TEXT NOT NULL,
    category        TEXT,
    brand           TEXT,
    cost_cents      BIGINT NOT NULL DEFAULT 0,
    retail_cents    BIGINT NOT NULL DEFAULT 0,
    qty_on_hand     INTEGER NOT NULL DEFAULT 0,
    reorder_point   INTEGER NOT NULL DEFAULT 0,
    reorder_qty     INTEGER NOT NULL DEFAULT 0,
    bin_location    TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (marina_id, part_number)
);

CREATE INDEX idx_parts_marina ON parts(marina_id) WHERE is_active;
```

## Fuel & Utilities

```sql
CREATE TABLE fuel_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    pump_id         TEXT NOT NULL,
    vessel_id       UUID REFERENCES vessels(id),
    customer_id     UUID REFERENCES customers(id),
    fuel_type       TEXT NOT NULL CHECK (fuel_type IN ('gasoline','diesel','ethanol_free')),
    quantity_litres NUMERIC(10,2) NOT NULL,
    unit_price_cents BIGINT NOT NULL,
    total_cents     BIGINT NOT NULL,
    cost_cents      BIGINT NOT NULL,
    attendant_id    UUID REFERENCES staff(id),
    fuelcloud_transaction_id TEXT,
    dispensed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fuel_marina ON fuel_transactions(marina_id, dispensed_at DESC);
CREATE INDEX idx_fuel_vessel ON fuel_transactions(vessel_id);

CREATE TABLE utility_readings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    slip_id         UUID NOT NULL REFERENCES slips(id),
    utility_type    TEXT NOT NULL CHECK (utility_type IN ('electric_kwh','water_gallons','pumpout')),
    previous_reading NUMERIC(12,2),
    current_reading NUMERIC(12,2) NOT NULL,
    usage           NUMERIC(12,2) NOT NULL,
    reading_date    DATE NOT NULL,
    read_by         UUID REFERENCES staff(id),
    billed          BOOLEAN NOT NULL DEFAULT false,
    invoice_id      UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_utility_slip ON utility_readings(slip_id, utility_type, reading_date DESC);
CREATE INDEX idx_utility_unbilled ON utility_readings(marina_id) WHERE NOT billed;
```

## Payments & Invoicing

```sql
CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    invoice_number  SERIAL,
    customer_id     UUID NOT NULL REFERENCES customers(id),
    source_type     TEXT NOT NULL CHECK (source_type IN ('reservation','contract','work_order',
                                                         'fuel','utility','retail','other')),
    source_id       UUID,
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','sent','viewed','paid','partial','overdue','void')),
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
CREATE INDEX idx_invoices_customer ON invoices(customer_id);

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

## Environmental Compliance & Communication

```sql
CREATE TABLE environmental_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    log_type        TEXT NOT NULL CHECK (log_type IN ('pumpout','waste_oil','fuel_spill',
                                                      'stormwater','bilge_water','absorbent_pad',
                                                      'battery_disposal','antifreeze')),
    quantity        NUMERIC(10,2),
    unit            TEXT CHECK (unit IN ('gallons','litres','pounds','count','kg')),
    location        TEXT,
    reported_by     UUID REFERENCES staff(id),
    incident_severity TEXT CHECK (incident_severity IN ('minor','moderate','major','reportable')),
    response_actions TEXT,
    epa_reported    BOOLEAN NOT NULL DEFAULT false,
    epa_report_number TEXT,
    photos          TEXT[] DEFAULT '{}',
    logged_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_envlog_marina ON environmental_logs(marina_id, log_type, logged_at DESC);
CREATE INDEX idx_envlog_unreported ON environmental_logs(marina_id) WHERE incident_severity = 'reportable' AND NOT epa_reported;

CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    reservation_id  UUID REFERENCES reservations(id),
    direction       TEXT NOT NULL CHECK (direction IN ('outbound','inbound')),
    channel         TEXT NOT NULL CHECK (channel IN ('sms','email','push','portal')),
    message_type    TEXT NOT NULL CHECK (message_type IN ('invoice','reservation_confirmation',
                        'arrival_briefing','weather_alert','renewal_offer','review_request',
                        'maintenance_reminder','general')),
    subject         TEXT,
    body            TEXT NOT NULL,
    delivered_at    TIMESTAMPTZ,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_messages_customer ON messages(customer_id);
```

## Integrations & Audit

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL REFERENCES marinas(id),
    provider        TEXT NOT NULL CHECK (provider IN ('quickbooks','xero','stripe','fuelcloud',
                                                      'dockwa','twilio','sendgrid','nmea','weather')),
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','inactive','error')),
    credentials_json JSONB NOT NULL DEFAULT '{}',
    config_json     JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (marina_id, provider)
);

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

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Marina & Infrastructure | 2 | marinas, slips |
| Staff | 1 | staff |
| Customer & Vessel | 2 | customers, vessels |
| Reservations & Contracts | 2 | reservations, contracts |
| Service & Parts | 3 | work_orders, work_order_lines, parts |
| Fuel & Utilities | 2 | fuel_transactions, utility_readings |
| Payments & Invoicing | 2 | invoices, payments |
| Environmental & Communication | 2 | environmental_logs, messages |
| Integrations & Audit | 2 | integrations, audit_log |
| **Total** | **18** | |

---

## Key Design Decisions

1. **Unified slip model with `slip_type`** — wet slips, dry-stack bays, moorings, rack positions, end ties, and face docks are all rows in the `slips` table with different `slip_type` values. This enables a single visual map renderer and unified occupancy analytics across storage types.

2. **ISO 8666 vessel dimensions in centimetres** — `loa_cm`, `beam_cm`, `draft_cm` store vessel principal data in centimetres (the ISO 8666 base unit), avoiding fractional feet/inches ambiguity. LOA drives slip assignment: `WHERE slips.max_loa_cm >= vessel.loa_cm`.

3. **USCG HIN as primary vessel identifier** — `vessels.hin` validates the 12-character HIN format (3-letter MIC prefix + 5 alphanumeric + 4 alphanumeric). CIN field supports international vessels per ISO 10087.

4. **Separate reservations and contracts** — reservations handle transient and short-term dockage (nightly/weekly); contracts handle seasonal and annual agreements. Different lifecycles, different billing patterns, different renewal workflows.

5. **Dynamic pricing as JSONB on reservation** — `reservations.dynamic_price_json` captures the AI yield management output (base rate, demand multiplier, event premium, final rate, model confidence) without embedding the pricing algorithm in the schema.

6. **Fuel transactions linked to vessel and customer** — enables per-vessel fuel consumption tracking, fuel margin analysis by pump, and FuelCloud webhook reconciliation. Cost field enables margin calculation.

7. **Utility readings as a time series** — `utility_readings` captures meter readings for electricity, water, and pump-out per slip. The `billed`/`invoice_id` linkage supports metered billing cycles without duplicating readings into invoices.

8. **EPA-aligned environmental logging** — `environmental_logs.log_type` categories match EPA Clean Marina audit checklist items. `incident_severity` and `epa_reported` fields support the reporting escalation workflow.

9. **ABYC E-13 lithium battery tracking** — `vessels.has_lithium_batteries` and `slips.lithium_charging_approved` enable slip assignment rules that enforce lithium battery charging protocol requirements.

10. **Invoices with polymorphic source** — `invoices.source_type` and `source_id` link invoices to reservations, contracts, work orders, fuel transactions, or utility readings. This supports the multiple revenue streams that distinguish marina operations from single-product businesses.

11. **Vessel compliance fields** — `registration_expiry` and `insurance_expiry` on vessels enable compliance dashboards that flag vessels with lapsed documentation, supporting marina liability management.

12. **Map coordinates on slips** — `map_x`, `map_y`, `map_rotation` store the slip's position on the visual marina map, enabling drag-and-drop rendering without a separate map metadata layer.
