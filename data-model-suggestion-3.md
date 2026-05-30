# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Non-Profit Fundraising CRM · Created: 2026-05-22

## Philosophy

This model uses a hybrid approach: core, frequently-queried fields are stored as typed relational columns with proper constraints and indexes, while variable, jurisdiction-specific, and tenant-customisable fields are stored in JSONB columns. This mirrors the pattern used by modern SaaS platforms (Neon CRM's Custom Objects, Salesforce's flexible schema) that must accommodate thousands of tenants with different data requirements without per-tenant schema migrations.

The fundamental insight is that nonprofit data has a stable core (contacts have names and emails; gifts have amounts and dates) surrounded by a highly variable periphery. A US nonprofit needs IRS EIN tracking and Schedule B flags. A UK charity needs Charity Commission registration numbers and Gift Aid declarations. A Canadian organisation needs CRA business numbers and tax shelter receipts. Rather than creating nullable columns for every jurisdiction or maintaining separate schemas, JSONB columns absorb this variation while the relational core provides type safety, referential integrity, and performant indexing where it matters most.

PostgreSQL's JSONB support (GIN indexes, containment operators, JSON path queries) makes this approach practical without sacrificing query performance. The hybrid model typically results in 40-50% fewer tables than a fully normalized model, dramatically faster schema evolution (add a field to a JSONB column without a migration), and the same referential integrity on core relationships.

**Best for:** Multi-country nonprofits, rapid MVP development, platforms serving diverse tenant types (education, healthcare, international development), and teams that need to ship features without waiting for database migrations.

**Trade-offs:**
- (+) Dramatically faster schema evolution — new fields added without ALTER TABLE
- (+) ~40% fewer tables than fully normalized model
- (+) Jurisdiction-specific fields accommodated without schema changes
- (+) Tenant-specific custom fields without EAV (Entity-Attribute-Value) anti-pattern
- (+) Core relational integrity preserved for essential relationships
- (-) JSONB fields not enforced at database level (application-layer validation required)
- (-) JSONB queries can be slower than indexed column queries if not properly indexed
- (-) Schema documentation must be maintained outside the database (JSON Schema definitions)
- (-) Harder for reporting tools to introspect JSONB columns
- (-) Risk of "JSONB sprawl" — undisciplined use can make the data model opaque

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FASB ASC 958 | Core `restriction_type` column on fund table; jurisdiction-specific accounting requirements in `regulatory_data` JSONB |
| IRS Form 990 | US-specific fields (`ein`, `schedule_b_flagged`, `990_category`) stored in `regulatory_data` JSONB on contact/org records |
| CASE Reporting Standards | `counting_method` as relational column; CASE-specific metadata in gift `extended_data` JSONB |
| PCI DSS 4.0 | Payment tokens in relational columns; processor-specific metadata in `payment_metadata` JSONB |
| GDPR / CCPA | Consent records as relational rows; jurisdiction-specific consent details in `consent_details` JSONB |
| ISO 3166 | `country_code` and `state_province` as indexed relational columns on all address records |
| ISO 4217 | `currency_code` as relational column; exchange rate metadata in JSONB |
| Charity Commission (UK) | UK registration numbers, Gift Aid flags stored in `regulatory_data` JSONB |
| CRA (Canada) | Canadian tax shelter numbers, eligible amount calculations in `regulatory_data` JSONB |
| IATI Standard | IATI activity identifiers and reporting metadata in `integration_data` JSONB on grant/gift records |
| SEPA / BACS | European payment mandate details in `payment_metadata` JSONB |
| JSON Schema Draft 2020-12 | All JSONB column structures documented as JSON Schema definitions for validation and API documentation |

---

## Core Constituent Management

### Contacts Table

```sql
CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    household_id    UUID REFERENCES household(id),

    -- Core relational fields (indexed, typed, constrained)
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email_primary   TEXT,
    phone_primary   TEXT,
    donor_type      TEXT NOT NULL DEFAULT 'individual',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_deceased     BOOLEAN NOT NULL DEFAULT false,

    -- Derived aggregates (maintained by triggers or application)
    lifetime_giving NUMERIC(15,2) DEFAULT 0,
    last_gift_date  DATE,
    first_gift_date DATE,
    total_gift_count INTEGER DEFAULT 0,
    engagement_score INTEGER DEFAULT 0,

    -- Flexible fields for contact details that vary by context
    demographics    JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "prefix": "Dr.",
    --   "suffix": "III",
    --   "middle_name": "Robert",
    --   "nickname": "Bob",
    --   "formal_name": "Dr. Robert J. Smith III",
    --   "gender": "male",
    --   "date_of_birth": "1965-03-15",
    --   "marital_status": "married",
    --   "preferred_language": "en",
    --   "ethnicity": "prefer_not_to_say"
    -- }

    -- Communication preferences (varies by jurisdiction and org type)
    communication_prefs JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "preferred_channel": "email",
    --   "email_frequency": "weekly",
    --   "do_not_call": false,
    --   "do_not_mail": false,
    --   "do_not_email": false,
    --   "do_not_sms": true,
    --   "seasonal_address_active": false,
    --   "acknowledgment_preference": "email"
    -- }

    -- Regulatory and compliance data (varies by country)
    regulatory_data JSONB DEFAULT '{}',
    -- US Example:
    -- {
    --   "schedule_b_flagged": true,
    --   "schedule_b_fiscal_year": 2025,
    --   "cumulative_annual_giving": 12500.00,
    --   "employer_match_eligible": true,
    --   "daf_donor": true
    -- }
    -- UK Example:
    -- {
    --   "gift_aid_eligible": true,
    --   "gift_aid_declaration_date": "2024-01-15",
    --   "higher_rate_taxpayer": true
    -- }

    -- AI and scoring data (updated by ML pipeline)
    ai_scores       JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "retention_risk": 0.23,
    --   "major_gift_propensity": 0.67,
    --   "planned_giving_propensity": 0.45,
    --   "recommended_ask_amount": 5000.00,
    --   "model_version": "v2.3",
    --   "scored_at": "2026-05-20T14:30:00Z",
    --   "upgrade_likelihood": 0.58,
    --   "churn_risk_factors": ["recency_declining", "no_event_attendance"]
    -- }

    -- Wealth screening data (from iWave, DonorSearch, etc.)
    wealth_data     JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "provider": "kindsight",
    --   "screened_at": "2026-03-15",
    --   "capacity_rating": "$1M-$5M",
    --   "capacity_score": 8,
    --   "propensity_score": 7,
    --   "affinity_score": 6,
    --   "real_estate_value": 1250000,
    --   "stock_holdings": 500000,
    --   "philanthropic_giving_total": 250000,
    --   "board_memberships": ["Red Cross", "United Way"]
    -- }

    -- Custom fields defined by tenant
    custom_fields   JSONB DEFAULT '{}',
    -- Example (university fundraising):
    -- {
    --   "class_year": 1992,
    --   "degree": "MBA",
    --   "school": "Business",
    --   "fraternity": "Delta Sigma Pi",
    --   "athletics_letter": true
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contact_tenant ON contact(tenant_id);
CREATE INDEX idx_contact_household ON contact(household_id);
CREATE INDEX idx_contact_name ON contact(tenant_id, last_name, first_name);
CREATE INDEX idx_contact_email ON contact(tenant_id, email_primary);
CREATE INDEX idx_contact_donor_type ON contact(tenant_id, donor_type);
CREATE INDEX idx_contact_engagement ON contact(tenant_id, engagement_score DESC);
CREATE INDEX idx_contact_last_gift ON contact(tenant_id, last_gift_date);

-- GIN index on JSONB for containment queries
CREATE INDEX idx_contact_custom_fields ON contact USING GIN (custom_fields);
CREATE INDEX idx_contact_ai_scores ON contact USING GIN (ai_scores);
CREATE INDEX idx_contact_regulatory ON contact USING GIN (regulatory_data);

-- Partial index for Schedule B flagged donors (US-specific, but performant)
CREATE INDEX idx_contact_schedule_b ON contact(tenant_id)
    WHERE (regulatory_data->>'schedule_b_flagged')::boolean = true;
```

### Households Table

```sql
CREATE TABLE household (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    primary_contact_id UUID,

    -- Flexible greeting and display preferences
    display_prefs   JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "formal_greeting": "Mr. and Mrs. John Smith",
    --   "informal_greeting": "John and Jane",
    --   "envelope_name": "The Smith Family",
    --   "recognition_name": "John & Jane Smith"
    -- }

    -- Derived household aggregates
    household_lifetime_giving NUMERIC(15,2) DEFAULT 0,
    household_last_gift_date DATE,

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_household_tenant ON household(tenant_id);
CREATE INDEX idx_household_name ON household(tenant_id, name);
```

### Organizations Table

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    org_type        TEXT NOT NULL DEFAULT 'nonprofit',

    -- Flexible org details
    org_details     JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "legal_name": "Smith Foundation Inc.",
    --   "tax_id": "12-3456789",
    --   "website": "https://smithfoundation.org",
    --   "industry": "philanthropy",
    --   "annual_revenue": 5000000,
    --   "employee_count": 25,
    --   "founding_year": 1998
    -- }

    -- Regulatory data (varies by jurisdiction)
    regulatory_data JSONB DEFAULT '{}',
    -- US Example: {"ein": "12-3456789", "irs_subsection": "501c3"}
    -- UK Example: {"charity_number": "1234567", "companies_house": "12345678"}

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_tenant ON organization(tenant_id);
CREATE INDEX idx_org_name ON organization(tenant_id, name);
CREATE INDEX idx_org_type ON organization(tenant_id, org_type);
```

### Addresses Table

```sql
CREATE TABLE address (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    addressable_type TEXT NOT NULL,          -- 'contact', 'household', 'organization'
    addressable_id  UUID NOT NULL,           -- polymorphic FK
    address_type    TEXT NOT NULL DEFAULT 'home',
    is_primary      BOOLEAN NOT NULL DEFAULT false,

    -- Core address fields (relational, indexed)
    street_line_1   TEXT NOT NULL,
    city            TEXT NOT NULL,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL DEFAULT 'US',

    -- Extended address data
    address_details JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "street_line_2": "Suite 400",
    --   "county": "Fairfax",
    --   "latitude": 38.8462,
    --   "longitude": -77.3064,
    --   "seasonal_start": "11-01",
    --   "seasonal_end": "04-30",
    --   "ncoa_updated_at": "2026-03-15",
    --   "ncoa_move_type": "individual",
    --   "delivery_point_barcode": "123456789012"
    -- }

    is_valid        BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_address_owner ON address(addressable_type, addressable_id);
CREATE INDEX idx_address_postal ON address(country_code, postal_code);
```

### Relationships Table

```sql
CREATE TABLE relationship (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id_a    UUID NOT NULL REFERENCES contact(id),
    contact_id_b    UUID NOT NULL REFERENCES contact(id),
    relationship_type TEXT NOT NULL,

    -- Flexible relationship metadata
    details         JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "reciprocal_type": "child",
    --   "is_primary": true,
    --   "start_date": "2015-06-20",
    --   "notes": "Introduced donor to planned giving programme"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rel_contact_a ON relationship(contact_id_a);
CREATE INDEX idx_rel_contact_b ON relationship(contact_id_b);
CREATE INDEX idx_rel_type ON relationship(tenant_id, relationship_type);
```

---

## Gift & Pledge Processing

### Gifts Table

```sql
CREATE TABLE gift (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    household_id    UUID REFERENCES household(id),
    campaign_id     UUID REFERENCES campaign(id),
    fund_id         UUID REFERENCES fund(id),
    pledge_id       UUID REFERENCES pledge(id),
    recurring_donation_id UUID REFERENCES recurring_donation(id),

    -- Core gift fields (relational, indexed, constrained)
    gift_type       TEXT NOT NULL DEFAULT 'cash',
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    gift_date       DATE NOT NULL,
    received_date   DATE,
    is_anonymous    BOOLEAN NOT NULL DEFAULT false,
    is_recurring    BOOLEAN NOT NULL DEFAULT false,
    acknowledgment_status TEXT DEFAULT 'pending',
    counting_method TEXT DEFAULT 'outright',

    -- Extended gift data (varies by gift type and jurisdiction)
    gift_details    JSONB DEFAULT '{}',
    -- Cash/Check Example:
    -- {
    --   "check_number": "1234",
    --   "batch_id": "uuid...",
    --   "appeal_code": "FY26-EOY-DM",
    --   "source_code": "WEBFORM-01"
    -- }
    -- Stock Example:
    -- {
    --   "ticker": "AAPL",
    --   "shares": 100,
    --   "fair_market_value_per_share": 185.50,
    --   "transfer_date": "2026-03-15",
    --   "broker": "Charles Schwab",
    --   "cost_basis": 12000.00
    -- }
    -- In-Kind Example:
    -- {
    --   "description": "Office furniture",
    --   "fair_market_value": 5000.00,
    --   "appraised": true,
    --   "appraiser_name": "Smith Appraisals",
    --   "appraisal_date": "2026-02-28"
    -- }
    -- DAF Example:
    -- {
    --   "daf_sponsor": "Fidelity Charitable",
    --   "daf_account_number": "DC-12345",
    --   "grant_recommendation_id": "GR-67890"
    -- }

    -- Tax receipt data (varies by jurisdiction)
    tax_data        JSONB DEFAULT '{}',
    -- US Example:
    -- {
    --   "tax_deductible_amount": 1000.00,
    --   "non_deductible_amount": 50.00,
    --   "benefit_description": "Dinner for two",
    --   "benefit_fair_market_value": 50.00,
    --   "receipt_number": "TR-2026-001234",
    --   "receipt_sent_date": "2026-01-05",
    --   "quid_pro_quo": true
    -- }
    -- UK Example:
    -- {
    --   "gift_aid_claimed": true,
    --   "gift_aid_amount": 250.00,
    --   "higher_rate_relief": 125.00,
    --   "gift_aid_declaration_id": "GAD-001"
    -- }
    -- Canada Example:
    -- {
    --   "eligible_amount": 1000.00,
    --   "advantage_amount": 0,
    --   "receipt_number": "2026-001234",
    --   "tax_shelter_number": null
    -- }

    -- Tribute/memorial data
    tribute_data    JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "tribute_type": "in_memory_of",
    --   "honoree_name": "Mary Johnson",
    --   "notify_contacts": ["uuid1", "uuid2"],
    --   "notification_sent": true,
    --   "notification_date": "2026-01-10"
    -- }

    -- Payment processor metadata
    payment_metadata JSONB DEFAULT '{}',
    -- Stripe Example:
    -- {
    --   "processor": "stripe",
    --   "charge_id": "ch_3PAbC...",
    --   "payment_intent_id": "pi_3PAbC...",
    --   "payment_method": "pm_1PQr...",
    --   "card_last_four": "4242",
    --   "card_brand": "visa",
    --   "receipt_url": "https://pay.stripe.com/receipts/..."
    -- }

    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gift_tenant ON gift(tenant_id);
CREATE INDEX idx_gift_contact ON gift(contact_id);
CREATE INDEX idx_gift_household ON gift(household_id);
CREATE INDEX idx_gift_date ON gift(tenant_id, gift_date);
CREATE INDEX idx_gift_campaign ON gift(campaign_id);
CREATE INDEX idx_gift_fund ON gift(fund_id);
CREATE INDEX idx_gift_type ON gift(tenant_id, gift_type);
CREATE INDEX idx_gift_pledge ON gift(pledge_id);
CREATE INDEX idx_gift_recurring ON gift(recurring_donation_id);
CREATE INDEX idx_gift_ack ON gift(tenant_id, acknowledgment_status) WHERE acknowledgment_status = 'pending';

-- GIN indexes on JSONB for specific query patterns
CREATE INDEX idx_gift_details ON gift USING GIN (gift_details);
CREATE INDEX idx_gift_tax ON gift USING GIN (tax_data);
```

### Soft Credits Table

```sql
CREATE TABLE soft_credit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gift_id         UUID NOT NULL REFERENCES gift(id) ON DELETE CASCADE,
    contact_id      UUID NOT NULL REFERENCES contact(id),
    credit_type     TEXT NOT NULL DEFAULT 'soft',
    amount          NUMERIC(15,2) NOT NULL,

    details         JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "matching_org_id": "uuid...",
    --   "match_ratio": "1:1",
    --   "solicitor_relationship": "board_member"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_soft_credit_gift ON soft_credit(gift_id);
CREATE INDEX idx_soft_credit_contact ON soft_credit(contact_id);
```

### Pledges Table

```sql
CREATE TABLE pledge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    campaign_id     UUID REFERENCES campaign(id),
    fund_id         UUID REFERENCES fund(id),
    pledge_amount   NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    balance         NUMERIC(15,2) NOT NULL,
    pledge_date     DATE NOT NULL,
    frequency       TEXT DEFAULT 'one_time',
    status          TEXT NOT NULL DEFAULT 'active',

    schedule_details JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "installment_amount": 500.00,
    --   "start_date": "2026-01-01",
    --   "end_date": "2028-12-31",
    --   "next_payment_date": "2026-07-01",
    --   "payment_day_of_month": 1,
    --   "auto_remind": true,
    --   "remind_days_before": 14,
    --   "write_off_policy": "after_3_missed"
    -- }

    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pledge_tenant ON pledge(tenant_id);
CREATE INDEX idx_pledge_contact ON pledge(contact_id);
CREATE INDEX idx_pledge_status ON pledge(tenant_id, status);
```

### Recurring Donations Table

```sql
CREATE TABLE recurring_donation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    fund_id         UUID REFERENCES fund(id),
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    frequency       TEXT NOT NULL DEFAULT 'monthly',
    status          TEXT NOT NULL DEFAULT 'active',
    next_charge_date DATE,

    -- Payment and processor details in JSONB (varies by processor)
    payment_config  JSONB DEFAULT '{}',
    -- Stripe Example:
    -- {
    --   "processor": "stripe",
    --   "subscription_id": "sub_1PQr...",
    --   "customer_id": "cus_1PQr...",
    --   "payment_method_id": "pm_1PQr...",
    --   "card_last_four": "4242",
    --   "card_expiry": "12/2028"
    -- }
    -- SEPA Example:
    -- {
    --   "processor": "sepa",
    --   "mandate_id": "MNDT-2026-001",
    --   "iban_last_four": "1234",
    --   "creditor_id": "DE98ZZZ09999999999",
    --   "mandate_date": "2026-01-15"
    -- }

    charge_history  JSONB DEFAULT '[]',
    -- Example: [{
    --   "date": "2026-05-01",
    --   "amount": 50.00,
    --   "status": "succeeded",
    --   "gift_id": "uuid..."
    -- }]
    -- Note: Only last N charges stored here for quick access;
    -- full history is in the gift table

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recurring_tenant ON recurring_donation(tenant_id);
CREATE INDEX idx_recurring_contact ON recurring_donation(contact_id);
CREATE INDEX idx_recurring_status ON recurring_donation(tenant_id, status);
CREATE INDEX idx_recurring_next ON recurring_donation(next_charge_date) WHERE status = 'active';
```

### Funds Table

```sql
CREATE TABLE fund (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    code            TEXT,
    fund_type       TEXT NOT NULL DEFAULT 'general',
    restriction_type TEXT NOT NULL DEFAULT 'without_restrictions',

    fund_details    JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "description": "Annual operating fund",
    --   "goal_amount": 500000.00,
    --   "start_date": "2026-01-01",
    --   "end_date": "2026-12-31",
    --   "gl_account_code": "4100",
    --   "reporting_category": "annual_fund"
    -- }

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fund_tenant ON fund(tenant_id);
CREATE INDEX idx_fund_type ON fund(tenant_id, fund_type);
```

---

## Campaigns & Appeals

### Campaigns Table

```sql
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_campaign_id UUID REFERENCES campaign(id),
    name            TEXT NOT NULL,
    campaign_type   TEXT NOT NULL DEFAULT 'annual_fund',
    status          TEXT NOT NULL DEFAULT 'planned',
    goal_amount     NUMERIC(15,2),
    raised_amount   NUMERIC(15,2) DEFAULT 0,
    start_date      DATE,
    end_date        DATE,

    campaign_details JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "description": "FY26 End of Year Appeal",
    --   "target_audience": "lybunt_donors",
    --   "channels": ["direct_mail", "email", "social"],
    --   "budget": 15000.00,
    --   "cost_per_dollar_raised": 0.12,
    --   "appeal_codes": ["FY26-EOY-DM", "FY26-EOY-EM"],
    --   "donor_count": 1245,
    --   "gift_count": 1578,
    --   "average_gift": 127.50,
    --   "new_donor_count": 234,
    --   "retention_rate": 0.62
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_tenant ON campaign(tenant_id);
CREATE INDEX idx_campaign_parent ON campaign(parent_campaign_id);
CREATE INDEX idx_campaign_status ON campaign(tenant_id, status);
CREATE INDEX idx_campaign_dates ON campaign(tenant_id, start_date, end_date);
```

---

## Planned Giving

### Planned Gifts Table

```sql
CREATE TABLE planned_gift (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    gift_vehicle    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'prospect',
    estimated_amount NUMERIC(15,2),
    fund_id         UUID REFERENCES fund(id),

    -- Vehicle-specific details in JSONB (avoids sparse columns)
    vehicle_details JSONB DEFAULT '{}',
    -- Bequest Example:
    -- {
    --   "bequest_type": "residuary",
    --   "bequest_percentage": 25.0,
    --   "estate_name": "Estate of John Smith",
    --   "will_date": "2023-06-15",
    --   "attorney_name": "Jane Doe, Esq.",
    --   "attorney_firm": "Doe & Associates",
    --   "attorney_phone": "555-0123",
    --   "codicil_date": null,
    --   "executor_name": "Robert Smith"
    -- }
    -- CRT Example:
    -- {
    --   "trust_type": "crut",
    --   "trust_name": "Smith Family Charitable Remainder Unitrust",
    --   "initial_funding": 500000.00,
    --   "payout_rate": 0.05,
    --   "payout_frequency": "quarterly",
    --   "income_beneficiaries": ["John Smith", "Jane Smith"],
    --   "trustee": "First National Bank",
    --   "trust_start_date": "2024-01-01",
    --   "trust_term": "lifetime",
    --   "estimated_remainder": 350000.00,
    --   "current_value": 520000.00,
    --   "last_valuation_date": "2026-03-31"
    -- }
    -- CGA Example:
    -- {
    --   "annuity_rate": 0.059,
    --   "acga_rate_date": "2024-07-01",
    --   "annual_payment": 2950.00,
    --   "payment_frequency": "quarterly",
    --   "funding_amount": 50000.00,
    --   "funding_asset": "cash",
    --   "age_at_funding": 72,
    --   "charitable_deduction": 23400.00,
    --   "tax_free_portion": 0.42
    -- }

    -- Timeline tracking
    timeline        JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "prospect_identified_date": "2025-06-15",
    --   "prospect_identified_by": "AI model v2.1",
    --   "first_conversation_date": "2025-08-20",
    --   "intent_documented_date": "2025-11-10",
    --   "intent_form_on_file": true,
    --   "legal_commitment_date": "2026-02-15",
    --   "documentation_on_file": true,
    --   "expected_maturity_year": null,
    --   "matured_date": null,
    --   "matured_amount": null
    -- }

    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_planned_gift_tenant ON planned_gift(tenant_id);
CREATE INDEX idx_planned_gift_contact ON planned_gift(contact_id);
CREATE INDEX idx_planned_gift_vehicle ON planned_gift(tenant_id, gift_vehicle);
CREATE INDEX idx_planned_gift_status ON planned_gift(tenant_id, status);
CREATE INDEX idx_planned_gift_details ON planned_gift USING GIN (vehicle_details);
```

---

## Grants

### Grant Records Table

```sql
CREATE TABLE grant_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    funder_org_id   UUID REFERENCES organization(id),
    funder_contact_id UUID REFERENCES contact(id),
    fund_id         UUID REFERENCES fund(id),
    name            TEXT NOT NULL,
    grant_type      TEXT NOT NULL DEFAULT 'project',
    status          TEXT NOT NULL DEFAULT 'prospect',
    amount_requested NUMERIC(15,2),
    amount_awarded  NUMERIC(15,2),
    amount_received NUMERIC(15,2) DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',

    grant_details   JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "application_deadline": "2026-03-15",
    --   "submitted_date": "2026-03-10",
    --   "award_date": "2026-05-01",
    --   "start_date": "2026-07-01",
    --   "end_date": "2027-06-30",
    --   "program_area": "youth_education",
    --   "reporting_schedule": [
    --     {"type": "interim", "due": "2026-12-31"},
    --     {"type": "final", "due": "2027-09-30"}
    --   ],
    --   "restrictions": "Youth STEM education only",
    --   "match_required": true,
    --   "match_ratio": "1:1",
    --   "match_amount": 50000.00,
    --   "iati_activity_id": "US-EIN-123456789-GRANT-001"
    -- }

    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_grant_tenant ON grant_record(tenant_id);
CREATE INDEX idx_grant_funder ON grant_record(funder_org_id);
CREATE INDEX idx_grant_status ON grant_record(tenant_id, status);
CREATE INDEX idx_grant_details ON grant_record USING GIN (grant_details);
```

---

## Interactions, Events & Membership

### Interactions Table

```sql
CREATE TABLE interaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    user_id         UUID REFERENCES app_user(id),
    interaction_type TEXT NOT NULL,
    interaction_date TIMESTAMPTZ NOT NULL DEFAULT now(),

    interaction_details JSONB DEFAULT '{}',
    -- Email Example:
    -- {
    --   "direction": "outbound",
    --   "subject": "Thank you for your gift",
    --   "body_preview": "Dear John, thank you for...",
    --   "email_id": "msg-123",
    --   "opened": true,
    --   "clicked": true,
    --   "campaign_id": "uuid..."
    -- }
    -- Meeting Example:
    -- {
    --   "direction": "in_person",
    --   "subject": "Major gift cultivation meeting",
    --   "location": "Donor's office",
    --   "duration_minutes": 60,
    --   "attendees": ["uuid1", "uuid2"],
    --   "outcome": "Positive - interested in naming opportunity",
    --   "next_action": "Send proposal",
    --   "next_action_date": "2026-06-01",
    --   "moves_management_stage": "cultivation"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_interaction_contact ON interaction(contact_id);
CREATE INDEX idx_interaction_date ON interaction(tenant_id, interaction_date);
CREATE INDEX idx_interaction_type ON interaction(tenant_id, interaction_type);
```

### Events Table

```sql
CREATE TABLE event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    campaign_id     UUID REFERENCES campaign(id),
    name            TEXT NOT NULL,
    event_type      TEXT NOT NULL DEFAULT 'fundraiser',
    status          TEXT NOT NULL DEFAULT 'planned',
    start_date      TIMESTAMPTZ,
    end_date        TIMESTAMPTZ,

    event_details   JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "location_name": "Grand Ballroom, Hilton",
    --   "location_address": "123 Main St, City, ST 12345",
    --   "capacity": 300,
    --   "ticket_price": 150.00,
    --   "vip_price": 500.00,
    --   "table_price": 2500.00,
    --   "goal_amount": 250000.00,
    --   "raised_amount": 0,
    --   "sponsor_levels": [
    --     {"name": "Platinum", "amount": 25000, "benefits": "..."},
    --     {"name": "Gold", "amount": 10000, "benefits": "..."}
    --   ],
    --   "auction_items": true,
    --   "dress_code": "black_tie"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_event_tenant ON event(tenant_id);
CREATE INDEX idx_event_dates ON event(tenant_id, start_date);
CREATE INDEX idx_event_campaign ON event(campaign_id);
```

### Event Registrations Table

```sql
CREATE TABLE event_registration (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES event(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    registration_type TEXT NOT NULL DEFAULT 'attendee',
    status          TEXT NOT NULL DEFAULT 'registered',
    amount_paid     NUMERIC(10,2) DEFAULT 0,

    registration_details JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "ticket_count": 2,
    --   "table_number": "12",
    --   "dietary_notes": "vegetarian",
    --   "checked_in_at": "2026-10-15T18:30:00Z",
    --   "guest_names": ["Jane Smith"],
    --   "parking_needed": true,
    --   "accessibility_needs": "wheelchair"
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_event_reg_event ON event_registration(event_id);
CREATE INDEX idx_event_reg_contact ON event_registration(contact_id);
```

### Membership Table

```sql
CREATE TABLE membership (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    level_name      TEXT NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE,
    status          TEXT NOT NULL DEFAULT 'active',
    amount          NUMERIC(10,2),

    membership_details JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "auto_renew": true,
    --   "renewal_date": "2027-01-01",
    --   "benefits": ["newsletter", "early_event_access", "member_directory"],
    --   "member_number": "M-2026-001234",
    --   "joined_date": "2020-01-15",
    --   "consecutive_years": 6
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_membership_tenant ON membership(tenant_id);
CREATE INDEX idx_membership_contact ON membership(contact_id);
CREATE INDEX idx_membership_status ON membership(tenant_id, status);
```

---

## Platform & Consent

### Tenant Table

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',

    -- Tenant configuration and customisation
    settings        JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "fiscal_year_start_month": 7,
    --   "tax_id": "12-3456789",
    --   "plan_type": "professional",
    --   "features_enabled": ["planned_giving", "grants", "volunteers"],
    --   "custom_field_definitions": {
    --     "contact": [
    --       {"key": "class_year", "label": "Class Year", "type": "integer"},
    --       {"key": "degree", "label": "Degree", "type": "text"}
    --     ]
    --   },
    --   "branding": {
    --     "primary_color": "#1a5276",
    --     "logo_url": "https://..."
    --   },
    --   "integrations": {
    --     "stripe_account_id": "acct_...",
    --     "mailchimp_list_id": "abc123"
    --   }
    -- }

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Consent Table

```sql
CREATE TABLE consent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    channel         TEXT NOT NULL,
    consent_status  TEXT NOT NULL DEFAULT 'opted_in',
    consent_date    TIMESTAMPTZ NOT NULL DEFAULT now(),

    consent_details JSONB DEFAULT '{}',
    -- GDPR Example:
    -- {
    --   "lawful_basis": "consent",
    --   "source": "web_form",
    --   "ip_address": "192.168.1.1",
    --   "form_url": "https://example.org/subscribe",
    --   "gdpr_article": "6.1.a",
    --   "expiry_date": "2028-05-20"
    -- }
    -- CAN-SPAM Example:
    -- {
    --   "lawful_basis": "implied",
    --   "source": "existing_donor_relationship",
    --   "unsubscribe_link_included": true
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_contact ON consent(contact_id);
CREATE INDEX idx_consent_channel ON consent(contact_id, channel);
```

### Audit Log Table

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL,
    changes         JSONB,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_date ON audit_log(tenant_id, created_at);
```

---

## JSONB Query Examples

### Find all UK donors eligible for Gift Aid

```sql
SELECT id, first_name, last_name, email_primary
FROM contact
WHERE tenant_id = $1
    AND (regulatory_data->>'gift_aid_eligible')::boolean = true;
```

### Find contacts with high AI retention risk

```sql
SELECT id, first_name, last_name,
       (ai_scores->>'retention_risk')::numeric AS risk_score,
       ai_scores->>'churn_risk_factors' AS risk_factors
FROM contact
WHERE tenant_id = $1
    AND (ai_scores->>'retention_risk')::numeric > 0.7
ORDER BY (ai_scores->>'retention_risk')::numeric DESC;
```

### Find stock gifts with specific ticker

```sql
SELECT g.id, g.amount, g.gift_date,
       c.first_name, c.last_name,
       g.gift_details->>'ticker' AS ticker,
       (g.gift_details->>'shares')::int AS shares
FROM gift g
JOIN contact c ON c.id = g.contact_id
WHERE g.tenant_id = $1
    AND g.gift_type = 'stock'
    AND g.gift_details @> '{"ticker": "AAPL"}';
```

### Find planned gifts by vehicle type with trust details

```sql
SELECT pg.id, c.first_name, c.last_name,
       pg.gift_vehicle, pg.status, pg.estimated_amount,
       pg.vehicle_details->>'trust_name' AS trust_name,
       (pg.vehicle_details->>'payout_rate')::numeric AS payout_rate,
       pg.vehicle_details->>'trustee' AS trustee
FROM planned_gift pg
JOIN contact c ON c.id = pg.contact_id
WHERE pg.tenant_id = $1
    AND pg.gift_vehicle = 'charitable_remainder_trust'
    AND pg.status IN ('legally_committed', 'irrevocable');
```

### Custom field query (tenant-specific)

```sql
-- University: Find all donors from class of 1992
SELECT id, first_name, last_name, lifetime_giving
FROM contact
WHERE tenant_id = $1
    AND custom_fields @> '{"class_year": 1992}'
ORDER BY lifetime_giving DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Constituent Management | 5 | contact, household, organization, address, relationship |
| Gift & Pledge Processing | 5 | gift, soft_credit, pledge, recurring_donation, fund |
| Campaigns | 1 | campaign (with JSONB details) |
| Planned Giving | 1 | planned_gift (vehicle details in JSONB) |
| Grant Management | 1 | grant_record (deliverables in JSONB) |
| Events & Membership | 3 | event, event_registration, membership |
| Interactions | 1 | interaction (type-specific details in JSONB) |
| Consent & Compliance | 1 | consent (jurisdiction-specific details in JSONB) |
| Platform | 3 | tenant, app_user, audit_log |
| **Total** | **~21 core tables** | ~40% fewer than normalized model; JSONB absorbs variable fields |

---

## Key Design Decisions

1. **JSONB for variable fields, relational for invariant fields** — Every table follows the same pattern: columns that are always present, always queried, and always constrained are relational; columns that vary by type, jurisdiction, or tenant configuration go in JSONB. The dividing line is: "Would I always JOIN or WHERE on this field?" If yes, it is relational. If not, JSONB.

2. **GIN indexes on JSONB columns** — PostgreSQL GIN indexes support efficient containment (`@>`) and existence (`?`) queries on JSONB. These are created on the JSONB columns most likely to be queried (custom_fields, ai_scores, regulatory_data, gift_details). This prevents the common complaint that JSONB queries are slow.

3. **Polymorphic addresses** — Rather than separate address tables for contacts, households, and organizations (which triples address management code), a single `address` table uses `addressable_type` + `addressable_id` polymorphic columns. This is a deliberate trade-off: less referential integrity at the database level, but dramatically simpler application code.

4. **Planned giving vehicle details in JSONB** — Bequests, CRTs, CLTs, CGAs, and life insurance policies have fundamentally different field structures. Rather than a sparse table with 40+ nullable columns or five separate tables, the `vehicle_details` JSONB column stores vehicle-specific data. JSON Schema definitions in the application layer validate that each vehicle type has the required fields.

5. **Multi-jurisdiction tax data in JSONB** — US, UK, and Canadian tax receipt requirements are structurally different (quid pro quo vs. Gift Aid vs. eligible amount). Storing these in a `tax_data` JSONB column means adding support for a new jurisdiction requires zero schema migrations — just a new JSON Schema validation rule and UI form.

6. **Custom field definitions in tenant settings** — Each tenant can define custom fields (stored in `settings` JSONB on the `tenant` table), and the actual values are stored in `custom_fields` JSONB on the entity tables. This avoids the EAV anti-pattern (which creates performance nightmares) while providing genuine tenant-level schema extensibility.

7. **Payment processor abstraction via JSONB** — Different payment processors (Stripe, PayPal, SEPA, BACS) return different metadata. Storing processor-specific data in `payment_metadata` JSONB means the schema does not need to change when adding a new payment processor.

8. **Campaign statistics in JSONB** — Campaign performance metrics (donor count, average gift, cost per dollar) are denormalised into the `campaign_details` JSONB. This avoids expensive aggregation queries on the gift table for dashboard rendering, while the JSONB structure allows adding new metrics without schema changes.

9. **Charge history on recurring donations** — The `charge_history` JSONB array stores the most recent charges for quick dashboard access. Full charge history lives in the gift table. This is a deliberate denormalization that eliminates a JOIN for the most common recurring donation query pattern.

10. **JSON Schema as the contract** — Every JSONB column has an associated JSON Schema definition (maintained in the application codebase, not shown here). This provides validation, API documentation (via OpenAPI 3.1 which uses JSON Schema natively), and client SDK generation — compensating for the loss of database-level type enforcement on JSONB fields.
