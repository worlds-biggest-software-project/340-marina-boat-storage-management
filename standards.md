# Standards & API Reference

> Project: Marina & Boat Storage Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 8666:2016 — Small Craft: Principal Data**
- URL: https://www.iso.org/standard/65424.html
- Defines standard measurement terms and procedures for principal dimensions of small craft with a hull length up to 24 m, including Length Overall (LOA), beam, draft, sail areas, and displacement. LOA is the primary unit used for marina slip sizing and tariff calculation, making this standard foundational to any vessel-data model in a marina system.

**ISO 12217-1:2022 — Small Craft: Stability and Buoyancy Assessment and Categorisation (Part 1)**
- URL: https://cdn.standards.iteh.ai/samples/79072/e54afb50d5a8440fa23ed8df3e3e5f16/ISO-12217-1-2022.pdf
- Specifies stability and buoyancy requirements and categorisation (Design Categories A–D) for sailing and power boats. Marina systems storing vessel data for insurance, haul-out, and regulatory purposes should capture the design category and stability certificate information.

**ISO 6185-3:2014 — Inflatable Boats (Part 3)**
- URL: https://www.iso.org/standard/60201.html
- Covers inflatable and rigid inflatable boats (RIBs) with hull length under 8 m and motor ratings ≥15 kW. Relevant for storage classification of inflatable tender berths and rack storage tiers.

### W3C & IETF Standards

**RFC 5545 — Internet Calendaring and Scheduling Core Object Specification (iCalendar)**
- URL: https://www.rfc-editor.org/rfc/rfc5545
- Defines the iCalendar data format (`.ics`) for representing calendar events, reservations, and scheduling information. Marina slip reservation systems can use iCalendar to export and import bookings into boater and marina-manager calendar applications (Google Calendar, Outlook, Apple Calendar).

**RFC 9073 — Event Publishing Extensions to iCalendar**
- URL: https://www.ietf.org/rfc/rfc9073.html
- Extends RFC 5545 with a `STRUCTURED-DATA` property for embedding domain-specific data directly within calendar events. Useful for embedding vessel details, slip assignment, and fuel pre-order data inside reservation calendar objects.

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Defines HTTP method semantics (GET, POST, PUT, PATCH, DELETE) and status codes that underpin all REST API design for marina management platforms.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the `Link` header and link relation types used in hypermedia REST APIs. Recommended for pagination, resource linking, and HATEOAS patterns in marina management API responses.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Defines the OAuth 2.0 framework for delegated authorisation. Marina APIs exposing reservation, billing, and vessel data to third-party integrations (accounting tools, fuel management systems, boater apps) should implement OAuth 2.0 flows.

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Adds identity layer over OAuth 2.0. Required for secure marina SaaS multi-tenancy, enabling marina operators and boaters to authenticate via their own identity providers (Google, Microsoft Entra ID, etc.).

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for describing RESTful APIs in a machine-readable format. Marina management systems exposing public or partner APIs should publish an OpenAPI 3.1 definition for automated SDK generation, documentation portals, and contract testing.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Standard for describing and validating JSON data structures. Defines the shape of vessel records, slip records, reservation objects, and invoice data exchanged via marina APIs.

**iCalendar RFC 5545 / RFC 9073** — see W3C & IETF section above.

### Marine Electronics & Data Interface Standards

**NMEA 0183 (v4.30)**
- URL: https://www.nmea.org/nmea-0183.html
- ASCII serial communications protocol for marine electronics (GPS, sonar, AIS transponders, depth sounders). Version 4.30 (December 2023) adds support for GNSS constellations including GLONASS, Galileo, BeiDou, QZSS, and NavIC. Marina systems integrating with vessel navigation equipment or AIS vessel-tracking feeds rely on NMEA 0183 sentence parsing.

**NMEA 2000 / IEC 61162-3**
- URL: https://www.nmea.org/nmea-2000.html
- Plug-and-play CAN bus network standard for connecting marine sensors and display units aboard vessels. Standardised as IEC 61162-3. Marina service and haul-out workflows may receive engine diagnostic data, tank levels, and bilge-pump status via NMEA 2000.

**NMEA OneNet**
- URL: https://www.nmea.org/nmea-onenet.html
- Next-generation marine networking standard built on IPv6 over standard Ethernet (IEEE 802.3), positioned as the successor to NMEA 2000. Enables high-bandwidth transfer of sonar, radar, and video data from vessels to dock-side marina systems. Marina software should plan for OneNet device discovery and data ingestion as vessel fleets modernise.

**NMEA Cloud API**
- URL: https://www.nmea.org/nmea_cloud.html
- Open API initiative for developers, researchers, and agencies to build applications consuming standardised, real-time marine data. Relevant for marina systems aggregating vessel telemetry, weather, and environmental sensor feeds.

**IEC 61162-1 — Maritime Navigation Digital Interfaces (Part 1: Single Talker)**
- URL: https://standards.globalspec.com/std/1298862/IEC%2061162-1
- Governs data communication between maritime electronic instruments for one-way serial transmission from a single talker to one or more listeners. The foundational IEC framing of the NMEA 0183 protocol.

**IEC 61162-450 / IEC 61162-460:2024 — Ethernet Marine Networks**
- URL: https://actisense.com/news/iec-61162-450-460-ethernet-marine-networks/
- IEC 61162-450 defines Ethernet-based high-speed shipboard digital communication. IEC 61162-460 (3rd edition, April 2024) adds enhanced safety and cybersecurity requirements. Relevant for modern marina-side equipment talking to vessels equipped with IEC 61162-450 Ethernet networks.

### Security & Compliance Standards

**PCI DSS v4.0 — Payment Card Industry Data Security Standard**
- URL: https://www.pcisecuritystandards.org/
- Mandatory for any marina system processing credit/debit card payments (slip rentals, fuel sales, boat storage fees, work orders, ship-store POS). Requires encrypted cardholder data at rest and in transit, tokenisation, and regular security audits. PCI DSS v4.0 is fully in force as of March 2025, deprecating v3.2.1.

**EMV (Europay, Mastercard, Visa) Chip Standard**
- URL: https://www.emvco.com/
- De-facto global standard for chip-based card transactions at marina POS terminals and fuel dock card readers. Marina software must ensure POS integrations are EMV-compliant to avoid chargeback liability shifts.

**3D Secure 2.0 (EMV 3-D Secure)**
- URL: https://www.emvco.com/emv-technologies/3d-secure/
- Cardholder authentication protocol for card-not-present marina transactions (online slip reservations, transient bookings, deposit collection). Reduces fraud liability for marinas accepting online payments.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr.eu/
- Applies to any marina operator storing personal data of EU/EEA residents. Governs collection of boater names, addresses, vessel details, bank information, and location data. Key requirements: lawful basis for processing, right to erasure, data breach notification within 72 hours, data minimisation. NMMA guidance (2018 and updated) addresses GDPR applicability across the boating industry.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/www-project-api-security/
- Lists the top API security risks including broken object-level authorisation, broken authentication, and unrestricted resource consumption. Critical reference for marina APIs exposing vessel locations, customer financial records, and access-gate control integrations.

**NIST SP 800-63B — Digital Identity Guidelines (Authentication)**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- US federal guidance on authentication assurance levels. Relevant for marina systems used by government-owned marinas (National Park Service, Army Corps of Engineers) or subject to federal procurement requirements.

### Environmental & Regulatory Standards

**EPA Clean Marina Program Guidelines**
- URL: https://www.epa.gov/npdes/clean-marinas
- Voluntary best-management-practices framework covering fuel spill prevention, sewage pump-out, waste oil collection, and stormwater management. Marina compliance-tracking modules should align data fields with EPA Clean Marina audit checklists.

**USCG Hull Identification Number (HIN) Standard**
- URL: https://uscgboating.org/regulations/State-Guidance/HIN/
- Federal regulation (effective 1972, revised 1984) requiring a 12-character HIN on every manufactured recreational vessel. The HIN is the primary unique vessel identifier used across US state registration systems, insurance records, and marina vessel-record databases. Marina software must validate and store HINs in conformance with USCG MIC-prefix format rules.

---

## Similar Products — Developer Documentation & APIs

### Harbour Assist

- **Description:** UK/Australia-origin cloud marina management platform covering berth management, billing, customer portal, and work orders. Used by harbour authorities and commercial marinas internationally.
- **API Documentation:** https://developer.harbourassist.com/
- **APIs List:** https://developer.harbourassist.com/apis
- **Developer Guide:** https://harbourassist.com/api/ — covers operational, vessel, customer, and invoice data; API keys issued on request via an early-adopter programme.
- **Standards:** REST/JSON; hosted on Microsoft Azure API Management for security, load balancing, and scalability.
- **Authentication:** API key (passed as header or query parameter).

### Marinas.com

- **Description:** Boater-facing marina directory and slip-reservation marketplace. Provides a public API for marina point-of-interest data, enabling third-party apps and chartplotters to display marina locations, amenities, and services.
- **API Documentation:** https://marinas.com/developers/api_documentation
- **Standards:** REST/JSON; returns `web_url`, `api_url`, and `icon_url` hyperlinks on all point-of-interest records to support database traversal.
- **Authentication:** API key passed as `access_token` query parameter or `Authorization` header.

### FuelCloud

- **Description:** Cloud-based fuel management platform used at marina fuel docks to monitor dispensing events, track inventory levels, and integrate fuel transactions with marina billing systems.
- **API Documentation:** https://developer.fuelcloud.com/ (access by request)
- **Integration Notes:** Integrates natively with Dockwa POS and Molo/Storable Marine. Data flow: FuelCloud dispenser events → marina management billing ledger via API.
- **Standards:** REST/JSON.
- **Authentication:** API key; access granted by contacting support@fuelcloud.com.

### Dockwa

- **Description:** Marina management and transient-dockage marketplace platform. Provides reservation, contract, and payment management for 1,000+ marinas. Engineering blog and GitHub organisation expose integration patterns.
- **API Reference:** https://api.dockwa.com/
- **Engineering Blog:** https://engineering.dockwa.com/
- **GitHub Organisation:** https://github.com/dockwa
- **Standards:** REST/JSON.
- **Authentication:** Not publicly documented; integration access via partner arrangements.

### DockMaster

- **Description:** Marine ERP platform for marinas, boatyards, and dealerships with 40+ years of domain logic. Offers REST API integrations with CRM (Salesforce), accounting, and partner tools.
- **API Overview:** https://www.dockmaster.com/blog/supporting-marine-industry-apis-integrations/
- **Integration Partners:** https://www.dockmaster.com/integration-partners
- **Community SDK:** https://github.com/excellenteasy/dockmaster-api (Node.js wrapper, community-maintained, not for production use)
- **Standards:** REST/JSON.
- **Authentication:** Not publicly documented; integration access via DockMaster partner programme.

### Molo / Storable Marine

- **Description:** Cloud marina management platform (Molo, acquired by Storable) with slip management, dry stack, fuel, billing, and work orders. Supports outbound JSON API for custom integrations to any cloud platform.
- **Integration Hub:** https://getmolo.com/connect/
- **Documentation:** Not publicly available; outbound JSON API access via Storable Marine account settings.
- **Key Ecosystem Integrations:** QuickBooks, Xero (accounting); FuelCloud (fuel); SpeedyDock (dry stack); Snag-A-Slip (transient bookings); Slack (notifications).
- **Standards:** REST/JSON.
- **Authentication:** Undisclosed; integration credentials via Storable Marine account.

### QuickBooks Online (Intuit)

- **Description:** Widely used SME accounting platform. Marina billing systems commonly sync invoices, payments, and customer records to QuickBooks via its REST API.
- **API Documentation:** https://developer.intuit.com/app/developer/qbo/docs/api/accounting/all-entities
- **OAuth 2.0 Guide:** https://developer.intuit.com/app/developer/qbo/docs/develop/authentication-and-authorization/oauth-2.0
- **SDKs:** JavaScript, Python, PHP, Java, .NET — https://developer.intuit.com/app/developer/qbo/docs/develop/sdks-and-samples
- **Standards:** REST/JSON, OpenAPI; webhooks for real-time sync.
- **Authentication:** OAuth 2.0 (access tokens expire in 1 hour; refresh tokens valid for 100 days on a rolling basis).

### Xero

- **Description:** Cloud accounting platform popular with marinas and boatyards in the UK, Australia, and New Zealand. Marina management platforms (Molo, Harbour Assist) sync invoices and payments via the Xero API.
- **API Documentation:** https://developer.xero.com/documentation/api/accounting/overview
- **OAuth 2.0 Guide:** https://developer.xero.com/documentation/guides/oauth2/overview/
- **SDKs:** JavaScript, Python, Ruby, .NET, Java, PHP — https://developer.xero.com/documentation/libraries/
- **Standards:** REST/JSON, OpenAPI 3.0; webhook event notifications available.
- **Authentication:** OAuth 2.0 (access tokens expire every 30 minutes; automatic token refresh required).

### Stripe

- **Description:** Payments infrastructure platform. Marina software uses Stripe for slip rental billing, transient dockage deposits, subscription contracts, and fuel sale card processing.
- **API Reference:** https://docs.stripe.com/api
- **Billing / Subscriptions Docs:** https://docs.stripe.com/billing
- **SDKs:** JavaScript, Python, Ruby, PHP, Java, Go, .NET — https://docs.stripe.com/libraries
- **Standards:** REST/JSON; OpenAPI spec published; webhooks via Stripe dashboard.
- **Authentication:** API key (secret key for server-side; publishable key for client-side). Supports 3D Secure 2.0 for card-not-present authentication.
- **PCI Compliance:** Stripe is a PCI DSS Level 1 certified service provider; using Stripe eliminates most PCI compliance burden for marina operators.

---

## Notes

**Emerging standard — NMEA OneNet adoption:** NMEA OneNet is in early-adopter phase as of 2026. Marina management software vendors should monitor NMEA OneNet device certification progress and plan for OneNet data ingestion alongside legacy NMEA 0183/2000 support.

**No universal marina data interchange standard:** Unlike hotel or airline reservations (which have GDS standards), the marina industry lacks a universal open data model for slip availability, vessel records, or transient reservation exchange. The closest approximation is the Marinas.com point-of-interest API and Dockwa's reservation marketplace, both proprietary. An open-source marina management platform has an opportunity to define and publish a reference OpenAPI schema that could become a de-facto standard for the segment.

**HIN as primary vessel key:** The USCG Hull Identification Number is the most reliable cross-system vessel identifier in North American waters. European equivalents include the CIN (Craft Identification Number) under ISO 10087:2006. Marina data models should store both HIN and CIN where applicable to support international operations.

**PCI DSS v4.0 transition:** PCI DSS v4.0 became the sole applicable standard on 31 March 2025. Marina systems still referencing v3.2.1 controls are technically non-compliant; any marina software evaluating compliance posture should audit against the v4.0 requirement set, particularly for fuel dock card readers and online booking payment flows.
