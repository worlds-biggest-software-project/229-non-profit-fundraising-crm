# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Non-Profit Fundraising CRM · Created: 2026-05-22

## Philosophy

This model treats every change to the system as an immutable domain event recorded in an append-only event store. The current state of any entity — a donor, a gift, a pledge, a planned giving commitment — is derived by replaying the sequence of events that created and modified it. Read models (projections) are materialised from the event stream and optimised for specific query patterns: dashboards, retention reports, LYBUNT/SYBUNT lists, and AI feature extraction.

The event-sourced approach is particularly powerful for nonprofit fundraising because the domain has inherent audit requirements. FASB ASC 958 requires tracking how gift restrictions change over time. IRS Form 990 Schedule B requires knowing cumulative giving at year-end. Planned giving vehicles change status over decades (prospect to intent to legal commitment to maturity). An event-sourced model captures all of this naturally — "what was true on December 31st?" is answered by replaying events up to that date, not by querying a snapshot that may have been overwritten.

This pattern is used in financial systems (banking ledgers, trading platforms) where immutability and auditability are non-negotiable. The CQRS (Command Query Responsibility Segregation) complement separates write operations (commands that produce events) from read operations (queries against materialised projections), allowing each side to be optimised independently. Write throughput is high (append-only), and read models can be purpose-built for specific use cases (donor retention dashboard, campaign performance report, AI feature store).

**Best for:** Organizations requiring complete audit trails, temporal queries ("what did this donor's record look like on date X?"), AI/ML analytics on change patterns, and regulatory compliance with time-dependent reporting requirements.

**Trade-offs:**
- (+) Complete, immutable audit trail — every change is preserved forever
- (+) Temporal queries are trivial — replay to any point in time
- (+) AI/ML feature extraction from event streams (recency, frequency, monetary patterns)
- (+) Independent scaling of reads and writes
- (+) New projections can be built from existing events without data migration
- (-) Higher implementation complexity than traditional CRUD
- (-) Event schema evolution requires careful versioning (upcasting)
- (-) Eventual consistency between event store and read models (typically milliseconds)
- (-) Larger storage requirements (events are never deleted)
- (-) Debugging requires understanding event replay, not just current state

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FASB ASC 958 | Gift restriction changes tracked as events (`GiftRestrictionChanged`); temporal replay produces ASC 958-compliant snapshots at any fiscal year-end |
| IRS Form 990 / Schedule B | Cumulative giving projections computed by replaying `GiftReceived` events within a fiscal year; Schedule B threshold alerts emitted as derived events |
| CASE Reporting Standards | Campaign gift counting derived from event streams with `counting_method` metadata; projections can apply different CASE counting rules to the same events |
| PCI DSS 4.0 | Payment events store only tokenised references; raw card data never enters the event store |
| GDPR / CCPA | Right-to-erasure implemented via crypto-shredding: donor-specific encryption keys deleted, making personal data in events unreadable while preserving aggregate event structure |
| AFP Donor Bill of Rights | Consent changes tracked as `ConsentGranted`/`ConsentRevoked` events with full provenance |
| ISO 4217 | Currency codes stored on all monetary events |
| ISO 3166 | Country/subdivision codes on address-related events |

---

## Event Store (Source of Truth)

### Domain Events Table

```sql
-- The single source of truth. All other tables are projections.
CREATE TABLE domain_event (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    aggregate_type  TEXT NOT NULL,
                    -- Contact, Household, Gift, Pledge, PlannedGift,
                    -- Campaign, Membership, Grant, Event, Interaction
    aggregate_id    UUID NOT NULL,           -- ID of the entity this event belongs to
    event_type      TEXT NOT NULL,           -- e.g., 'ContactCreated', 'GiftReceived'
    event_version   INTEGER NOT NULL,        -- schema version for upcasting
    sequence_number BIGINT NOT NULL,         -- per-aggregate ordering
    payload         JSONB NOT NULL,          -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
                    -- {user_id, ip_address, correlation_id, causation_id, source}
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    stored_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE(aggregate_type, aggregate_id, sequence_number)
);

-- Append-only: no UPDATE or DELETE triggers/policies should be applied
-- Partitioned by month for storage management
CREATE INDEX idx_event_aggregate ON domain_event(aggregate_type, aggregate_id, sequence_number);
CREATE INDEX idx_event_tenant ON domain_event(tenant_id, occurred_at);
CREATE INDEX idx_event_type ON domain_event(event_type, occurred_at);
CREATE INDEX idx_event_occurred ON domain_event(occurred_at);
```

### Event Type Catalogue

Below are the core domain events. Each event's `payload` contains the fields listed.

**Contact Aggregate:**
- `ContactCreated` — {first_name, last_name, email, phone, household_id, donor_type, source}
- `ContactUpdated` — {changed_fields: {field: {old, new}}}
- `ContactMerged` — {surviving_id, merged_id, fields_from_merged}
- `ContactDeceased` — {deceased_date}
- `ContactAddressAdded` — {address_type, street, city, state, postal_code, country_code}
- `ContactAddressUpdated` — {address_id, changed_fields}
- `ContactAddressNcoaUpdated` — {address_id, ncoa_move_type, new_address}
- `ContactRelationshipAdded` — {related_contact_id, relationship_type}
- `ContactAffiliationAdded` — {organization_id, role, title}
- `ContactEngagementScoreUpdated` — {old_score, new_score, model_version, factors}

**Gift Aggregate:**
- `GiftReceived` — {contact_id, amount, currency, gift_type, gift_date, fund_id, campaign_id, appeal_id, batch_id}
- `GiftAllocated` — {gift_id, allocations: [{fund_id, amount}]}
- `GiftSoftCredited` — {gift_id, contact_id, credit_type, amount}
- `GiftAcknowledged` — {gift_id, acknowledgment_type, sent_date}
- `GiftTaxReceiptIssued` — {gift_id, deductible_amount, receipt_number}
- `GiftReversed` — {gift_id, reason, reversed_by}
- `GiftRestrictionChanged` — {gift_id, old_restriction, new_restriction, reason}

**Pledge Aggregate:**
- `PledgeCreated` — {contact_id, amount, frequency, start_date, end_date, fund_id, campaign_id}
- `PledgePaymentReceived` — {pledge_id, gift_id, amount, remaining_balance}
- `PledgeWrittenOff` — {pledge_id, write_off_amount, reason}
- `PledgeCancelled` — {pledge_id, reason}

**Recurring Donation Aggregate:**
- `RecurringDonationCreated` — {contact_id, amount, frequency, payment_method, fund_id}
- `RecurringDonationCharged` — {recurring_id, gift_id, amount, status}
- `RecurringDonationFailed` — {recurring_id, failure_reason, consecutive_failures}
- `RecurringDonationPaused` — {recurring_id, reason}
- `RecurringDonationCancelled` — {recurring_id, reason}
- `RecurringDonationAmountChanged` — {recurring_id, old_amount, new_amount}

**Planned Gift Aggregate:**
- `PlannedGiftProspectIdentified` — {contact_id, vehicle, estimated_amount, identification_source}
- `PlannedGiftIntentDocumented` — {planned_gift_id, intent_date, documentation_type}
- `PlannedGiftLegallyCommitted` — {planned_gift_id, legal_date, face_value, vehicle_details}
- `PlannedGiftMatured` — {planned_gift_id, matured_amount, maturity_date}
- `PlannedGiftRevoked` — {planned_gift_id, revocation_date, reason}

**Campaign Aggregate:**
- `CampaignCreated` — {name, campaign_type, goal_amount, start_date, end_date}
- `CampaignUpdated` — {changed_fields}
- `CampaignGoalReached` — {campaign_id, goal_amount, actual_amount, reached_date}
- `CampaignCompleted` — {campaign_id, total_raised, total_donors, total_gifts}

**Consent Aggregate:**
- `ConsentGranted` — {contact_id, channel, lawful_basis, source, ip_address}
- `ConsentRevoked` — {contact_id, channel, reason, source}
- `DataDeletionRequested` — {contact_id, request_source, gdpr_article}
- `DataDeletionCompleted` — {contact_id, crypto_shredding_key_deleted}

---

## Read Model Projections

Projections are materialised views rebuilt from the event stream. They are disposable and rebuildable — if a projection's schema needs to change, drop it and replay events.

### Contact Projection (Current State)

```sql
CREATE TABLE contact_projection (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    household_id    UUID,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email_primary   TEXT,
    phone_primary   TEXT,
    donor_type      TEXT,
    communication_preference TEXT,
    is_deceased     BOOLEAN DEFAULT false,
    is_active       BOOLEAN DEFAULT true,

    -- Derived aggregates (updated by event handlers)
    lifetime_giving     NUMERIC(15,2) DEFAULT 0,
    ytd_giving          NUMERIC(15,2) DEFAULT 0,
    last_gift_amount    NUMERIC(15,2),
    last_gift_date      DATE,
    first_gift_date     DATE,
    largest_gift        NUMERIC(15,2),
    total_gift_count    INTEGER DEFAULT 0,
    total_pledge_balance NUMERIC(15,2) DEFAULT 0,
    engagement_score    INTEGER DEFAULT 0,

    -- AI scores (updated by score events)
    retention_risk_score    NUMERIC(5,4),
    major_gift_propensity   NUMERIC(5,4),
    planned_giving_propensity NUMERIC(5,4),
    recommended_ask_amount  NUMERIC(15,2),

    last_event_at   TIMESTAMPTZ,
    projection_version BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cp_tenant ON contact_projection(tenant_id);
CREATE INDEX idx_cp_household ON contact_projection(household_id);
CREATE INDEX idx_cp_name ON contact_projection(tenant_id, last_name, first_name);
CREATE INDEX idx_cp_email ON contact_projection(tenant_id, email_primary);
CREATE INDEX idx_cp_engagement ON contact_projection(tenant_id, engagement_score DESC);
CREATE INDEX idx_cp_retention_risk ON contact_projection(tenant_id, retention_risk_score DESC);
CREATE INDEX idx_cp_last_gift ON contact_projection(tenant_id, last_gift_date);
```

### Gift Projection (Current State)

```sql
CREATE TABLE gift_projection (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    contact_id      UUID NOT NULL,
    household_id    UUID,
    campaign_id     UUID,
    appeal_id       UUID,
    fund_id         UUID,
    pledge_id       UUID,
    gift_type       TEXT NOT NULL,
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) DEFAULT 'USD',
    gift_date       DATE NOT NULL,
    is_anonymous    BOOLEAN DEFAULT false,
    is_recurring    BOOLEAN DEFAULT false,
    tribute_type    TEXT,
    tribute_name    TEXT,
    acknowledgment_status TEXT,
    acknowledgment_date DATE,
    tax_deductible_amount NUMERIC(15,2),
    counting_method TEXT,
    is_reversed     BOOLEAN DEFAULT false,
    projection_version BIGINT NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gp_tenant ON gift_projection(tenant_id);
CREATE INDEX idx_gp_contact ON gift_projection(contact_id);
CREATE INDEX idx_gp_date ON gift_projection(tenant_id, gift_date);
CREATE INDEX idx_gp_campaign ON gift_projection(campaign_id);
CREATE INDEX idx_gp_fund ON gift_projection(fund_id);
CREATE INDEX idx_gp_ack ON gift_projection(tenant_id, acknowledgment_status) WHERE acknowledgment_status = 'pending';
```

### Retention Analytics Projection

```sql
-- Purpose-built read model for donor retention analysis
CREATE TABLE retention_projection (
    tenant_id       UUID NOT NULL,
    fiscal_year     INTEGER NOT NULL,
    contact_id      UUID NOT NULL,
    household_id    UUID,
    donor_segment   TEXT NOT NULL,
                    -- new, retained, upgraded, downgraded, reactivated, lapsed
    prior_year_giving NUMERIC(15,2) DEFAULT 0,
    current_year_giving NUMERIC(15,2) DEFAULT 0,
    prior_year_gift_count INTEGER DEFAULT 0,
    current_year_gift_count INTEGER DEFAULT 0,
    first_gift_date DATE,
    last_gift_date  DATE,
    giving_streak_years INTEGER DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, fiscal_year, contact_id)
);

CREATE INDEX idx_retention_segment ON retention_projection(tenant_id, fiscal_year, donor_segment);
CREATE INDEX idx_retention_lapsed ON retention_projection(tenant_id, fiscal_year, donor_segment)
    WHERE donor_segment = 'lapsed';
```

### Campaign Performance Projection

```sql
CREATE TABLE campaign_performance_projection (
    campaign_id     UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    campaign_name   TEXT NOT NULL,
    campaign_type   TEXT NOT NULL,
    goal_amount     NUMERIC(15,2),
    total_raised    NUMERIC(15,2) DEFAULT 0,
    total_donors    INTEGER DEFAULT 0,
    total_gifts     INTEGER DEFAULT 0,
    average_gift    NUMERIC(15,2) DEFAULT 0,
    median_gift     NUMERIC(15,2) DEFAULT 0,
    new_donors      INTEGER DEFAULT 0,
    retained_donors INTEGER DEFAULT 0,
    percent_to_goal NUMERIC(5,2) DEFAULT 0,
    cost            NUMERIC(15,2),
    cost_per_dollar NUMERIC(8,4),           -- cost to raise $1
    start_date      DATE,
    end_date        DATE,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (campaign_id)
);

CREATE INDEX idx_camp_perf_tenant ON campaign_performance_projection(tenant_id);
```

### AI Feature Store Projection

```sql
-- Materialised feature vectors for ML model training and inference
CREATE TABLE ai_feature_store (
    contact_id      UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    feature_date    DATE NOT NULL,           -- date this snapshot represents

    -- RFM features
    recency_days    INTEGER,                 -- days since last gift
    frequency_12m   INTEGER,                 -- gifts in last 12 months
    frequency_lifetime INTEGER,
    monetary_12m    NUMERIC(15,2),
    monetary_lifetime NUMERIC(15,2),
    monetary_avg    NUMERIC(15,2),

    -- Engagement features
    interactions_30d INTEGER DEFAULT 0,
    interactions_90d INTEGER DEFAULT 0,
    interactions_365d INTEGER DEFAULT 0,
    email_opens_90d INTEGER DEFAULT 0,
    email_clicks_90d INTEGER DEFAULT 0,
    events_attended_12m INTEGER DEFAULT 0,
    volunteer_hours_12m NUMERIC(10,2) DEFAULT 0,

    -- Giving pattern features
    giving_streak_months INTEGER DEFAULT 0,
    months_since_first_gift INTEGER,
    gift_amount_trend TEXT,                  -- increasing, stable, decreasing
    seasonal_giving_month INTEGER,           -- most common giving month
    is_recurring_donor BOOLEAN DEFAULT false,
    recurring_amount NUMERIC(15,2),
    has_pledge BOOLEAN DEFAULT false,
    pledge_fulfillment_rate NUMERIC(5,4),

    -- Planned giving features
    donor_age       INTEGER,
    donor_tenure_years INTEGER,
    wealth_capacity_score INTEGER,
    has_planned_gift_intent BOOLEAN DEFAULT false,

    -- Derived from event patterns
    communication_response_rate NUMERIC(5,4),
    days_between_gifts_avg NUMERIC(10,2),
    gift_amount_coefficient_of_variation NUMERIC(8,4),

    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (contact_id, feature_date)
);

CREATE INDEX idx_ai_feature_tenant ON ai_feature_store(tenant_id, feature_date);
CREATE INDEX idx_ai_feature_date ON ai_feature_store(feature_date);
```

### LYBUNT/SYBUNT Projection

```sql
-- Last Year But Unfortunately Not This Year / Some Years But Unfortunately Not This Year
CREATE TABLE lybunt_sybunt_projection (
    tenant_id       UUID NOT NULL,
    contact_id      UUID NOT NULL,
    fiscal_year     INTEGER NOT NULL,
    classification  TEXT NOT NULL,           -- lybunt, sybunt
    prior_year_total NUMERIC(15,2),
    prior_year_count INTEGER,
    last_gift_date  DATE,
    last_gift_amount NUMERIC(15,2),
    years_of_giving INTEGER,                -- for SYBUNT depth
    suggested_ask   NUMERIC(15,2),          -- from AI or prior year amount
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, fiscal_year, contact_id)
);

CREATE INDEX idx_lybunt_class ON lybunt_sybunt_projection(tenant_id, fiscal_year, classification);
```

---

## Event Handlers (Projection Updaters)

Event handlers subscribe to the event stream and update projections. Here is example logic in pseudocode:

```sql
-- Example: Handler for GiftReceived events
-- This updates multiple projections from a single event

-- 1. Insert into gift_projection
-- 2. Update contact_projection:
--    SET lifetime_giving = lifetime_giving + event.amount,
--        ytd_giving = ytd_giving + event.amount,
--        last_gift_date = event.gift_date,
--        last_gift_amount = event.amount,
--        total_gift_count = total_gift_count + 1,
--        largest_gift = GREATEST(largest_gift, event.amount)
-- 3. Update retention_projection for current fiscal year
-- 4. Update campaign_performance_projection
-- 5. Update ai_feature_store (increment frequency, update monetary)
-- 6. Check Schedule B threshold → emit ScheduleBThresholdReached if exceeded
```

---

## Temporal Query Examples

### "What was this donor's lifetime giving as of December 31, 2025?"

```sql
SELECT
    aggregate_id AS contact_id,
    SUM((payload->>'amount')::numeric) AS lifetime_giving_at_date
FROM domain_event
WHERE aggregate_type = 'Gift'
    AND event_type = 'GiftReceived'
    AND (payload->>'contact_id')::uuid = '...'
    AND occurred_at <= '2025-12-31 23:59:59-05'
    AND (payload->>'is_reversed')::boolean IS DISTINCT FROM true
GROUP BY aggregate_id;
```

### "Show me every change to this contact's record"

```sql
SELECT
    event_type,
    payload,
    metadata->>'user_id' AS changed_by,
    occurred_at
FROM domain_event
WHERE aggregate_type = 'Contact'
    AND aggregate_id = '...'
ORDER BY sequence_number ASC;
```

### "Reconstruct this pledge's state as of June 30, 2025"

```sql
-- Replay all pledge events up to the date
SELECT event_type, payload, occurred_at
FROM domain_event
WHERE aggregate_type = 'Pledge'
    AND aggregate_id = '...'
    AND occurred_at <= '2025-06-30 23:59:59-04'
ORDER BY sequence_number ASC;
-- Application code replays these events to rebuild the Pledge aggregate
```

### "Which donors had their engagement score drop in the last 30 days?"

```sql
SELECT DISTINCT
    (payload->>'contact_id')::uuid AS contact_id,
    (payload->>'old_score')::int AS old_score,
    (payload->>'new_score')::int AS new_score,
    occurred_at
FROM domain_event
WHERE event_type = 'ContactEngagementScoreUpdated'
    AND (payload->>'new_score')::int < (payload->>'old_score')::int
    AND occurred_at >= now() - interval '30 days'
ORDER BY occurred_at DESC;
```

---

## GDPR Crypto-Shredding Pattern

```sql
-- Each contact has a unique encryption key stored separately
CREATE TABLE contact_encryption_key (
    contact_id      UUID PRIMARY KEY,
    encryption_key  BYTEA NOT NULL,         -- AES-256 key
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Personal data in event payloads is encrypted with the contact's key.
-- To exercise right-to-erasure: DELETE FROM contact_encryption_key WHERE contact_id = '...';
-- Events remain intact (preserving aggregate integrity) but personal data becomes unreadable.
-- Non-personal data (amounts, dates, fund IDs) is stored unencrypted.
```

---

## Snapshotting (Performance Optimisation)

```sql
-- Periodic snapshots avoid replaying full event history for long-lived aggregates
CREATE TABLE aggregate_snapshot (
    aggregate_type  TEXT NOT NULL,
    aggregate_id    UUID NOT NULL,
    snapshot_version BIGINT NOT NULL,        -- sequence_number at time of snapshot
    state           JSONB NOT NULL,          -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, snapshot_version)
);

-- To load an aggregate:
-- 1. Find latest snapshot for (type, id)
-- 2. Replay events with sequence_number > snapshot_version
-- 3. Apply events to snapshot state
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | domain_event (partitioned by month) |
| Contact Projections | 1 | contact_projection (denormalised current state) |
| Gift Projections | 1 | gift_projection (current state of gifts) |
| Analytics Projections | 4 | retention, campaign_performance, lybunt_sybunt, ai_feature_store |
| Infrastructure | 3 | aggregate_snapshot, contact_encryption_key, projection_checkpoint |
| **Total** | **~10 core tables** | Projections are disposable and rebuildable; new ones can be added without schema migration |

---

## Key Design Decisions

1. **Single event table, not per-aggregate tables** — A single `domain_event` table partitioned by month keeps the write path simple and enables cross-aggregate event stream processing. The `aggregate_type` + `aggregate_id` composite index provides efficient per-entity replay. This is the standard pattern used in EventStoreDB, Marten, and Axon Framework.

2. **JSONB payloads with versioned schemas** — Event payloads are stored as JSONB with an `event_version` field. When event schemas evolve (e.g., adding a field to `GiftReceived`), upcasters transform old versions to new during replay. This avoids the relational migration problem entirely for the event store.

3. **Projections are disposable** — Read model tables can be dropped and rebuilt from the event stream at any time. This means schema changes to read models are zero-risk migrations: create the new projection, replay events, swap, drop the old one. This is transformative for a rapidly-evolving product.

4. **AI feature store as a first-class projection** — The `ai_feature_store` table materialises ML-ready feature vectors directly from the event stream. This eliminates the typical ETL pipeline between operational database and ML platform. Features are always fresh because they are updated in real time by event handlers.

5. **GDPR via crypto-shredding, not event deletion** — Deleting events from an append-only store breaks aggregate integrity. Instead, personal data in event payloads is encrypted with per-contact keys. Deleting the key makes personal data unrecoverable while preserving the event structure for aggregate replay and analytics (with anonymised data).

6. **Retention and LYBUNT/SYBUNT as dedicated projections** — These are the most critical analytical reports in nonprofit fundraising. By materialising them as purpose-built projections updated in real time, the system delivers instant dashboard performance without running expensive aggregation queries against the gift table.

7. **Metadata for causation tracking** — Every event carries metadata including `correlation_id` (linking events across aggregates in a single user action) and `causation_id` (the event that caused this event). This enables tracing chains of causation: "this stewardship email was sent because this engagement score dropped because this donor's recency exceeded 180 days."

8. **Snapshot optimisation for long-lived aggregates** — Contacts and planned gifts can accumulate thousands of events over years or decades. Periodic snapshots (every 100 events) avoid replaying full history on every read while preserving the ability to do temporal queries by replaying from any point.

9. **Event-driven Schedule B threshold detection** — Rather than running a nightly batch to check IRS Schedule B thresholds, the system emits a `ScheduleBThresholdReached` event when a `GiftReceived` event pushes a donor's cumulative annual giving past $5,000. This enables real-time compliance alerting.

10. **Campaign performance as a real-time projection** — Campaign dashboards update within milliseconds of a gift being recorded, not after a nightly ETL. The `campaign_performance_projection` is updated by the `GiftReceived` event handler, giving fundraising staff live visibility into campaign progress.
