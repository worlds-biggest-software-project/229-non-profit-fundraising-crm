# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Non-Profit Fundraising CRM · Created: 2026-05-22

## Philosophy

This model follows a fully normalized relational design where every domain concept has its own dedicated table with explicit foreign key relationships. The schema is modeled after the proven patterns established by Salesforce NPSP (Household Accounts, Opportunities as gifts, Contact Roles for attribution) and the Microsoft Common Data Model for Nonprofits (90+ entity definitions covering constituent management, fundraising, awards, and program delivery). Every relationship is explicit, every constraint is enforced at the database level, and every field has a defined type.

The normalized approach treats the database as the system of record and the enforcer of business rules. Gift attribution rules (soft credits, matching gifts, split gifts), pledge schedules, planned giving vehicles, and fund designations are all represented as distinct tables with referential integrity constraints. This prevents orphaned records, enforces accounting consistency required by FASB ASC 958, and makes the schema self-documenting.

This is the most conservative and well-understood approach. It maps directly to how enterprise nonprofit CRMs (Blackbaud Raiser's Edge, DonorPerfect) have structured their data for decades. The trade-off is a higher table count and more complex joins, but the payoff is data integrity that regulators and auditors can trust.

**Best for:** Organizations requiring strict data integrity, complex gift accounting (soft credits, matching gifts, planned giving vehicles), and regulatory compliance with FASB ASC 958 and IRS Form 990 reporting.

**Trade-offs:**
- (+) Maximum data integrity — foreign keys prevent orphaned records and enforce business rules
- (+) Standard SQL queries — no proprietary query language needed
- (+) Auditor-friendly — schema maps directly to accounting concepts
- (+) Well-understood by most development teams
- (-) High table count (~55-65 tables) increases schema complexity
- (-) Schema migrations required for new entity types or fields
- (-) Complex joins for cross-entity queries (e.g., "all giving by household across all funds")
- (-) Less flexible for jurisdiction-specific or tenant-specific custom fields

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FASB ASC 958 | Net asset classification (with/without donor restrictions) modeled as `restriction_type` on fund designations and gift records; enables ASC 958-compliant financial reporting |
| IRS Form 990 / Schedule B | Cumulative annual giving tracked per constituent to flag donors meeting Schedule B thresholds ($5,000 or 2% of revenue) |
| CASE Reporting Standards | Gift counting rules (outright gifts, pledges, planned gifts) modeled with `gift_type` and `counting_method` fields on gift records |
| PCI DSS 4.0 | Payment card data never stored; `payment_method` table references tokenized payment processor references only |
| GDPR / CCPA | Consent records stored in dedicated `consent` table with lawful basis, timestamp, and channel; supports right-to-erasure workflows |
| ISO 3166 | Country and subdivision codes used for all address records and jurisdiction modeling |
| ISO 4217 | Currency codes on all monetary fields to support multi-currency giving |
| vCard / RFC 6350 | Contact fields (name, email, phone, address) aligned with vCard properties for import/export portability |
| NCOA | Address records include `ncoa_updated_at` and `ncoa_move_type` fields for USPS address hygiene tracking |
| IATI Standard | Gift and grant records include fields mappable to IATI activity and transaction elements for international reporting |
| OAuth 2.0 / RFC 6749 | API authentication modeled with `oauth_client` and `oauth_token` tables |

---

## Core Constituent Management

### Organizations Table

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,                    -- EIN for US orgs
    org_type        TEXT NOT NULL DEFAULT 'nonprofit',
                    -- nonprofit, foundation, corporation, government, other
    website         TEXT,
    industry        TEXT,
    annual_revenue  NUMERIC(15,2),
    employee_count  INTEGER,
    notes           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organization_tenant ON organization(tenant_id);
CREATE INDEX idx_organization_name ON organization(tenant_id, name);
CREATE INDEX idx_organization_type ON organization(tenant_id, org_type);
```

### Households Table

```sql
CREATE TABLE household (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,           -- e.g., "Smith Family"
    formal_greeting TEXT,                    -- "Mr. and Mrs. John Smith"
    informal_greeting TEXT,                  -- "John and Jane"
    primary_contact_id UUID,                -- FK set after contacts created
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_household_tenant ON household(tenant_id);
CREATE INDEX idx_household_name ON household(tenant_id, name);
```

### Contacts Table

```sql
CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    household_id    UUID REFERENCES household(id),
    prefix          TEXT,                    -- Mr., Mrs., Dr., etc.
    first_name      TEXT NOT NULL,
    middle_name     TEXT,
    last_name       TEXT NOT NULL,
    suffix          TEXT,                    -- Jr., III, Esq., etc.
    nickname        TEXT,
    formal_name     TEXT,                    -- computed or overridden
    email_primary   TEXT,
    email_secondary TEXT,
    phone_primary   TEXT,
    phone_mobile    TEXT,
    phone_work      TEXT,
    date_of_birth   DATE,
    gender          TEXT,
    deceased        BOOLEAN NOT NULL DEFAULT false,
    deceased_date   DATE,
    donor_type      TEXT NOT NULL DEFAULT 'individual',
                    -- individual, major_donor, planned_giving_prospect, lapsed
    communication_preference TEXT DEFAULT 'email',
                    -- email, mail, phone, sms, do_not_contact
    engagement_score INTEGER,               -- 0-100 computed score
    lifetime_giving NUMERIC(15,2) DEFAULT 0,
    first_gift_date DATE,
    last_gift_date  DATE,
    largest_gift    NUMERIC(15,2),
    total_gift_count INTEGER DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
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
```

### Addresses Table

```sql
CREATE TABLE address (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID REFERENCES contact(id),
    household_id    UUID REFERENCES household(id),
    organization_id UUID REFERENCES organization(id),
    address_type    TEXT NOT NULL DEFAULT 'home',
                    -- home, work, seasonal, other
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    street_line_1   TEXT NOT NULL,
    street_line_2   TEXT,
    city            TEXT NOT NULL,
    state_province  TEXT,                    -- ISO 3166-2 subdivision code
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL DEFAULT 'US',  -- ISO 3166-1 alpha-2
    latitude        NUMERIC(10,7),
    longitude       NUMERIC(10,7),
    seasonal_start  DATE,                   -- for snowbird addresses
    seasonal_end    DATE,
    ncoa_updated_at TIMESTAMPTZ,            -- last NCOA processing date
    ncoa_move_type  TEXT,                   -- individual, family, business
    is_valid        BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT address_owner_check CHECK (
        (contact_id IS NOT NULL)::int +
        (household_id IS NOT NULL)::int +
        (organization_id IS NOT NULL)::int = 1
    )
);

CREATE INDEX idx_address_contact ON address(contact_id);
CREATE INDEX idx_address_household ON address(household_id);
CREATE INDEX idx_address_postal ON address(postal_code);
```

### Relationships Table

```sql
CREATE TABLE relationship (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id_a    UUID NOT NULL REFERENCES contact(id),
    contact_id_b    UUID NOT NULL REFERENCES contact(id),
    relationship_type TEXT NOT NULL,
                    -- spouse, parent, child, sibling, employer, employee,
                    -- advisor, friend, board_member, referrer
    reciprocal_type TEXT,                   -- auto-set inverse (parent <-> child)
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    notes           TEXT,
    start_date      DATE,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT relationship_not_self CHECK (contact_id_a != contact_id_b)
);

CREATE INDEX idx_relationship_contact_a ON relationship(contact_id_a);
CREATE INDEX idx_relationship_contact_b ON relationship(contact_id_b);
CREATE INDEX idx_relationship_type ON relationship(tenant_id, relationship_type);
```

### Affiliations Table

```sql
CREATE TABLE affiliation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    role            TEXT,                    -- CEO, Board Member, Volunteer, etc.
    title           TEXT,
    department      TEXT,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    start_date      DATE,
    end_date        DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_affiliation_contact ON affiliation(contact_id);
CREATE INDEX idx_affiliation_org ON affiliation(organization_id);
```

---

## Gift & Pledge Processing

### Funds / General Accounting Units

```sql
CREATE TABLE fund (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    code            TEXT,                    -- short code for accounting
    description     TEXT,
    fund_type       TEXT NOT NULL DEFAULT 'general',
                    -- general, restricted, endowment, capital_campaign
    restriction_type TEXT NOT NULL DEFAULT 'without_restrictions',
                    -- without_restrictions, with_donor_restrictions
                    -- aligns with FASB ASC 958 / ASU 2016-14
    goal_amount     NUMERIC(15,2),
    start_date      DATE,
    end_date        DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fund_tenant ON fund(tenant_id);
CREATE INDEX idx_fund_type ON fund(tenant_id, fund_type);
```

### Campaigns Table

```sql
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_campaign_id UUID REFERENCES campaign(id),
    name            TEXT NOT NULL,
    campaign_type   TEXT NOT NULL DEFAULT 'annual_fund',
                    -- annual_fund, capital_campaign, major_gift,
                    -- planned_giving, peer_to_peer, event, direct_mail, digital
    status          TEXT NOT NULL DEFAULT 'planned',
                    -- planned, active, completed, cancelled
    goal_amount     NUMERIC(15,2),
    raised_amount   NUMERIC(15,2) DEFAULT 0,
    start_date      DATE,
    end_date        DATE,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_tenant ON campaign(tenant_id);
CREATE INDEX idx_campaign_parent ON campaign(parent_campaign_id);
CREATE INDEX idx_campaign_status ON campaign(tenant_id, status);
CREATE INDEX idx_campaign_dates ON campaign(tenant_id, start_date, end_date);
```

### Appeals Table

```sql
CREATE TABLE appeal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    campaign_id     UUID REFERENCES campaign(id),
    name            TEXT NOT NULL,
    appeal_type     TEXT NOT NULL DEFAULT 'direct_mail',
                    -- direct_mail, email, phone, digital, event, personal_visit
    code            TEXT,                    -- appeal source code for tracking
    mail_date       DATE,
    response_deadline DATE,
    goal_amount     NUMERIC(15,2),
    cost            NUMERIC(15,2),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_appeal_campaign ON appeal(campaign_id);
CREATE INDEX idx_appeal_tenant ON appeal(tenant_id);
```

### Gifts Table

```sql
CREATE TABLE gift (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    household_id    UUID REFERENCES household(id),
    campaign_id     UUID REFERENCES campaign(id),
    appeal_id       UUID REFERENCES appeal(id),
    fund_id         UUID REFERENCES fund(id),
    pledge_id       UUID REFERENCES pledge(id),     -- if this is a pledge payment
    gift_type       TEXT NOT NULL DEFAULT 'cash',
                    -- cash, check, credit_card, ach, wire, stock, in_kind,
                    -- matching, daf, cryptocurrency, other
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    exchange_rate   NUMERIC(12,6) DEFAULT 1.0,
    amount_usd      NUMERIC(15,2),                  -- normalised for reporting
    gift_date       DATE NOT NULL,
    received_date   DATE,
    batch_id        UUID REFERENCES gift_batch(id),
    check_number    TEXT,
    reference_number TEXT,                           -- payment processor ref
    is_anonymous    BOOLEAN NOT NULL DEFAULT false,
    is_recurring    BOOLEAN NOT NULL DEFAULT false,
    recurring_donation_id UUID REFERENCES recurring_donation(id),
    tribute_type    TEXT,                            -- in_honor_of, in_memory_of
    tribute_name    TEXT,
    acknowledgment_status TEXT DEFAULT 'pending',
                    -- pending, sent, not_required
    acknowledgment_date DATE,
    tax_receipt_status TEXT DEFAULT 'pending',
                    -- pending, sent, not_required
    tax_deductible_amount NUMERIC(15,2),
    non_deductible_amount NUMERIC(15,2) DEFAULT 0,  -- e.g., fair market value of benefit
    counting_method TEXT DEFAULT 'outright',
                    -- outright, pledge, planned, matching (CASE standards)
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gift_tenant ON gift(tenant_id);
CREATE INDEX idx_gift_contact ON gift(contact_id);
CREATE INDEX idx_gift_household ON gift(household_id);
CREATE INDEX idx_gift_campaign ON gift(campaign_id);
CREATE INDEX idx_gift_fund ON gift(fund_id);
CREATE INDEX idx_gift_date ON gift(tenant_id, gift_date);
CREATE INDEX idx_gift_type ON gift(tenant_id, gift_type);
CREATE INDEX idx_gift_pledge ON gift(pledge_id);
CREATE INDEX idx_gift_batch ON gift(batch_id);
CREATE INDEX idx_gift_recurring ON gift(recurring_donation_id);
CREATE INDEX idx_gift_ack_status ON gift(tenant_id, acknowledgment_status) WHERE acknowledgment_status = 'pending';
```

### Gift Batch Table

```sql
CREATE TABLE gift_batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    batch_number    TEXT NOT NULL,
    batch_date      DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'open',
                    -- open, closed, posted
    expected_count  INTEGER,
    expected_amount NUMERIC(15,2),
    actual_count    INTEGER DEFAULT 0,
    actual_amount   NUMERIC(15,2) DEFAULT 0,
    entered_by      UUID REFERENCES app_user(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gift_batch_tenant ON gift_batch(tenant_id);
CREATE INDEX idx_gift_batch_status ON gift_batch(tenant_id, status);
```

### Fund Allocations (Split Gifts)

```sql
CREATE TABLE gift_fund_allocation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gift_id         UUID NOT NULL REFERENCES gift(id) ON DELETE CASCADE,
    fund_id         UUID NOT NULL REFERENCES fund(id),
    amount          NUMERIC(15,2) NOT NULL,
    percentage      NUMERIC(5,2),           -- optional, for display
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gift_fund_alloc_gift ON gift_fund_allocation(gift_id);
CREATE INDEX idx_gift_fund_alloc_fund ON gift_fund_allocation(fund_id);
```

### Soft Credits Table

```sql
CREATE TABLE soft_credit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    gift_id         UUID NOT NULL REFERENCES gift(id) ON DELETE CASCADE,
    contact_id      UUID NOT NULL REFERENCES contact(id),
    credit_type     TEXT NOT NULL DEFAULT 'soft',
                    -- soft, matching, solicitor, household, influencer
    amount          NUMERIC(15,2) NOT NULL,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_soft_credit_gift ON soft_credit(gift_id);
CREATE INDEX idx_soft_credit_contact ON soft_credit(contact_id);
CREATE INDEX idx_soft_credit_type ON soft_credit(credit_type);
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
    balance         NUMERIC(15,2) NOT NULL,  -- remaining unpaid
    pledge_date     DATE NOT NULL,
    start_date      DATE,
    end_date        DATE,
    frequency       TEXT DEFAULT 'one_time',
                    -- one_time, monthly, quarterly, semi_annual, annual
    installment_amount NUMERIC(15,2),
    next_payment_date DATE,
    status          TEXT NOT NULL DEFAULT 'active',
                    -- active, fulfilled, written_off, cancelled
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pledge_tenant ON pledge(tenant_id);
CREATE INDEX idx_pledge_contact ON pledge(contact_id);
CREATE INDEX idx_pledge_status ON pledge(tenant_id, status);
CREATE INDEX idx_pledge_next_payment ON pledge(tenant_id, next_payment_date) WHERE status = 'active';
```

### Recurring Donations Table

```sql
CREATE TABLE recurring_donation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    fund_id         UUID REFERENCES fund(id),
    campaign_id     UUID REFERENCES campaign(id),
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    frequency       TEXT NOT NULL DEFAULT 'monthly',
                    -- weekly, biweekly, monthly, quarterly, annual
    payment_method  TEXT NOT NULL,
                    -- credit_card, ach, sepa_direct_debit, paypal
    payment_token   TEXT,                    -- tokenised payment reference
    processor_subscription_id TEXT,          -- Stripe subscription ID, etc.
    start_date      DATE NOT NULL,
    end_date        DATE,
    next_charge_date DATE,
    last_charge_date DATE,
    last_charge_status TEXT,                 -- succeeded, failed, pending
    consecutive_failures INTEGER DEFAULT 0,
    total_given     NUMERIC(15,2) DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'active',
                    -- active, paused, cancelled, failed, completed
    cancellation_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_recurring_tenant ON recurring_donation(tenant_id);
CREATE INDEX idx_recurring_contact ON recurring_donation(contact_id);
CREATE INDEX idx_recurring_status ON recurring_donation(tenant_id, status);
CREATE INDEX idx_recurring_next_charge ON recurring_donation(next_charge_date) WHERE status = 'active';
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
                    -- bequest, charitable_remainder_trust, charitable_lead_trust,
                    -- charitable_gift_annuity, retained_life_estate,
                    -- life_insurance, retirement_plan, daf
    status          TEXT NOT NULL DEFAULT 'prospect',
                    -- prospect, cultivating, intent_documented, legally_committed,
                    -- irrevocable, matured, revoked
    estimated_amount NUMERIC(15,2),
    face_value      NUMERIC(15,2),          -- for insurance/annuity
    matured_amount  NUMERIC(15,2),          -- actual received amount
    fund_id         UUID REFERENCES fund(id),
    campaign_id     UUID REFERENCES campaign(id),

    -- Bequest-specific fields
    bequest_type    TEXT,                    -- specific, residuary, percentage, contingent
    bequest_percentage NUMERIC(5,2),
    estate_name     TEXT,
    attorney_name   TEXT,
    attorney_contact TEXT,

    -- Trust/Annuity-specific fields
    trust_type      TEXT,                   -- crat, crut, clat, clut
    payout_rate     NUMERIC(5,4),           -- e.g., 0.0500 for 5%
    payout_frequency TEXT,                  -- monthly, quarterly, semi_annual, annual
    remainder_value NUMERIC(15,2),
    income_beneficiary TEXT,
    trustee         TEXT,
    trust_start_date DATE,
    trust_term_years INTEGER,

    -- ACGA annuity rates
    annuity_rate    NUMERIC(5,4),           -- ACGA recommended rate
    annual_payment  NUMERIC(15,2),

    intent_date     DATE,
    legal_commitment_date DATE,
    maturity_date   DATE,
    revocation_date DATE,
    documentation_on_file BOOLEAN DEFAULT false,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_planned_gift_tenant ON planned_gift(tenant_id);
CREATE INDEX idx_planned_gift_contact ON planned_gift(contact_id);
CREATE INDEX idx_planned_gift_vehicle ON planned_gift(tenant_id, gift_vehicle);
CREATE INDEX idx_planned_gift_status ON planned_gift(tenant_id, status);
```

---

## Grant Management

### Grants Table

```sql
CREATE TABLE grant_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    funder_org_id   UUID REFERENCES organization(id),
    funder_contact_id UUID REFERENCES contact(id),
    fund_id         UUID REFERENCES fund(id),
    campaign_id     UUID REFERENCES campaign(id),
    name            TEXT NOT NULL,
    grant_type      TEXT NOT NULL DEFAULT 'project',
                    -- project, operating, capital, capacity_building, fellowship
    status          TEXT NOT NULL DEFAULT 'prospect',
                    -- prospect, loi_submitted, application_submitted,
                    -- awarded, declined, active, reporting, closed
    amount_requested NUMERIC(15,2),
    amount_awarded  NUMERIC(15,2),
    amount_received NUMERIC(15,2) DEFAULT 0,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    application_deadline DATE,
    submitted_date  DATE,
    award_date      DATE,
    start_date      DATE,
    end_date        DATE,
    reporting_requirements TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_grant_tenant ON grant_record(tenant_id);
CREATE INDEX idx_grant_funder ON grant_record(funder_org_id);
CREATE INDEX idx_grant_status ON grant_record(tenant_id, status);
CREATE INDEX idx_grant_deadline ON grant_record(tenant_id, application_deadline);
```

### Grant Deliverables Table

```sql
CREATE TABLE grant_deliverable (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grant_id        UUID NOT NULL REFERENCES grant_record(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    due_date        DATE,
    status          TEXT NOT NULL DEFAULT 'pending',
                    -- pending, in_progress, submitted, accepted
    submitted_date  DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_grant_deliverable_grant ON grant_deliverable(grant_id);
CREATE INDEX idx_grant_deliverable_due ON grant_deliverable(due_date) WHERE status != 'accepted';
```

---

## Interactions & Engagement

### Interactions Table

```sql
CREATE TABLE interaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    user_id         UUID REFERENCES app_user(id),     -- staff member who logged it
    interaction_type TEXT NOT NULL,
                    -- email_sent, email_received, phone_call, meeting,
                    -- personal_visit, letter_sent, event_attendance,
                    -- volunteer_activity, social_media, note
    direction       TEXT,                    -- inbound, outbound
    subject         TEXT,
    body            TEXT,
    interaction_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    campaign_id     UUID REFERENCES campaign(id),
    appeal_id       UUID REFERENCES appeal(id),
    duration_minutes INTEGER,
    outcome         TEXT,
    next_action     TEXT,
    next_action_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_interaction_contact ON interaction(contact_id);
CREATE INDEX idx_interaction_tenant_date ON interaction(tenant_id, interaction_date);
CREATE INDEX idx_interaction_type ON interaction(tenant_id, interaction_type);
CREATE INDEX idx_interaction_user ON interaction(user_id);
```

### Tasks Table

```sql
CREATE TABLE task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    contact_id      UUID REFERENCES contact(id),
    assigned_to     UUID NOT NULL REFERENCES app_user(id),
    created_by      UUID NOT NULL REFERENCES app_user(id),
    task_type       TEXT NOT NULL DEFAULT 'follow_up',
                    -- follow_up, call, meeting, proposal, acknowledgment,
                    -- stewardship, data_entry, other
    subject         TEXT NOT NULL,
    description     TEXT,
    due_date        DATE,
    priority        TEXT NOT NULL DEFAULT 'normal',
                    -- low, normal, high, urgent
    status          TEXT NOT NULL DEFAULT 'open',
                    -- open, in_progress, completed, cancelled
    completed_date  DATE,
    campaign_id     UUID REFERENCES campaign(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_task_assigned ON task(assigned_to, status);
CREATE INDEX idx_task_contact ON task(contact_id);
CREATE INDEX idx_task_due ON task(tenant_id, due_date) WHERE status IN ('open', 'in_progress');
```

---

## Events & Membership

### Events Table

```sql
CREATE TABLE event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    campaign_id     UUID REFERENCES campaign(id),
    name            TEXT NOT NULL,
    event_type      TEXT NOT NULL DEFAULT 'fundraiser',
                    -- fundraiser, gala, auction, luncheon, golf_tournament,
                    -- webinar, volunteer_activity, stewardship, other
    status          TEXT NOT NULL DEFAULT 'planned',
                    -- planned, registration_open, in_progress, completed, cancelled
    start_date      TIMESTAMPTZ,
    end_date        TIMESTAMPTZ,
    location_name   TEXT,
    location_address TEXT,
    capacity        INTEGER,
    ticket_price    NUMERIC(10,2),
    goal_amount     NUMERIC(15,2),
    raised_amount   NUMERIC(15,2) DEFAULT 0,
    description     TEXT,
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
                    -- attendee, sponsor, volunteer, speaker, vip
    ticket_count    INTEGER DEFAULT 1,
    amount_paid     NUMERIC(10,2) DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'registered',
                    -- registered, confirmed, attended, cancelled, no_show
    checked_in_at   TIMESTAMPTZ,
    table_number    TEXT,
    dietary_notes   TEXT,
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
    membership_level_id UUID NOT NULL REFERENCES membership_level(id),
    start_date      DATE NOT NULL,
    end_date        DATE,
    renewal_date    DATE,
    status          TEXT NOT NULL DEFAULT 'active',
                    -- active, expired, cancelled, pending_renewal
    auto_renew      BOOLEAN NOT NULL DEFAULT false,
    amount          NUMERIC(10,2),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_membership_tenant ON membership(tenant_id);
CREATE INDEX idx_membership_contact ON membership(contact_id);
CREATE INDEX idx_membership_status ON membership(tenant_id, status);
CREATE INDEX idx_membership_renewal ON membership(tenant_id, renewal_date) WHERE status = 'active';
```

### Membership Levels Table

```sql
CREATE TABLE membership_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    description     TEXT,
    annual_amount   NUMERIC(10,2),
    sort_order      INTEGER DEFAULT 0,
    benefits        TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Volunteer Management

### Volunteer Records Table

```sql
CREATE TABLE volunteer_record (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID NOT NULL REFERENCES contact(id) UNIQUE,
    skills          TEXT[],
    availability    TEXT,
    emergency_contact_name TEXT,
    emergency_contact_phone TEXT,
    background_check_status TEXT DEFAULT 'not_started',
                    -- not_started, pending, passed, failed
    background_check_date DATE,
    total_hours     NUMERIC(10,2) DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_volunteer_contact ON volunteer_record(contact_id);
```

### Volunteer Hours Table

```sql
CREATE TABLE volunteer_hours (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    volunteer_id    UUID NOT NULL REFERENCES volunteer_record(id),
    event_id        UUID REFERENCES event(id),
    activity_name   TEXT NOT NULL,
    hours           NUMERIC(6,2) NOT NULL,
    activity_date   DATE NOT NULL,
    status          TEXT NOT NULL DEFAULT 'logged',
                    -- logged, approved, rejected
    approved_by     UUID REFERENCES app_user(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_vol_hours_volunteer ON volunteer_hours(volunteer_id);
CREATE INDEX idx_vol_hours_date ON volunteer_hours(activity_date);
```

---

## Wealth Screening & AI Scoring

### Wealth Screening Table

```sql
CREATE TABLE wealth_screening (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    provider        TEXT NOT NULL,           -- iwave, donorsearch, kindsight
    screening_date  TIMESTAMPTZ NOT NULL,
    capacity_rating TEXT,                    -- e.g., "$1M-$5M"
    capacity_score  INTEGER,                -- 1-10 or provider-specific
    propensity_score INTEGER,               -- likelihood to give
    affinity_score  INTEGER,                -- connection to cause
    major_gift_likelihood NUMERIC(5,4),     -- 0.0000-1.0000
    philanthropic_giving_total NUMERIC(15,2),
    real_estate_value NUMERIC(15,2),
    stock_holdings  NUMERIC(15,2),
    business_interests TEXT,
    board_memberships TEXT,
    political_giving NUMERIC(15,2),
    raw_data        JSONB,                  -- full provider response
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wealth_contact ON wealth_screening(contact_id);
CREATE INDEX idx_wealth_date ON wealth_screening(contact_id, screening_date DESC);
CREATE INDEX idx_wealth_capacity ON wealth_screening(capacity_score DESC);
```

### AI Scores Table

```sql
CREATE TABLE ai_score (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    score_type      TEXT NOT NULL,
                    -- retention_risk, major_gift_propensity,
                    -- planned_giving_propensity, upgrade_likelihood,
                    -- churn_risk, ask_amount_recommendation
    score_value     NUMERIC(8,4) NOT NULL,
    confidence      NUMERIC(5,4),
    model_version   TEXT NOT NULL,
    features_used   JSONB,                  -- input features for explainability
    explanation     TEXT,                    -- human-readable reasoning
    recommended_action TEXT,
    scored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ
);

CREATE INDEX idx_ai_score_contact ON ai_score(contact_id);
CREATE INDEX idx_ai_score_type ON ai_score(contact_id, score_type);
CREATE INDEX idx_ai_score_value ON ai_score(score_type, score_value DESC);
```

---

## Communication & Consent

### Communication Preferences & Consent

```sql
CREATE TABLE consent (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contact_id      UUID NOT NULL REFERENCES contact(id),
    channel         TEXT NOT NULL,
                    -- email, sms, phone, direct_mail, push_notification
    consent_status  TEXT NOT NULL DEFAULT 'opted_in',
                    -- opted_in, opted_out, not_set
    lawful_basis    TEXT,                    -- consent, legitimate_interest (GDPR)
    source          TEXT,                    -- web_form, import, manual, preference_centre
    consent_date    TIMESTAMPTZ NOT NULL DEFAULT now(),
    expiry_date     TIMESTAMPTZ,
    ip_address      INET,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_contact ON consent(contact_id);
CREATE INDEX idx_consent_channel ON consent(contact_id, channel);
```

### Email Communications Table

```sql
CREATE TABLE email_communication (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    campaign_id     UUID REFERENCES campaign(id),
    appeal_id       UUID REFERENCES appeal(id),
    subject         TEXT NOT NULL,
    from_name       TEXT,
    from_email      TEXT,
    template_id     UUID,
    sent_count      INTEGER DEFAULT 0,
    open_count      INTEGER DEFAULT 0,
    click_count     INTEGER DEFAULT 0,
    bounce_count    INTEGER DEFAULT 0,
    unsubscribe_count INTEGER DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'draft',
                    -- draft, scheduled, sending, sent, cancelled
    scheduled_at    TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_email_comm_tenant ON email_communication(tenant_id);
CREATE INDEX idx_email_comm_campaign ON email_communication(campaign_id);
```

---

## Multi-Tenant & Platform

### Tenant Table

```sql
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    tax_id          TEXT,
    country_code    CHAR(2) NOT NULL DEFAULT 'US',
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    fiscal_year_start_month INTEGER NOT NULL DEFAULT 1,
    plan_type       TEXT NOT NULL DEFAULT 'free',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Application Users Table

```sql
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'staff',
                    -- admin, manager, staff, readonly, api_only
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_user_tenant ON app_user(tenant_id);
CREATE INDEX idx_user_email ON app_user(email);
```

### Audit Log Table

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID REFERENCES app_user(id),
    entity_type     TEXT NOT NULL,           -- contact, gift, pledge, etc.
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL,           -- create, update, delete, merge
    changes         JSONB,                  -- {field: {old: x, new: y}}
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_date ON audit_log(tenant_id, created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Constituent Management | 7 | contact, household, organization, address, relationship, affiliation, consent |
| Gift & Pledge Processing | 8 | gift, gift_batch, gift_fund_allocation, soft_credit, pledge, recurring_donation, fund, appeal |
| Campaigns | 2 | campaign (hierarchical), email_communication |
| Planned Giving | 1 | planned_gift (consolidated with vehicle-specific fields) |
| Grant Management | 2 | grant_record, grant_deliverable |
| Events & Membership | 4 | event, event_registration, membership, membership_level |
| Interactions & Tasks | 2 | interaction, task |
| Volunteer Management | 2 | volunteer_record, volunteer_hours |
| Wealth Screening & AI | 2 | wealth_screening, ai_score |
| Platform & Multi-Tenant | 4 | tenant, app_user, audit_log, oauth tables |
| **Total** | **~34 core tables** | Additional junction and lookup tables bring full count to ~40-45 |

---

## Key Design Decisions

1. **Household model** — Contacts belong to households, mirroring the Salesforce NPSP pattern that has become the nonprofit CRM industry standard. Gifts can be attributed at both contact and household level, and household lifetime giving is derivable from member contacts.

2. **Fund/designation alignment with FASB ASC 958** — The `fund` table models general accounting units with `restriction_type` using the two-class system (with/without donor restrictions) mandated by ASU 2016-14. Every gift can be allocated across multiple funds via the `gift_fund_allocation` junction table.

3. **Consolidated planned giving table** — Rather than separate tables per gift vehicle (which would create sparse, rarely-used tables), a single `planned_gift` table uses vehicle-type-specific nullable fields. This is a deliberate denormalization trade-off: it reduces join complexity for a domain area where query patterns are simpler and record volumes are low.

4. **Soft credit as a separate table** — Following DonorPerfect and Blackbaud patterns, soft credits are first-class records that can attribute a single gift to multiple contacts (solicitor credit, household credit, matching gift credit). This enables accurate LYBUNT/SYBUNT reporting where a contact can be credited for giving they influenced but did not directly make.

5. **Batch gift entry support** — The `gift_batch` table supports high-volume annual fund processing workflows where staff enter checks in batches and reconcile totals before posting, a critical workflow for organizations processing hundreds of direct mail responses.

6. **Tenant-scoped indexes** — All indexes include `tenant_id` as the leading column to support row-level security and efficient multi-tenant queries without schema-per-tenant overhead.

7. **NCOA integration in address records** — Address records carry `ncoa_updated_at` and `ncoa_move_type` fields to track USPS National Change of Address processing, which nonprofits run quarterly before major mailings.

8. **AI score explainability** — The `ai_score` table stores not just the score value but the model version, input features (as JSONB), and human-readable explanation. This supports the AI-native positioning and enables gift officers to understand and trust ML recommendations.

9. **Multi-currency with USD normalization** — All monetary fields carry `currency_code` (ISO 4217) and an optional `amount_usd` for normalized reporting, supporting international nonprofits while keeping reporting simple.

10. **Separate interaction and task tables** — Interactions record what happened (past tense); tasks record what needs to happen (future tense). Both link to contacts, enabling the complete chronological timeline that every nonprofit CRM requires.
