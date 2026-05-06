# Rail Network Operations

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-augmented platform for train scheduling, capacity management, and maintenance planning across the full rail network lifecycle.

Rail Network Operations is a candidate project to build an integrated, standards-based alternative to the closed proprietary suites that dominate rail operations today. It is aimed at infrastructure managers, train operators, and rail consultancies that need to coordinate timetables, capacity, possessions, rolling-stock maintenance, and real-time control without locking into a single vendor's monolithic stack.

---

## Why Rail Network Operations?

- Incumbent platforms (HAFAS/TPS, RailSys, Trapeze, DELMIA Rail Planning, Hitachi TMS, Thales ARAMIS) are closed-source, carry multi-million-euro licensing, and impose long, expensive integration projects.
- Power-user desktop clients and dated UX (e.g. HAFAS, RailSys) create steep learning curves and slow onboarding for smaller operators and academics.
- Most tools still produce overnight plans; live "what-if" replanning during incidents is an underserved capability.
- Heterogeneous rolling-stock fleets are poorly served because predictive-maintenance offerings (e.g. Alstom Flexcare) work best when paired with a single OEM's hardware.
- Information silos between scheduling, maintenance, and operations control slow decision-making at exactly the moments when speed matters most.

---

## Key Features

### Timetabling and Capacity

- Web-based graphical timetable (train-graph) editor with conflict detection
- Capacity utilisation analysis aligned with UIC 406-style methods
- Path request management and multi-operator slot allocation
- Possession planning for track access during maintenance windows
- Scenario comparison for alternative timetable variants

### Simulation

- Microsimulation of train runs respecting signalling logic and gradients
- Stochastic delay propagation analysis
- ETCS-aware simulation extensions
- Conflict detection grounded in train dynamics and platform occupation

### Real-Time Operations

- Live train tracking via vendor-neutral feed adapters
- Delay propagation modelling and automated re-pathing/re-timing
- Crew and rolling-stock rostering adjustments during disruption
- Operator decision support for dispatchers

### Maintenance and Assets

- Track, signalling, and structure inspection workflows
- Defect recording and prioritisation
- Predictive-maintenance hooks for rolling stock based on sensor and odometer data
- Maintenance work-order management and parts inventory integration

### Crew and Passenger Information

- Duty planning with rule-based legal-compliance checks
- Disruption recovery rostering
- GTFS / GTFS-RT export for passenger information systems
- Revised timetable publication and delay notifications

---

## AI-Native Advantage

AI is positioned as augmentation rather than replacement of safety-critical control logic. Candidate capabilities include conflict-resolution suggestions during live operations learned from dispatcher behaviour, natural-language querying of operational data, demand forecasting that feeds capacity allocation, predictive-maintenance models trained across heterogeneous fleets, and generative scenario planning for disruption response (strikes, weather, climate-resilience events such as heat buckling, flooding, and leaf fall).

---

## Tech Stack & Deployment

The platform is intended to be web-first and modular, in contrast to monolithic incumbents. It builds on open data standards: RailML for infrastructure and timetable models, GTFS and GTFS-RT for passenger-facing exchange, and TAF/TAP TSI for cross-operator data. Adapters are required to bridge legacy SCADA, interlocking, and traffic-management protocols. Safety-critical components must respect CENELEC EN 50128/50129 development processes where deployed in regulated environments, and IEC 62443 cybersecurity controls apply to trackside-connected systems.

---

## Market Context

Gartner tracks the category as "Rail Operation Management Systems," and analyst attention is shifting from batch planning-cycle tools toward real-time decisioning, automation, and analytics. The incumbent landscape is split between European interoperability-focused vendors (Siemens/HaCon, RMcon, Hitachi, Thales, Alstom), North American freight-oriented vendors (Wabtec, Oliver Wyman MultiRail, CloudMoyo), and PLM-integrated platforms (Dassault DELMIA). Primary buyers are infrastructure managers, passenger and freight train operating companies, and rail consultancies.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
