# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Rail Network Operations · Created: 2026-05-26

## Philosophy

Rail networks are inherently graph-structured: stations connected by track sections, trains traversing paths through the graph, crews assigned to duties over those paths, and possessions blocking sections for maintenance. A normalized relational model makes this topology explicit through foreign keys, enabling the join-heavy queries that power conflict detection ("does train path A overlap in time and space with possession B on track section C?") and capacity analysis ("how many trains traverse section X in time window Y?").

Each domain concept — station, track section, rolling stock, train service, crew member, possession, disruption — gets its own table with constraints enforcing domain rules. The infrastructure topology follows railTopoModel (IRS 30100) conventions with track sections as directed edges between nodes (stations, junctions), enabling UIC 406-style capacity utilisation calculations as SQL queries over the graph. Timetable data aligns with RailML 3.x entity structure, and passenger-facing output targets GTFS export.

This architecture prioritises data integrity for safety-adjacent operations where a scheduling conflict or missed possession can have serious consequences. Every relationship is enforced at the database level, and the complete state is always queryable without replaying logs.

**Best for:** Infrastructure managers and large train operators needing rigorous timetabling, capacity analysis, and multi-operator coordination with full auditability.

**Trade-offs:**
- Pro: Full referential integrity across network topology, timetable, crew, and maintenance
- Pro: RailML / railTopoModel alignment enables standards-based import/export
- Pro: Conflict detection queries are natural joins across track_sections and train paths
- Con: 14 tables with complex spatial-temporal joins for capacity analysis
- Con: Real-time position data requires partitioning discipline
- Con: Schema changes needed when new infrastructure element types are added

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RailML 3.x | Infrastructure, timetable, and rolling stock entity naming and structure |
| railTopoModel (IRS 30100) | Track section topology as directed edges between nodes |
| UIC 406 | Capacity utilisation calculations over track sections |
| GTFS | Station/stop fields for passenger information export |
| GTFS-RT | Real-time position and delay feed structure |
| NeTEx (CEN TS 16614) | Timetable and network data exchange alignment |
| SIRI (CEN TS 15531) | Real-time service information structure |
| TAF/TAP TSI | Cross-operator freight and passenger data exchange |
| ERTMS/ETCS | Signalling system fields on track sections |
| EN 50126 | RAMS fields on rolling stock and infrastructure |
| IEC 61375 | Onboard telemetry source for rolling stock sensor data |
| ISO 55000 | Asset management fields on infrastructure and rolling stock |
| ISO 8601 | All timestamps and durations |
| IEC 62443 / NIS2 | Security classification on audit trail |
| GDPR | Crew personal data handling |
| AsyncAPI 3.0 | Event-driven train position streaming |
| MCP | Exposing operational data to AI assistants |

---

## Core Tables

### operators

```sql
CREATE TABLE operators (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          TEXT NOT NULL,
  operator_type TEXT NOT NULL CHECK (operator_type IN (
                  'infrastructure_manager', 'passenger_toc', 'freight_operator',
                  'open_access', 'concession', 'regional_authority'
                )),
  country_code  CHAR(2) NOT NULL,        -- ISO 3166-1
  uic_company_code TEXT,                 -- UIC company code (RICS)
  era_entity_id TEXT,                    -- ERA entity reference
  timezone      TEXT NOT NULL DEFAULT 'UTC',
  settings_json JSONB DEFAULT '{}',
  is_active     BOOLEAN NOT NULL DEFAULT true,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### stations

```sql
CREATE TABLE stations (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  name          TEXT NOT NULL,
  station_code  TEXT NOT NULL,            -- national station code (e.g., NLC, DS100)
  ibnr          TEXT,                    -- International station number (UIC/IBNR)
  gtfs_stop_id  TEXT,                    -- GTFS stop_id for export
  station_type  TEXT NOT NULL CHECK (station_type IN (
                  'mainline', 'regional', 'suburban', 'metro', 'freight_terminal',
                  'junction', 'depot', 'maintenance_facility', 'halt'
                )),
  latitude      NUMERIC(10, 7) NOT NULL,
  longitude     NUMERIC(10, 7) NOT NULL,
  elevation_m   NUMERIC(8, 2),
  platform_count INTEGER,
  platforms_json JSONB DEFAULT '[]',     -- [{number, length_m, height_mm, track_ids, accessibility}]
  facilities_json JSONB DEFAULT '{}',   -- {ticket_office, waiting_room, parking, bike_storage, accessibility}
  timezone      TEXT NOT NULL DEFAULT 'UTC',
  country_code  CHAR(2) NOT NULL,
  region        TEXT,
  is_active     BOOLEAN NOT NULL DEFAULT true,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stations_operator ON stations(operator_id);
CREATE INDEX idx_stations_code ON stations(station_code);
CREATE INDEX idx_stations_ibnr ON stations(ibnr) WHERE ibnr IS NOT NULL;
CREATE INDEX idx_stations_type ON stations(station_type);
CREATE INDEX idx_stations_gtfs ON stations(gtfs_stop_id) WHERE gtfs_stop_id IS NOT NULL;
```

### track_sections

```sql
CREATE TABLE track_sections (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  name          TEXT NOT NULL,
  section_code  TEXT NOT NULL,
  from_station_id UUID NOT NULL REFERENCES stations(id),
  to_station_id UUID NOT NULL REFERENCES stations(id),
  length_km     NUMERIC(8, 3) NOT NULL,
  track_count   INTEGER NOT NULL DEFAULT 1,
  max_speed_kph INTEGER NOT NULL,
  line_category TEXT,                    -- UIC line category
  electrification TEXT CHECK (electrification IN (
                  'none', 'ac_25kv', 'ac_15kv', 'dc_3kv', 'dc_1500v',
                  'dc_750v_third_rail', 'dc_750v_fourth_rail', 'dual'
                )),
  gauge_mm      INTEGER DEFAULT 1435,    -- standard gauge
  signalling_system TEXT CHECK (signalling_system IN (
                  'etcs_l1', 'etcs_l2', 'etcs_l3', 'national_atp',
                  'ptc', 'abs', 'dark_territory', 'token', 'axle_counter'
                )),
  gradient_profile JSONB DEFAULT '[]',   -- [{km_start, km_end, gradient_permille}]
  capacity_trains_per_hour INTEGER,      -- UIC 406 theoretical capacity
  status        TEXT NOT NULL DEFAULT 'operational' CHECK (status IN (
                  'operational', 'degraded', 'closed', 'under_construction',
                  'possession', 'decommissioned'
                )),
  speed_restrictions JSONB DEFAULT '[]', -- [{km_start, km_end, speed_kph, reason, valid_until}]
  asset_condition TEXT CHECK (asset_condition IN ('good', 'fair', 'poor', 'critical')),
  last_renewal_date DATE,
  next_renewal_due DATE,
  geometry_geojson JSONB,                -- GeoJSON LineString
  is_active     BOOLEAN NOT NULL DEFAULT true,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tracks_operator ON track_sections(operator_id);
CREATE INDEX idx_tracks_from ON track_sections(from_station_id);
CREATE INDEX idx_tracks_to ON track_sections(to_station_id);
CREATE INDEX idx_tracks_status ON track_sections(status);
CREATE INDEX idx_tracks_signalling ON track_sections(signalling_system);
CREATE INDEX idx_tracks_condition ON track_sections(asset_condition) WHERE asset_condition IN ('poor', 'critical');
```

### rolling_stock

```sql
CREATE TABLE rolling_stock (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  unit_number   TEXT NOT NULL,            -- fleet number
  vehicle_type  TEXT NOT NULL CHECK (vehicle_type IN (
                  'emu', 'dmu', 'locomotive', 'coach', 'hst',
                  'freight_wagon', 'multiple_unit', 'railcar', 'shunter'
                )),
  class_designation TEXT NOT NULL,        -- e.g., 'Class 390', 'ICE 4', 'TGV-M'
  manufacturer  TEXT,
  build_year    INTEGER,
  max_speed_kph INTEGER NOT NULL,
  traction      TEXT CHECK (traction IN ('electric', 'diesel', 'bi_mode', 'battery', 'hydrogen')),
  gauge_mm      INTEGER DEFAULT 1435,
  seats_first   INTEGER DEFAULT 0,
  seats_standard INTEGER DEFAULT 0,
  standing_capacity INTEGER DEFAULT 0,
  length_m      NUMERIC(8, 2),
  weight_tonnes NUMERIC(8, 2),
  axle_load_tonnes NUMERIC(6, 2),
  etcs_equipped BOOLEAN NOT NULL DEFAULT false,
  etcs_baseline TEXT,                    -- 'B2', 'B3R1', 'B3R2'
  home_depot_id UUID REFERENCES stations(id),
  status        TEXT NOT NULL DEFAULT 'available' CHECK (status IN (
                  'available', 'in_service', 'maintenance', 'overhaul',
                  'defective', 'stored', 'withdrawn'
                )),
  mileage_km    NUMERIC(12, 1) DEFAULT 0,
  next_exam_due_at TIMESTAMPTZ,
  next_exam_type TEXT,
  rams_data_json JSONB DEFAULT '{}',     -- EN 50126 RAMS metrics: {mtbf_hours, mttr_hours, availability_pct}
  sensor_config_json JSONB DEFAULT '{}', -- IEC 61375 onboard sensor mapping
  specs_json    JSONB DEFAULT '{}',
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rs_operator ON rolling_stock(operator_id);
CREATE INDEX idx_rs_type ON rolling_stock(vehicle_type);
CREATE INDEX idx_rs_class ON rolling_stock(class_designation);
CREATE INDEX idx_rs_status ON rolling_stock(status);
CREATE INDEX idx_rs_depot ON rolling_stock(home_depot_id);
CREATE INDEX idx_rs_exam ON rolling_stock(next_exam_due_at) WHERE next_exam_due_at IS NOT NULL;
```

### train_services

```sql
CREATE TABLE train_services (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  service_code  TEXT NOT NULL,            -- headcode or train number pattern
  service_type  TEXT NOT NULL CHECK (service_type IN (
                  'express', 'regional', 'suburban', 'intercity', 'high_speed',
                  'freight', 'charter', 'empty_stock', 'engineering',
                  'light_locomotive'
                )),
  origin_id     UUID NOT NULL REFERENCES stations(id),
  destination_id UUID NOT NULL REFERENCES stations(id),
  via_stations  UUID[] DEFAULT '{}',
  days_of_operation TEXT[] DEFAULT '{}', -- ['monday', 'tuesday', ...] or GTFS calendar_id
  valid_from    DATE NOT NULL,
  valid_until   DATE,
  timetable_version TEXT,
  category      TEXT CHECK (category IN ('scheduled', 'additional', 'diverted', 'cancelled')),
  priority      INTEGER DEFAULT 1,       -- path priority for conflict resolution
  is_active     BOOLEAN NOT NULL DEFAULT true,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ts_operator ON train_services(operator_id);
CREATE INDEX idx_ts_code ON train_services(service_code);
CREATE INDEX idx_ts_type ON train_services(service_type);
CREATE INDEX idx_ts_origin ON train_services(origin_id);
CREATE INDEX idx_ts_dest ON train_services(destination_id);
CREATE INDEX idx_ts_valid ON train_services(valid_from, valid_until);
```

### train_runs

```sql
CREATE TABLE train_runs (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  service_id    UUID NOT NULL REFERENCES train_services(id),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  run_date      DATE NOT NULL,
  train_number  TEXT NOT NULL,            -- actual headcode for this run
  rolling_stock_id UUID REFERENCES rolling_stock(id),
  consist_json  JSONB DEFAULT '[]',      -- [{unit_number, position, orientation}]
  status        TEXT NOT NULL DEFAULT 'planned' CHECK (status IN (
                  'planned', 'confirmed', 'running', 'completed',
                  'cancelled', 'part_cancelled', 'diverted'
                )),
  origin_id     UUID NOT NULL REFERENCES stations(id),
  destination_id UUID NOT NULL REFERENCES stations(id),
  scheduled_departure TIMESTAMPTZ NOT NULL,
  scheduled_arrival TIMESTAMPTZ NOT NULL,
  actual_departure TIMESTAMPTZ,
  actual_arrival TIMESTAMPTZ,
  delay_minutes INTEGER DEFAULT 0,
  delay_reason_code TEXT,                -- industry delay attribution code
  delay_cause_operator UUID REFERENCES operators(id),
  cancellation_reason TEXT,
  path_segments JSONB NOT NULL DEFAULT '[]',
  -- [{
  --   "sequence": 1,
  --   "station_id": "...",
  --   "station_code": "PAD",
  --   "platform": "1",
  --   "track_section_id": "...",
  --   "scheduled_arrival": null,
  --   "scheduled_departure": "2026-06-01T08:00:00Z",
  --   "actual_arrival": null,
  --   "actual_departure": "2026-06-01T08:02:00Z",
  --   "dwell_seconds": 60,
  --   "stopping_pattern": "stop"
  -- }]
  capacity_json JSONB DEFAULT '{}',      -- {first: 120, standard: 450, standing: 200}
  load_factor_pct NUMERIC(5, 2),
  pax_count_estimated INTEGER,
  gtfs_trip_id  TEXT,                    -- for GTFS export
  notes         TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (service_id, run_date)
);

CREATE INDEX idx_runs_service ON train_runs(service_id);
CREATE INDEX idx_runs_date ON train_runs(run_date);
CREATE INDEX idx_runs_operator ON train_runs(operator_id);
CREATE INDEX idx_runs_status ON train_runs(status);
CREATE INDEX idx_runs_rs ON train_runs(rolling_stock_id);
CREATE INDEX idx_runs_delay ON train_runs(delay_minutes) WHERE delay_minutes > 0;
CREATE INDEX idx_runs_gtfs ON train_runs(gtfs_trip_id) WHERE gtfs_trip_id IS NOT NULL;
```

### crew_members

```sql
CREATE TABLE crew_members (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  employee_number TEXT NOT NULL,
  name          TEXT NOT NULL,
  role          TEXT NOT NULL CHECK (role IN (
                  'driver', 'conductor', 'guard', 'dispatcher',
                  'shunter', 'maintenance_technician', 'train_manager'
                )),
  home_depot_id UUID REFERENCES stations(id),
  traction_knowledge TEXT[] DEFAULT '{}', -- ['class_390', 'class_350', 'emu_general']
  route_knowledge TEXT[] DEFAULT '{}',    -- ['wcml_euston_crewe', 'ecml_kings_cross_york']
  certifications_json JSONB DEFAULT '[]', -- [{cert, issued, expires, authority}]
  contract_hours_weekly NUMERIC(5, 2),
  max_driving_hours_daily NUMERIC(5, 2),
  rest_rules_json JSONB DEFAULT '{}',    -- jurisdiction-specific rest requirements
  is_active     BOOLEAN NOT NULL DEFAULT true,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (operator_id, employee_number)
);

CREATE INDEX idx_crew_operator ON crew_members(operator_id);
CREATE INDEX idx_crew_role ON crew_members(role);
CREATE INDEX idx_crew_depot ON crew_members(home_depot_id);
CREATE INDEX idx_crew_traction ON crew_members USING GIN (traction_knowledge);
CREATE INDEX idx_crew_route ON crew_members USING GIN (route_knowledge);
```

### crew_duties

```sql
CREATE TABLE crew_duties (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  crew_member_id UUID NOT NULL REFERENCES crew_members(id),
  train_run_id  UUID REFERENCES train_runs(id),
  duty_date     DATE NOT NULL,
  duty_type     TEXT NOT NULL CHECK (duty_type IN (
                  'driving', 'conducting', 'shunting', 'standby',
                  'training', 'rest', 'annual_leave', 'sick'
                )),
  sign_on_station_id UUID REFERENCES stations(id),
  sign_off_station_id UUID REFERENCES stations(id),
  planned_start TIMESTAMPTZ NOT NULL,
  planned_end   TIMESTAMPTZ NOT NULL,
  actual_start  TIMESTAMPTZ,
  actual_end    TIMESTAMPTZ,
  driving_hours NUMERIC(5, 2) DEFAULT 0,
  total_hours   NUMERIC(5, 2) DEFAULT 0,
  is_compliant  BOOLEAN NOT NULL DEFAULT true,
  compliance_violations JSONB DEFAULT '[]', -- [{rule, description}]
  is_disruption_roster BOOLEAN NOT NULL DEFAULT false,
  notes         TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_duties_crew ON crew_duties(crew_member_id);
CREATE INDEX idx_duties_run ON crew_duties(train_run_id);
CREATE INDEX idx_duties_date ON crew_duties(duty_date);
CREATE INDEX idx_duties_type ON crew_duties(duty_type);
CREATE INDEX idx_duties_compliant ON crew_duties(is_compliant) WHERE is_compliant = false;
```

### possessions

```sql
CREATE TABLE possessions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  track_section_ids UUID[] NOT NULL,     -- affected track sections
  station_ids   UUID[] DEFAULT '{}',     -- affected stations
  possession_type TEXT NOT NULL CHECK (possession_type IN (
                  'total_blockade', 'single_line_working', 'speed_restriction',
                  'platform_closure', 'signal_possession', 'overhead_line',
                  'track_renewal', 'ballast', 'drainage'
                )),
  title         TEXT NOT NULL,
  description   TEXT,
  planned_start TIMESTAMPTZ NOT NULL,
  planned_end   TIMESTAMPTZ NOT NULL,
  actual_start  TIMESTAMPTZ,
  actual_end    TIMESTAMPTZ,
  status        TEXT NOT NULL DEFAULT 'planned' CHECK (status IN (
                  'planned', 'confirmed', 'active', 'overrun',
                  'completed', 'cancelled'
                )),
  impact_level  TEXT CHECK (impact_level IN ('none', 'minor', 'moderate', 'major', 'total')),
  affected_services JSONB DEFAULT '[]', -- [{service_id, impact: "cancelled"|"diverted"|"delayed"}]
  alternative_transport JSONB DEFAULT '[]', -- [{mode: "bus"|"taxi", provider, route}]
  work_orders   UUID[] DEFAULT '{}',
  approved_by   UUID,
  approved_at   TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_poss_operator ON possessions(operator_id);
CREATE INDEX idx_poss_tracks ON possessions USING GIN (track_section_ids);
CREATE INDEX idx_poss_status ON possessions(status);
CREATE INDEX idx_poss_planned ON possessions(planned_start, planned_end);
CREATE INDEX idx_poss_active ON possessions(status) WHERE status IN ('active', 'overrun');
```

### maintenance_records

```sql
CREATE TABLE maintenance_records (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  record_type   TEXT NOT NULL CHECK (record_type IN (
                  'inspection', 'defect', 'work_order', 'exam',
                  'overhaul', 'modification', 'sensor_alert'
                )),
  asset_type    TEXT NOT NULL CHECK (asset_type IN (
                  'track', 'signal', 'overhead_line', 'structure',
                  'station', 'rolling_stock', 'points', 'level_crossing'
                )),
  asset_id      UUID,                    -- references rolling_stock, track_sections, or stations
  asset_reference TEXT,                  -- human-readable asset ID
  track_section_id UUID REFERENCES track_sections(id),
  rolling_stock_id UUID REFERENCES rolling_stock(id),
  station_id    UUID REFERENCES stations(id),
  priority      TEXT NOT NULL CHECK (priority IN ('safety_critical', 'urgent', 'planned', 'routine')),
  status        TEXT NOT NULL DEFAULT 'open' CHECK (status IN (
                  'open', 'assigned', 'in_progress', 'completed',
                  'deferred', 'cancelled'
                )),
  title         TEXT NOT NULL,
  description   TEXT,
  defect_code   TEXT,                    -- industry defect classification
  location_km   NUMERIC(8, 3),           -- track chainage for infrastructure defects
  reported_by   UUID,
  assigned_to   UUID,
  planned_start TIMESTAMPTZ,
  planned_end   TIMESTAMPTZ,
  actual_start  TIMESTAMPTZ,
  actual_end    TIMESTAMPTZ,
  linked_possession_id UUID REFERENCES possessions(id),
  parts_used    JSONB DEFAULT '[]',
  labour_hours  NUMERIC(8, 2) DEFAULT 0,
  cost_cents    BIGINT DEFAULT 0,
  photos        TEXT[] DEFAULT '{}',
  sensor_data_json JSONB,               -- triggering sensor data for predictive maintenance
  is_predictive BOOLEAN NOT NULL DEFAULT false,
  ai_confidence NUMERIC(5, 4),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_maint_operator ON maintenance_records(operator_id);
CREATE INDEX idx_maint_type ON maintenance_records(record_type);
CREATE INDEX idx_maint_asset ON maintenance_records(asset_type);
CREATE INDEX idx_maint_track ON maintenance_records(track_section_id);
CREATE INDEX idx_maint_rs ON maintenance_records(rolling_stock_id);
CREATE INDEX idx_maint_status ON maintenance_records(status);
CREATE INDEX idx_maint_priority ON maintenance_records(priority) WHERE priority IN ('safety_critical', 'urgent');
CREATE INDEX idx_maint_predictive ON maintenance_records(is_predictive) WHERE is_predictive = true;
```

### disruptions

```sql
CREATE TABLE disruptions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  disruption_type TEXT NOT NULL CHECK (disruption_type IN (
                  'infrastructure_failure', 'rolling_stock_failure', 'signal_failure',
                  'overhead_line', 'points_failure', 'trespass', 'fatality',
                  'weather_heat', 'weather_flood', 'weather_wind', 'weather_snow',
                  'weather_leaf_fall', 'landslip', 'level_crossing_incident',
                  'strike', 'staff_shortage', 'congestion', 'other'
                )),
  severity      TEXT NOT NULL CHECK (severity IN ('minor', 'moderate', 'major', 'severe')),
  title         TEXT NOT NULL,
  description   TEXT,
  affected_track_sections UUID[] DEFAULT '{}',
  affected_stations UUID[] DEFAULT '{}',
  started_at    TIMESTAMPTZ NOT NULL,
  estimated_end TIMESTAMPTZ,
  actual_end    TIMESTAMPTZ,
  status        TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                  'active', 'mitigated', 'resolved', 'closed'
                )),
  trains_affected INTEGER DEFAULT 0,
  total_delay_minutes INTEGER DEFAULT 0,
  services_cancelled INTEGER DEFAULT 0,
  services_diverted INTEGER DEFAULT 0,
  mitigation_actions JSONB DEFAULT '[]', -- [{action, decision_by, decided_at}]
  passenger_information JSONB DEFAULT '{}', -- {message, channels: ["siri", "gtfs_rt", "twitter"]}
  cause_operator_id UUID REFERENCES operators(id),
  root_cause    TEXT,
  lessons_learned TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_disr_operator ON disruptions(operator_id);
CREATE INDEX idx_disr_type ON disruptions(disruption_type);
CREATE INDEX idx_disr_severity ON disruptions(severity);
CREATE INDEX idx_disr_status ON disruptions(status);
CREATE INDEX idx_disr_started ON disruptions(started_at);
CREATE INDEX idx_disr_tracks ON disruptions USING GIN (affected_track_sections);
CREATE INDEX idx_disr_active ON disruptions(status) WHERE status IN ('active', 'mitigated');
```

### train_positions

```sql
CREATE TABLE train_positions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  train_run_id  UUID NOT NULL REFERENCES train_runs(id),
  operator_id   UUID NOT NULL,
  track_section_id UUID REFERENCES track_sections(id),
  latitude      NUMERIC(10, 7),
  longitude     NUMERIC(10, 7),
  speed_kph     NUMERIC(6, 1),
  heading_degrees NUMERIC(5, 1),
  delay_seconds INTEGER DEFAULT 0,
  next_station_id UUID REFERENCES stations(id),
  distance_to_next_km NUMERIC(8, 3),
  signal_aspect TEXT,                    -- current signal aspect
  source        TEXT NOT NULL CHECK (source IN (
                  'gps', 'track_circuit', 'axle_counter', 'balise',
                  'gsm_r', 'frmcs', 'estimated', 'manual'
                )),
  recorded_at   TIMESTAMPTZ NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_pos_run ON train_positions(train_run_id);
CREATE INDEX idx_pos_operator ON train_positions(operator_id);
CREATE INDEX idx_pos_section ON train_positions(track_section_id);
CREATE INDEX idx_pos_recorded ON train_positions(recorded_at);
CREATE INDEX idx_pos_delay ON train_positions(delay_seconds) WHERE delay_seconds > 300;
```

### ai_suggestions

```sql
CREATE TABLE ai_suggestions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL,
  suggestion_type TEXT NOT NULL CHECK (suggestion_type IN (
                  'conflict_resolution', 'repathing', 'retiming',
                  'crew_reassignment', 'rolling_stock_swap',
                  'demand_forecast', 'predictive_maintenance',
                  'disruption_scenario', 'possession_optimisation',
                  'incident_report_draft', 'natural_language_query',
                  'climate_resilience'
                )),
  target_type   TEXT NOT NULL,
  target_id     UUID NOT NULL,
  model_id      TEXT NOT NULL,
  model_version TEXT,
  input_json    JSONB NOT NULL,
  output_json   JSONB NOT NULL,
  confidence    NUMERIC(5, 4),
  status        TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                  'pending', 'accepted', 'rejected', 'auto_applied', 'expired'
                )),
  reviewed_by   UUID,
  reviewed_at   TIMESTAMPTZ,
  feedback_json JSONB,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_operator ON ai_suggestions(operator_id);
CREATE INDEX idx_ai_type ON ai_suggestions(suggestion_type);
CREATE INDEX idx_ai_target ON ai_suggestions(target_type, target_id);
CREATE INDEX idx_ai_status ON ai_suggestions(status);
```

### audit_log

```sql
CREATE TABLE audit_log (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL,
  actor_id      UUID,
  actor_type    TEXT NOT NULL CHECK (actor_type IN (
                  'user', 'system', 'ai', 'scada', 'interlocking',
                  'tms', 'pss_feed', 'gtfs_publisher', 'scheduler'
                )),
  action        TEXT NOT NULL,
  resource_type TEXT NOT NULL,
  resource_id   UUID NOT NULL,
  changes_json  JSONB,
  ip_address    INET,
  session_id    TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_operator ON audit_log(operator_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_id);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Operator & Infrastructure | 3 | operators, stations (GTFS/NeTEx aligned), track_sections (railTopoModel) |
| Rolling Stock | 1 | With EN 50126 RAMS, ETCS, IEC 61375 sensor config |
| Timetable | 2 | train_services (patterns), train_runs (individual runs with path segments) |
| Crew | 2 | crew_members (with route/traction knowledge), crew_duties (rostered duties) |
| Maintenance & Possessions | 2 | possessions (track access windows), maintenance_records |
| Operations | 2 | disruptions, train_positions (partitioned real-time) |
| AI | 1 | ai_suggestions (12 suggestion types) |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **14** | |

---

## Key Design Decisions

1. **Track sections as directed edges** — `from_station_id` → `to_station_id` models the network as a directed graph aligned with railTopoModel (IRS 30100), enabling path-finding, conflict detection, and UIC 406 capacity queries via recursive CTEs.

2. **Signalling system on track sections** — ETCS level and national ATP system are explicit fields, enabling simulation engines to apply correct braking curves and movement authorities per section.

3. **Path segments as JSONB on train_runs** — Individual stopping patterns with per-station actual times embedded in the run row, since the path is always queried as a unit. Avoids a separate junction table with one row per stop.

4. **Dual delay attribution** — `delay_reason_code` and `delay_cause_operator` support the multi-operator delay attribution workflow where an infrastructure manager's failure causes a train operator's delay.

5. **Crew route and traction knowledge as arrays** — GIN-indexed TEXT[] arrays enable efficient queries: "find all drivers qualified on Class 390 who know the WCML route and are available on Tuesday."

6. **Possessions linked to track sections** — UUID array of affected track sections enables conflict detection: "does this train path traverse any section under possession during this window?"

7. **Maintenance covers both infrastructure and rolling stock** — A single `maintenance_records` table with `asset_type` discriminator handles track defects, signal faults, and rolling stock exams without duplicating workflow logic.

8. **Predictive maintenance flag** — `is_predictive` and `ai_confidence` distinguish AI-triggered maintenance from scheduled/reactive, enabling ROI tracking for predictive maintenance models.

9. **Climate-resilience disruption types** — `weather_heat`, `weather_flood`, `weather_leaf_fall`, and `landslip` as first-class disruption types support climate resilience scenario planning.

10. **Train position source tracking** — GPS, track circuit, axle counter, balise, GSM-R, and FRMCS as position sources reflect the heterogeneous location systems in rail and track data provenance.

11. **GTFS export fields** — `gtfs_stop_id` on stations and `gtfs_trip_id` on train_runs enable direct GTFS feed generation for passenger information systems.

12. **Possession impact tracking** — `affected_services` JSONB on possessions pre-computes the impact on train services (cancelled, diverted, delayed), supporting the possession planning workflow.
