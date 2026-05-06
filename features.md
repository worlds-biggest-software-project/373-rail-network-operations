# Rail Network Operations — Feature & Functionality Survey

> Candidate #373 · Researched: 2026-05-06

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| HAFAS / TPS (Siemens / HaCon) | Timetable planning, optimisation, passenger information | Commercial, proprietary | https://www.hacon.de/ |
| RailSys (RMcon) | Timetabling, capacity, microsimulation | Commercial, proprietary | https://www.rmcon-int.de/railsys-en/ |
| OpenTrack | Microsimulation of railway operations | Commercial (academic origin), proprietary | https://www.opentrack.ch/ |
| Trapeze Rail (Modaxo) | Scheduling, crew rostering, real-time ops | Commercial, proprietary | https://www.modaxo.com/ |
| DELMIA Rail Planning (Dassault) | Capacity planning, possession management | Commercial, enterprise SaaS / on-prem | https://www.3ds.com/products/delmia |
| CloudMoyo Rail Operations Mgmt | Cloud-native rail ops, freight + passenger | Commercial SaaS | https://rail.cloudmoyo.com/ |
| Alstom Flexcare | Rolling-stock predictive maintenance | Commercial service + platform | https://www.alstom.com/ |
| Oliver Wyman MultiRail | Freight network optimisation | Commercial (consulting + software) | https://www.oliverwyman.com/multirail |
| Wabtec / GE Trip Optimizer & NetworkOptimizer | Traffic management, energy-optimal driving | Commercial, proprietary | https://www.wabteccorp.com/ |
| Hitachi Rail Traffic Management System (TMS) | ETCS-aligned TMS, real-time control | Commercial, proprietary | https://www.hitachirail.com/ |
| Thales ARAMIS / NAIA | Traffic management, dispatch | Commercial, proprietary | https://www.thalesgroup.com/ |
| Trenolab | Open-source/open-data leaning consulting & tools | Mixed; some open tooling | https://www.trenolab.com/ |

## Feature Analysis by Solution

### HAFAS / TPS (Siemens / HaCon)
**Core features**
- Timetable construction and editing across long planning horizons
- Routing engine for passenger journey planning (used by Deutsche Bahn, SBB)
- Path conflict detection and resolution
- Multi-operator path allocation and slot management
- Integration with passenger information systems

**Differentiating features**
- Industry-leading routing engine reused across Europe's largest passenger railways
- Tight feedback loop between planning and operational data

**UX patterns**
- Power-user desktop clients for planners; web modules for downstream consumers
- Heavy reliance on graphical timetable (train graph) editors

**Integration points**
- HAFAS API used by countless travel-information apps; TAF/TAP TSI data exchange

**Known gaps**
- Dated client UX; long onboarding curve; expensive integration projects

**Licence / IP notes**
- Closed-source, multi-million-euro enterprise licensing

### RailSys (RMcon)
**Core features**
- Integrated timetable construction and capacity assessment (UIC 406)
- Microsimulation of train runs for conflict detection
- Possession and construction-window planning
- Stochastic delay propagation analysis

**Differentiating features**
- Strong academic lineage (University of Hannover); favoured by infrastructure managers for capacity studies

**UX patterns**
- Scenario tree management; side-by-side comparison of timetable variants

**Integration points**
- Imports from RailML; exports to operational TMS

**Known gaps**
- Steep learning curve; limited real-time operations module

**Licence / IP notes**
- Proprietary; named-user licensing

### OpenTrack
**Core features**
- High-fidelity microsimulation including signalling logic, train dynamics, ETCS
- Infrastructure modelling (tracks, signals, gradients)
- Stochastic simulation with Monte Carlo runs
- Animation of train movements and occupancy charts

**Differentiating features**
- Detailed signalling-system modelling including ETCS levels 1/2/3 and national systems
- Widely used by universities and consultancies — de facto reference for capacity studies

**UX patterns**
- Visual editor for infrastructure; scripted scenario runs

**Integration points**
- RailML import/export; bespoke COM/scripting

**Known gaps**
- Single-user desktop app; limited collaboration features; no live ops

**Licence / IP notes**
- Commercial proprietary (ETH spin-off)

### Trapeze Rail (Modaxo)
**Core features**
- Crew and duty rostering with rules-based legal compliance
- Vehicle/rolling-stock scheduling
- Real-time operations control and incident management
- Workforce mobile apps for drivers and conductors

**Differentiating features**
- Strong crew-rostering optimisation (genetic / constraint-based)

**UX patterns**
- Modular suite; web + mobile clients

**Integration points**
- Connectors to payroll, HR, fleet telematics

**Known gaps**
- Less depth on infrastructure-side capacity simulation

**Licence / IP notes**
- Proprietary; enterprise pricing

### DELMIA Rail Planning (Dassault)
**Core features**
- Long-term capacity planning at network and corridor level
- Possession planning aligned with track-renewal programmes
- Timetable scenario comparison
- Integration with 3DEXPERIENCE platform

**Differentiating features**
- Native link to broader engineering/PLM models of the network

**UX patterns**
- Web-based dashboards with role-specific views

**Integration points**
- 3DEXPERIENCE; ERP and asset systems

**Known gaps**
- Heavyweight platform commitment; rail module less mature than core PLM

**Licence / IP notes**
- Proprietary, subscription

### CloudMoyo Rail Operations Management System
**Core features**
- Cloud-native operations dashboards
- Crew, locomotive, and yard management
- Service-level reporting and KPIs
- Integration with safety and regulatory reporting

**Differentiating features**
- Built on Azure with modern data pipeline; preferred by North American Class I and short-line freight operators

**UX patterns**
- Browser-first responsive dashboards

**Integration points**
- REST APIs; PTC and dispatch system integrations

**Known gaps**
- Less coverage of European interoperability standards (ETCS, TAF/TAP TSI)

**Licence / IP notes**
- SaaS subscription

### Alstom Flexcare
**Core features**
- Predictive maintenance based on onboard sensor telemetry
- Maintenance work-order management
- Spare-parts inventory integration
- Reliability analytics and root-cause analysis

**Differentiating features**
- Leverages Alstom HealthHub IIoT platform; deep domain models for specific rolling-stock families

**UX patterns**
- Operations centre dashboards; mobile work-order apps

**Integration points**
- Onboard CCU data; ERP/EAM integration

**Known gaps**
- Best when paired with Alstom rolling stock; heterogeneous fleets harder

**Licence / IP notes**
- Service-led contract; proprietary

### Oliver Wyman MultiRail
**Core features**
- Freight network optimisation: blocking, train-makeup, routing
- Yard and terminal optimisation
- Service design and what-if analysis
- Demand-driven capacity allocation

**Differentiating features**
- Long heritage in North American freight; deep operations-research models

**UX patterns**
- Analyst-oriented; scenario workbench

**Integration points**
- Connects to TMS/dispatch; data warehouse exports

**Known gaps**
- Freight focus; limited passenger applicability

**Licence / IP notes**
- Proprietary; consulting-led delivery

### Wabtec NetworkOptimizer / Trip Optimizer
**Core features**
- Movement planning across freight networks
- Energy-optimal driving recommendations
- Locomotive utilisation optimisation
- Integration with PTC

**Differentiating features**
- Embedded in major Class I deployments; tight coupling to onboard locomotive systems

**UX patterns**
- Dispatcher consoles plus in-cab displays

**Integration points**
- Locomotive electronics; dispatching systems

**Known gaps**
- Closed ecosystem; vendor lock-in

**Licence / IP notes**
- Proprietary

### Hitachi Rail TMS
**Core features**
- Real-time traffic management aligned with ERTMS/ETCS
- Automatic route setting and conflict resolution
- Integration with interlocking and signalling
- Operator decision support

**Differentiating features**
- ETCS-native; deployed in major European projects (e.g. UK East Coast)

**UX patterns**
- Control-room workstations with large-screen overviews

**Integration points**
- Interlockings, ETCS RBC, passenger info

**Known gaps**
- Expensive deployments; long delivery cycles

**Licence / IP notes**
- Proprietary; integrator-delivered

### Thales ARAMIS / NAIA
**Core features**
- Traffic management for mainline and metro
- Conflict detection and resolution
- Decision-support tooling for dispatchers
- Integration with safety systems

**Differentiating features**
- Strong urban/metro footprint; CBTC integration

**UX patterns**
- Mission-critical control-room UI

**Integration points**
- Signalling, depot management, passenger info

**Known gaps**
- Tied to Thales signalling; high integration cost

**Licence / IP notes**
- Proprietary

### Trenolab tools
**Core features**
- Microsimulation, capacity studies, timetable robustness analysis
- Open-data and open-source orientation around RailML and GTFS

**Differentiating features**
- Bridges open-source and academic methods with consulting practice

**UX patterns**
- Notebook-driven analytics; some open libraries

**Integration points**
- RailML, GTFS, OpenStreetMap

**Known gaps**
- Smaller than incumbents; limited turn-key product

**Licence / IP notes**
- Mixed; some open-source components

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Graphical timetable (train-graph) editor with conflict detection
- Microsimulation of train runs respecting signalling and gradients
- Capacity assessment (UIC 406-style) and possession planning
- Real-time train tracking with delay propagation
- Crew/duty rostering with legal-rule compliance
- Rolling-stock maintenance scheduling and work orders
- RailML, GTFS, GTFS-RT, and TAF/TAP TSI import/export

### Differentiating Features
- ETCS-aware microsimulation and TMS integration
- Multi-operator path allocation and slot auctioning
- Predictive maintenance fed by onboard telemetry
- Energy-optimal driving advisories
- Cross-modal integration (rail + bus + on-demand)
- Open data exchange across operators and infrastructure managers

### Underserved Areas / Opportunities
- Open-source, modular alternative to monolithic incumbents
- Lightweight web-based timetable editors usable by smaller operators and academics
- Live "what-if" replanning when an incident occurs (most tools still produce overnight plans)
- Transparent, auditable optimisation explanations for regulators
- Cross-vendor integration for heterogeneous rolling-stock fleets
- Climate-resilience scenario planning (heat buckling, flooding, leaf fall)

### AI-Augmentation Candidates
- Conflict resolution suggestions during live operations using learned dispatcher behaviour
- Natural-language querying of operational data ("show me trains delayed >10 min on the Brighton Main Line in the last hour")
- Demand forecasting feeding capacity allocation decisions
- Predictive maintenance models trained on cross-fleet sensor data
- Automated possession-window optimisation balancing maintenance, capacity, and passenger impact
- Generative scenario planning for disruption response (strikes, weather)
- LLM-assisted incident reporting and lessons-learned synthesis

## Legal & IP Summary

The category is dominated by closed-source proprietary platforms with substantial patent footprints around traffic-management algorithms, ETCS integration (Wabtec, Hitachi, Thales, Siemens), and onboard predictive-maintenance pipelines (Alstom, Wabtec). Any new entrant must avoid reproducing patented dispatching or movement-authority logic and should rely instead on published interoperability standards (RailML, GTFS, TAF/TAP TSI, ERTMS public specifications). Open data formats — RailML (CC-BY-derived community licence), GTFS (open), and OpenStreetMap rail data (ODbL) — are safe building blocks. Care is required when ingesting commercial schedule feeds, which often carry restrictive redistribution terms. No copyright-restricted material was incorporated into this survey; all features described are derived from public marketing pages, vendor datasheets, and academic literature.

## Recommended Feature Scope

**Must-have (MVP)**
- RailML-based infrastructure and timetable data model
- Web-based graphical timetable editor with conflict detection
- Capacity utilisation analysis (UIC 406-style)
- Possession planning workflow
- GTFS / GTFS-RT export for passenger information
- Real-time train tracking ingestion (vendor-neutral feed adapter)

**Should-have (v1.1)**
- Microsimulation engine for run-time and conflict analysis
- Crew rostering with rule-based compliance checks
- Predictive-maintenance hooks for rolling stock
- AI-assisted what-if replanning during disruptions
- Multi-operator slot allocation workflow

**Nice-to-have (backlog)**
- ETCS-aware simulation extensions
- Climate-resilience scenario library
- Natural-language operational-data querying
- Cross-modal (rail + bus + micromobility) capacity coordination
- Generative incident-response playbooks
