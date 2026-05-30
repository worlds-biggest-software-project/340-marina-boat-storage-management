# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Marina & Boat Storage Management · Created: 2026-05-25

## Philosophy

This model treats every state change in the marina as an immutable event — a vessel checking in, a slip being reassigned, fuel being dispensed, a pump-out being logged, a work order advancing, a contract being renewed. The `event_store` is the single source of truth; all queryable state is materialised into read-model tables via projections (CQRS pattern).

Marinas operate across multiple revenue streams (dockage, fuel, service, utilities, retail) with seasonal cycles, environmental compliance requirements, and dynamic pricing — all of which benefit from event-sourced architecture. Occupancy changes are events that drive yield management. Fuel dispenser readings are events that feed demand forecasting and leak detection. Pump-out logs are events that generate EPA compliance reports. Work order progression events power technician efficiency analytics. The complete event history enables temporal queries ("which vessels were docked during the storm on March 15?") and new read models derived from historical events without schema migration.

Events follow the CloudEvents specification for interoperability with FuelCloud webhooks, Stripe payment events, and QuickBooks change notifications. The marina's event stream is a natural integration surface for the fragmented hardware ecosystem (fuel dispensers, utility meters, gate controllers).

**Best for:** Marina groups and operators that want full operational audit trails, environmental compliance reporting by construction, fuel anomaly detection from dispenser event streams, and the ability to derive new analytics (occupancy trends, pricing optimisation, maintenance prediction) from historical events.

**Trade-offs:**
- (+) Complete audit trail — every slip assignment, fuel sale, and environmental incident is immutable
- (+) Temporal queries — replay to any date for occupancy, revenue, or compliance snapshots
- (+) New read models derived from existing events without schema migration
- (+) EPA compliance reports generated from event replay rather than separate logging
- (+) Natural fit for hardware event ingestion (FuelCloud, utility meters)
- (-) Eventual consistency — read models lag behind event writes
- (-) Event schema evolution requires careful versioning
- (-) Higher storage volume than state-only models
- (-) Visual slip map requires read model, not direct event queries

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0.2 | `event_store` columns: `ce_source`, `ce_type`, `ce_specversion`, `ce_time` |
| ISO 8666 | Vessel dimensions in vessel stream events and `rm_vessels` read model |
| USCG HIN | HIN in vessel stream events; validated on `vessel_registered` |
| EPA Clean Marina | Environmental events (pumpout, spill, waste) materialise into compliance read model |
| ABYC E-13 | Lithium battery status in vessel events; slip assignment rules in marina events |
| NMEA 0183/2000 | Telematics data ingested as vessel events (engine hours, tank levels) |
| PCI DSS v4.0 | Payment events reference Stripe tokens only |
| Stripe | Payment events mirror Stripe webhook event types |

---

## Infrastructure Tables

### event_store

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
                        'reservation','vessel','slip','work_order',
                        'fuel','utility','payment','marina','ai')),
    stream_id       UUID NOT NULL,
    version         INTEGER NOT NULL,
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',

    -- CloudEvents envelope
    ce_source       TEXT NOT NULL DEFAULT '/marina-management',
    ce_type         TEXT NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',
    ce_time         TIMESTAMPTZ NOT NULL DEFAULT now(),

    marina_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('staff','customer','system','integration','ai')),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_es_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_es_marina ON event_store(marina_id, created_at DESC);
CREATE INDEX idx_es_type ON event_store(event_type, created_at DESC);
CREATE INDEX idx_es_actor ON event_store(actor_id, created_at DESC);
```

### stream_snapshots

```sql
CREATE TABLE stream_snapshots (
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    version         INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);
```

### projection_checkpoints

```sql
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Catalogue

### Reservation Stream (`stream_type = 'reservation'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `reservation_created` | marina_id, slip_id, vessel_id, customer_id, type, start_date, end_date, hold_type | Booking initiated |
| `reservation_priced` | rate_per_foot_cents, flat_rate_cents, total_cents, dynamic_pricing | Rate set or adjusted |
| `reservation_confirmed` | confirmed_at, confirmation_number | Booking confirmed |
| `check_in_completed` | check_in_at, slip_number, vessel_loa_cm | Vessel arrived |
| `slip_swapped` | old_slip_id, new_slip_id, reason | Slip reassignment |
| `check_out_completed` | check_out_at, total_stay_days | Vessel departed |
| `reservation_cancelled` | reason, cancelled_at, refund_cents | Cancellation |
| `no_show_recorded` | expected_date | No-show flagged |
| `contract_signed` | contract_type, annual_rate_cents, signed_at, signature_url | Seasonal/annual agreement |
| `contract_renewed` | new_start_date, new_end_date, new_rate_cents | Renewal processed |
| `arrival_briefing_sent` | channel, content | AI-generated arrival info |
| `weather_alert_sent` | alert_type, severity, message | Weather notification |
| `review_requested` | channel, sent_at | Post-visit survey |
| `message_sent` | direction, channel, type, body | Communication logged |

### Vessel Stream (`stream_type = 'vessel'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `vessel_registered` | hin, cin, name, make, model, year, loa_cm, beam_cm, draft_cm, vessel_type, customer_id | First visit |
| `vessel_updated` | changed_fields | Profile changes |
| `registration_updated` | number, state, expiry | Registration renewal |
| `insurance_updated` | policy, expiry | Insurance renewal |
| `engine_hours_logged` | engine_hours, logged_at | Engine hour reading |
| `telematics_connected` | nmea_device_id, capabilities[] | NMEA device linked |
| `telematics_reading` | engine_hours, fuel_level_pct, battery_volts, bilge_status | Periodic reading |
| `lithium_battery_status_changed` | has_lithium, battery_type, capacity_kwh | Battery status update |
| `maintenance_due` | service_type, due_engine_hours, due_date, oem_schedule | AI or OEM alert |
| `churn_risk_flagged` | risk_score, factors[], last_visit | AI retention signal |

### Slip Stream (`stream_type = 'slip'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `slip_created` | slip_number, slip_type, dock, section, max_loa_cm, amenities, map_coords | Infrastructure added |
| `slip_status_changed` | from_status, to_status, reason | Available ↔ occupied ↔ maintenance |
| `slip_assigned` | vessel_id, reservation_id, assigned_at | Vessel placed in slip |
| `slip_vacated` | vessel_id, vacated_at | Vessel removed |
| `slip_rate_updated` | old_seasonal, new_seasonal, old_transient, new_transient | Rate change |
| `slip_maintenance_started` | reason, expected_completion | Out of service |
| `slip_maintenance_completed` | completed_at, work_performed | Back in service |
| `lithium_approval_changed` | approved, reason | E-13 policy change |

### Work Order Stream (`stream_type = 'work_order'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `wo_requested` | vessel_id, customer_id, wo_type, description, priority | Service request |
| `wo_estimated` | line_items[], estimate_cents | Estimate created |
| `wo_approved` | approved_at, approved_by | Customer approved |
| `line_added` | line_id, line_type, description, technician_id, price_cents, hours_estimated | Line item added |
| `work_started` | technician_id, started_at | Work begins |
| `time_logged` | technician_id, line_id, hours, clock_in, clock_out | Time capture |
| `part_consumed` | part_number, quantity, line_id | Part used |
| `photo_uploaded` | line_id, url, description | Before/after photo |
| `wo_completed` | completed_at, subtotal_cents, tax_cents, total_cents | Work finished |
| `haul_out_scheduled` | scheduled_date, crane_required, vessel_weight | Haul-out planned |
| `launch_completed` | launched_at, slip_id | Vessel back in water |

### Fuel Stream (`stream_type = 'fuel'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `fuel_dispensed` | pump_id, fuel_type, quantity_litres, unit_price_cents, total_cents, cost_cents, vessel_id, customer_id, attendant_id | Fuel sale |
| `tank_reading` | pump_id, fuel_type, level_litres, capacity_litres, reading_at | Tank level reading |
| `delivery_received` | pump_id, fuel_type, quantity_litres, cost_cents, supplier, delivery_at | Fuel delivery |
| `anomaly_detected` | pump_id, anomaly_type, expected_litres, actual_litres, variance_pct | AI leak/theft detection |
| `price_updated` | pump_id, fuel_type, old_price_cents, new_price_cents | Price change |

### Utility Stream (`stream_type = 'utility'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `meter_read` | slip_id, utility_type, previous_reading, current_reading, usage, read_by | Meter reading |
| `usage_billed` | slip_id, utility_type, usage, rate_cents, total_cents, invoice_id | Billed to invoice |
| `pumpout_completed` | slip_id, vessel_id, quantity_gallons, performed_by | Pump-out logged |
| `pumpout_station_serviced` | station_id, service_type, serviced_by | Equipment maintenance |

### Marina Stream (`stream_type = 'marina'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `marina_created` | name, address, timezone, total_slips | Initial setup |
| `settings_updated` | changed_fields | Rates, hours, policies |
| `staff_hired` | staff_id, full_name, role, certifications | New staff |
| `integration_connected` | provider, config | QuickBooks, FuelCloud, Stripe |
| `environmental_incident` | log_type, quantity, unit, location, severity, response_actions | Spill, waste event |
| `epa_report_filed` | report_number, period, filed_at | EPA report submitted |
| `clean_marina_audit` | audit_date, findings[], score, auditor | EPA audit result |
| `part_catalogued` | part_number, description, cost_cents, retail_cents, qty | Inventory added |
| `part_restocked` | part_number, quantity, supplier, cost_cents | Restocked |

### Payment Stream (`stream_type = 'payment'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `invoice_generated` | customer_id, source_type, source_id, line_items[], total_cents, due_date | Invoice created |
| `invoice_sent` | channel, sent_at | Invoice delivered |
| `invoice_viewed` | viewed_at | Customer opened |
| `payment_initiated` | invoice_id, amount_cents, method, stripe_pi_id | Payment started |
| `payment_succeeded` | amount_cents, paid_at, stripe_pi_id | Confirmed |
| `payment_failed` | amount_cents, failure_reason | Failed |
| `payment_refunded` | refund_cents, reason | Refund |
| `qbo_synced` | qbo_invoice_id or qbo_payment_id, synced_at | Accounting sync |

### AI Stream (`stream_type = 'ai'`)

| Event Type | Payload Fields | Notes |
|-----------|---------------|-------|
| `dynamic_price_calculated` | slip_id, base_rate, demand_multiplier, event_premium, final_rate, confidence | Yield management |
| `maintenance_predicted` | vessel_id, service_type, predicted_date, confidence, factors[] | Predictive maintenance |
| `fuel_demand_forecast` | pump_id, period, predicted_litres, confidence, factors[] | Fuel delivery optimisation |
| `fuel_anomaly_flagged` | pump_id, anomaly_type, variance_pct, recommended_action | Leak/theft detection |
| `churn_risk_calculated` | customer_id, risk_score, factors[] | Retention signal |
| `guest_briefing_generated` | reservation_id, content, weather_data, local_events | Personalised arrival info |
| `compliance_report_generated` | report_type, period, summary | EPA Clean Marina report |

---

## Read Model Tables

### rm_slip_map

```sql
CREATE TABLE rm_slip_map (
    id              UUID PRIMARY KEY,
    marina_id       UUID NOT NULL,
    slip_number     TEXT NOT NULL,
    slip_type       TEXT NOT NULL,
    dock            TEXT,
    section         TEXT,
    max_loa_cm      INTEGER NOT NULL,
    max_beam_cm     INTEGER,
    amenities       JSONB NOT NULL DEFAULT '{}',
    -- Example: {"power_amps":50,"power_voltage":240,"water":true,"pumpout":false,"wifi":true,
    --   "lithium_approved":false}
    status          TEXT NOT NULL,
    map_x           NUMERIC(8,2),
    map_y           NUMERIC(8,2),
    map_rotation    NUMERIC(5,2),
    seasonal_rate_cents BIGINT,
    transient_rate_cents BIGINT,

    current_vessel  JSONB,
    -- Example: {"vessel_id":"uuid","name":"Sea Breeze","hin":"ABC12345D607",
    --   "loa_cm":1220,"customer_name":"John Smith","reservation_type":"seasonal",
    --   "check_in":"2026-04-01","end_date":"2026-10-31"}

    next_reservation JSONB,
    -- Example: {"reservation_id":"uuid","vessel_name":"Wind Song","start_date":"2026-11-01"}

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (marina_id, slip_number)
);

CREATE INDEX idx_rm_slip_marina ON rm_slip_map(marina_id, status);
CREATE INDEX idx_rm_slip_size ON rm_slip_map(marina_id, max_loa_cm);
```

### rm_revenue

```sql
CREATE TABLE rm_revenue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    marina_id       UUID NOT NULL,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','weekly','monthly')),
    department      TEXT NOT NULL CHECK (department IN ('dockage','fuel','service','utilities','retail')),

    revenue_cents        BIGINT NOT NULL DEFAULT 0,
    cost_cents           BIGINT NOT NULL DEFAULT 0,
    gross_profit_cents   BIGINT NOT NULL DEFAULT 0,
    gross_margin_pct     NUMERIC(5,2) NOT NULL DEFAULT 0,
    transaction_count    INTEGER NOT NULL DEFAULT 0,

    -- Dockage-specific
    occupancy_pct   NUMERIC(5,2),
    transient_nights INTEGER,
    seasonal_slips  INTEGER,
    revenue_per_foot_cents BIGINT,
    avg_dynamic_multiplier NUMERIC(4,2),

    -- Fuel-specific
    litres_dispensed NUMERIC(12,2),
    fuel_margin_cents BIGINT,

    -- Service-specific
    wo_completed    INTEGER,
    hours_billed    NUMERIC(8,2),

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (marina_id, period, period_type, department)
);

CREATE INDEX idx_rm_rev ON rm_revenue(marina_id, period_type, department, period DESC);
```

### rm_vessels

```sql
CREATE TABLE rm_vessels (
    id              UUID PRIMARY KEY,
    marina_id       UUID NOT NULL,
    hin             TEXT,
    name            TEXT,
    make            TEXT,
    model           TEXT,
    year            SMALLINT,
    loa_cm          INTEGER NOT NULL,
    beam_cm         INTEGER,
    vessel_type     TEXT,
    customer_id     UUID NOT NULL,
    customer_name   TEXT NOT NULL,
    customer_phone  TEXT,
    customer_email  TEXT,
    has_lithium_batteries BOOLEAN NOT NULL DEFAULT false,
    engine_hours    INTEGER,
    registration_expiry DATE,
    insurance_expiry DATE,
    current_slip    TEXT,
    current_reservation_id UUID,
    next_maintenance JSONB,
    -- Example: {"service_type":"engine_oil","due_date":"2026-07-15","due_engine_hours":500}
    last_visit      TIMESTAMPTZ,
    total_visits     INTEGER NOT NULL DEFAULT 0,
    lifetime_revenue_cents BIGINT NOT NULL DEFAULT 0,
    ai_churn_risk   NUMERIC(4,3),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_vessels_marina ON rm_vessels(marina_id);
CREATE INDEX idx_rm_vessels_compliance ON rm_vessels(registration_expiry) WHERE registration_expiry IS NOT NULL;
CREATE INDEX idx_rm_vessels_insurance ON rm_vessels(insurance_expiry) WHERE insurance_expiry IS NOT NULL;
```

### rm_compliance

```sql
CREATE TABLE rm_compliance (
    marina_id       UUID PRIMARY KEY,

    -- EPA Clean Marina
    pumpouts_ytd    INTEGER NOT NULL DEFAULT 0,
    pumpouts_this_month INTEGER NOT NULL DEFAULT 0,
    waste_oil_gallons_ytd NUMERIC(10,2) NOT NULL DEFAULT 0,
    spill_incidents_ytd INTEGER NOT NULL DEFAULT 0,
    reportable_incidents_ytd INTEGER NOT NULL DEFAULT 0,
    unreported_incidents INTEGER NOT NULL DEFAULT 0,
    last_epa_audit  TIMESTAMPTZ,
    epa_audit_score NUMERIC(5,2),
    clean_marina_certified BOOLEAN NOT NULL DEFAULT false,

    -- Fuel
    fuel_anomalies_30d INTEGER NOT NULL DEFAULT 0,
    fuel_variance_pct_30d NUMERIC(5,2),

    -- Vessel compliance
    vessels_registration_expired INTEGER NOT NULL DEFAULT 0,
    vessels_insurance_expired INTEGER NOT NULL DEFAULT 0,
    vessels_with_lithium INTEGER NOT NULL DEFAULT 0,

    -- ABYC
    lithium_approved_slips INTEGER NOT NULL DEFAULT 0,
    lithium_policy TEXT,

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### rm_marina_dashboard

```sql
CREATE TABLE rm_marina_dashboard (
    marina_id       UUID PRIMARY KEY,
    marina_name     TEXT NOT NULL,

    -- Occupancy
    total_slips     INTEGER NOT NULL DEFAULT 0,
    occupied_slips  INTEGER NOT NULL DEFAULT 0,
    available_slips INTEGER NOT NULL DEFAULT 0,
    maintenance_slips INTEGER NOT NULL DEFAULT 0,
    occupancy_pct   NUMERIC(5,2) NOT NULL DEFAULT 0,

    -- Today
    arrivals_today  INTEGER NOT NULL DEFAULT 0,
    departures_today INTEGER NOT NULL DEFAULT 0,
    today_revenue_cents BIGINT NOT NULL DEFAULT 0,
    today_fuel_litres NUMERIC(10,2) NOT NULL DEFAULT 0,

    -- Service
    open_work_orders INTEGER NOT NULL DEFAULT 0,
    waiting_parts   INTEGER NOT NULL DEFAULT 0,
    techs_on_site   INTEGER NOT NULL DEFAULT 0,
    haul_outs_scheduled INTEGER NOT NULL DEFAULT 0,

    -- Fuel
    fuel_levels     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"diesel":{"level_litres":7200,"capacity":10000,"pct":72},
    --   "gasoline":{"level_litres":3400,"capacity":5000,"pct":68}}

    -- Compliance
    compliance_alerts JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type":"fuel_anomaly","pump":"P1","severity":"medium","message":"5% variance"},
    --   {"type":"vessel_insurance_expired","count":3}]

    -- AI insights
    ai_insights     JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type":"dynamic_pricing","message":"Demand spike expected July 4 weekend — 1.5x rates recommended"},
    --   {"type":"maintenance_due","message":"4 vessels due for service this month"}]

    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Event Replay Queries

### Occupancy history for any date range

```sql
SELECT date_trunc('day', ce_time) AS day,
       COUNT(*) FILTER (WHERE event_type = 'check_in_completed') AS arrivals,
       COUNT(*) FILTER (WHERE event_type = 'check_out_completed') AS departures
FROM event_store
WHERE stream_type = 'reservation'
  AND marina_id = $1
  AND ce_time BETWEEN $2 AND $3
GROUP BY day ORDER BY day;
```

### Fuel dispensing anomaly detection (variance between tank readings and sales)

```sql
WITH sales AS (
    SELECT (payload->>'pump_id') AS pump_id,
           SUM((payload->>'quantity_litres')::numeric) AS dispensed
    FROM event_store
    WHERE stream_type = 'fuel' AND event_type = 'fuel_dispensed'
      AND marina_id = $1 AND ce_time BETWEEN $2 AND $3
    GROUP BY payload->>'pump_id'
),
readings AS (
    SELECT payload->>'pump_id' AS pump_id,
           FIRST_VALUE((payload->>'level_litres')::numeric) OVER w AS start_level,
           LAST_VALUE((payload->>'level_litres')::numeric) OVER w AS end_level
    FROM event_store
    WHERE stream_type = 'fuel' AND event_type = 'tank_reading'
      AND marina_id = $1 AND ce_time BETWEEN $2 AND $3
    WINDOW w AS (PARTITION BY payload->>'pump_id' ORDER BY ce_time
                 ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
),
deliveries AS (
    SELECT payload->>'pump_id' AS pump_id,
           SUM((payload->>'quantity_litres')::numeric) AS delivered
    FROM event_store
    WHERE stream_type = 'fuel' AND event_type = 'delivery_received'
      AND marina_id = $1 AND ce_time BETWEEN $2 AND $3
    GROUP BY payload->>'pump_id'
)
SELECT s.pump_id, s.dispensed, d.delivered,
       r.start_level, r.end_level,
       (r.start_level + COALESCE(d.delivered,0) - r.end_level) AS expected_dispensed,
       s.dispensed - (r.start_level + COALESCE(d.delivered,0) - r.end_level) AS variance_litres
FROM sales s
LEFT JOIN readings r ON r.pump_id = s.pump_id
LEFT JOIN deliveries d ON d.pump_id = s.pump_id;
```

### EPA compliance report (pump-out and incident history)

```sql
SELECT event_type,
       COUNT(*) AS event_count,
       SUM(CASE WHEN event_type = 'pumpout_completed'
           THEN (payload->>'quantity_gallons')::numeric ELSE 0 END) AS total_pumpout_gallons,
       COUNT(*) FILTER (WHERE event_type = 'environmental_incident'
           AND payload->>'severity' = 'reportable') AS reportable_incidents
FROM event_store
WHERE marina_id = $1
  AND event_type IN ('pumpout_completed', 'environmental_incident')
  AND ce_time BETWEEN $2 AND $3
GROUP BY event_type;
```

### Dynamic pricing effectiveness

```sql
WITH priced AS (
    SELECT stream_id,
           (payload->'dynamic_pricing'->>'demand_multiplier')::numeric AS multiplier,
           (payload->>'total_cents')::bigint AS revenue_cents
    FROM event_store
    WHERE stream_type = 'reservation' AND event_type = 'reservation_priced'
      AND marina_id = $1 AND ce_time >= now() - interval '6 months'
),
completed AS (
    SELECT stream_id FROM event_store
    WHERE stream_type = 'reservation' AND event_type = 'check_out_completed'
      AND marina_id = $1
)
SELECT CASE WHEN p.multiplier > 1.0 THEN 'dynamic_premium' ELSE 'base_rate' END AS pricing_tier,
       COUNT(*) AS reservations,
       AVG(p.revenue_cents) AS avg_revenue_cents,
       SUM(p.revenue_cents) AS total_revenue_cents
FROM priced p
JOIN completed c ON c.stream_id = p.stream_id
GROUP BY pricing_tier;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_slip_map, rm_revenue, rm_vessels, rm_compliance, rm_marina_dashboard |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Nine stream types cover the full marina** — reservation, vessel, slip, work_order, fuel, utility, payment, marina, and ai. Each maps to a distinct operational domain. The fuel and utility streams are separated from the slip stream because they generate high-volume machine events from dispensers and meters.

2. **Fuel stream enables anomaly detection** — `fuel_dispensed`, `tank_reading`, and `delivery_received` events create a closed reconciliation loop. Variance between tank drawdown and recorded sales flags potential leaks or theft — a capability no analysed incumbent offers.

3. **Environmental compliance from event replay** — `pumpout_completed` events on the utility stream and `environmental_incident` events on the marina stream generate EPA Clean Marina compliance reports by replaying events over any audit period. No separate compliance logging system needed.

4. **Slip stream captures infrastructure lifecycle** — slip creation, status changes, rate updates, and maintenance periods are events. The `rm_slip_map` read model materialises current state for the visual map, but the full slip history (rate changes, occupancy patterns) is available via replay.

5. **Dynamic pricing as event pair** — `dynamic_price_calculated` (AI recommendation) and `reservation_priced` (applied price) events enable pricing effectiveness analysis: what rate did AI recommend, what was actually charged, and did the reservation convert?

6. **Revenue read model by department** — `rm_revenue` breaks down by department (dockage, fuel, service, utilities, retail) and period, enabling the multi-stream P&L view that differentiates marina from single-revenue-stream businesses.

7. **Vessel telematics as events** — `telematics_reading` events from NMEA devices (engine hours, fuel level, battery voltage, bilge status) feed predictive maintenance models and vessel health monitoring without a separate telemetry database.

8. **Compliance read model** — `rm_compliance` aggregates EPA metrics (pump-outs, waste oil, spill incidents), fuel anomalies, vessel documentation status, and ABYC lithium battery policy data. All derived from event replay.

9. **Guest experience events** — `arrival_briefing_sent`, `weather_alert_sent`, and `review_requested` events on the reservation stream track personalised guest communications, enabling the AI to optimise timing, content, and channel based on historical engagement patterns.

10. **AI stream for all predictions** — dynamic pricing, maintenance prediction, fuel demand forecasting, anomaly detection, churn risk, and guest briefing generation all produce events in the AI stream. This enables accuracy tracking (did the predicted maintenance need actually occur?) and model performance monitoring.
