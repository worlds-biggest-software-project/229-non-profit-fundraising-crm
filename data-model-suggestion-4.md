# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Non-Profit Fundraising CRM · Created: 2026-05-22

## Philosophy

This model combines a relational PostgreSQL backbone for operational CRUD (gift processing, pledge management, campaign tracking) with a graph layer for relationship-intensive queries. The graph layer models the complex web of connections between donors, households, organizations, board memberships, referral chains, and influence networks that are central to major gift fundraising and prospect identification.

In nonprofit fundraising, relationships drive revenue. A major gift officer needs to know: "Who on our board knows this prospect? Which donors are connected to this foundation? Who referred the donors that gave the most last year? Which advisors (attorneys, financial planners) influence multiple planned giving prospects?" These are fundamentally graph traversal problems. In a traditional relational model, answering them requires multiple self-joins, recursive CTEs, and brittle hand-tuned queries. In a graph model, they are simple traversals.

The hybrid approach uses PostgreSQL for the 90% of operations that are standard CRUD (recording gifts, managing pledges, sending acknowledgments) and adds a graph layer — implemented either as a property graph in PostgreSQL using `graph_node` and `graph_edge` tables with Apache AGE extension, or as a separate Neo4j instance for organizations that need advanced graph analytics. The relational tables are the system of record; the graph is a synchronised secondary index optimised for relationship queries, prospect identification, and AI-powered network analysis.

**Best for:** Organizations with complex donor networks, major gift programmes that rely on relationship mapping, planned giving prospecting based on advisor connections, peer-to-peer fundraising network analysis, and AI-powered influence scoring.

**Trade-offs:**
- (+) Natural modeling of donor-to-donor, donor-to-organization, and influence relationships
- (+) Multi-hop relationship queries are orders of magnitude faster than relational JOINs
- (+) AI-powered network analysis (influence scoring, community detection, referral path optimization)
- (+) Prospect identification through relationship proximity to existing major donors
- (+) Relationship visualization enables gift officers to see connection maps
- (-) Dual-storage adds operational complexity (sync between relational and graph)
- (-) Graph query languages (Cypher, openCypher, GQL) are less widely known than SQL
- (-) Additional infrastructure if using a separate graph database (Neo4j)
- (-) Risk of data drift between relational and graph layers if sync fails
- (-) Overkill for small nonprofits with simple donor bases

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| FASB ASC 958 | Fund restriction tracking in relational layer; gift flow analysis across connected entities via graph |
| IRS Form 990 / Schedule B | Cumulative giving in relational layer; Form 990 public data ingested as graph nodes for prospect research |
| CASE Reporting Standards | Gift counting in relational layer; relationship-influenced giving analysis via graph traversal |
| PCI DSS 4.0 | Payment data in relational layer only; never stored in graph |
| GDPR / CCPA | Consent in relational layer; graph nodes carry no personal data beyond IDs and relationship metadata |
| ISO 3166 / ISO 4217 | Country codes and currency codes in relational layer |
| AFP Donor Bill of Rights | Relationship data used for cultivation, not surveillance; consent-based contact only |
| GQL (ISO/IEC 39075) | Graph queries written in emerging GQL standard or openCypher for portability |

---

## Relational Layer (System of Record)

The relational tables follow the same structure as Data Model Suggestion 1 (normalized relational) for operational CRUD. Below are the key tables with additions specific to the graph-relational hybrid.

### Contacts Table (Relational)

```sql
CREATE TABLE contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    household_id    UUID REFERENCES household(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email_primary   TEXT,
    phone_primary   TEXT,
    donor_type      TEXT NOT NULL DEFAULT 'individual',
    communication_preference TEXT DEFAULT 'email',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_deceased     BOOLEAN NOT NULL DEFAULT false,

    -- Aggregate giving fields (maintained by triggers)
    lifetime_giving     NUMERIC(15,2) DEFAULT 0,
    last_gift_date      DATE,
    first_gift_date     DATE,
    total_gift_count    INTEGER DEFAULT 0,
    engagement_score    INTEGER DEFAULT 0,

    -- Graph-derived fields (updated by graph sync)
    network_influence_score NUMERIC(5,4) DEFAULT 0,
        -- PageRank-like score based on connections to high-value donors
    relationship_count  INTEGER DEFAULT 0,
    connection_to_major_donors INTEGER DEFAULT 0,
        -- count of relationships within 2 hops of major donors ($10k+)
    referral_value      NUMERIC(15,2) DEFAULT 0,
        -- total giving by contacts this person referred

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_contact_tenant ON contact(tenant_id);
CREATE INDEX idx_contact_household ON contact(household_id);
CREATE INDEX idx_contact_name ON contact(tenant_id, last_name, first_name);
CREATE INDEX idx_contact_email ON contact(tenant_id, email_primary);
CREATE INDEX idx_contact_influence ON contact(tenant_id, network_influence_score DESC);
CREATE INDEX idx_contact_referral_value ON contact(tenant_id, referral_value DESC);
```

### Organizations Table (Relational)

```sql
CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    org_type        TEXT NOT NULL DEFAULT 'nonprofit',
    website         TEXT,
    industry        TEXT,
    annual_revenue  NUMERIC(15,2),

    -- Graph-derived fields
    connected_donor_count INTEGER DEFAULT 0,
        -- number of donors affiliated with this org
    connected_donor_giving NUMERIC(15,2) DEFAULT 0,
        -- total giving by affiliated donors
    board_overlap_count INTEGER DEFAULT 0,
        -- number of shared board members with other connected orgs

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_tenant ON organization(tenant_id);
CREATE INDEX idx_org_name ON organization(tenant_id, name);
CREATE INDEX idx_org_connected_giving ON organization(tenant_id, connected_donor_giving DESC);
```

### Gifts Table (Relational)

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
    referred_by_contact_id UUID REFERENCES contact(id),
        -- tracks which contact influenced this gift (graph-informed)
    gift_type       TEXT NOT NULL DEFAULT 'cash',
    amount          NUMERIC(15,2) NOT NULL,
    currency_code   CHAR(3) NOT NULL DEFAULT 'USD',
    gift_date       DATE NOT NULL,
    received_date   DATE,
    is_anonymous    BOOLEAN NOT NULL DEFAULT false,
    is_recurring    BOOLEAN NOT NULL DEFAULT false,
    acknowledgment_status TEXT DEFAULT 'pending',
    counting_method TEXT DEFAULT 'outright',
    check_number    TEXT,
    reference_number TEXT,
    tax_deductible_amount NUMERIC(15,2),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gift_tenant ON gift(tenant_id);
CREATE INDEX idx_gift_contact ON gift(contact_id);
CREATE INDEX idx_gift_date ON gift(tenant_id, gift_date);
CREATE INDEX idx_gift_campaign ON gift(campaign_id);
CREATE INDEX idx_gift_fund ON gift(fund_id);
CREATE INDEX idx_gift_referred_by ON gift(referred_by_contact_id);
```

### Other Relational Tables

The following tables follow standard normalized relational patterns (see Data Model Suggestion 1 for full DDL):

- `household` — household groupings
- `address` — multi-type addresses with NCOA tracking
- `fund` — fund designations with FASB ASC 958 restriction types
- `campaign` — hierarchical campaign structure
- `appeal` — campaign appeals with source codes
- `pledge` — pledge commitments and schedules
- `recurring_donation` — sustainer programmes
- `soft_credit` — multi-contact gift attribution
- `planned_gift` — planned giving vehicles
- `grant_record` — funder relationships and awards
- `event` — fundraising events
- `event_registration` — attendee registrations
- `membership` — member records
- `interaction` — contact activity log
- `task` — staff task management
- `consent` — GDPR/CCPA consent records
- `audit_log` — change tracking
- `tenant` — multi-tenant configuration
- `app_user` — staff users

---

## Graph Layer

### Graph Node Table (PostgreSQL Implementation)

```sql
-- Generic node table for the property graph
-- Each node corresponds to an entity in the relational layer
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    node_type       TEXT NOT NULL,
                    -- person, household, organization, foundation,
                    -- advisor, board, campaign, event, fund
    entity_id       UUID NOT NULL,           -- FK to the relational table (contact.id, org.id, etc.)
    label           TEXT NOT NULL,            -- display name for visualization
    properties      JSONB DEFAULT '{}',
                    -- cached properties for graph queries without joining relational tables
                    -- Example for person node:
                    -- {
                    --   "donor_type": "major_donor",
                    --   "lifetime_giving": 250000,
                    --   "engagement_score": 85,
                    --   "city": "Washington",
                    --   "state": "DC"
                    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, node_type, entity_id)
);

CREATE INDEX idx_gn_tenant ON graph_node(tenant_id);
CREATE INDEX idx_gn_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_gn_entity ON graph_node(entity_id);
CREATE INDEX idx_gn_properties ON graph_node USING GIN (properties);
CREATE INDEX idx_gn_label ON graph_node(tenant_id, label);
```

### Graph Edge Table (PostgreSQL Implementation)

```sql
-- Directed edges between nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    source_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL,
                    -- SPOUSE_OF, PARENT_OF, CHILD_OF, SIBLING_OF,
                    -- EMPLOYED_BY, BOARD_MEMBER_OF, ADVISES,
                    -- REFERRED, INFLUENCED_GIFT, SOLICITED,
                    -- MEMBER_OF_HOUSEHOLD, AFFILIATED_WITH,
                    -- GAVE_TO (campaign/fund), ATTENDED (event),
                    -- VOLUNTEERS_FOR, KNOWS, FRIEND_OF
    weight          NUMERIC(10,4) DEFAULT 1.0,
                    -- relationship strength (e.g., giving amount, interaction count)
    properties      JSONB DEFAULT '{}',
                    -- Edge-specific metadata
                    -- Example for REFERRED edge:
                    -- {
                    --   "referral_date": "2025-06-15",
                    --   "referral_type": "personal_introduction",
                    --   "resulting_gift_amount": 25000,
                    --   "resulting_gift_date": "2025-09-01"
                    -- }
                    -- Example for BOARD_MEMBER_OF edge:
                    -- {
                    --   "role": "Chair",
                    --   "start_date": "2020-01-01",
                    --   "committee": "Finance",
                    --   "term_end": "2026-12-31"
                    -- }
                    -- Example for ADVISES edge:
                    -- {
                    --   "advisor_type": "estate_attorney",
                    --   "firm": "Smith & Associates",
                    --   "relationship_since": "2018-03-01"
                    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_tenant ON graph_edge(tenant_id);
CREATE INDEX idx_ge_source ON graph_edge(source_node_id);
CREATE INDEX idx_ge_target ON graph_edge(target_node_id);
CREATE INDEX idx_ge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_ge_weight ON graph_edge(tenant_id, edge_type, weight DESC);
CREATE INDEX idx_ge_properties ON graph_edge USING GIN (properties);

-- Composite index for bidirectional traversal
CREATE INDEX idx_ge_source_type ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_ge_target_type ON graph_edge(target_node_id, edge_type);
```

### Graph Analytics Results Table

```sql
-- Pre-computed graph analytics (updated periodically by background jobs)
CREATE TABLE graph_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    node_id         UUID NOT NULL REFERENCES graph_node(id),
    metric_type     TEXT NOT NULL,
                    -- pagerank, betweenness_centrality, community_id,
                    -- closeness_centrality, influence_score,
                    -- degree_centrality, clustering_coefficient
    metric_value    NUMERIC(15,8) NOT NULL,
    algorithm_version TEXT NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, node_id, metric_type, algorithm_version)
);

CREATE INDEX idx_ga_tenant ON graph_analytics(tenant_id);
CREATE INDEX idx_ga_node ON graph_analytics(node_id);
CREATE INDEX idx_ga_metric ON graph_analytics(tenant_id, metric_type, metric_value DESC);
```

### Referral Chain Table

```sql
-- Materialised referral chains for attribution and influence tracking
CREATE TABLE referral_chain (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    referrer_contact_id UUID NOT NULL REFERENCES contact(id),
    referred_contact_id UUID NOT NULL REFERENCES contact(id),
    depth           INTEGER NOT NULL DEFAULT 1,
                    -- 1 = direct referral, 2 = referred by someone who was referred, etc.
    referral_date   DATE,
    referred_lifetime_giving NUMERIC(15,2) DEFAULT 0,
    referred_gift_count INTEGER DEFAULT 0,
    chain_path      UUID[] NOT NULL,         -- ordered array of contact IDs in the chain
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ref_referrer ON referral_chain(referrer_contact_id);
CREATE INDEX idx_ref_referred ON referral_chain(referred_contact_id);
CREATE INDEX idx_ref_depth ON referral_chain(tenant_id, depth);
CREATE INDEX idx_ref_giving ON referral_chain(tenant_id, referred_lifetime_giving DESC);
```

---

## Graph Query Examples

### Find all connections within 2 hops of a major donor

```sql
-- Using recursive CTE for graph traversal in PostgreSQL
WITH RECURSIVE connections AS (
    -- Start from the target donor
    SELECT
        gn.id AS node_id,
        gn.entity_id,
        gn.node_type,
        gn.label,
        0 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_node gn
    WHERE gn.entity_id = $1  -- target donor contact_id
        AND gn.tenant_id = $2

    UNION ALL

    -- Traverse edges (both directions for undirected relationships)
    SELECT
        next_node.id,
        next_node.entity_id,
        next_node.node_type,
        next_node.label,
        c.depth + 1,
        c.path || next_node.id
    FROM connections c
    JOIN graph_edge ge ON (
        (ge.source_node_id = c.node_id OR ge.target_node_id = c.node_id)
        AND ge.is_active = true
    )
    JOIN graph_node next_node ON (
        next_node.id = CASE
            WHEN ge.source_node_id = c.node_id THEN ge.target_node_id
            ELSE ge.source_node_id
        END
    )
    WHERE c.depth < 2
        AND next_node.id != ALL(c.path)  -- prevent cycles
        AND next_node.is_active = true
)
SELECT DISTINCT ON (entity_id)
    node_type, entity_id, label, depth
FROM connections
WHERE depth > 0
ORDER BY entity_id, depth ASC;
```

### Find board members shared between two organizations

```sql
SELECT
    c.id, c.first_name, c.last_name,
    o1.name AS org_1,
    o2.name AS org_2,
    e1.properties->>'role' AS role_at_org1,
    e2.properties->>'role' AS role_at_org2
FROM graph_node gn_person
JOIN graph_edge e1 ON e1.source_node_id = gn_person.id
    AND e1.edge_type = 'BOARD_MEMBER_OF'
JOIN graph_node gn_org1 ON gn_org1.id = e1.target_node_id
    AND gn_org1.entity_id = $1  -- org 1 id
JOIN graph_edge e2 ON e2.source_node_id = gn_person.id
    AND e2.edge_type = 'BOARD_MEMBER_OF'
JOIN graph_node gn_org2 ON gn_org2.id = e2.target_node_id
    AND gn_org2.entity_id = $2  -- org 2 id
JOIN contact c ON c.id = gn_person.entity_id
JOIN organization o1 ON o1.id = gn_org1.entity_id
JOIN organization o2 ON o2.id = gn_org2.entity_id
WHERE gn_person.node_type = 'person';
```

### Find the shortest referral path between two donors

```sql
WITH RECURSIVE referral_path AS (
    SELECT
        gn.id AS node_id,
        gn.entity_id,
        gn.label,
        ARRAY[gn.id] AS path,
        ARRAY[gn.label] AS name_path,
        0 AS depth
    FROM graph_node gn
    WHERE gn.entity_id = $1  -- source contact
        AND gn.tenant_id = $3

    UNION ALL

    SELECT
        next_node.id,
        next_node.entity_id,
        next_node.label,
        rp.path || next_node.id,
        rp.name_path || next_node.label,
        rp.depth + 1
    FROM referral_path rp
    JOIN graph_edge ge ON ge.source_node_id = rp.node_id
        AND ge.edge_type IN ('REFERRED', 'KNOWS', 'FRIEND_OF')
        AND ge.is_active = true
    JOIN graph_node next_node ON next_node.id = ge.target_node_id
    WHERE rp.depth < 6  -- max path length
        AND next_node.id != ALL(rp.path)
)
SELECT path, name_path, depth
FROM referral_path
WHERE entity_id = $2  -- target contact
ORDER BY depth ASC
LIMIT 5;
```

### Identify planned giving prospects through advisor networks

```sql
-- Find donors who share an estate attorney with existing planned giving donors
SELECT
    prospect.entity_id AS prospect_id,
    prospect.label AS prospect_name,
    prospect.properties->>'lifetime_giving' AS lifetime_giving,
    advisor.label AS shared_advisor,
    advisor_edge.properties->>'advisor_type' AS advisor_type,
    existing_pg.label AS existing_pg_donor,
    pg.gift_vehicle,
    pg.estimated_amount
FROM graph_node advisor
JOIN graph_edge advisor_edge ON advisor_edge.target_node_id = advisor.id
    AND advisor_edge.edge_type = 'ADVISES'
    AND (advisor_edge.properties->>'advisor_type') IN ('estate_attorney', 'financial_planner')
JOIN graph_node existing_pg_node ON existing_pg_node.id = advisor_edge.source_node_id
    AND existing_pg_node.node_type = 'person'
JOIN planned_gift pg ON pg.contact_id = existing_pg_node.entity_id
    AND pg.status IN ('intent_documented', 'legally_committed', 'irrevocable', 'matured')
-- Find other clients of the same advisor
JOIN graph_edge prospect_edge ON prospect_edge.target_node_id = advisor.id
    AND prospect_edge.edge_type = 'ADVISES'
    AND prospect_edge.source_node_id != existing_pg_node.id
JOIN graph_node prospect ON prospect.id = prospect_edge.source_node_id
    AND prospect.node_type = 'person'
-- Exclude donors who already have planned gifts
LEFT JOIN planned_gift existing ON existing.contact_id = prospect.entity_id
WHERE existing.id IS NULL
    AND advisor.tenant_id = $1
ORDER BY (prospect.properties->>'lifetime_giving')::numeric DESC;
```

### Calculate network influence score (simplified PageRank)

```sql
-- Iterative PageRank-like computation
-- In practice, run this as a background job and store results in graph_analytics

WITH node_degrees AS (
    SELECT
        gn.id AS node_id,
        COUNT(ge.id) AS out_degree
    FROM graph_node gn
    LEFT JOIN graph_edge ge ON ge.source_node_id = gn.id AND ge.is_active = true
    WHERE gn.tenant_id = $1 AND gn.node_type = 'person'
    GROUP BY gn.id
),
-- Simplified single-iteration influence propagation
influence AS (
    SELECT
        gn.id AS node_id,
        gn.entity_id,
        gn.label,
        -- Base score from own giving
        COALESCE((gn.properties->>'lifetime_giving')::numeric, 0) AS own_giving,
        -- Incoming influence from connected donors
        COALESCE(SUM(
            (source_node.properties->>'lifetime_giving')::numeric
            * ge.weight
            / GREATEST(nd.out_degree, 1)
        ), 0) AS network_influence
    FROM graph_node gn
    LEFT JOIN graph_edge ge ON ge.target_node_id = gn.id AND ge.is_active = true
    LEFT JOIN graph_node source_node ON source_node.id = ge.source_node_id
    LEFT JOIN node_degrees nd ON nd.node_id = ge.source_node_id
    WHERE gn.tenant_id = $1 AND gn.node_type = 'person'
    GROUP BY gn.id, gn.entity_id, gn.label, gn.properties
)
SELECT
    entity_id,
    label,
    own_giving,
    network_influence,
    (own_giving * 0.4 + network_influence * 0.6) AS composite_influence_score
FROM influence
ORDER BY (own_giving * 0.4 + network_influence * 0.6) DESC
LIMIT 100;
```

---

## Graph Synchronization

### Relational-to-Graph Sync Triggers

```sql
-- Trigger to sync contact changes to graph layer
CREATE OR REPLACE FUNCTION sync_contact_to_graph() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
        VALUES (
            NEW.tenant_id,
            'person',
            NEW.id,
            NEW.first_name || ' ' || NEW.last_name,
            jsonb_build_object(
                'donor_type', NEW.donor_type,
                'lifetime_giving', NEW.lifetime_giving,
                'engagement_score', NEW.engagement_score,
                'is_deceased', NEW.is_deceased
            )
        );
    ELSIF TG_OP = 'UPDATE' THEN
        UPDATE graph_node
        SET label = NEW.first_name || ' ' || NEW.last_name,
            properties = jsonb_build_object(
                'donor_type', NEW.donor_type,
                'lifetime_giving', NEW.lifetime_giving,
                'engagement_score', NEW.engagement_score,
                'is_deceased', NEW.is_deceased
            ),
            updated_at = now()
        WHERE entity_id = NEW.id AND node_type = 'person';
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE graph_node SET is_active = false, updated_at = now()
        WHERE entity_id = OLD.id AND node_type = 'person';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_contact_graph_sync
AFTER INSERT OR UPDATE OR DELETE ON contact
FOR EACH ROW EXECUTE FUNCTION sync_contact_to_graph();
```

```sql
-- Trigger to sync relationships to graph edges
CREATE OR REPLACE FUNCTION sync_relationship_to_graph() RETURNS TRIGGER AS $$
DECLARE
    source_node UUID;
    target_node UUID;
BEGIN
    IF TG_OP = 'INSERT' THEN
        SELECT id INTO source_node FROM graph_node
            WHERE entity_id = NEW.contact_id_a AND node_type = 'person';
        SELECT id INTO target_node FROM graph_node
            WHERE entity_id = NEW.contact_id_b AND node_type = 'person';

        IF source_node IS NOT NULL AND target_node IS NOT NULL THEN
            INSERT INTO graph_edge (tenant_id, source_node_id, target_node_id, edge_type, properties)
            VALUES (
                NEW.tenant_id,
                source_node,
                target_node,
                UPPER(NEW.relationship_type),
                jsonb_build_object(
                    'relational_id', NEW.id,
                    'start_date', NEW.start_date
                )
            );
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_relationship_graph_sync
AFTER INSERT ON relationship
FOR EACH ROW EXECUTE FUNCTION sync_relationship_to_graph();
```

---

## Neo4j Alternative (For Advanced Graph Needs)

For organizations requiring advanced graph algorithms (community detection, link prediction, graph neural networks), a separate Neo4j instance can replace the PostgreSQL graph tables. The Cypher queries are more expressive than recursive CTEs:

### Cypher: Find all connections within 2 hops

```cypher
MATCH path = (donor:Person {entity_id: $contactId})-[*1..2]-(connected)
WHERE connected <> donor
RETURN DISTINCT connected.entity_id AS contact_id,
       connected.label AS name,
       connected.lifetime_giving AS giving,
       length(path) AS distance
ORDER BY connected.lifetime_giving DESC
```

### Cypher: Community detection for donor segmentation

```cypher
CALL gds.louvain.stream('donor-network', {
    nodeLabels: ['Person'],
    relationshipTypes: ['KNOWS', 'REFERRED', 'SPOUSE_OF', 'BOARD_MEMBER_OF']
})
YIELD nodeId, communityId
WITH gds.util.asNode(nodeId) AS person, communityId
RETURN communityId,
       count(person) AS community_size,
       sum(person.lifetime_giving) AS total_giving,
       collect(person.label)[0..5] AS sample_members
ORDER BY total_giving DESC
```

### Cypher: Link prediction for prospect identification

```cypher
// Predict which non-donors are most likely to become donors
// based on their structural similarity to existing major donors
CALL gds.nodeSimilarity.stream('donor-network', {
    nodeLabels: ['Person'],
    relationshipTypes: ['KNOWS', 'BOARD_MEMBER_OF', 'AFFILIATED_WITH']
})
YIELD node1, node2, similarity
WITH gds.util.asNode(node1) AS person1, gds.util.asNode(node2) AS person2, similarity
WHERE person1.lifetime_giving > 10000 AND person2.lifetime_giving = 0
RETURN person2.entity_id AS prospect_id,
       person2.label AS prospect_name,
       person1.label AS similar_to_donor,
       person1.lifetime_giving AS donor_giving,
       similarity
ORDER BY similarity DESC
LIMIT 50
```

---

## Graph-Informed AI Features

### Network-Enhanced Donor Scoring

The graph layer enables AI features that are impossible with relational data alone:

```sql
-- Graph-enhanced feature vector for ML models
CREATE VIEW graph_enhanced_features AS
SELECT
    c.id AS contact_id,
    c.tenant_id,
    c.lifetime_giving,
    c.engagement_score,
    c.total_gift_count,
    c.network_influence_score,
    c.connection_to_major_donors,
    c.referral_value,

    -- Graph centrality metrics
    COALESCE(ga_pr.metric_value, 0) AS pagerank,
    COALESCE(ga_bc.metric_value, 0) AS betweenness_centrality,
    COALESCE(ga_cc.metric_value, 0) AS closeness_centrality,
    COALESCE(ga_comm.metric_value, 0) AS community_id,

    -- Relationship counts by type
    (SELECT COUNT(*) FROM graph_edge ge
     JOIN graph_node gn ON gn.id = ge.source_node_id
     WHERE gn.entity_id = c.id AND ge.edge_type = 'BOARD_MEMBER_OF'
    ) AS board_memberships,

    (SELECT COUNT(*) FROM graph_edge ge
     JOIN graph_node gn ON gn.id = ge.source_node_id
     WHERE gn.entity_id = c.id AND ge.edge_type = 'REFERRED'
    ) AS referrals_made,

    (SELECT COUNT(*) FROM graph_edge ge
     JOIN graph_node gn ON gn.id = ge.source_node_id OR gn.id = ge.target_node_id
     WHERE gn.entity_id = c.id AND ge.edge_type = 'ADVISES'
    ) AS advisor_connections

FROM contact c
LEFT JOIN graph_node gn ON gn.entity_id = c.id AND gn.node_type = 'person'
LEFT JOIN graph_analytics ga_pr ON ga_pr.node_id = gn.id AND ga_pr.metric_type = 'pagerank'
LEFT JOIN graph_analytics ga_bc ON ga_bc.node_id = gn.id AND ga_bc.metric_type = 'betweenness_centrality'
LEFT JOIN graph_analytics ga_cc ON ga_cc.node_id = gn.id AND ga_cc.metric_type = 'closeness_centrality'
LEFT JOIN graph_analytics ga_comm ON ga_comm.node_id = gn.id AND ga_comm.metric_type = 'community_id';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Constituent Management | 7 | contact, household, organization, address, relationship, affiliation, consent |
| Gift & Pledge Processing | 7 | gift, soft_credit, pledge, recurring_donation, fund, appeal, gift_batch |
| Campaigns | 1 | campaign |
| Planned Giving | 1 | planned_gift |
| Grant Management | 2 | grant_record, grant_deliverable |
| Events & Membership | 4 | event, event_registration, membership, membership_level |
| Interactions & Tasks | 2 | interaction, task |
| Graph Layer | 4 | graph_node, graph_edge, graph_analytics, referral_chain |
| Platform | 4 | tenant, app_user, audit_log, oauth tables |
| **Total** | **~32 relational + 4 graph = ~36 tables** | Graph tables add ~12% overhead to normalized model |

---

## Key Design Decisions

1. **Graph as secondary index, not source of truth** — The relational tables are the system of record. The graph layer is a synchronised secondary representation optimised for relationship queries. If the graph becomes corrupted, it can be rebuilt entirely from relational data. This avoids the complexity of making a graph database transactionally consistent with CRUD operations.

2. **PostgreSQL-native graph with Neo4j upgrade path** — The `graph_node` and `graph_edge` tables implement a property graph model in PostgreSQL using standard tables and recursive CTEs. Organizations needing advanced graph algorithms (community detection, link prediction, graph neural networks) can add Neo4j as a secondary store without changing the relational layer. Apache AGE PostgreSQL extension is another option for Cypher query support within PostgreSQL.

3. **Trigger-based synchronization** — PostgreSQL triggers keep the graph layer synchronized with relational changes in real time. This ensures the graph is always current for queries without requiring application-level dual-write logic. For high-throughput environments, triggers can be replaced with asynchronous event-driven sync via a message queue.

4. **Referral chain materialisation** — The `referral_chain` table pre-computes multi-hop referral paths with their associated giving impact. This avoids running expensive recursive CTEs on every dashboard load and enables "referral value" as a first-class metric for each contact.

5. **Graph-derived fields on relational entities** — Key graph metrics (`network_influence_score`, `connection_to_major_donors`, `referral_value`) are denormalised onto the `contact` table. This means standard relational queries (donor lists, segments, exports) can filter and sort by graph-derived metrics without touching the graph tables — critical for report performance.

6. **Edge weight for relationship strength** — Graph edges carry a `weight` property that quantifies relationship strength (e.g., total giving influenced, interaction frequency, years of relationship). This enables weighted graph algorithms (weighted PageRank, weighted shortest path) that produce more meaningful influence scores than unweighted traversal.

7. **Graph analytics as background computation** — PageRank, betweenness centrality, and community detection are computed as background jobs (hourly or daily) and stored in `graph_analytics`. This avoids running expensive graph algorithms during interactive queries while keeping analytics reasonably fresh.

8. **Advisor network for planned giving prospecting** — The graph explicitly models ADVISES edges between estate attorneys, financial planners, and donors. This enables the "shared advisor" prospect identification query — finding non-planned-giving donors who share advisors with existing planned giving donors — which is a high-value use case that is nearly impossible in a relational-only model.

9. **Gift referral attribution** — The `referred_by_contact_id` column on the gift table, combined with the REFERRED edge type in the graph, enables tracking not just who gave, but who caused the giving. This supports referral value calculations and helps identify "connectors" whose network influence generates more revenue than their own giving.

10. **Community detection for donor segmentation** — Graph community detection algorithms (Louvain, Label Propagation) can identify natural donor clusters that cut across traditional segments (major gift, annual fund, planned giving). A donor who gives modestly but is deeply connected to a high-value community may be a better cultivation target than an isolated high-capacity donor.
