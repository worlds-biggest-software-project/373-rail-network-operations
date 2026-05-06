# Standards & API Reference

> Project: Rail Network Operations · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards
- **ISO/IEC 27001 — Information Security Management** — https://www.iso.org/standard/27001 — Required by many infrastructure managers for any system that processes operational or safety-related data.
- **ISO 55000 / 55001 — Asset Management** — https://www.iso.org/standard/55088.html — Framework for managing physical rail assets (track, rolling stock, structures); aligns with maintenance modules.
- **ISO 9001 — Quality Management** — https://www.iso.org/iso-9001-quality-management.html — Common procurement prerequisite for rail-software vendors.
- **ISO 8601 — Date and Time** — https://www.iso.org/iso-8601-date-and-time-format.html — Required for unambiguous timetable representation across time zones and DST transitions.

### CENELEC / IEC Rail-Specific Standards
- **EN 50126 — RAMS (Reliability, Availability, Maintainability, Safety) for Railway Applications** — https://standards.cencenelec.eu/ — Lifecycle process for railway systems.
- **EN 50128 — Software for Railway Control and Protection Systems** — https://standards.cencenelec.eu/ — Software safety integrity levels (SIL); applies to scheduling/control software interacting with safety functions.
- **EN 50129 — Safety-Related Electronic Systems for Signalling** — https://standards.cencenelec.eu/ — Hardware safety case requirements adjacent to TMS.
- **EN 50159 — Safety-Related Communication in Transmission Systems** — https://standards.cencenelec.eu/ — Used for trackside-to-control-centre data integrity.
- **IEC 62443 — Industrial Automation & Control Systems Security** — https://www.iec.ch/cyber-security — De facto cybersecurity baseline now mandated for many rail control systems.
- **IEC 61375 — Train Communication Network (TCN)** — https://webstore.iec.ch/publication/61375 — Onboard data network used by predictive-maintenance feeds.

### European Interoperability (TSI) Standards
- **TSI CCS (Control-Command and Signalling)** — https://www.era.europa.eu/ — Defines ERTMS/ETCS and GSM-R/FRMCS interoperability requirements.
- **TAF TSI — Telematics Applications for Freight** — https://www.era.europa.eu/domains/telematic-applications/taf_en — Mandatory data exchange between RUs and IMs for freight.
- **TAP TSI — Telematics Applications for Passengers** — https://www.era.europa.eu/domains/telematic-applications/tap_en — Passenger timetable, tariff, reservation and real-time information exchange.
- **ERTMS / ETCS Subsets (e.g. SUBSET-026)** — https://www.era.europa.eu/domains/european-rail-traffic-management-system-ertms_en — System Requirements Specification for European Train Control System.

### Data Model & Exchange Standards
- **RailML 3.x** — https://www.railml.org/ — XML schema for infrastructure, rolling stock, timetable and interlocking data; widely used as industry interchange.
- **railTopoModel (IRS 30100, UIC)** — https://www.uic.org/ — Topological data model underlying RailML 3 infrastructure.
- **GTFS (General Transit Feed Specification)** — https://gtfs.org/schedule/reference/ — Open schedule format consumed by most journey planners.
- **GTFS-Realtime** — https://gtfs.org/realtime/reference/ — Trip updates, vehicle positions, service alerts in protobuf.
- **NeTEx (CEN TS 16614)** — https://netex-cen.eu/ — European standard for exchanging public transport network and timetable data.
- **SIRI (CEN TS 15531)** — https://www.siri-cen.eu/ — Real-time public transport information service interface.
- **OCIT-O / OCIT-C** — https://www.ocit.org/ — Common protocols in German-speaking markets for traffic-control communications.
- **RailwayML / Eulynx** — https://www.eulynx.eu/ — Standard interfaces between signalling subsystems.

### API & Service Specifications
- **OpenAPI 3.1** — https://spec.openapis.org/oas/v3.1.0 — Recommended for the platform's REST surface.
- **AsyncAPI 3.0** — https://www.asyncapi.com/docs/reference/specification/v3.0.0 — Event-driven specification for streaming train-position and alert data.
- **JSON Schema 2020-12** — https://json-schema.org/specification — Validation for configuration and exchange payloads.
- **GraphQL** — https://spec.graphql.org/ — Flexible queries for analytics dashboards.
- **gRPC / Protocol Buffers** — https://grpc.io/ — Common for high-frequency trackside telemetry.

### Security & Authentication Standards
- **OAuth 2.0 (RFC 6749) and OAuth 2.1 draft** — https://datatracker.ietf.org/doc/html/rfc6749 — Authorisation framework for B2B integrations.
- **OpenID Connect Core 1.0** — https://openid.net/specs/openid-connect-core-1_0.html — Operator/dispatcher SSO.
- **mTLS (RFC 8705)** — https://datatracker.ietf.org/doc/html/rfc8705 — Service-to-service trust for trackside connections.
- **OWASP ASVS 4.x** — https://owasp.org/www-project-application-security-verification-standard/ — Security verification baseline.
- **NIST SP 800-82r3 — Industrial Control Systems Security** — https://csrc.nist.gov/publications/detail/sp/800-82/rev-3/final — Reference for OT/IT boundaries.
- **NIS2 Directive (EU 2022/2555)** — https://eur-lex.europa.eu/eli/dir/2022/2555/oj — Cybersecurity obligations for transport operators.
- **GDPR (EU 2016/679)** — https://eur-lex.europa.eu/eli/reg/2016/679/oj — Applies to crew, passenger, and CCTV data.

### MCP Server Specifications
- **Model Context Protocol (MCP)** — https://modelcontextprotocol.io/ — Could be used to expose timetable, capacity, and incident data to AI assistants for natural-language analytics and decision support.

## Similar Products — Developer Documentation & APIs

### HAFAS / HaCon
- **Description:** Long-established timetable, routing, and passenger-information platform from Siemens/HaCon.
- **API Documentation:** https://www.hacon.de/en/portfolio/data-services/ (commercial; partner-only detailed docs)
- **SDKs/Libraries:** Community-reverse-engineered clients (e.g., `hafas-client` https://github.com/public-transport/hafas-client)
- **Developer Guide:** https://github.com/public-transport/hafas-client/tree/main/docs
- **Standards:** Bespoke JSON over HTTPS; TAP TSI exports
- **Authentication:** API key per integration

### Deutsche Bahn Open-Data API
- **Description:** Public station, timetable, and disruption data from Deutsche Bahn.
- **API Documentation:** https://developers.deutschebahn.com/
- **SDKs/Libraries:** Community SDKs (Node, Python)
- **Developer Guide:** https://developers.deutschebahn.com/db-api-marketplace/apis/
- **Standards:** REST/JSON, some XML legacy
- **Authentication:** API key, OAuth 2.0

### National Rail Enquiries (UK) Darwin / RTT
- **Description:** UK real-time train running data and timetable feeds.
- **API Documentation:** https://wiki.openraildata.com/index.php/Darwin
- **SDKs/Libraries:** OpenLDBWS clients in multiple languages
- **Developer Guide:** https://wiki.openraildata.com/
- **Standards:** SOAP/XML (Darwin), STOMP feeds (TRUST, TD)
- **Authentication:** Token-based

### SBB (Swiss Federal Railways) Open Data
- **Description:** Comprehensive Swiss rail timetable, real-time, and infrastructure data.
- **API Documentation:** https://data.sbb.ch/
- **SDKs/Libraries:** GTFS / GTFS-RT consumers; Transport API community wrappers
- **Developer Guide:** https://developer.sbb.ch/
- **Standards:** GTFS, GTFS-RT, NeTEx, SIRI
- **Authentication:** API key for some endpoints; many open

### Open Transport Data Switzerland (opentransportdata.swiss)
- **Description:** National public-transport data hub.
- **API Documentation:** https://opentransportdata.swiss/en/cookbook/
- **SDKs/Libraries:** Standard GTFS/NeTEx tooling
- **Developer Guide:** https://opentransportdata.swiss/en/
- **Standards:** GTFS, GTFS-RT, NeTEx, SIRI
- **Authentication:** API key

### Trafiklab (Sweden)
- **Description:** Swedish public transport open API platform.
- **API Documentation:** https://www.trafiklab.se/api/
- **SDKs/Libraries:** GTFS standard libraries
- **Developer Guide:** https://www.trafiklab.se/docs/
- **Standards:** GTFS, GTFS-RT, SIRI
- **Authentication:** API key

### Network Rail Open Data (UK)
- **Description:** UK infrastructure manager feeds (TRUST, TD, SCHEDULE, VSTP).
- **API Documentation:** https://wiki.openraildata.com/index.php/Network_Rail_Feeds
- **SDKs/Libraries:** Community STOMP / NROD clients
- **Developer Guide:** https://wiki.openraildata.com/
- **Standards:** STOMP, JSON, CIF (schedule)
- **Authentication:** Account credentials

### Amtrak / FRA APIs (US)
- **Description:** US passenger and freight regulator/operator data feeds.
- **API Documentation:** https://www.fra.dot.gov/ (datasets); Amtrak operates limited public APIs
- **SDKs/Libraries:** Community Amtrak status scrapers
- **Developer Guide:** https://railroads.dot.gov/
- **Standards:** REST/CSV downloads; GTFS for some operators
- **Authentication:** Mostly open

### TransitFeeds / MobilityData
- **Description:** Aggregator of GTFS and GTFS-RT feeds worldwide; reference validators.
- **API Documentation:** https://mobilitydatabase.org/
- **SDKs/Libraries:** `gtfs-validator` https://github.com/MobilityData/gtfs-validator
- **Developer Guide:** https://mobilitydata.org/
- **Standards:** GTFS, GTFS-RT
- **Authentication:** API key (for hosted service)

### OpenTripPlanner
- **Description:** Open-source multimodal journey planner.
- **API Documentation:** https://docs.opentripplanner.org/en/latest/apis/
- **SDKs/Libraries:** Java core; REST & GraphQL APIs
- **Developer Guide:** https://docs.opentripplanner.org/
- **Standards:** GTFS, GTFS-RT, NeTEx, SIRI, OSM
- **Authentication:** Configurable

### CloudMoyo Rail
- **Description:** Cloud-native rail operations management with REST APIs for integration.
- **API Documentation:** https://rail.cloudmoyo.com/ (partner-only)
- **SDKs/Libraries:** None public
- **Developer Guide:** Partner portal
- **Standards:** REST/JSON
- **Authentication:** OAuth 2.0

## Notes

The rail-software ecosystem combines mature European interoperability standards (TSI CCS, TAF/TAP TSI, RailML, NeTEx, SIRI) with North American conventions (GTFS-derived, AAR data formats). Open APIs are rich on the passenger-information and infrastructure-data side (DB, SBB, Network Rail, MobilityData) but are typically gated for traffic-management and safety-critical systems. Emerging areas include FRMCS (5G replacement for GSM-R), Eulynx standardised signalling interfaces, and cross-border ETCS data exchange — all of which are still evolving and worth ongoing monitoring. MCP server exposure of operational data to AI assistants is a greenfield opportunity not addressed by any incumbent vendor.
