# Rail Network Operations — Phased Development Plan

> Project: 373-rail-network-operations · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three
`data-model-suggestion-*.md` files into a concrete, buildable specification for an open,
standards-based rail network operations platform: timetabling, capacity analysis, possession
planning, real-time operations, maintenance, crew, passenger-information export, and AI augmentation.

---

## Core Requirements (Synthesis)

**What it does.** An integrated, web-first, vendor-neutral platform that connects timetable
planning, capacity allocation, possession/maintenance planning, real-time operations control,
rolling-stock and infrastructure maintenance, and crew rostering across the rail network lifecycle —
built on open standards (RailML 3.x, railTopoModel/IRS 30100, GTFS, GTFS-RT, NeTEx, SIRI, TAF/TAP TSI)
rather than a closed proprietary stack.

**Who uses it.** Infrastructure managers (capacity, possessions, conflict detection), passenger and
freight train operating companies (timetables, crew, rolling stock, real-time control), dispatchers
(live decision support), maintenance teams (defects, work orders, predictive alerts), and rail
consultancies/academics (capacity studies, simulation).

**Key differentiators (vs HAFAS, RailSys, OpenTrack, Trapeze, DELMIA, CloudMoyo, Alstom, Hitachi,
Thales).** Open-source and modular; lightweight web-based train-graph editor usable without a
power-user desktop client; live "what-if" replanning during incidents (most incumbents still produce
overnight plans); transparent, auditable optimisation explanations; cross-vendor support for
heterogeneous rolling-stock fleets; climate-resilience scenario planning; and AI exposed as
*augmentation* of (never replacement for) safety-critical control logic — including natural-language
operational querying via an MCP server, a greenfield gap no incumbent fills.

**MVP scope (from features.md "Must-have").** RailML-based infrastructure + timetable data model;
web-based graphical train-graph editor with conflict detection; UIC 406-style capacity utilisation
analysis; possession planning workflow; GTFS/GTFS-RT export; vendor-neutral real-time train-tracking
ingestion.

**Post-MVP (Should-have / Nice-to-have).** Microsimulation engine; crew rostering with rule-based
compliance; predictive-maintenance hooks; AI-assisted what-if replanning; multi-operator slot
allocation; ETCS-aware simulation; climate-resilience scenario library; natural-language querying;
cross-modal coordination; generative incident-response playbooks.

**Deployment model.** Self-hostable (Docker Compose / Kubernetes) for infrastructure managers with
data-sovereignty needs, and SaaS-deployable for smaller operators. API-first (OpenAPI 3.1 REST +
AsyncAPI 3.0 event streaming + an MCP server). The safety-critical boundary is explicit: this
platform is a **planning, analysis, and decision-support** system. It never issues movement
authorities or controls interlockings; where it integrates with TMS/interlocking it does so read-only
or advisory, so it stays outside the EN 50128 SIL scope while still respecting EN 50126 RAMS
documentation discipline and IEC 62443 / NIS2 security controls on trackside-connected adapters.

**Integration surface.** RailML 3.x import/export; GTFS + GTFS-RT export; NeTEx/SIRI; vendor-neutral
real-time feed adapters (GTFS-RT, Network Rail STOMP/NROD, SBB/DB open data, generic webhook/MQTT);
LLM provider (for AI features) via a pluggable gateway; MCP server for AI assistants.

**Data model decision.** The **normalised relational model (data-model-suggestion-1)** is the
canonical schema: rail topology is a directed graph, and conflict detection / UIC 406 capacity are
naturally join-and-CTE queries over `track_sections` and train paths, with full referential
integrity for safety-adjacent data. We adopt two ideas from the other suggestions:
(a) the **append-only `domain_events`/audit log** and CloudEvents envelope from
data-model-suggestion-3 for real-time provenance, what-if replay, and AI training data; and
(b) **materialised read-model views** (à la suggestion-3's `rm_*`) for the dispatcher live board and
daily performance, refreshed from the relational tables, to keep dashboards fast without a second
write store. Suggestion-2's pure-JSONB aggregate approach is rejected as canonical because
cross-service analytics and conflict detection would require fragile JSONB extraction, but its
"network overview / service day" framing informs the read models.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | Domain is heavy on standards parsing (RailML XML, GTFS), constraint/optimisation, simulation, and ML/LLM (predictive maintenance, demand forecasting, NL querying). Python has the richest libraries for all four (`lxml`, `gtfs` tooling, OR-Tools, NumPy/SciPy, scikit-learn, OpenAI/Anthropic SDKs) and keeps the AI work first-class rather than bolted on. |
| API framework | FastAPI + Uvicorn | Async (needed for streaming positions and concurrent feed adapters), generates OpenAPI 3.1 automatically (a stated standard), Pydantic v2 validation maps directly to JSON Schema 2020-12 (another stated standard). |
| Data validation | Pydantic v2 | Single source of truth for request/response models, config, and exchange payloads; emits JSON Schema 2020-12. |
| Database | PostgreSQL 16 + PostGIS + TimescaleDB | The canonical model is relational with recursive-CTE graph queries (suggestion-1). PostGIS handles `geometry_geojson` LineStrings, station coordinates, and spatial conflict queries. TimescaleDB hypertables handle high-frequency `train_positions` and `domain_events` (replaces manual `PARTITION BY RANGE`). PostgreSQL JSONB covers RailML-flavoured nested fields. |
| ORM / migrations | SQLAlchemy 2.0 (async) + Alembic | Mature async ORM; Alembic for versioned migrations (a Definition-of-Done requirement per phase). |
| Spatial / graph | PostGIS + NetworkX | PostGIS for in-DB spatial joins; NetworkX in-memory for pathfinding and capacity graph algorithms on the loaded network. |
| Optimisation | Google OR-Tools (CP-SAT) | Constraint programming for conflict resolution, possession-window optimisation, slot allocation, and crew rostering — published, patent-free, avoids reproducing vendors' proprietary dispatching algorithms. |
| Task queue / async jobs | Celery + Redis | Simulation runs, RailML/GTFS import/export, capacity studies, and AI calls are long-running; Celery isolates them from request latency. Redis doubles as cache and pub/sub for live position fan-out. |
| Real-time transport | WebSocket (FastAPI) + Redis pub/sub; AsyncAPI 3.0 spec | Sub-second push of positions and alerts to the control-room UI; AsyncAPI documents the event channels (a stated standard). |
| Streaming ingestion | aiomqtt, stomp.py, httpx pollers | Vendor-neutral adapters: MQTT, STOMP (Network Rail NROD/Darwin), and HTTP pollers for GTFS-RT / DB / SBB open data. |
| Frontend | React 18 + TypeScript + Vite | The train-graph (time–distance) editor and control-room board are interactive SPAs; React + TypeScript is the standard for complex, stateful operational UIs. |
| Graph rendering | D3.js (custom SVG/Canvas) + MapLibre GL | The train graph is a bespoke time–distance diagram (D3); the geographic network view uses MapLibre GL (open, no token lock-in) over the PostGIS geometry. |
| UI components | shadcn/ui + Tailwind CSS | Fast, accessible (OWASP/WCAG) component layer for dashboards and forms. |
| Frontend data | TanStack Query + Zustand | Server-state caching for REST, local store for editor/optimistic state. |
| AuthN / AuthZ | OAuth 2.1 + OpenID Connect (Authlib); mTLS for trackside adapters | OIDC for dispatcher/planner SSO; OAuth 2.0 client-credentials for B2B integrations; mTLS (RFC 8705) for trackside service-to-service — all stated standards. RBAC enforced per operator. |
| AI / LLM gateway | Pluggable provider abstraction (Anthropic / OpenAI / local) | AI features (NL query, scenario generation, incident drafting) must not lock to one vendor; a thin gateway with prompt-caching keeps providers swappable. |
| ML | scikit-learn + ONNX Runtime | Predictive-maintenance and demand-forecasting models trained offline, served via ONNX for portability across heterogeneous fleets. |
| MCP server | Python MCP SDK | Exposes timetable/capacity/incident read models to AI assistants for NL analytics — the differentiating greenfield capability. |
| Standards libs | `lxml` (+ RailML 3.x XSDs), `gtfs-realtime-bindings` (protobuf), in-house NeTEx/SIRI serialisers | RailML/railTopoModel import-export, GTFS-RT protobuf encoding, NeTEx/SIRI XML for European interoperability. |
| Containerisation | Docker + Docker Compose (dev) / Helm chart (prod) | Self-host parity; reproducible RAMS/audit environments. |
| Testing | pytest + pytest-asyncio + Testcontainers; Vitest + Playwright (frontend) | Unit/integration with real Postgres via Testcontainers; Playwright for the editor E2E. |
| Code quality | Ruff (lint+format) + mypy --strict; ESLint + Prettier + tsc | Strict typing supports the EN 50126 documentation/quality expectation. |
| Package mgmt | uv (Python) + pnpm (frontend) | Fast, reproducible lockfiles. |
| Observability | OpenTelemetry + Prometheus + structured JSON logs | NIS2/IEC 62443 expect auditable security and operational telemetry. |
| API specs | OpenAPI 3.1, AsyncAPI 3.0, JSON Schema 2020-12, GraphQL (analytics, later) | Explicit standards from `standards.md`. GraphQL added in a later phase for flexible analytics dashboards. |

### Project Structure

```
rail-network-operations/
├── pyproject.toml
├── uv.lock
├── README.md
├── docker-compose.yml                 # postgres+postgis+timescale, redis, api, worker, web
├── Dockerfile                         # backend image
├── Dockerfile.worker                  # celery worker image
├── alembic.ini
├── .env.example
├── docs/
│   ├── openapi/                        # exported OpenAPI 3.1 snapshots (CI artefact)
│   ├── asyncapi/                       # AsyncAPI 3.0 channel spec
│   └── adr/                            # architecture decision records (RAMS evidence)
├── migrations/                         # Alembic versions
├── data/
│   ├── fixtures/                        # sample RailML, GTFS, GTFS-RT, position feeds
│   └── schemas/                         # RailML 3.x XSDs, GTFS-RT .proto
├── src/
│   └── rno/
│       ├── __init__.py
│       ├── main.py                      # FastAPI app factory, router wiring
│       ├── config.py                    # Pydantic Settings
│       ├── db/
│       │   ├── base.py                  # async engine/session
│       │   ├── models/                  # SQLAlchemy ORM (one module per aggregate)
│       │   └── read_models/             # materialised-view definitions + refreshers
│       ├── schemas/                     # Pydantic request/response models
│       ├── domain/
│       │   ├── topology/                # network graph (NetworkX), railTopoModel
│       │   ├── timetable/               # services, runs, path segments
│       │   ├── conflicts/               # conflict detection engine
│       │   ├── capacity/                # UIC 406 utilisation
│       │   ├── possessions/             # possession planning + clash checks
│       │   ├── simulation/              # microsimulation engine (later phase)
│       │   ├── crew/                    # rostering + compliance rules
│       │   ├── maintenance/             # defects, work orders, predictive hooks
│       │   ├── disruptions/             # disruption + replan
│       │   └── optimisation/            # OR-Tools models (conflict, possession, slot, crew)
│       ├── integrations/
│       │   ├── railml/                  # import/export
│       │   ├── gtfs/                    # GTFS static export
│       │   ├── gtfs_rt/                 # GTFS-RT protobuf export
│       │   ├── netex_siri/              # NeTEx/SIRI serialisers
│       │   ├── feeds/                   # real-time adapters (gtfs_rt, stomp, mqtt, http)
│       │   └── taf_tap/                 # cross-operator exchange (later)
│       ├── api/
│       │   ├── routers/                 # FastAPI routers per resource
│       │   ├── deps.py                  # auth, db session, operator scoping
│       │   ├── ws/                      # WebSocket endpoints
│       │   └── errors.py                # RFC 9457 problem+json handlers
│       ├── ai/
│       │   ├── gateway.py               # provider-agnostic LLM client w/ caching
│       │   ├── nlq/                     # natural-language querying
│       │   ├── scenarios/               # generative scenario planning
│       │   ├── suggestions/             # conflict/repath/crew suggestion services
│       │   └── ml/                      # predictive maintenance, demand forecast
│       ├── mcp/
│       │   └── server.py                # MCP server exposing read models
│       ├── events/
│       │   ├── store.py                 # append domain_events (CloudEvents envelope)
│       │   └── bus.py                   # Redis pub/sub fan-out
│       ├── auth/                        # OIDC/OAuth, RBAC, mTLS
│       ├── workers/                     # Celery tasks
│       └── observability/               # OTel, logging, metrics
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/
└── web/
    ├── package.json
    ├── vite.config.ts
    └── src/
        ├── main.tsx
        ├── api/                          # generated OpenAPI client
        ├── components/
        ├── features/
        │   ├── train-graph/              # D3 time–distance editor
        │   ├── network-map/              # MapLibre network view
        │   ├── capacity/                 # UIC 406 dashboards
        │   ├── possessions/
        │   ├── control-room/             # live board (WebSocket)
        │   ├── crew/
        │   ├── maintenance/
        │   └── nlq/                       # natural-language query console
        └── store/
```

---

## Phase 1: Foundation, Topology & Data Model

### Purpose
Establish the repository, tooling, configuration, database, authentication, and the canonical
relational schema (from data-model-suggestion-1) with the network-topology graph layer. After this
phase the platform can model an operator's stations and directed track sections, persist them with
full referential integrity, and answer basic graph queries (shortest path, sections between two
stations) — the substrate every later phase builds on.

### Tasks

#### 1.1 — Project scaffold, config, and CI

**What**: Create the buildable skeleton: `pyproject.toml` (uv), FastAPI app factory, Pydantic
settings, Docker Compose (Postgres+PostGIS+TimescaleDB, Redis, api, worker), linting, typing, CI.

**Design**:
- `config.py` Pydantic `Settings`:
  ```python
  class Settings(BaseSettings):
      database_url: str          # postgresql+asyncpg://...
      redis_url: str = "redis://localhost:6379/0"
      jwt_issuer: str
      jwt_audience: str
      oidc_discovery_url: str | None = None
      llm_provider: Literal["anthropic", "openai", "none"] = "none"
      llm_api_key: SecretStr | None = None
      environment: Literal["dev", "test", "staging", "prod"] = "dev"
      log_level: str = "INFO"
      model_config = SettingsConfigDict(env_prefix="RNO_", env_file=".env")
  ```
- `main.py` `create_app() -> FastAPI` wires routers, OTel middleware, RFC 9457 error handlers, CORS,
  and a `/healthz` (liveness) and `/readyz` (DB+Redis ping) endpoint.
- `docker-compose.yml` services: `db` (image `timescale/timescaledb-ha:pg16` with PostGIS),
  `redis`, `api`, `worker`, `web`.
- CI (GitHub Actions): `ruff check`, `ruff format --check`, `mypy --strict src`, `pytest`,
  `docker build`.

**Testing**:
- `Unit: Settings loads from env with RNO_ prefix → correct typed values; missing required field → ValidationError naming the field.`
- `Integration: GET /healthz → 200 {"status":"ok"}.`
- `Integration (Testcontainers Postgres+Redis): GET /readyz with both up → 200; with DB down → 503.`
- `CI: ruff, mypy --strict, pytest all green on empty skeleton.`

#### 1.2 — Canonical relational schema (Alembic migration 0001)

**What**: Implement the 14-table normalised schema from data-model-suggestion-1 as SQLAlchemy 2.0
models plus the first Alembic migration, with `train_positions` and `audit_log` as TimescaleDB
hypertables instead of native range partitions.

**Design**:
- One ORM module per aggregate under `db/models/`: `operators`, `stations`, `track_sections`,
  `rolling_stock`, `train_services`, `train_runs`, `crew_members`, `crew_duties`, `possessions`,
  `maintenance_records`, `disruptions`, `train_positions`, `ai_suggestions`, `audit_log`.
- Field definitions, CHECK constraints, FKs, and indexes exactly as specified in
  data-model-suggestion-1 (operators/stations/track_sections/rolling_stock/train_services/
  train_runs/crew_members/crew_duties/possessions/maintenance_records/disruptions/train_positions/
  ai_suggestions/audit_log).
- `track_sections` stores `geometry_geojson` as PostGIS `geometry(LineString, 4326)` (override the
  JSONB suggestion) plus a generated `length_km` validation check; stations store
  `geometry(Point,4326)` derived from lat/long.
- Convert to hypertables in the migration:
  ```sql
  SELECT create_hypertable('train_positions', 'recorded_at', chunk_time_interval => INTERVAL '1 day');
  SELECT create_hypertable('audit_log', 'created_at', chunk_time_interval => INTERVAL '7 days');
  ```
- Multi-tenancy: every table is scoped by `operator_id`; a Postgres Row-Level-Security policy keyed
  on a `app.current_operator` session GUC enforces isolation.
- Enums implemented as Python `StrEnum` mirrored by the SQL CHECK constraints for type-safe code.

**Testing**:
- `Integration (Testcontainers): alembic upgrade head → all 14 tables, indexes, hypertables present (query pg_catalog / timescaledb_information.hypertables).`
- `Integration: alembic downgrade base then upgrade head → idempotent, no errors.`
- `Unit: insert track_section with from==to station → CHECK/constraint or service-layer validation rejects self-loop.`
- `Integration: insert train_run with duplicate (service_id, run_date) → UNIQUE violation.`
- `Integration: RLS — session GUC operator A cannot SELECT operator B's stations.`
- `Unit: StrEnum values exactly match SQL CHECK lists (parametrised test over every enum).`

#### 1.3 — Topology graph layer (railTopoModel)

**What**: A `domain/topology` module that loads `stations` + `track_sections` into a directed
NetworkX graph and exposes pathfinding and adjacency queries aligned to railTopoModel (IRS 30100).

**Design**:
```python
@dataclass(frozen=True)
class NetworkGraph:
    operator_id: UUID
    g: nx.MultiDiGraph        # nodes = station ids, edges = track_section ids (directed)

class TopologyService:
    async def load(self, operator_id: UUID) -> NetworkGraph: ...
    def shortest_path(self, ng: NetworkGraph, src: UUID, dst: UUID,
                      weight: Literal["length_km", "min_runtime"] = "length_km"
                      ) -> list[UUID]:  # ordered track_section ids
        ...
    def sections_between(self, ng: NetworkGraph, src: UUID, dst: UUID) -> list[UUID]: ...
    def neighbours(self, ng: NetworkGraph, station_id: UUID) -> list[UUID]: ...
```
- `min_runtime` weight derives from `length_km / max_speed_kph` as a first approximation (refined by
  simulation in Phase 6).
- Graph is cached in Redis per operator with invalidation on any track_section/station write
  (via the event bus from Phase 5; until then, TTL cache).

**Testing**:
- `Unit (fixture network): shortest_path A→D over a 4-node line → expected ordered section ids.`
- `Unit: shortest_path with a possession-closed section excluded → alternative route returned.`
- `Unit: disconnected stations → NoPathError raised.`
- `Integration: load() builds graph matching DB rows; node/edge counts equal table counts.`

#### 1.4 — AuthN/AuthZ and operator scoping

**What**: OIDC/OAuth 2.1 authentication, RBAC roles, and per-request operator scoping dependency.

**Design**:
- `auth/` validates JWTs against `oidc_discovery_url` (Authlib); client-credentials flow for B2B.
- Roles: `planner`, `dispatcher`, `controller`, `maintenance`, `crew_manager`, `viewer`, `admin`,
  `integration` (machine). RBAC matrix maps role → allowed `(resource, action)`.
- FastAPI dependency `current_principal()` returns `Principal(user_id, operator_id, roles)`; sets the
  Postgres `app.current_operator` GUC for RLS for the request's transaction.
- `require(*perms)` dependency factory guards routers.

**Testing**:
- `Unit: valid JWT → Principal with correct claims; expired/invalid signature → 401.`
- `Unit: RBAC — dispatcher hitting an admin-only route → 403.`
- `Integration: request sets app.current_operator GUC; cross-operator read blocked by RLS.`
- `Integration (mocked OIDC): client-credentials token with scope=integration → integration role.`

---

## Phase 2: Infrastructure & Fleet CRUD + RailML Import/Export

### Purpose
Expose the network and fleet as a managed API and make it interoperable with the rail ecosystem from
day one by importing/exporting RailML 3.x. After this phase, an operator can populate their entire
infrastructure (stations, track sections with signalling/electrification/gradients), rolling-stock
fleet, and base timetable either through the API/UI or by importing an existing RailML file — the
prerequisite for every planning feature.

### Tasks

#### 2.1 — CRUD APIs for infrastructure & fleet

**What**: REST resources for `operators`, `stations`, `track_sections`, `rolling_stock`,
`train_services`, `crew_members`.

**Design**:
- Endpoints (OpenAPI 3.1, all under `/api/v1`, operator-scoped, paginated, filterable):
  ```
  POST   /stations                  201 StationOut
  GET    /stations?type=&q=&page=    200 Page[StationOut]
  GET    /stations/{id}             200 StationOut | 404 problem+json
  PATCH  /stations/{id}             200 StationOut
  DELETE /stations/{id}             204 (soft-delete → is_active=false)
  ... identical pattern for track-sections, rolling-stock, train-services, crew-members
  ```
- Pydantic models mirror the schema; `StationCreate` requires name, station_code, station_type,
  latitude, longitude; computes PostGIS point on write.
- `TrackSectionCreate` validates `from_station_id != to_station_id`, both stations exist and belong to
  the operator, and `max_speed_kph > 0`.
- Errors use RFC 9457 `application/problem+json`.

**Testing**:
- `Unit: TrackSectionCreate with from==to → ValidationError.`
- `Integration: POST station then GET → round-trips lat/long and station_type.`
- `Integration: POST track_section referencing another operator's station → 404 (RLS hides it).`
- `Integration: DELETE station → is_active=false, still retrievable with include_inactive=true.`
- `Integration: pagination + ?type=mainline filter returns only matching, correct total count.`

#### 2.2 — RailML 3.x import

**What**: Import a RailML 3.x file (infrastructure + rolling stock + base timetable) into the schema.

**Design**:
- `integrations/railml/importer.py`:
  ```python
  class RailMLImporter:
      def parse(self, xml: bytes) -> RailMLDocument        # lxml + XSD validation
      async def import_document(self, operator_id: UUID, doc: RailMLDocument,
                                mode: Literal["create", "upsert"]) -> ImportReport
  @dataclass
  class ImportReport:
      stations: int; track_sections: int; rolling_stock: int
      services: int; warnings: list[str]; errors: list[str]
  ```
- Validate against the RailML 3.x XSDs in `data/schemas/`. Map `<infrastructure>` netElements/
  netRelations (railTopoModel) → stations + directed track_sections; `<rollingstock>` →
  rolling_stock; `<timetable>` → train_services + path segments.
- Long imports run as a Celery task; endpoint `POST /imports/railml` (multipart) returns `202` with a
  job id; `GET /imports/{id}` returns the `ImportReport`.
- Idempotency in `upsert` mode keyed on `section_code` / `unit_number` / `service_code`.

**Testing**:
- `Unit: parse() valid RailML → RailMLDocument; schema-invalid XML → RailMLValidationError listing failures.`
- `Integration (fixture RailML): import_document create → expected row counts; warnings for unmapped elements.`
- `Integration: re-import same file in upsert mode → no duplicates, counts stable.`
- `E2E: POST /imports/railml multipart → 202 job; poll → completed ImportReport.`

#### 2.3 — RailML 3.x export

**What**: Export an operator's infrastructure, fleet, and timetable as a schema-valid RailML 3.x file.

**Design**:
- `integrations/railml/exporter.py: export(operator_id, scope: set[Literal["infrastructure","rollingstock","timetable"]]) -> bytes`.
- Output validated against the same XSDs before return.
- `GET /exports/railml?scope=infrastructure,timetable` → `200 application/xml`.

**Testing**:
- `Integration: export then re-import into a clean operator → row counts match the source (round-trip).`
- `Unit: exported document validates against RailML 3.x XSD.`
- `Integration: scope=infrastructure omits rolling stock and timetable elements.`

---

## Phase 3: Timetabling & Conflict Detection (Core Value #1)

### Purpose
Deliver the heart of the planner-facing product: construct train services and dated runs with path
segments, and detect timetable conflicts (headway, platform occupation, single-line, possession
clashes) over the network graph. This is the capability incumbents charge millions for; shipping it
early proves the platform's value. After this phase a planner can build a timetable and immediately
see conflicts.

### Tasks

#### 3.1 — Timetable construction API (services, runs, path segments)

**What**: APIs to create/edit `train_services`, generate `train_runs` for a date range, and edit the
ordered `path_segments` (stations, platforms, scheduled arr/dep, dwell, track section per leg).

**Design**:
- `PathSegment` Pydantic model mirrors the JSONB shape in `train_runs.path_segments`
  (sequence, station_id, station_code, platform, track_section_id, scheduled_arrival/departure,
  dwell_seconds, stopping_pattern ∈ {stop, pass, set_down_only, pick_up_only, request}).
- `POST /services/{id}/generate-runs {from, to, days_of_operation}` → creates `train_runs` per the
  service calendar (skips suppressed dates), defaulting `scheduled_departure/arrival` from the
  service path template.
- `PUT /runs/{id}/path` replaces path segments; service-layer validation: monotonic sequence,
  monotonic non-decreasing times, every `track_section_id` adjacent in the topology graph, dwell ≥ 0.
- On every path write, enqueue conflict re-evaluation (Task 3.2) for affected sections/runs.

**Testing**:
- `Unit: path with non-monotonic times → ValidationError naming the offending segment.`
- `Unit: path referencing non-adjacent sections → TopologyError.`
- `Integration: generate-runs over a Mon–Fri service for one week → 5 runs, weekend skipped.`
- `Integration: PUT path then GET run → segments round-trip with platforms and dwell.`

#### 3.2 — Conflict detection engine

**What**: Detect conflicts across all runs on a date: headway violations, platform/station occupation
overlaps, single-line opposing movements, and possession clashes.

**Design**:
```python
class ConflictType(StrEnum):
    HEADWAY = "headway"; PLATFORM_OCCUPATION = "platform_occupation"
    SINGLE_LINE = "single_line"; POSSESSION_CLASH = "possession_clash"
    DWELL_OVERRUN = "dwell_overrun"

@dataclass
class Conflict:
    type: ConflictType
    severity: Literal["info","warning","critical"]
    run_a: UUID; run_b: UUID | None
    track_section_id: UUID | None; station_id: UUID | None
    time_window: tuple[datetime, datetime]
    description: str
    suggested_resolution: str | None   # filled by AI in Phase 8

class ConflictDetector:
    def detect(self, runs: list[TrainRunView], graph: NetworkGraph,
               possessions: list[PossessionView],
               min_headway_s: int = 180) -> list[Conflict]: ...
```
- Algorithm: build per-track-section occupation intervals from each run's path segments
  (entry→exit times computed from adjacent segment times); for each section, sort intervals and flag
  pairs whose gap < `min_headway_s` (HEADWAY) or that overlap on a single-track section in opposing
  directions (SINGLE_LINE). Per (station, platform), flag overlapping dwell intervals
  (PLATFORM_OCCUPATION). For each section under a possession in the run's time window →
  POSSESSION_CLASH. Headway configurable per section via `track_sections.capacity_trains_per_hour`.
- `POST /timetable/conflicts {date}` → `200 list[Conflict]`; results cached and invalidated on path
  writes. Heavy runs use Celery.

**Testing**:
- `Unit: two runs on a single-track section <180s apart → one HEADWAY critical conflict.`
- `Unit: opposing runs on single-line overlapping → SINGLE_LINE conflict.`
- `Unit: two runs dwelling same platform overlapping → PLATFORM_OCCUPATION conflict.`
- `Unit: run path crossing a section under possession in window → POSSESSION_CLASH.`
- `Unit: well-separated runs → empty list.`
- `Integration (fixture timetable with 3 known conflicts): detect returns exactly those 3.`

---

## Phase 4: Capacity Analysis & Possession Planning (Core Value #2)

### Purpose
Add the infrastructure-manager-facing core: UIC 406-style capacity utilisation across the network and
a possession-planning workflow that books track-access windows and surfaces their service impact.
After this phase an infrastructure manager can identify bottlenecks and plan maintenance windows with
quantified passenger/capacity impact — directly competing with RailSys and DELMIA.

### Tasks

#### 4.1 — UIC 406 capacity utilisation

**What**: Compute capacity utilisation per track section over a time window using the UIC 406
compression method.

**Design**:
```python
@dataclass
class CapacityResult:
    track_section_id: UUID
    window: tuple[datetime, datetime]
    train_count: int
    occupation_time_s: int          # sum of blocking times
    compressed_time_s: int          # after UIC 406 compression
    utilisation_pct: float          # compressed / window
    rating: Literal["satisfactory","near_capacity","overloaded"]  # UIC 406 thresholds

class CapacityAnalyser:
    def utilisation(self, section: TrackSectionView, runs: list[TrainRunView],
                    window: tuple[datetime, datetime]) -> CapacityResult: ...
```
- UIC 406 compression: take each train's blocking-time stairway on the section (entry→exit + headway
  buffer), then compress trains together minimising idle time without overtaking changes; utilisation
  = compressed occupation ÷ window length. Thresholds per UIC 406 leaflet (e.g. 60% peak / 75%
  daily as `near_capacity`, above → `overloaded`; configurable).
- `POST /capacity/analyse {section_ids?, window}` → `200 list[CapacityResult]`; network-wide bottleneck
  scan ranks sections by utilisation.

**Testing**:
- `Unit: single train on empty section → utilisation ≈ blocking/window, rating satisfactory.`
- `Unit: section packed beyond threshold → overloaded.`
- `Unit: compression with non-overtaking constraint reduces idle correctly (golden value from a worked UIC 406 example).`
- `Integration: bottleneck scan ranks the known-busiest fixture section first.`

#### 4.2 — Possession planning workflow

**What**: Create, approve, and activate possessions; auto-compute affected services and clashing
runs; manage the lifecycle (planned → confirmed → active → overrun → completed/cancelled).

**Design**:
- `POST /possessions` (track_section_ids, type, window, title) → computes `affected_services` by
  intersecting each planned run's path with the possessed sections in the window and classifying
  impact (cancelled / diverted via topology alt-route / delayed).
- State machine enforced server-side; illegal transitions → `409 problem+json`.
- `POST /possessions/{id}/approve` (role: planner/admin) sets approved_by/at, status=confirmed.
- `GET /possessions/{id}/impact` → list of affected runs with proposed mitigation (diversion route
  from `TopologyService.shortest_path` excluding possessed sections; replacement bus if none).
- Possessions feed the POSSESSION_CLASH conflict check (Task 3.2) and capacity analysis.

**Testing**:
- `Unit: state machine — confirm→active legal, completed→active illegal (409).`
- `Unit: affected-services computation flags exactly the runs whose path crosses possessed sections in-window.`
- `Unit: diversion suggested when an alt-route exists; replacement-bus flagged when none.`
- `Integration: create possession → run that clashes now reports POSSESSION_CLASH in conflict detection.`

---

## Phase 5: Real-Time Ingestion, Event Store & Live Operations

### Purpose
Bring the platform to life: ingest vendor-neutral real-time feeds, persist positions and an immutable
event log, fan updates to a live control-room board over WebSocket, and model delay propagation.
After this phase dispatchers see live train movements and delays, and the append-only event store
(from data-model-suggestion-3) provides provenance, replay, and AI training data.

### Tasks

#### 5.1 — Domain event store & bus

**What**: Append-only `domain_events` table (CloudEvents 1.0 envelope, TimescaleDB hypertable) plus a
Redis pub/sub bus, used by all subsequent state changes.

**Design**:
- `domain_events` adapts data-model-suggestion-3's `event_store`: stream_type, stream_id, sequence_no,
  event_type, event_data JSONB, CloudEvents fields (ce_source/type/specversion/time), actor_id,
  actor_type, operator_id, location_ref, correlation_id, causation_id; hypertable on `ce_time`;
  UNIQUE(stream_type, stream_id, sequence_no).
- `events/store.py: append(event: DomainEvent) -> None` (transactional with the originating write);
  `events/bus.py: publish(event)` to Redis channel `ops:{operator_id}`.
- Service-layer writes (path edits, possessions, disruptions, positions) emit events.

**Testing**:
- `Unit: append assigns next sequence_no per stream; duplicate sequence → UNIQUE violation.`
- `Integration: state change writes both the domain row and a domain_event in one transaction (rollback leaves neither).`
- `Integration: publish → subscriber on ops:{operator} receives the CloudEvents JSON.`

#### 5.2 — Vendor-neutral feed adapters

**What**: Pluggable adapters that normalise external real-time feeds into `train.position_reported`
and `train.delayed` events and `train_positions` rows.

**Design**:
```python
class FeedAdapter(Protocol):
    name: str
    async def run(self, operator_id: UUID, on_event: Callable[[DomainEvent], Awaitable[None]]): ...
```
- Built-in adapters: `GtfsRtAdapter` (poll GTFS-RT VehiclePositions/TripUpdates protobuf),
  `StompAdapter` (Network Rail NROD/Darwin TD/TRUST), `MqttAdapter` (generic), `HttpPollAdapter`
  (DB/SBB open data JSON). Each maps external ids → `train_runs` via `train_number`/`gtfs_trip_id`.
- Adapters run as Celery/asyncio long-lived tasks; config in `feeds` table or settings; mTLS for
  trackside connections.
- Each position → insert `train_positions` row + emit `train.position_reported`; threshold breach →
  `train.delayed`.

**Testing**:
- `Unit (recorded GTFS-RT fixture): GtfsRtAdapter decodes protobuf → expected position events.`
- `Unit: position with no matching run → buffered/dropped with warning, no crash.`
- `Integration (mock STOMP broker): TD message → train_positions row + event published.`
- `Integration: delay over threshold → train.delayed emitted with computed delay_seconds.`

#### 5.3 — Live control-room board (WebSocket) & delay propagation

**What**: WebSocket endpoint streaming live positions/delays/disruptions, backed by a materialised
read model; first-cut delay-propagation estimator.

**Design**:
- Materialised read models `rm_train_board`, `rm_network_status` (adapted from
  data-model-suggestion-3), refreshed from the relational tables and the event stream.
- `WS /ws/control-room` (auth via token query/subprotocol): on connect, snapshot of current board;
  then incremental updates from the Redis `ops:{operator}` channel.
- `domain/disruptions/propagation.py`: when a run is delayed at a node, propagate to downstream
  segments and to dependent runs sharing rolling stock/crew/platform, computing estimated knock-on
  delays via the topology graph and consist/duty links.

**Testing**:
- `Integration: WS connect → receives snapshot; a published position event → client receives the delta.`
- `Unit: delay at section X propagates to the next-stop ETA correctly.`
- `Unit: a unit's late inbound arrival propagates to its outbound run's estimated departure.`
- `Integration: read-model refresh after an event reflects new delay within the refresh interval.`

---

## Phase 6: Passenger Information Export (GTFS / GTFS-RT / NeTEx / SIRI)

### Purpose
Make the platform publishable to the wider transit ecosystem and journey planners — a stated MVP
must-have and a low-friction integration win. After this phase the operator's timetable and live
status flow to any GTFS/SIRI consumer (journey planners, station screens, apps).

### Tasks

#### 6.1 — GTFS static export

**What**: Generate a spec-valid GTFS feed (zip of agency, stops, routes, trips, stop_times, calendar,
calendar_dates) from services, runs, and stations.

**Design**:
- `integrations/gtfs/exporter.py: build_feed(operator_id, date_range) -> bytes (zip)`.
- Map `stations.gtfs_stop_id` → stops.txt; `train_services` → routes/trips; `train_runs.path_segments`
  → stop_times; `days_of_operation` / valid_from/until → calendar + calendar_dates.
- `GET /exports/gtfs?from=&to=` → `200 application/zip`. Validate with MobilityData `gtfs-validator`
  in CI against committed output.

**Testing**:
- `Integration: exported zip passes gtfs-validator with zero errors on the fixture network.`
- `Unit: a run with set_down_only stop → correct pickup_type/drop_off_type in stop_times.`
- `Integration: calendar_dates reflects suppressed/added dates.`

#### 6.2 — GTFS-RT, NeTEx & SIRI

**What**: GTFS-RT protobuf feeds (TripUpdates, VehiclePositions, ServiceAlerts) from live data, plus
NeTEx (timetable) and SIRI (real-time) XML for European interoperability.

**Design**:
- `integrations/gtfs_rt/exporter.py` builds `FeedMessage` from `rm_train_board` + active disruptions
  using `gtfs-realtime-bindings`. Served at `GET /feeds/gtfs-rt/{trip-updates|vehicle-positions|alerts}`
  → `application/x-protobuf`, regenerated on a short interval.
- NeTEx serialiser (timetable/network) and SIRI-ET/SIRI-SX (estimated timetable / situation exchange)
  XML generated from the same read models; disruptions → SIRI Situation Exchange + GTFS-RT alerts.

**Testing**:
- `Unit: a delayed run → TripUpdate with correct stop_time_update delays (decode round-trip).`
- `Unit: active disruption → GTFS-RT ServiceAlert + SIRI-SX situation with matching scope.`
- `Integration: vehicle-positions feed reflects latest train_positions.`

---

## Phase 7: Crew Rostering, Maintenance & Predictive Hooks

### Purpose
Complete the operational picture with crew duties (rule-based legal compliance), infrastructure and
rolling-stock maintenance workflows, and predictive-maintenance ingestion — the Trapeze/Alstom
feature space, delivered openly and fleet-agnostically.

### Tasks

#### 7.1 — Crew rostering with compliance rules

**What**: Roster `crew_duties` against runs and validate working-time rules (max driving hours, min
rest, route/traction knowledge).

**Design**:
- `domain/crew/compliance.py` — pluggable rule engine; rules from `crew_members.rest_rules_json` /
  operator settings:
  ```python
  class ComplianceRule(Protocol):
      code: str
      def check(self, member: CrewMemberView, duties: list[DutyView]) -> list[Violation]: ...
  ```
  Built-ins: `MaxDrivingHoursDaily`, `MinRestBetweenDuties`, `MaxTotalHoursWeekly`,
  `RouteKnowledgeRequired`, `TractionKnowledgeRequired`.
- `POST /duties` and `POST /duties/validate` → returns `compliance_violations`; assigning an
  unqualified driver to a run → violation, not hard block (planner override with audit event).
- Disruption rostering: `POST /duties/reassign` finds available qualified crew via GIN-indexed
  route/traction arrays.

**Testing**:
- `Unit: 9h driving with 8h limit → MaxDrivingHoursDaily violation.`
- `Unit: 6h rest with 12h minimum → MinRestBetweenDuties violation.`
- `Unit: driver lacking route knowledge for the run → RouteKnowledgeRequired violation.`
- `Integration: compliant duty → is_compliant=true, no violations.`

#### 7.2 — Maintenance & predictive-maintenance hooks

**What**: Defect/inspection/work-order workflow over the unified `maintenance_records` table, plus an
ingestion endpoint for rolling-stock sensor telemetry (IEC 61375) that can raise predictive alerts.

**Design**:
- CRUD + lifecycle (open → assigned → in_progress → completed/deferred) for maintenance records on
  track/signal/structure/rolling-stock assets; link to possessions for infrastructure work.
- `POST /telemetry/rolling-stock` (mTLS, role integration): batches of sensor readings →
  `domain_events` (`unit.sensor_reading`); a pluggable scorer (ML model from Phase 8) may create a
  `maintenance_record` with `is_predictive=true`, `ai_confidence`, linked sensor data.
- Safety-critical records (`priority=safety_critical`) raise alerts on the control-room board.

**Testing**:
- `Unit: maintenance lifecycle illegal transition → 409.`
- `Integration: telemetry batch → sensor-reading events stored; threshold breach → predictive maintenance_record created with confidence.`
- `Integration: safety_critical defect → alert event on control-room channel.`

---

## Phase 8: AI Augmentation & Optimisation

### Purpose
Deliver the AI-native differentiators as *augmentation* — suggestions a human accepts/rejects, never
autonomous control. Conflict-resolution and possession/slot/crew optimisation (OR-Tools), what-if
replanning, demand forecasting, and predictive-maintenance models. Every suggestion is logged to
`ai_suggestions` with confidence and an accept/reject feedback loop that becomes training data.

### Tasks

#### 8.1 — Pluggable LLM gateway & AI suggestion service

**What**: Provider-agnostic LLM client (prompt-cached) and the `ai_suggestions` workflow
(pending → accepted/rejected/auto_applied/expired) feeding human-in-the-loop review.

**Design**:
```python
class LLMGateway(Protocol):
    async def complete(self, system: str, user: str, *, cache_key: str | None,
                       json_schema: dict | None) -> LLMResult: ...
```
- Providers: Anthropic, OpenAI, local — selected by `settings.llm_provider`. Prompt caching on stable
  system prompts (network description, rule catalogue).
- `SuggestionService.create(type, target, input)` persists an `ai_suggestions` row; `accept/reject`
  records reviewer + feedback and emits `ai.suggestion_accepted/rejected` events.

**Testing**:
- `Unit (mock gateway): create suggestion → row persisted status=pending with input/output JSON.`
- `Unit: accept → status accepted, reviewer recorded, event emitted.`
- `Unit: gateway respects json_schema (mock returns invalid → re-prompt/raise).`

#### 8.2 — Optimisation models (OR-Tools)

**What**: CP-SAT models for conflict resolution (retime/repath), possession-window optimisation,
multi-operator slot allocation, and disruption crew reassignment.

**Design**:
- `domain/optimisation/conflict.py`: variables = per-run timing offsets within tolerance; constraints
  = headway, platform occupation, possession blocks, dwell minima; objective = minimise total
  weighted delay (priority-weighted). Returns proposed `retiming` per run + explanation.
- `possession.py`: choose windows minimising weighted passenger impact subject to required work
  duration and capacity constraints. `slot.py`: allocate competing path requests by priority/fairness.
  `crew.py`: reassign to cover uncovered runs respecting compliance rules from 7.1.
- Each result is wrapped as an `ai_suggestion` with a human-readable, **auditable** explanation
  (constraints satisfied, objective value) — the "transparent optimisation" differentiator.

**Testing**:
- `Unit: two conflicting runs → retiming suggestion that, re-fed to ConflictDetector, yields zero conflicts.`
- `Unit: infeasible scenario → returns INFEASIBLE status with explanation, no crash.`
- `Unit: possession optimiser prefers the lower-passenger-impact window between two candidates.`
- `Unit: slot allocation respects priority ordering under contention.`

#### 8.3 — What-if replanning, forecasting & predictive ML

**What**: Live "what-if" disruption replanning (the underserved gap), demand forecasting feeding
capacity, and predictive-maintenance models served via ONNX.

**Design**:
- What-if: clone the current `service_day` state into a sandbox scenario, apply a disruption +
  candidate dispatcher actions, run propagation (5.3) + optimisation (8.2), and diff projected KPIs
  vs the live baseline. `POST /scenarios/what-if` → `200 ScenarioResult`.
- Demand forecast: scikit-learn model on historical run load factors → per-run `pax` forecast feeding
  capacity decisions; served from `ai/ml/`.
- Predictive maintenance: model consuming `unit.sensor_reading` event histories → component RUL /
  failure probability, emitting `unit.failure_predicted` → predictive `maintenance_record` (7.2).
- Generative scenario library for climate-resilience (heat buckling, flooding, leaf fall) seeds
  disruptions for what-if.

**Testing**:
- `Integration: what-if injecting a section blockage → ScenarioResult shows increased projected delay vs baseline; suggested reroute reduces it.`
- `Unit (fixture history): demand model predicts within tolerance on a holdout run.`
- `Unit: sensor history breaching learned threshold → unit.failure_predicted with RUL and probability.`

---

## Phase 9: Web UI — Train-Graph Editor, Network Map & Control Room

### Purpose
Deliver the lightweight, web-based UX that is the project's core differentiator against power-user
desktop incumbents: an interactive train-graph (time–distance) editor with live conflict overlay, a
geographic network/capacity map, and a real-time control-room board.

### Tasks

#### 9.1 — Train-graph (time–distance) editor

**What**: D3-based time–distance diagram where planners view and drag train paths, with conflicts
overlaid live.

**Design**:
- `web/src/features/train-graph`: x-axis = time, y-axis = distance along a chosen corridor (ordered
  stations); each run drawn as a polyline through its path segments. Dragging a node adjusts the
  scheduled time (PATCH path); on drop, refetch conflicts (3.2) and highlight conflicting segments
  red with tooltips (and AI suggestion from 8.2 when present).
- Optimistic update via Zustand; server reconciliation via TanStack Query.

**Testing**:
- `Vitest: time/distance scales map segment times → correct pixel coordinates.`
- `Vitest: conflict list → correct segments flagged.`
- `Playwright E2E: load corridor, drag a path to create a conflict → red overlay + tooltip appear; undo restores.`

#### 9.2 — Network/capacity map & control-room board

**What**: MapLibre geographic view coloured by capacity utilisation/condition, and a live board over
the Phase 5 WebSocket.

**Design**:
- Network map renders PostGIS section geometries; choropleth by UIC 406 `utilisation_pct` or asset
  condition; clicking a section opens its capacity result and possessions.
- Control room: live train list/positions with delay, disruption banner, possession overlay; consumes
  `WS /ws/control-room`; safety-critical maintenance alerts surfaced.

**Testing**:
- `Vitest: utilisation→colour mapping correct at threshold boundaries.`
- `Playwright E2E (mock WS): position update → train row delay updates live; disruption event → banner shows.`

---

## Phase 10: Multi-Operator Exchange, MCP Server & Hardening

### Purpose
Finish the interoperability and AI-access story and harden for regulated deployment: TAF/TAP TSI
cross-operator exchange, the MCP server for natural-language analytics (the greenfield gap), GraphQL
analytics, and security/compliance hardening (OWASP ASVS, IEC 62443, NIS2, GDPR, EN 50126 evidence).

### Tasks

#### 10.1 — TAF/TAP TSI exchange & multi-operator slot allocation

**What**: Cross-operator data exchange messages (TAF/TAP TSI) and the path-request → allocation
workflow across operators sharing infrastructure.

**Design**:
- `integrations/taf_tap`: generate/consume TAF/TAP TSI messages (train composition, path request,
  path details, train running information) over the operator boundary.
- Path-request workflow: a TOC submits a path request to an infrastructure manager; conflict check +
  slot optimisation (8.2) proposes allocation; acceptance creates the service/run. Audited end-to-end.

**Testing**:
- `Unit: path request → TAF/TAP message validates against schema.`
- `Integration: two operators requesting the same slot → optimiser allocates per priority/fairness; loser gets an alternative.`

#### 10.2 — MCP server & GraphQL analytics

**What**: An MCP server exposing read models for natural-language operational querying, plus a GraphQL
endpoint for flexible analytics dashboards.

**Design**:
- `mcp/server.py` tools: `query_trains(filters)`, `network_status()`, `capacity(section, window)`,
  `disruptions(active)`, `daily_performance(date)` — all operator-scoped and read-only, backed by the
  read models. Enables prompts like "show trains delayed >10 min on the Brighton Main Line in the last
  hour" (the research's stated NL example).
- GraphQL (Strawberry) over `rm_*` read models for dashboards; persisted queries only in prod.

**Testing**:
- `Integration: MCP query_trains(delayed>10m, line) → matches the SQL read-model result.`
- `Unit: MCP tools reject cross-operator access.`
- `Integration: GraphQL delayed-trains query equals REST equivalent.`

#### 10.3 — Security, compliance & RAMS hardening

**What**: Security controls and compliance evidence for regulated deployment.

**Design**:
- OWASP ASVS pass: authz tests on every route, rate limiting, input validation, secrets in a vault,
  audit_log on all mutating actions. mTLS enforced on trackside adapters (RFC 8705). GDPR: crew PII
  field-level encryption + retention/erasure jobs. IEC 62443/NIS2: network segmentation guidance,
  signed audit log, security event telemetry via OTel. EN 50126: ADRs in `docs/adr/` and traceability
  matrix linking requirements → code → tests as RAMS evidence (the platform stays advisory/out-of-SIL,
  documented explicitly).
- `SECURITY.md`, threat model, and a compliance checklist generated in CI.

**Testing**:
- `Integration: every mutating endpoint writes an audit_log row (parametrised sweep).`
- `Integration: trackside adapter without client cert → connection rejected.`
- `Integration: GDPR erasure request → crew PII redacted, audit entry retained.`
- `Security: ASVS automated checks (auth, headers, rate-limit) green; dependency/SAST scan clean.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Topology & Data Model        ── required by everything
    │
Phase 2: Infrastructure & Fleet CRUD + RailML     ── requires P1
    │
Phase 3: Timetabling & Conflict Detection         ── requires P2
    │
    ├── Phase 4: Capacity & Possession Planning    ── requires P3
    │
    ├── Phase 5: Real-Time, Event Store & Live Ops ── requires P2 (P3 for conflict-aware delay)
    │       │
    │       └── Phase 6: Passenger Info Export      ── requires P3 (static) + P5 (realtime)
    │
    └── Phase 7: Crew, Maintenance & Predictive     ── requires P2 (P3 for run links)

Phase 8: AI Augmentation & Optimisation           ── requires P3, P4, P5, P7
    │
Phase 9: Web UI (train-graph, map, control room)  ── requires P3, P4, P5 (P8 for suggestion overlay)
    │
Phase 10: Multi-Operator, MCP & Hardening         ── requires P3–P8 (MCP needs read models from P5)
```

**Parallelism opportunities (after Phase 3):**
- **Phase 4** (capacity/possessions), **Phase 5** (real-time), and **Phase 7** (crew/maintenance) can
  be developed concurrently — they share the Phase 2/3 schema but touch independent domain modules.
- **Phase 6** (passenger export) can start its static-GTFS half right after Phase 3, in parallel with
  Phase 5; its real-time half (GTFS-RT/SIRI) waits for Phase 5.
- **Phase 9** (web UI) front-end work can begin against Phase 3/4/5 APIs in parallel with Phase 7/8
  backend work, using mocked endpoints for not-yet-built features.
- Within **Phase 8**, the OR-Tools optimisers (8.2) and the ML models (8.3) are independent of each
  other once the suggestion plumbing (8.1) exists.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented behind the documented APIs/interfaces.
2. All unit and integration tests for the phase pass; coverage on new domain logic ≥ 85%.
3. `ruff check` and `ruff format --check` pass; `mypy --strict src` passes (frontend: ESLint +
   `tsc --noEmit` pass).
4. `docker compose build` succeeds and `docker compose up` brings the stack healthy
   (`/healthz` + `/readyz` green).
5. The phase's headline capability works end-to-end against the committed fixtures (RailML, GTFS,
   recorded feeds) — demonstrated by at least one E2E/integration test.
6. New Alembic migration(s) created, and `upgrade head` → `downgrade base` → `upgrade head` is clean.
7. New REST endpoints appear in the auto-generated OpenAPI 3.1 spec; new event channels appear in the
   AsyncAPI 3.0 spec; both snapshots committed under `docs/`.
8. New config options documented in `.env.example` and the README.
9. All mutating endpoints write to `audit_log`/`domain_events`; authorization is enforced and tested.
10. Standards touched by the phase are validated (RailML against XSD, GTFS against gtfs-validator,
    GTFS-RT protobuf round-trip, SIRI/NeTEx schema-valid) where applicable.
11. An ADR is recorded in `docs/adr/` for any non-trivial design decision (EN 50126 RAMS evidence).
```
