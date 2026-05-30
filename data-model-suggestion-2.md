# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Rail Network Operations · Created: 2026-05-26

## Philosophy

Rail operations have two natural aggregates that dominate day-to-day workflows: the **network** (infrastructure topology with current state) and the **service day** (everything that happened or is planned for a specific date across the network). A dispatcher's primary view is "what is the state of the network right now?" — which infrastructure is available, which possessions are active, which disruptions are ongoing. A planner's primary view is "what does the timetable look like on this date?" — services, paths, crew assignments, rolling stock allocations.

By embedding related data into these aggregates as JSONB, the platform renders a complete network overview or daily service plan with minimal joins. Track sections embed their current speed restrictions, possessions, and condition. Train services embed their path, crew, rolling stock, and real-time status. This matters for the web-based timetable editor, which must load and manipulate complex service patterns without waiting for multi-table joins across infrastructure, timetable, crew, and rolling stock tables.

The trade-off is managed denormalisation: infrastructure state appears both on the track section aggregate and referenced from possessions and disruptions. But rail operations are overwhelmingly read-heavy — dispatchers and planners query constantly, while infrastructure changes are infrequent — so optimising the read path justifies the write-path complexity.

**Best for:** Smaller operators and consultancies needing a fast-to-deploy, web-first timetable editor and operations dashboard with maximum flexibility for different network configurations.

**Trade-offs:**
- Pro: Network overview and daily service plan as single-aggregate reads
- Pro: 6 tables vs. 14 — simpler deployment, backup, and schema management
- Pro: New infrastructure element types or crew rules added without migrations
- Con: Cross-service analytics require JSONB extraction
- Con: Infrastructure state consistency between track_sections and referencing entities must be maintained by application
- Con: Large JSONB blobs on busy service days with hundreds of train runs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RailML 3.x | Infrastructure and timetable entity naming in JSONB structures |
| railTopoModel (IRS 30100) | Topology encoded in track_sections connectivity_json |
| UIC 406 | Capacity fields pre-computed on track_sections |
| GTFS | Station codes and trip IDs embedded for export |
| GTFS-RT | Real-time delay and position data embedded on train services |
| NeTEx / SIRI | Data exchange alignment for passenger information |
| ERTMS/ETCS | Signalling system and ETCS baseline embedded on track sections |
| EN 50126 | RAMS metrics embedded on rolling stock in service_day |
| IEC 61375 | Sensor config for rolling stock telemetry |
| ISO 55000 | Asset condition and renewal tracking on track sections |
| MCP | Aggregates exposed via MCP for AI assistant queries |

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
  country_code  CHAR(2) NOT NULL,
  uic_company_code TEXT,
  timezone      TEXT NOT NULL DEFAULT 'UTC',

  settings_json JSONB DEFAULT '{}',
  -- {
  --   "crew_rest_rules": {"min_rest_hours": 12, "max_driving_hours": 8},
  --   "delay_threshold_minutes": 5,
  --   "gtfs_agency_id": "LNER",
  --   "siri_participant_ref": "LNER"
  -- }

  crew_json     JSONB DEFAULT '[]',
  -- [
  --   {
  --     "crew_id": "...",
  --     "employee_number": "E1234",
  --     "name": "J. Smith",
  --     "role": "driver",
  --     "home_depot": "Kings Cross",
  --     "traction_knowledge": ["class_800", "class_91"],
  --     "route_knowledge": ["ecml_kgx_york", "ecml_york_edinburgh"],
  --     "certifications": [{"cert": "ptsrules", "expires": "2027-03-15"}],
  --     "max_driving_hours": 8,
  --     "is_active": true
  --   }
  -- ]

  rolling_stock_json JSONB DEFAULT '[]',
  -- [
  --   {
  --     "rs_id": "...",
  --     "unit_number": "800101",
  --     "class": "Class 800",
  --     "type": "emu",
  --     "traction": "bi_mode",
  --     "max_speed_kph": 201,
  --     "seats_first": 42,
  --     "seats_standard": 379,
  --     "etcs_equipped": true,
  --     "etcs_baseline": "B3R1",
  --     "status": "available",
  --     "mileage_km": 450000,
  --     "next_exam": {"type": "C6", "due_at": "2026-07-15"},
  --     "home_depot": "Bounds Green",
  --     "rams": {"mtbf_hours": 12000, "availability_pct": 97.2}
  --   }
  -- ]

  is_active     BOOLEAN NOT NULL DEFAULT true,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_operators_type ON operators(operator_type);
CREATE INDEX idx_operators_crew ON operators USING GIN (crew_json);
CREATE INDEX idx_operators_rs ON operators USING GIN (rolling_stock_json);
```

### track_sections

Infrastructure aggregate — each track section carries its full current state.

```sql
CREATE TABLE track_sections (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  name          TEXT NOT NULL,
  section_code  TEXT NOT NULL,

  -- Topology (railTopoModel)
  from_station_code TEXT NOT NULL,
  from_station_name TEXT NOT NULL,
  to_station_code TEXT NOT NULL,
  to_station_name TEXT NOT NULL,
  length_km     NUMERIC(8, 3) NOT NULL,
  track_count   INTEGER NOT NULL DEFAULT 1,
  is_active     BOOLEAN NOT NULL DEFAULT true,

  -- Physical characteristics
  characteristics_json JSONB NOT NULL DEFAULT '{}',
  -- {
  --   "max_speed_kph": 200,
  --   "electrification": "ac_25kv",
  --   "gauge_mm": 1435,
  --   "signalling_system": "etcs_l2",
  --   "etcs_baseline": "B3R1",
  --   "line_category": "UIC_D4",
  --   "gradient_profile": [{"km": 0, "permille": 0}, {"km": 5.2, "permille": 12}],
  --   "geometry_geojson": {"type": "LineString", "coordinates": [...]},
  --   "axle_load_limit_tonnes": 25
  -- }

  -- Current operational state
  state_json    JSONB NOT NULL DEFAULT '{}',
  -- {
  --   "status": "operational",
  --   "capacity_trains_per_hour": 24,
  --   "current_utilisation_pct": 68,
  --   "speed_restrictions": [
  --     {"km_start": 12.5, "km_end": 14.0, "speed_kph": 40, "reason": "track_defect", "until": "2026-06-15"}
  --   ],
  --   "active_possessions": [],
  --   "active_disruptions": []
  -- }

  -- Asset condition (ISO 55000)
  condition_json JSONB DEFAULT '{}',
  -- {
  --   "overall": "good",
  --   "track": "good",
  --   "signalling": "fair",
  --   "overhead_line": "good",
  --   "last_inspection": {"date": "2026-04-20", "type": "track_geometry", "result": "pass"},
  --   "last_renewal": "2020-03-15",
  --   "next_renewal_due": "2035-03-15",
  --   "defects_open": 0,
  --   "maintenance_cost_ytd_cents": 1500000
  -- }

  -- Connectivity for graph traversal
  connectivity_json JSONB DEFAULT '{}',
  -- {
  --   "from_connections": ["section_id_1", "section_id_2"],
  --   "to_connections": ["section_id_3"],
  --   "junctions": [{"name": "Hitchin Jn", "diverging_sections": ["section_id_4"]}]
  -- }

  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ts_operator ON track_sections(operator_id);
CREATE INDEX idx_ts_from ON track_sections(from_station_code);
CREATE INDEX idx_ts_to ON track_sections(to_station_code);
CREATE INDEX idx_ts_state ON track_sections USING GIN (state_json);
CREATE INDEX idx_ts_condition ON track_sections USING GIN (condition_json);
```

### service_days

Daily aggregate — everything about train operations on a specific date for an operator.

```sql
CREATE TABLE service_days (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  operator_id   UUID NOT NULL REFERENCES operators(id),
  service_date  DATE NOT NULL,
  status        TEXT NOT NULL DEFAULT 'planning' CHECK (status IN (
                  'planning', 'confirmed', 'operational', 'completed', 'reviewed'
                )),

  -- Train services for this day
  services_json JSONB NOT NULL DEFAULT '[]',
  -- [
  --   {
  --     "run_id": "...",
  --     "service_code": "1A23",
  --     "service_type": "intercity",
  --     "origin": "KGX",
  --     "destination": "EDH",
  --     "scheduled_departure": "2026-06-01T08:00:00Z",
  --     "scheduled_arrival": "2026-06-01T12:22:00Z",
  --     "status": "running",
  --     "rolling_stock": {"unit": "800101", "class": "Class 800", "seats": 421},
  --     "crew": [
  --       {"role": "driver", "name": "J. Smith", "sign_on": "KGX", "sign_off": "YRK"},
  --       {"role": "driver", "name": "A. Jones", "sign_on": "YRK", "sign_off": "EDH"}
  --     ],
  --     "path": [
  --       {"seq": 1, "station": "KGX", "arr": null, "dep": "08:00", "act_dep": "08:02", "platform": "8"},
  --       {"seq": 2, "station": "PBO", "arr": "08:48", "dep": "08:49", "act_arr": "08:50", "platform": "1"},
  --       {"seq": 3, "station": "YRK", "arr": "09:52", "dep": "09:54", "platform": "5"}
  --     ],
  --     "delay_minutes": 2,
  --     "delay_reason": "late_departure",
  --     "load_factor_pct": 78,
  --     "gtfs_trip_id": "LNER_1A23_20260601"
  --   }
  -- ]

  -- Possessions active on this day
  possessions_json JSONB DEFAULT '[]',
  -- [
  --   {
  --     "possession_id": "...",
  --     "type": "total_blockade",
  --     "sections": ["Welwyn-Hitchin"],
  --     "start": "2026-06-01T01:00:00Z",
  --     "end": "2026-06-01T05:00:00Z",
  --     "status": "completed",
  --     "affected_services": 3,
  --     "replacement_transport": [{"mode": "bus", "route": "Welwyn-Hitchin"}]
  --   }
  -- ]

  -- Disruptions on this day
  disruptions_json JSONB DEFAULT '[]',
  -- [
  --   {
  --     "disruption_id": "...",
  --     "type": "signal_failure",
  --     "severity": "moderate",
  --     "location": "Peterborough",
  --     "started_at": "2026-06-01T10:30:00Z",
  --     "resolved_at": "2026-06-01T11:45:00Z",
  --     "trains_affected": 12,
  --     "total_delay_minutes": 180,
  --     "mitigation": [{"action": "single_line_working", "decided_at": "10:35"}]
  --   }
  -- ]

  -- Maintenance activities on this day
  maintenance_json JSONB DEFAULT '[]',
  -- [
  --   {
  --     "record_id": "...",
  --     "type": "inspection",
  --     "asset_type": "track",
  --     "asset_ref": "ECML km 105-120",
  --     "status": "completed",
  --     "is_predictive": false,
  --     "defects_found": 0
  --   }
  -- ]

  -- Daily performance summary
  performance_json JSONB DEFAULT '{}',
  -- {
  --   "services_planned": 120,
  --   "services_ran": 118,
  --   "services_cancelled": 2,
  --   "services_diverted": 1,
  --   "ppm_pct": 92.4,
  --   "right_time_pct": 78.3,
  --   "avg_delay_minutes": 3.2,
  --   "disruptions_count": 1,
  --   "possessions_count": 1,
  --   "pax_estimated": 45000,
  --   "crew_compliance_issues": 0
  -- }

  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (operator_id, service_date)
);

CREATE INDEX idx_sd_operator ON service_days(operator_id);
CREATE INDEX idx_sd_date ON service_days(service_date);
CREATE INDEX idx_sd_status ON service_days(status);
CREATE INDEX idx_sd_services ON service_days USING GIN (services_json);
CREATE INDEX idx_sd_disruptions ON service_days USING GIN (disruptions_json);
CREATE INDEX idx_sd_performance ON service_days USING GIN (performance_json);
```

### train_positions

High-frequency real-time data remains relational and partitioned.

```sql
CREATE TABLE train_positions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id        UUID NOT NULL,
  operator_id   UUID NOT NULL,
  train_number  TEXT NOT NULL,
  latitude      NUMERIC(10, 7),
  longitude     NUMERIC(10, 7),
  speed_kph     NUMERIC(6, 1),
  delay_seconds INTEGER DEFAULT 0,
  track_section_code TEXT,
  next_station_code TEXT,
  source        TEXT NOT NULL CHECK (source IN (
                  'gps', 'track_circuit', 'axle_counter', 'balise',
                  'gsm_r', 'frmcs', 'estimated'
                )),
  recorded_at   TIMESTAMPTZ NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_pos_run ON train_positions(run_id);
CREATE INDEX idx_pos_operator ON train_positions(operator_id);
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

## Example Queries

### Network overview (all track sections with current state)

```sql
SELECT section_code, name, from_station_name, to_station_name,
       characteristics_json->>'max_speed_kph' AS max_speed,
       state_json->>'status' AS status,
       state_json->>'capacity_trains_per_hour' AS capacity,
       state_json->'speed_restrictions' AS restrictions,
       condition_json->>'overall' AS condition
FROM track_sections
WHERE operator_id = $1 AND is_active = true
ORDER BY section_code;
```

### Daily service plan with crew and rolling stock

```sql
SELECT service_date, status,
       services_json, possessions_json, disruptions_json,
       maintenance_json, performance_json
FROM service_days
WHERE operator_id = $1 AND service_date = $2;
```

### Delayed trains today

```sql
SELECT svc->>'service_code' AS service,
       svc->>'origin' AS origin,
       svc->>'destination' AS dest,
       (svc->>'delay_minutes')::int AS delay,
       svc->>'delay_reason' AS reason
FROM service_days sd,
     jsonb_array_elements(sd.services_json) AS svc
WHERE sd.operator_id = $1
  AND sd.service_date = CURRENT_DATE
  AND (svc->>'delay_minutes')::int > 5
ORDER BY (svc->>'delay_minutes')::int DESC;
```

### Track sections needing maintenance

```sql
SELECT section_code, name,
       condition_json->>'overall' AS condition,
       condition_json->>'defects_open' AS defects,
       condition_json->>'next_renewal_due' AS renewal_due
FROM track_sections
WHERE operator_id = $1
  AND condition_json->>'overall' IN ('poor', 'critical')
ORDER BY condition_json->>'overall';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Operators | 1 | Crew and rolling stock fleet embedded as JSONB |
| Infrastructure | 1 | track_sections with topology, state, condition, connectivity embedded |
| Service Days | 1 | Daily aggregate: services, crew, possessions, disruptions, maintenance, performance |
| Real-Time | 1 | train_positions (partitioned) |
| AI | 1 | ai_suggestions |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **6** | |

---

## Key Design Decisions

1. **Crew and rolling stock embedded on operator** — Fleet and crew rosters are operator-level concerns that change infrequently. Embedding them avoids separate tables while providing the complete resource pool for duty and consist planning.

2. **Track sections as the infrastructure aggregate** — Each track section carries its physical characteristics, current operational state (speed restrictions, active possessions, disruptions), asset condition, and graph connectivity. A network overview is a single table scan.

3. **Service day as the operational aggregate** — Everything that happens on a specific date — train services with paths, crew assignments, rolling stock allocations, possessions, disruptions, maintenance, and daily KPIs — in a single row per operator per day. This maps to how dispatchers and controllers think: "what's happening today?"

4. **Path embedded in service** — Each train service's stopping pattern (stations, scheduled/actual times, platforms) is embedded in the service object within services_json, since paths are always viewed in the context of their service.

5. **Train positions remain partitioned relational** — Position updates at 30-second intervals for hundreds of trains create high-volume time-series data unsuitable for JSONB embedding. Partitioned by recorded_at with indexed run_id.

6. **Performance KPIs pre-computed** — PPM (Public Performance Measure), right-time percentage, average delay, and cancellation counts are pre-computed in performance_json, avoiding real-time aggregation across hundreds of service objects.

7. **Connectivity graph in JSONB** — Track section connections (from/to neighbouring sections, junctions) encoded as JSONB enable application-level graph traversal for pathfinding without a separate graph database.

8. **Stations denormalised onto track sections** — Station names and codes are embedded directly on track_sections rather than requiring a separate stations table, since the primary use is displaying the network topology.

9. **Possessions appear on both track_sections and service_days** — Active possessions are embedded in the track section's state_json (infrastructure view) and in the service day's possessions_json (operational view). Application maintains consistency.

10. **GTFS trip IDs embedded per service** — Each train service object carries its GTFS trip_id, enabling direct GTFS feed generation by extracting from the service day aggregate.
