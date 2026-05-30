# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Rail Network Operations · Created: 2026-05-26

## Philosophy

Rail operations are a continuous stream of events: a timetable is published, a path is allocated, a crew member signs on, a train departs, a signal fails, a dispatcher reroutes trains, a possession starts and ends. Every one of these events changes the state of the network. An event-sourced architecture captures each event as an immutable record, and current state — which trains are running, which sections are blocked, which crew are on duty — is derived by projecting events into read models.

This maps naturally to how rail operations actually work. A dispatcher responding to a signal failure needs to know not just "section X is blocked" but "when did the failure start, what trains are affected, what rerouting decisions have been made, and in what order?" That temporal chain of events is the native data structure of event sourcing. The same event stream supports what-if replanning: "if I had rerouted train A via the diversionary route at 10:35 instead of waiting for the repair at 10:50, what would the cascade delay have been?"

The event store also provides the training data for AI models: conflict resolution models learn from the sequence of dispatcher decisions during disruptions, predictive maintenance models consume rolling stock sensor event streams, and demand forecasting models use boarding/alighting event patterns. The immutable log simultaneously serves regulatory audit requirements — EN 50126 RAMS lifecycle evidence, NIS2 cybersecurity event trails, and delay attribution records for performance regimes.

**Best for:** Infrastructure managers and regulators who need full temporal auditability, disruption replay capability, and AI training data grounded in operational event sequences.

**Trade-offs:**
- Pro: Complete temporal audit trail for delay attribution, RAMS evidence, and regulatory compliance
- Pro: Disruption replay and what-if analysis are native capabilities
- Pro: Event streams directly feed AI model training (conflict resolution, predictive maintenance)
- Pro: New read models added without touching the event store
- Con: Higher storage requirements (event store + read model projections)
- Con: Eventual consistency between events and read models
- Con: Event schema versioning requires governance
- Con: Real-time dashboards must read from projections, not the event store

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RailML 3.x | Entity naming in event payloads |
| railTopoModel (IRS 30100) | Infrastructure events reference topological identifiers |
| UIC 406 | Capacity utilisation computed from train movement events |
| GTFS / GTFS-RT | GTFS-RT alerts and trip updates generated from event projections |
| NeTEx / SIRI | SIRI service delivery messages generated from disruption events |
| ERTMS/ETCS | Signalling state changes as events |
| EN 50126 | RAMS lifecycle evidence captured as event sequences |
| IEC 61375 | Onboard sensor events from Train Communication Network |
| CloudEvents 1.0 | Event envelope format |
| AsyncAPI 3.0 | Event streaming specification for train positions and alerts |
| IEC 62443 / NIS2 | Cybersecurity event logging and actor provenance |
| GDPR | Crew personal data events with retention handling |
| MCP | Read models exposed via MCP for AI assistant queries |

---

## Infrastructure Tables

### event_store

```sql
CREATE TABLE event_store (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stream_type    TEXT NOT NULL CHECK (stream_type IN (
                   'network', 'infrastructure', 'rolling_stock', 'timetable',
                   'train', 'crew', 'possession', 'maintenance',
                   'disruption', 'passenger_info', 'ai', 'config'
                 )),
  stream_id      UUID NOT NULL,
  sequence_no    BIGINT NOT NULL,
  event_type     TEXT NOT NULL,
  event_data     JSONB NOT NULL,
  metadata       JSONB NOT NULL DEFAULT '{}',

  -- CloudEvents envelope
  ce_source      TEXT NOT NULL,
  ce_type        TEXT NOT NULL,
  ce_specversion TEXT NOT NULL DEFAULT '1.0',
  ce_time        TIMESTAMPTZ NOT NULL DEFAULT now(),

  -- Actor provenance
  actor_id       UUID,
  actor_type     TEXT NOT NULL CHECK (actor_type IN (
                   'dispatcher', 'planner', 'controller', 'driver',
                   'maintenance_tech', 'system', 'ai',
                   'interlocking', 'tms', 'scada', 'etcs_rbc',
                   'onboard_sensor', 'track_circuit', 'axle_counter',
                   'gsm_r', 'frmcs', 'api_client', 'scheduler',
                   'gtfs_publisher'
                 )),

  -- Context
  operator_id    UUID NOT NULL,
  location_ref   TEXT,                   -- station code or track section code

  -- Correlation
  correlation_id UUID,
  causation_id   UUID,

  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (stream_type, stream_id, sequence_no)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id);
CREATE INDEX idx_events_type ON event_store(event_type);
CREATE INDEX idx_events_operator ON event_store(operator_id);
CREATE INDEX idx_events_time ON event_store(ce_time);
CREATE INDEX idx_events_actor ON event_store(actor_id);
CREATE INDEX idx_events_correlation ON event_store(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_events_location ON event_store(location_ref) WHERE location_ref IS NOT NULL;
```

### stream_snapshots

```sql
CREATE TABLE stream_snapshots (
  stream_type    TEXT NOT NULL,
  stream_id      UUID NOT NULL,
  sequence_no    BIGINT NOT NULL,
  snapshot_data  JSONB NOT NULL,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (stream_type, stream_id, sequence_no)
);
```

### projection_checkpoints

```sql
CREATE TABLE projection_checkpoints (
  projection_name TEXT PRIMARY KEY,
  last_event_id   UUID NOT NULL,
  last_sequence   BIGINT NOT NULL,
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Taxonomy

### Infrastructure Stream
| Event Type | Key Fields |
|------------|------------|
| `station.registered` | name, station_code, ibnr, type, location, platforms |
| `station.platform_modified` | platform_number, length_m, accessibility |
| `track_section.registered` | from_station, to_station, length_km, max_speed, signalling, electrification |
| `track_section.speed_restriction_imposed` | km_start, km_end, speed_kph, reason |
| `track_section.speed_restriction_lifted` | km_start, km_end |
| `track_section.status_changed` | from_status, to_status, reason |
| `track_section.condition_assessed` | overall, track, signalling, ohl, inspector |
| `signal.state_changed` | signal_id, from_aspect, to_aspect |

### Rolling Stock Stream
| Event Type | Key Fields |
|------------|------------|
| `unit.registered` | unit_number, class, type, traction, max_speed, seats, etcs |
| `unit.status_changed` | from_status, to_status, reason |
| `unit.mileage_updated` | new_mileage_km, delta_km |
| `unit.exam_completed` | exam_type, result, next_due |
| `unit.defect_reported` | defect_code, severity, description |
| `unit.sensor_reading` | sensor_type, value, unit, anomaly_score (IEC 61375) |
| `unit.failure_predicted` | component, probability, rul_km, model_id |

### Timetable Stream
| Event Type | Key Fields |
|------------|------------|
| `timetable.published` | version, valid_from, valid_until, services_count |
| `service.created` | service_code, type, origin, destination, days_of_operation, path |
| `service.modified` | changes (path, timing, stopping_pattern) |
| `service.cancelled` | reason, affected_date_range |
| `path.allocated` | service_id, track_sections, timing_points, priority |
| `path.conflict_detected` | service_a, service_b, section, time_window, conflict_type |
| `path.conflict_resolved` | resolution (repathed/retimed/prioritised), decided_by |

### Train Stream (per-run)
| Event Type | Key Fields |
|------------|------------|
| `train.formed` | run_id, service_code, date, rolling_stock, consist |
| `train.departed` | station, platform, scheduled_time, actual_time, delay_seconds |
| `train.arrived` | station, platform, scheduled_time, actual_time, delay_seconds |
| `train.passed` | station, scheduled_time, actual_time |
| `train.position_reported` | latitude, longitude, speed_kph, track_section, source |
| `train.delayed` | cause, delay_seconds, cause_operator, attribution_code |
| `train.diverted` | original_path, new_path, reason |
| `train.cancelled_en_route` | station, reason |
| `train.terminated_short` | station, reason |
| `train.completed` | final_delay_seconds, pax_estimated |

### Crew Stream
| Event Type | Key Fields |
|------------|------------|
| `crew.registered` | employee_number, role, depot, traction_knowledge, route_knowledge |
| `crew.duty_rostered` | duty_date, train_runs, sign_on, sign_off, planned_hours |
| `crew.signed_on` | station, actual_time |
| `crew.signed_off` | station, actual_time, driving_hours, total_hours |
| `crew.reassigned` | from_run, to_run, reason (disruption_roster) |
| `crew.compliance_violated` | rule, description, severity |
| `crew.certification_updated` | cert, issued, expires |

### Possession Stream
| Event Type | Key Fields |
|------------|------------|
| `possession.planned` | type, sections, stations, planned_start, planned_end |
| `possession.confirmed` | approved_by, affected_services |
| `possession.started` | actual_start |
| `possession.overrun` | new_estimated_end, reason |
| `possession.completed` | actual_end, work_summary |
| `possession.cancelled` | reason |
| `replacement_transport.arranged` | mode, route, provider |

### Maintenance Stream
| Event Type | Key Fields |
|------------|------------|
| `inspection.scheduled` | asset_type, asset_ref, type, planned_date |
| `inspection.completed` | result, defects_found, inspector |
| `defect.reported` | asset_type, asset_ref, defect_code, severity, location_km |
| `defect.repaired` | repair_method, parts_used, labour_hours |
| `work_order.created` | type, asset, priority, description |
| `work_order.completed` | actual_end, cost_cents |
| `predictive_alert.generated` | unit_number, component, probability, model_id |

### Disruption Stream
| Event Type | Key Fields |
|------------|------------|
| `disruption.started` | type, severity, location, sections, stations |
| `disruption.mitigated` | action, decided_by |
| `disruption.escalated` | new_severity, reason |
| `disruption.resolved` | resolution, duration_minutes |
| `disruption.impact_assessed` | trains_affected, total_delay_minutes, cancellations |
| `disruption.lessons_learned` | findings, recommendations |

### Passenger Info Stream
| Event Type | Key Fields |
|------------|------------|
| `pax.alert_published` | channel (siri/gtfs_rt/app), message, affected_services |
| `pax.timetable_revised` | affected_date, changes_count |
| `pax.delay_notification` | train_number, delay_minutes, reason_text |

### AI Stream
| Event Type | Key Fields |
|------------|------------|
| `ai.conflict_resolution_suggested` | conflict_id, options, recommended, confidence |
| `ai.repathing_suggested` | train_id, original_path, suggested_path, delay_saved_min |
| `ai.demand_forecast_generated` | service_id, date, forecast_pax, model_id |
| `ai.maintenance_predicted` | unit_number, component, rul_km, probability |
| `ai.disruption_scenario_generated` | scenario_type, parameters, projected_impact |
| `ai.possession_optimised` | possession_id, original_window, suggested_window, impact_reduction |
| `ai.suggestion_accepted` | suggestion_id, accepted_by |
| `ai.suggestion_rejected` | suggestion_id, rejected_by, reason |

---

## Read Model Tables

### rm_network_status

```sql
CREATE TABLE rm_network_status (
  section_id     UUID PRIMARY KEY,
  operator_id    UUID NOT NULL,
  section_code   TEXT NOT NULL,
  name           TEXT NOT NULL,
  from_station   TEXT NOT NULL,
  to_station     TEXT NOT NULL,
  length_km      NUMERIC(8, 3) NOT NULL,

  -- Current state
  status         TEXT NOT NULL,
  max_speed_kph  INTEGER NOT NULL,
  signalling     TEXT,
  electrification TEXT,
  capacity_tph   INTEGER,
  utilisation_pct NUMERIC(5, 2),

  -- Active restrictions
  speed_restrictions JSONB DEFAULT '[]',
  active_possessions JSONB DEFAULT '[]',
  active_disruptions JSONB DEFAULT '[]',

  -- Asset condition
  condition_overall TEXT,
  defects_open   INTEGER DEFAULT 0,
  last_inspection JSONB DEFAULT '{}',
  next_renewal_due DATE,

  -- Trains currently on section
  trains_on_section JSONB DEFAULT '[]',

  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_net_operator ON rm_network_status(operator_id);
CREATE INDEX idx_rm_net_status ON rm_network_status(status);
CREATE INDEX idx_rm_net_condition ON rm_network_status(condition_overall)
  WHERE condition_overall IN ('poor', 'critical');
```

### rm_train_board

Live departure/arrival board and service tracking.

```sql
CREATE TABLE rm_train_board (
  run_id         UUID PRIMARY KEY,
  operator_id    UUID NOT NULL,
  service_code   TEXT NOT NULL,
  service_type   TEXT NOT NULL,
  run_date       DATE NOT NULL,
  origin         TEXT NOT NULL,
  destination    TEXT NOT NULL,
  status         TEXT NOT NULL,

  -- Rolling stock and crew
  rolling_stock  JSONB DEFAULT '{}',
  crew           JSONB DEFAULT '[]',

  -- Path with actual times
  path           JSONB NOT NULL DEFAULT '[]',

  -- Timing
  scheduled_departure TIMESTAMPTZ NOT NULL,
  scheduled_arrival TIMESTAMPTZ NOT NULL,
  actual_departure TIMESTAMPTZ,
  actual_arrival TIMESTAMPTZ,
  current_delay_seconds INTEGER DEFAULT 0,
  delay_attribution JSONB DEFAULT '{}',

  -- Current position
  last_reported_station TEXT,
  last_position  JSONB DEFAULT '{}',
  next_station   TEXT,

  -- Performance
  load_factor_pct NUMERIC(5, 2),
  pax_estimated  INTEGER,
  gtfs_trip_id   TEXT,

  -- Timeline of key events
  timeline       JSONB DEFAULT '[]',

  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_tb_operator ON rm_train_board(operator_id);
CREATE INDEX idx_rm_tb_date ON rm_train_board(run_date);
CREATE INDEX idx_rm_tb_status ON rm_train_board(status);
CREATE INDEX idx_rm_tb_delay ON rm_train_board(current_delay_seconds)
  WHERE current_delay_seconds > 300;
CREATE INDEX idx_rm_tb_service ON rm_train_board(service_code);
```

### rm_disruption_board

```sql
CREATE TABLE rm_disruption_board (
  disruption_id  UUID PRIMARY KEY,
  operator_id    UUID NOT NULL,
  disruption_type TEXT NOT NULL,
  severity       TEXT NOT NULL,
  title          TEXT NOT NULL,
  status         TEXT NOT NULL,

  -- Location
  affected_sections JSONB DEFAULT '[]',
  affected_stations JSONB DEFAULT '[]',
  location_description TEXT,

  -- Timeline
  started_at     TIMESTAMPTZ NOT NULL,
  estimated_end  TIMESTAMPTZ,
  actual_end     TIMESTAMPTZ,
  duration_minutes INTEGER,

  -- Impact
  trains_affected INTEGER DEFAULT 0,
  total_delay_minutes INTEGER DEFAULT 0,
  services_cancelled INTEGER DEFAULT 0,
  services_diverted INTEGER DEFAULT 0,

  -- Response
  mitigation_actions JSONB DEFAULT '[]',
  replacement_transport JSONB DEFAULT '[]',
  passenger_alerts JSONB DEFAULT '[]',

  -- Attribution
  cause_operator TEXT,
  root_cause     TEXT,
  lessons_learned TEXT,

  -- AI suggestions
  ai_suggestions JSONB DEFAULT '[]',

  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_db_operator ON rm_disruption_board(operator_id);
CREATE INDEX idx_rm_db_status ON rm_disruption_board(status) WHERE status IN ('active', 'mitigated');
CREATE INDEX idx_rm_db_severity ON rm_disruption_board(severity);
CREATE INDEX idx_rm_db_started ON rm_disruption_board(started_at);
```

### rm_daily_performance

```sql
CREATE TABLE rm_daily_performance (
  operator_id    UUID NOT NULL,
  performance_date DATE NOT NULL,

  -- Service metrics
  services_planned INTEGER DEFAULT 0,
  services_ran   INTEGER DEFAULT 0,
  services_cancelled INTEGER DEFAULT 0,
  services_part_cancelled INTEGER DEFAULT 0,
  services_diverted INTEGER DEFAULT 0,

  -- Punctuality
  ppm_pct        NUMERIC(5, 2),          -- Public Performance Measure
  right_time_pct NUMERIC(5, 2),
  avg_delay_minutes NUMERIC(6, 2),
  delays_by_cause JSONB DEFAULT '{}',    -- {"signal_failure": 45, "rolling_stock": 20, "weather": 15}
  delay_attribution JSONB DEFAULT '{}',  -- by responsible operator

  -- Disruptions
  disruptions_count INTEGER DEFAULT 0,
  disruption_minutes INTEGER DEFAULT 0,
  possessions_count INTEGER DEFAULT 0,

  -- Crew
  duties_planned INTEGER DEFAULT 0,
  duties_covered INTEGER DEFAULT 0,
  compliance_violations INTEGER DEFAULT 0,

  -- Maintenance
  inspections_completed INTEGER DEFAULT 0,
  defects_found INTEGER DEFAULT 0,
  defects_resolved INTEGER DEFAULT 0,
  predictive_alerts INTEGER DEFAULT 0,

  -- Rolling stock
  fleet_availability_pct NUMERIC(5, 2),
  units_in_service INTEGER DEFAULT 0,
  units_in_maintenance INTEGER DEFAULT 0,

  -- Passenger
  pax_estimated  INTEGER DEFAULT 0,

  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (operator_id, performance_date)
);

CREATE INDEX idx_rm_dp_operator ON rm_daily_performance(operator_id);
CREATE INDEX idx_rm_dp_date ON rm_daily_performance(performance_date);
CREATE INDEX idx_rm_dp_ppm ON rm_daily_performance(ppm_pct);
```

### rm_crew_roster

```sql
CREATE TABLE rm_crew_roster (
  crew_id        UUID NOT NULL,
  operator_id    UUID NOT NULL,
  duty_date      DATE NOT NULL,
  employee_number TEXT NOT NULL,
  name           TEXT NOT NULL,
  role           TEXT NOT NULL,
  home_depot     TEXT NOT NULL,

  -- Current duty
  duty_type      TEXT NOT NULL,
  train_runs     JSONB DEFAULT '[]',
  sign_on_station TEXT,
  sign_off_station TEXT,
  planned_start  TIMESTAMPTZ,
  planned_end    TIMESTAMPTZ,
  actual_start   TIMESTAMPTZ,
  actual_end     TIMESTAMPTZ,

  -- Hours tracking
  driving_hours  NUMERIC(5, 2) DEFAULT 0,
  total_hours    NUMERIC(5, 2) DEFAULT 0,
  hours_remaining_today NUMERIC(5, 2),
  is_compliant   BOOLEAN NOT NULL DEFAULT true,
  compliance_issues JSONB DEFAULT '[]',

  -- Availability for reassignment
  is_available_for_reassignment BOOLEAN NOT NULL DEFAULT false,
  traction_knowledge TEXT[] DEFAULT '{}',
  route_knowledge TEXT[] DEFAULT '{}',

  -- Disruption rostering
  is_disruption_roster BOOLEAN NOT NULL DEFAULT false,
  reassignment_history JSONB DEFAULT '[]',

  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (crew_id, duty_date)
);

CREATE INDEX idx_rm_cr_operator ON rm_crew_roster(operator_id);
CREATE INDEX idx_rm_cr_date ON rm_crew_roster(duty_date);
CREATE INDEX idx_rm_cr_role ON rm_crew_roster(role);
CREATE INDEX idx_rm_cr_available ON rm_crew_roster(is_available_for_reassignment)
  WHERE is_available_for_reassignment = true;
CREATE INDEX idx_rm_cr_compliant ON rm_crew_roster(is_compliant) WHERE is_compliant = false;
```

---

## Event Replay Examples

### Disruption timeline reconstruction

```sql
SELECT event_type, event_data, actor_type, ce_time
FROM event_store
WHERE stream_type = 'disruption'
  AND stream_id = $1
ORDER BY sequence_no ASC;
```

Returns the full lifecycle: started → mitigated → escalated → resolved → impact assessed → lessons learned, with every dispatcher decision timestamped.

### What-if disruption replay

```sql
-- Get all events during a disruption window for affected area
SELECT event_type, event_data, ce_time, stream_type, actor_type
FROM event_store
WHERE ce_time BETWEEN $1 AND $2  -- disruption window
  AND location_ref = ANY($3)     -- affected section codes
  AND stream_type IN ('train', 'disruption', 'crew', 'timetable')
ORDER BY ce_time ASC;
```

Feed to simulation engine with alternative dispatcher decisions to compare outcomes.

### Delay attribution chain

```sql
-- Trace delay propagation from root cause
WITH RECURSIVE delay_chain AS (
  SELECT id, event_type, event_data, ce_time, causation_id, stream_id
  FROM event_store
  WHERE id = $1  -- root delay event
  UNION ALL
  SELECT e.id, e.event_type, e.event_data, e.ce_time, e.causation_id, e.stream_id
  FROM event_store e
  JOIN delay_chain d ON e.causation_id = d.id
  WHERE e.event_type = 'train.delayed'
)
SELECT * FROM delay_chain ORDER BY ce_time ASC;
```

### Predictive maintenance training data

```sql
SELECT event_type, event_data, ce_time
FROM event_store
WHERE stream_type = 'rolling_stock'
  AND stream_id = $1  -- unit stream ID
  AND event_type IN ('unit.sensor_reading', 'unit.defect_reported', 'unit.exam_completed')
ORDER BY ce_time ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_network_status, rm_train_board, rm_disruption_board, rm_daily_performance, rm_crew_roster |
| **Total** | **8** | |

---

## Key Design Decisions

1. **Twelve stream types covering the full rail domain** — Network, infrastructure, rolling stock, timetable, train, crew, possession, maintenance, disruption, passenger info, AI, and config streams separate concerns while correlation_id links causally related events across streams.

2. **Signalling-system-aware actor types** — `interlocking`, `etcs_rbc`, `track_circuit`, `axle_counter`, `gsm_r`, and `frmcs` as first-class actor types track the provenance of machine-generated events from safety-critical systems, essential for EN 50126 RAMS evidence and IEC 62443 security audit.

3. **Location reference on events** — `location_ref` (station code or track section code) enables spatial querying of events: "show me all events on the Brighton Main Line in the last hour."

4. **Train lifecycle as an event sequence** — formed → departed → arrived → passed → delayed → diverted → completed captures the complete run lifecycle, enabling per-train performance analysis and delay propagation modelling.

5. **Crew events for compliance auditing** — Sign-on, sign-off, driving hours, and compliance violations as events create a tamper-evident record for working-time directive compliance.

6. **Possession lifecycle events** — planned → confirmed → started → overrun → completed captures the full possession lifecycle, enabling possession performance analysis (how often do possessions overrun?).

7. **Disruption-to-train causation** — Disruption events are linked to affected train events via correlation_id, enabling automatic delay attribution and cascade delay analysis.

8. **Passenger information as events** — Alert publication, timetable revision, and delay notifications are events, providing a complete record of what passengers were told and when — important for service quality reviews.

9. **rm_train_board as the dispatcher's read model** — Pre-computed current position, delay, path with actual times, and event timeline enable the control room display without querying the event store.

10. **rm_daily_performance for regulatory reporting** — PPM, right-time percentage, delay attribution by cause and operator, and fleet availability are pre-computed daily, supporting performance regime reporting.

11. **rm_crew_roster with reassignment availability** — Pre-computed hours remaining, traction/route knowledge, and availability flag enable rapid crew reassignment during disruptions.

12. **Climate-resilience disruption types** — Weather events (heat, flood, wind, snow, leaf fall, landslip) as first-class disruption types feed climate resilience analysis and scenario planning models.
