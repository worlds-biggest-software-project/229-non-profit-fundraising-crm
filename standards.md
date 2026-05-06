# Standards & API Reference

> Project: Non-Profit Fundraising CRM · Generated: 2026-05-03

## Industry Standards & Specifications

### Accounting & Regulatory Standards

**FASB ASC 958 — Nonprofit Entities**
- URL: https://asc.fasb.org/958
- The primary US GAAP standard governing financial reporting for nonprofit organisations. Defines how contributions (gifts, pledges, bequests, in-kind donations), conditional and unconditional promises to give, and endowment assets are recognised on financial statements. A fundraising CRM must track the data fields required by ASC 958, including gift restriction type (unrestricted, temporarily restricted, permanently restricted under legacy GAAP; or with/without donor restrictions under ASU 2016-14).

**IRS Form 990 and Schedule B**
- URL: https://www.irs.gov/forms-pubs/about-form-990
- Schedule B instructions: https://www.irs.gov/instructions/i990sb
- The annual information return filed by US tax-exempt organisations. Schedule B (Schedule of Contributors) requires disclosure of donors contributing more than $5,000 or more than 2% of total revenues. A nonprofit CRM must track cumulative annual giving per constituent and flag major donors meeting Schedule B thresholds. Form 990 data is publicly accessible and serves as a primary source for prospect research and wealth screening.

**IRS Charitable Gift Annuity and Planned Giving Rules**
- URL: https://www.irs.gov/charities-non-profits/charitable-organizations/charitable-gift-annuities
- Governs the tax treatment of charitable remainder trusts (CRTs), charitable lead trusts (CLTs), gift annuities, and bequests. A planned giving module must model payout rates, remainder values, and IRS-required substantiation letters for deferred gift vehicles. American Council on Gift Annuities (ACGA) sets recommended maximum annuity rates updated periodically.

**CASE (Council for Advancement and Support of Education) Reporting Standards**
- URL: https://www.case.org/resources/case-reporting-standards-management-guidelines
- Fundraising reporting standards widely adopted across higher education and adjacent nonprofits. Defines how to count and report campaign gifts, pledges, planned gifts, and matching gifts consistently for benchmarking. Systems serving educational institutions must produce CASE-compliant reports.

**AFP (Association of Fundraising Professionals) Donor Bill of Rights**
- URL: https://afpglobal.org/donor-bill-rights
- Ethical standards co-authored by AFP, AHP, CASE, and the Giving Institute governing donor data use, privacy, transparency in solicitation, and the right to know how gifts are used. Fundraising CRMs should facilitate compliance by enabling consent capture, preference management, and transparent gift designation tracking.

---

### Payment Card Industry Standards

**PCI DSS 4.0 (Payment Card Industry Data Security Standard)**
- URL: https://www.pcisecuritystandards.org/document_library/
- Mandatory compliance standard for any system that processes, stores, or transmits payment card data. PCI DSS 4.0 became mandatory in March 2025, introducing stricter requirements for web-based payment forms (SAQ A-EP scope), customised confirmation pages, and third-party script management. Nonprofit CRMs handling online donation payment processing must be PCI DSS 4.0 compliant or use a compliant payment processor (Stripe, Braintree, PayPal) that handles card data out of scope. Key requirements relevant to fundraising platforms: tokenised card storage, real-time fraud scoring, encrypted data transmission, and annual penetration testing.

**EMV (Europay, Mastercard, Visa) Standards**
- URL: https://www.emvco.com/
- Governs chip card and contactless payment specifications relevant to in-person donation collection at events. EMV compliance is required for card-present transactions to achieve liability shift protection.

---

### Data Privacy & Consent Standards

**GDPR (General Data Protection Regulation) — EU Regulation 2016/679**
- URL: https://gdpr.eu/
- Applies to any organisation collecting or processing personal data of EU residents regardless of where the organisation is based. Relevant requirements for nonprofit CRMs: lawful basis for processing donor data (typically legitimate interest or explicit consent), right to access/correction/erasure (right to be forgotten), data portability obligations, breach notification within 72 hours, and cross-border data transfer restrictions. Non-compliance fines reach €20 million or 4% of annual global revenue.

**CCPA / CPRA (California Consumer Privacy Act / California Privacy Rights Act)**
- URL: https://oag.ca.gov/privacy/ccpa
- US state privacy law granting California residents similar rights to GDPR subjects: right to know, delete, correct, and opt out of sale of personal information. Nonprofit CRMs must provide donor data export and deletion workflows to satisfy CCPA/CPRA requests.

**CAN-SPAM Act (US) and CASL (Canada)**
- URL (CAN-SPAM): https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- URL (CASL): https://crtc.gc.ca/eng/internet/anti.htm
- Govern commercial electronic messaging and fundraising solicitation emails. CAN-SPAM requires unsubscribe mechanisms and accurate sender identification. CASL is stricter, requiring express or implied consent before sending commercial email. Nonprofit CRMs must maintain opt-in/opt-out consent records per constituent.

**NCOA (National Change of Address) — USPS**
- URL: https://postalpro.usps.com/ncoa
- USPS service enabling organisations to update mailing addresses for constituents who have moved and registered address changes with the postal service. NCOA processing is typically run quarterly or before major direct mail campaigns. Nonprofit CRMs should support NCOA file submission and automated address update workflows.

---

### Authentication & API Security Standards

**OAuth 2.0 — RFC 6749**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The industry-standard authorisation framework governing how third-party applications obtain delegated access to CRM data on behalf of users. All major nonprofit CRM APIs (Blackbaud SKY API, Salesforce, Virtuous, Neon CRM v2) use OAuth 2.0 for API authentication. A nonprofit CRM must implement OAuth 2.0 to support ecosystem integrations from wealth screening, email, accounting, and payment platforms.

**OpenID Connect (OIDC) Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer built on OAuth 2.0 providing standardised user identity tokens (ID tokens). Used by Salesforce and enterprise CRM platforms for single sign-on (SSO) integration with nonprofit IT environments. Relevant for nonprofit CRMs targeting enterprise organisations with existing identity providers (Azure AD, Okta, Google Workspace).

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de facto standard for describing REST APIs in a machine-readable format. Enables auto-generation of client SDKs, interactive documentation (Swagger UI), and automated API testing. Nonprofit CRM REST APIs should publish an OpenAPI 3.1 specification to lower developer integration friction.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/specification
- Standard for describing and validating JSON data structures. Used alongside OpenAPI 3.1 to define CRM data model payloads (constituent records, gift objects, pledge schedules). Publishing JSON Schema definitions enables third-party developers to validate integration data without proprietary SDKs.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Authoritative checklist for API security vulnerabilities relevant to CRM platforms: broken object-level authorisation (BOLA), broken authentication, excessive data exposure, and rate limiting. Fundraising CRMs expose sensitive constituent and financial data and must be designed against this checklist.

**NIST Cybersecurity Framework 2.0**
- URL: https://www.nist.gov/cyberframework
- US federal cybersecurity framework providing identify/protect/detect/respond/recover guidance. Increasingly required by nonprofit organisations receiving federal grants or working with healthcare-adjacent data. Relevant for enterprise CRM deployment security posture.

---

### Data Exchange & Interoperability Standards

**IATI (International Aid Transparency Initiative) Standard**
- URL: https://iatistandard.org/en/
- Open XML-based data standard for publishing development and humanitarian funding information. 1,690+ organisations publishing as of 2024. Relevant for international nonprofit organisations required to report funding flows to bilateral donors, UN agencies, or DAC members. A fundraising CRM serving international development organisations should support IATI XML export for funder reporting compliance.

**vCard / RFC 6350 (Contact Data Format)**
- URL: https://datatracker.ietf.org/doc/html/rfc6350
- IETF standard format for contact/address book data. Relevant for constituent data import/export portability between CRM systems, especially for small nonprofits migrating from consumer contact management tools.

**CSV / RFC 4180 (Comma-Separated Values)**
- URL: https://datatracker.ietf.org/doc/html/rfc4180
- De facto standard for flat-file data exchange. All nonprofit CRM platforms must support CSV import/export for constituent records and gift history as the lowest-common-denominator data portability format.

**SEPA Direct Debit (Single Euro Payments Area)**
- URL: https://www.europeanpaymentscouncil.eu/what-we-do/sepa-payment-schemes/sepa-direct-debit
- EU payment standard for recurring debit mandates. Relevant for nonprofit CRMs targeting European organisations processing recurring donations from EU bank accounts. Mandates require specific XML (pain.008) format for batch submission to banks.

---

## Similar Products — Developer Documentation & APIs

### Blackbaud SKY API

- **Description:** Open REST API platform providing developer access to all Blackbaud products including Raiser's Edge NXT, Altru, and Blackbaud CRM. Covers constituent management, gift processing, event management, and more.
- **API Documentation:** https://developer.blackbaud.com/skyapi/docs
- **API Reference:** https://developer.sky.blackbaud.com/api
- **Getting Started:** https://developer.blackbaud.com/skyapi/docs/getting-started
- **Standards:** REST/JSON, OAuth 2.0 (three-legged), OpenAPI-described endpoints
- **Authentication:** OAuth 2.0 (authorisation code flow); SKY Developer account required
- **Webhooks:** Event subscriptions available for constituent and gift events via SKY API
- **SDKs/Libraries:** Official .NET SDK; community SDKs for Python and JavaScript

---

### Salesforce Agentforce Nonprofit — Fundraising Business APIs

- **Description:** REST APIs native to the Salesforce platform for the Agentforce Nonprofit (formerly Nonprofit Cloud) fundraising data model. Covers Donation, Pledge, Campaign, Opportunity, and Fundraising Contact objects.
- **API Documentation:** https://developer.salesforce.com/docs/atlas.en-us.nonprofit_cloud.meta/nonprofit_cloud/npc_fundraising_dev_guide.htm
- **Fundraising Business APIs Reference:** https://developer.salesforce.com/docs/atlas.en-us.nonprofit_cloud.meta/nonprofit_cloud/npc_fundraising_business_api_rest_reference.htm
- **Developer Guide PDF (Spring 2026):** https://resources.docs.salesforce.com/latest/latest/en-us/sfdc/pdf/nonprofit_cloud.pdf
- **Standards:** REST/JSON, SOAP, Bulk API, Streaming API (all Salesforce-standard)
- **Authentication:** OAuth 2.0 (Connected App + JWT or authorisation code flow), OpenID Connect for SSO
- **SDKs/Libraries:** Salesforce DX CLI, Apex (native), JavaScript (JSforce), Python (simple-salesforce), Java (Force.com Toolkit)

---

### Virtuous CRM+ API

- **Description:** REST API for the Virtuous responsive fundraising platform covering contacts, gifts, projects, communications, and event data. Includes webhook support for real-time integration.
- **API Documentation:** https://docs.virtuoussoftware.com/
- **Webhook Guide:** https://support.virtuous.org/hc/en-us/articles/360051385392-How-Do-I-Integrate-with-Virtuous-CRM-Webhooks
- **Standards:** REST/JSON; resource-oriented URLs; standard HTTP response codes
- **Authentication:** API key (bearer token); OAuth 2.0 for third-party integrations
- **Webhooks:** Contact Create/Update, Gift Create/Update/Delete, Form Submission, Event, Project events; HTTPS POST with JSON payload; auto-retry up to 5 times over 24 hours

---

### Neon CRM Developer API

- **Description:** REST API providing access to constituent, membership, event, donation, volunteer, and custom object data within Neon CRM. API v2 is a complete rebuild of v1 adopting modern RESTful conventions.
- **Developer Centre:** https://developer.neoncrm.com/
- **API v2 Documentation:** https://developer.neoncrm.com/api-v2/
- **Getting Started:** https://developer.neoncrm.com/getting-started/
- **Changelog:** https://developer.neoncrm.com/updates/
- **Standards:** REST/JSON (v2); legacy XML-RPC available via v1 (deprecated path)
- **Authentication:** API key + organisation ID pair (HTTP Basic Auth); OAuth 2.0 for marketplace integrations
- **SDKs/Libraries:** No official SDK; community Python and PHP wrappers available

---

### Bloomerang REST API

- **Description:** Private-key REST API for server-to-server integration with the Bloomerang donor management platform. Covers constituents, transactions, interactions, campaigns, and funds.
- **API Documentation:** https://bloomerang.com/api/rest-api/
- **Swagger UI:** Available at the documentation URL above
- **Legacy API (deprecated):** https://bloomerang.com/api/rest-api-v1/
- **Standards:** REST/JSON; versioned endpoints; Swagger/OpenAPI documented
- **Authentication:** Private API key via HTTP Basic Auth; public key (Bloomerang.js) for front-end form embedding
- **SDKs/Libraries:** Unofficial Ruby gem (chiperific/bloomerang_api on GitHub); Parsons Python library includes Bloomerang connector

---

### DonorPerfect API

- **Description:** XML-based API providing programmatic access to DonorPerfect constituent, gift, pledge, and acknowledgment data. Used for data import/export and third-party system integration.
- **API Factsheet:** https://www.donorperfect.com/factsheets/api-access-feature/
- **XML API Documentation:** https://softerware-uploads.s3.amazonaws.com/community/donorperfect/attachments/DPO_SUP_Manual_XML_API_Documentation%2009202023.pdf
- **API Key Request:** api@softerware.com
- **Standards:** XML-based API; RESTful JSON API in development alongside legacy XML
- **Authentication:** API key credential pair
- **Webhooks:** Supported for real-time donation data push to external systems
- **SDKs/Libraries:** No official SDK; server-side scripting (PHP, ASP, Perl) examples in documentation

---

### Givebutter REST API

- **Description:** REST API for the Givebutter fundraising platform covering campaigns, transactions, contacts, events, and financial data. Supports webhook subscriptions for event-driven integrations.
- **API Reference:** https://docs.givebutter.com/reference/reference-getting-started
- **Help Centre:** https://help.givebutter.com/en/collections/3094658-api-and-more
- **Webhook Guide:** https://help.givebutter.com/en/articles/8828428-how-to-automate-workflows-and-data-using-webhooks
- **Standards:** REST/JSON; resource-oriented; organised around REST conventions
- **Authentication:** API key; webhook payloads include HMAC signature for verification (Signature header)
- **Webhooks:** Event subscriptions including campaign.created, donation.completed, and others; signing secret for payload verification

---

### CharityEngine API

- **Description:** REST and SOAP APIs enabling custom integration and asset management for the CharityEngine all-in-one nonprofit platform. Covers CRM, payments, campaigns, and advocacy data.
- **Developer Portal:** https://charityengine.net/developers-home/
- **Standards:** REST/JSON and SOAP/XML; JavaScript/jQuery/HTML5 for front-end form customisation
- **Authentication:** API credentials provided via enterprise account setup
- **SDKs/Libraries:** No public SDK; JavaScript embedding for donation forms and campaign widgets

---

### iWave (Kindsight) API

- **Description:** Prospect research and wealth screening API providing donor capacity scores, philanthropic giving history (from Form 990 public data), and affinity ratings drawn from 44+ vetted data sources.
- **Product Page:** https://kindsight.io/iwave/
- **Integration Guide:** https://www.donorperfect.com/integrations/prospect-research/iwave/
- **Standards:** REST/JSON; integrates via webhooks or scheduled batch screening jobs into CRM constituent records
- **Authentication:** API key / OAuth token depending on CRM integration path
- **Note:** Kindsight (formerly iWave) is the rebranded name post-acquisition; DonorSearch is a competing platform with 40+ CRM integrations

---

### Fundraise Up API

- **Description:** REST API and CRM integration layer for the Fundraise Up online fundraising platform, enabling real-time donation data sync to Salesforce, DonorPerfect, Bloomerang, and other CRMs.
- **API Documentation:** https://fundraiseup.com/platform/rest-api/
- **Integration Docs:** https://fundraiseup.com/docs/develop/
- **Standards:** REST/JSON; CRM integrations use OAuth where available (e.g., Salesforce) or API keys
- **Authentication:** OAuth 2.0 for CRM integrations; API key for standalone REST access
- **Webhooks:** Real-time donation event push to configured CRM endpoints

---

## Notes

**Emerging standards and gaps:**

- The nonprofit sector lacks a universal data portability standard equivalent to healthcare's FHIR. Constituent record portability between platforms requires bespoke ETL processes; no single XML or JSON schema describes the full nonprofit CRM data model in a vendor-neutral way.
- The IATI standard covers international aid transparency but is not widely implemented by domestic US nonprofits; most US fundraising CRMs have no IATI export capability.
- PCI DSS 4.0 (mandatory from March 2025) has raised the compliance bar for web-based donation form implementations, particularly around inline scripting, content security policies, and third-party payment form rendering. All platforms are adapting; documentation on compliance specifics is evolving.
- AI agent standards for nonprofit fundraising (automated donor outreach, volunteer scheduling) are nascent. Salesforce's Agentforce is the only platform publishing a formal agent capability model; broader MCP (Model Context Protocol) adoption in nonprofit CRM integrations has not yet been established.
- SEPA XML (pain.008) and UK BACS formats for recurring donation processing remain outside the scope of US-centric CRM platforms, representing an opportunity for international market differentiation.
