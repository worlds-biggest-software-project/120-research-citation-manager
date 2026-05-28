# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Research & Citation Manager · Created: 2026-05-19

## Philosophy

This model uses a dual-layer architecture: relational PostgreSQL tables for operational CRUD (library management, user accounts, attachments, systematic reviews) and a property graph layer for relationship-intensive queries (citation networks, author collaboration graphs, concept maps, literature neighbourhood exploration). The graph layer can be implemented either as PostgreSQL tables modeling nodes and edges (using `graph_nodes` / `graph_edges` tables with JSONB properties) or as a dedicated graph database (Neo4j, Amazon Neptune) synchronized from the relational layer.

Citation management is fundamentally a graph problem. The core questions researchers ask — "What papers cite this one?", "Who are the most influential authors in this subfield?", "What is the shortest citation path between these two papers?", "Show me the neighbourhood of related work around my seed paper" — are graph traversal queries that relational databases handle awkwardly (recursive CTEs, multiple self-joins) but graph databases answer naturally. Connected Papers and Research Rabbit have demonstrated that citation graph visualization is one of the most valued features in modern research tools; this model makes the graph a first-class citizen rather than a derived view.

The OpenAlex data model — which structures 209M+ works, 2B+ authors, and their relationships as a heterogeneous directed graph with typed nodes and edges — provides a proven reference architecture for this approach. This model adapts that pattern for a personal/group library context where the graph is a subset of the global academic graph, enriched with the user's annotations, tags, and AI analyses.

**Best for:** Products where citation graph exploration, author network analysis, literature neighbourhood visualization, and conflict-of-interest detection are core features rather than afterthoughts.

**Trade-offs:**
- (+) Citation chain queries (multi-hop traversals) are natural and performant
- (+) Visual graph exploration is backed by native graph structure, not derived from relational joins
- (+) Author collaboration networks, co-citation analysis, and bibliographic coupling are first-class queries
- (+) Graph algorithms (PageRank, community detection, shortest path) run natively
- (+) Extensible: new node and edge types require no schema migration
- (-) Dual-layer architecture increases operational complexity
- (-) Graph consistency must be maintained via sync if using a separate graph DB
- (-) Graph databases have less mature tooling for backups, monitoring, and migrations than PostgreSQL
- (-) Simple CRUD operations (add a paper, update metadata) don't benefit from the graph layer
- (-) Team needs graph query language skills (Cypher or SQL graph CTEs)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CSL-JSON / CSL 1.0.2 | Item metadata stored as CSL-JSON in relational layer; graph nodes reference relational items by ID |
| DOI (ISO 26324) | Graph nodes for works use DOI as canonical external identifier |
| ORCID | Graph nodes for persons use ORCID as canonical identifier; edges connect persons to works |
| OpenAlex Entity Model | Graph node types (Work, Author, Source, Institution, Concept) mirror OpenAlex entities |
| CiTO (Citation Typing Ontology) | Edge types between Work nodes use CiTO predicates: `cito:cites`, `cito:supports`, `cito:disputes`, `cito:extends` |
| ISNI (ISO 27729) | Secondary person identifier stored on Author graph nodes |
| ROR (Research Organization Registry) | Institution nodes use ROR IDs for unambiguous institution identification |
| Dublin Core (DCMI) | Graph node properties align with DC terms where applicable |
| W3C Web Annotation Data Model | Annotations stored in relational layer following W3C model |
| Schema.org / ScholarlyArticle | Graph export can serialize to Schema.org vocabulary for web embedding |

---

## Relational Layer (Operational CRUD)

```sql
-- ============================================================
-- USERS & AUTH (same as other models)
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    orcid_id        TEXT,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    auth_subject    TEXT,
    password_hash   TEXT,
    preferences     JSONB NOT NULL DEFAULT '{}',
    storage_used_bytes BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_orcid ON users (orcid_id) WHERE orcid_id IS NOT NULL;

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    org_type        TEXT NOT NULL DEFAULT 'team',
    ror_id          TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, user_id)
);

-- ============================================================
-- LIBRARIES & COLLECTIONS
-- ============================================================

CREATE TABLE libraries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_user_id   UUID REFERENCES users(id) ON DELETE CASCADE,
    owner_org_id    UUID REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL DEFAULT 'My Library',
    library_type    TEXT NOT NULL DEFAULT 'personal',
    is_public       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_library_owner CHECK (
        (owner_user_id IS NOT NULL) != (owner_org_id IS NOT NULL)
    )
);

CREATE TABLE library_members (
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'reader',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (library_id, user_id)
);

CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES collections(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collections_library ON collections (library_id);

CREATE TABLE collection_items (
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (collection_id, item_id)
);

-- ============================================================
-- ITEMS (Relational — operational records)
-- ============================================================

CREATE TABLE items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    item_type       TEXT NOT NULL,
    item_key        TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',   -- CSL-JSON
    enrichment      JSONB NOT NULL DEFAULT '{}',
    
    -- Denormalized for fast sorting/filtering
    doi             TEXT,
    title           TEXT,
    pub_year        INTEGER,
    first_author    TEXT,
    
    -- Graph node reference
    graph_node_id   UUID,                          -- links to graph_nodes for this work
    
    is_retracted    BOOLEAN NOT NULL DEFAULT FALSE,
    version         INTEGER NOT NULL DEFAULT 1,
    date_added      TIMESTAMPTZ NOT NULL DEFAULT now(),
    date_modified   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (library_id, item_key)
);

CREATE INDEX idx_items_library ON items (library_id);
CREATE INDEX idx_items_doi ON items (doi) WHERE doi IS NOT NULL;
CREATE INDEX idx_items_graph_node ON items (graph_node_id) WHERE graph_node_id IS NOT NULL;
CREATE INDEX idx_items_metadata ON items USING GIN (metadata jsonb_path_ops);

-- ============================================================
-- TAGS
-- ============================================================

CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    tag_type        TEXT NOT NULL DEFAULT 'user',
    color           TEXT,
    UNIQUE (library_id, name, tag_type)
);

CREATE TABLE item_tags (
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (item_id, tag_id)
);

-- ============================================================
-- ATTACHMENTS & ANNOTATIONS
-- ============================================================

CREATE TABLE attachments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    file_name       TEXT NOT NULL,
    content_type    TEXT NOT NULL,
    file_size_bytes BIGINT,
    storage_key     TEXT NOT NULL,
    md5_hash        TEXT,
    link_mode       TEXT NOT NULL DEFAULT 'stored',
    url             TEXT,
    fulltext        TEXT,
    fulltext_ts     TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', COALESCE(fulltext, ''))) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachments_item ON attachments (item_id);
CREATE INDEX idx_attachments_fulltext ON attachments USING GIN (fulltext_ts);

CREATE TABLE annotations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attachment_id   UUID NOT NULL REFERENCES attachments(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    annotation_type TEXT NOT NULL,
    body            JSONB NOT NULL DEFAULT '{}',
    selector        JSONB NOT NULL DEFAULT '{}',
    page_number     INTEGER,
    color           TEXT,
    sort_index      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_annotations_attachment ON annotations (attachment_id);
```

## Graph Layer (Citation Network & Knowledge Graph)

```sql
-- ============================================================
-- GRAPH NODES — Typed entities in the academic knowledge graph
-- ============================================================

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       TEXT NOT NULL,
    -- Node types (inspired by OpenAlex):
    --   'work'        — a scholarly output (paper, book, dataset, preprint)
    --   'author'      — a person who creates works
    --   'source'      — a journal, repository, or conference proceedings
    --   'institution' — a university, lab, or research organization
    --   'concept'     — a topic or research area
    --   'funder'      — a funding body

    -- Canonical identifiers
    external_ids    JSONB NOT NULL DEFAULT '{}',
    -- Example for a work node:
    -- {"doi": "10.1126/science.178.4060.471", "pmid": "17754304", "openalex": "W2023271753"}
    -- Example for an author node:
    -- {"orcid": "0000-0001-1234-5678", "openalex": "A5023888391"}
    -- Example for an institution node:
    -- {"ror": "https://ror.org/042nb2s44", "openalex": "I27837315"}

    -- Node properties (varies by type)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for a work node:
    -- {
    --   "title": "Citation analysis as a tool in journal evaluation",
    --   "pub_year": 1972,
    --   "type": "article-journal",
    --   "citation_count": 4523,
    --   "is_retracted": false,
    --   "open_access": true
    -- }
    -- Example for an author node:
    -- {
    --   "display_name": "Eugene Garfield",
    --   "works_count": 287,
    --   "cited_by_count": 45000,
    --   "h_index": 52
    -- }
    -- Example for a concept node:
    -- {
    --   "display_name": "Bibliometrics",
    --   "level": 2,
    --   "description": "Statistical analysis of scientific publications"
    -- }

    -- Full-text search
    display_name    TEXT,                      -- denormalized for search
    
    -- Computed graph metrics (updated by background jobs)
    pagerank        REAL,
    in_degree       INTEGER DEFAULT 0,         -- citation count for works
    out_degree      INTEGER DEFAULT 0,         -- reference count for works
    betweenness     REAL,                      -- betweenness centrality

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes (node_type);
CREATE INDEX idx_graph_nodes_name ON graph_nodes (display_name);
CREATE INDEX idx_graph_nodes_name_fts ON graph_nodes USING GIN (to_tsvector('english', COALESCE(display_name, '')));
CREATE INDEX idx_graph_nodes_external_ids ON graph_nodes USING GIN (external_ids jsonb_path_ops);
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN (properties jsonb_path_ops);
CREATE INDEX idx_graph_nodes_pagerank ON graph_nodes (pagerank DESC NULLS LAST) WHERE node_type = 'work';

-- ============================================================
-- GRAPH EDGES — Typed, directed relationships
-- ============================================================

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL,
    -- Edge types (CiTO ontology and OpenAlex-inspired):
    --   'cites'           — work -> work (basic citation)
    --   'supports'        — work -> work (Scite: supporting citation)
    --   'disputes'        — work -> work (Scite: contrasting citation)
    --   'extends'         — work -> work (builds upon)
    --   'authored_by'     — work -> author
    --   'published_in'    — work -> source (journal/proceedings)
    --   'affiliated_with' — author -> institution
    --   'funded_by'       — work -> funder
    --   'tagged_with'     — work -> concept
    --   'co_authored'     — author -> author (derived)
    --   'related_to'      — work -> work (similarity, not citation)

    -- Edge properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example for 'cites' edge:
    -- {
    --   "citation_context": "As demonstrated by Garfield (1972)...",
    --   "section": "introduction",
    --   "confidence": 0.95,
    --   "is_self_citation": false
    -- }
    -- Example for 'authored_by' edge:
    -- {
    --   "position": 0,
    --   "is_corresponding": true,
    --   "raw_affiliation": "University of Pennsylvania"
    -- }
    -- Example for 'tagged_with' edge:
    -- {
    --   "score": 0.92,
    --   "source": "openalex_classifier"
    -- }

    weight          REAL DEFAULT 1.0,          -- edge weight for graph algorithms
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges (source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_node_id, edge_type);
CREATE INDEX idx_graph_edges_type ON graph_edges (edge_type);
CREATE INDEX idx_graph_edges_properties ON graph_edges USING GIN (properties jsonb_path_ops);

-- Prevent duplicate edges of the same type between the same nodes
CREATE UNIQUE INDEX idx_graph_edges_unique ON graph_edges (source_node_id, target_node_id, edge_type);

-- ============================================================
-- GRAPH NODE EMBEDDINGS (for semantic similarity edges)
-- ============================================================

CREATE TABLE graph_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_id         UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    embedding_scope TEXT NOT NULL,             -- 'title_abstract', 'fulltext'
    model_id        TEXT NOT NULL,
    embedding       vector(1536),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (node_id, embedding_scope, model_id)
);

CREATE INDEX idx_graph_embeddings_vector ON graph_embeddings USING ivfflat (embedding vector_cosine_ops);
```

## AI Features & Systematic Review

```sql
-- ============================================================
-- AI OUTPUTS
-- ============================================================

CREATE TABLE ai_outputs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_type     TEXT NOT NULL,
    target_id       UUID NOT NULL,
    output_type     TEXT NOT NULL,
    model_id        TEXT NOT NULL,
    model_version   TEXT NOT NULL,
    content         JSONB NOT NULL,
    input_node_ids  UUID[],                    -- graph nodes used as input
    token_count     INTEGER,
    feedback_score  INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_outputs_target ON ai_outputs (target_type, target_id);

-- ============================================================
-- SYSTEMATIC REVIEW
-- ============================================================

CREATE TABLE review_projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    protocol        TEXT NOT NULL DEFAULT 'prisma',
    config          JSONB NOT NULL DEFAULT '{}',
    status          TEXT NOT NULL DEFAULT 'screening',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE review_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review_projects(id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    reviewer_id     UUID NOT NULL REFERENCES users(id),
    stage           TEXT NOT NULL,
    decision        TEXT NOT NULL,
    reason          TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, item_id, reviewer_id, stage)
);

CREATE TABLE data_extractions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review_projects(id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    extractor_id    UUID NOT NULL REFERENCES users(id),
    extracted_data  JSONB NOT NULL DEFAULT '{}',
    confidence      TEXT DEFAULT 'medium',
    extracted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, item_id, extractor_id)
);

-- ============================================================
-- READING HISTORY (for research memory)
-- ============================================================

CREATE TABLE reading_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    action          TEXT NOT NULL,
    duration_seconds INTEGER,
    progress        REAL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reading_history_user ON reading_history (user_id, created_at DESC);
```

## Example Graph Queries

```sql
-- ============================================================
-- Citation chain: 2-hop citation neighbourhood around a seed paper
-- ============================================================
WITH RECURSIVE citation_chain AS (
    -- Start from the seed paper's graph node
    SELECT gn.id AS node_id, gn.display_name, 0 AS depth
    FROM graph_nodes gn
    JOIN items i ON i.graph_node_id = gn.id
    WHERE i.id = '...'  -- seed item ID

    UNION ALL

    -- Follow citation edges outward (papers that cite or are cited by)
    SELECT
        CASE WHEN ge.source_node_id = cc.node_id THEN ge.target_node_id
             ELSE ge.source_node_id END AS node_id,
        gn2.display_name,
        cc.depth + 1
    FROM citation_chain cc
    JOIN graph_edges ge ON (ge.source_node_id = cc.node_id OR ge.target_node_id = cc.node_id)
        AND ge.edge_type IN ('cites', 'supports', 'disputes', 'extends')
    JOIN graph_nodes gn2 ON gn2.id = CASE
        WHEN ge.source_node_id = cc.node_id THEN ge.target_node_id
        ELSE ge.source_node_id END
    WHERE cc.depth < 2  -- limit to 2 hops
)
SELECT DISTINCT node_id, display_name, MIN(depth) AS min_depth
FROM citation_chain
GROUP BY node_id, display_name
ORDER BY min_depth, display_name;

-- ============================================================
-- Author collaboration network: find co-authors of co-authors
-- ============================================================
WITH direct_coauthors AS (
    SELECT DISTINCT ge2.target_node_id AS coauthor_id
    FROM graph_nodes author
    JOIN graph_edges ge1 ON ge1.target_node_id = author.id AND ge1.edge_type = 'authored_by'
    JOIN graph_edges ge2 ON ge2.source_node_id = ge1.source_node_id AND ge2.edge_type = 'authored_by'
    WHERE author.external_ids->>'orcid' = '0000-0001-1234-5678'
      AND ge2.target_node_id != author.id
)
SELECT gn.display_name, gn.properties->>'h_index' AS h_index,
       gn.properties->>'works_count' AS works_count,
       gn.external_ids->>'orcid' AS orcid
FROM direct_coauthors dc
JOIN graph_nodes gn ON gn.id = dc.coauthor_id
ORDER BY (gn.properties->>'cited_by_count')::int DESC;

-- ============================================================
-- Bibliographic coupling: papers that share many references
-- ============================================================
SELECT gn2.display_name AS related_paper,
       gn2.properties->>'pub_year' AS year,
       COUNT(*) AS shared_references
FROM graph_edges ge1
JOIN graph_edges ge2 ON ge2.target_node_id = ge1.target_node_id
    AND ge2.edge_type = 'cites'
    AND ge2.source_node_id != ge1.source_node_id
JOIN graph_nodes gn2 ON gn2.id = ge2.source_node_id
WHERE ge1.source_node_id = '...'  -- seed paper node ID
  AND ge1.edge_type = 'cites'
GROUP BY gn2.id, gn2.display_name, gn2.properties->>'pub_year'
HAVING COUNT(*) >= 3
ORDER BY shared_references DESC
LIMIT 20;

-- ============================================================
-- Co-citation analysis: papers frequently cited together
-- ============================================================
SELECT gn.display_name, gn.properties->>'pub_year' AS year,
       COUNT(*) AS co_citation_count
FROM graph_edges ge1
JOIN graph_edges ge2 ON ge2.source_node_id = ge1.source_node_id
    AND ge2.edge_type = 'cites'
    AND ge2.target_node_id != ge1.target_node_id
JOIN graph_nodes gn ON gn.id = ge2.target_node_id
WHERE ge1.target_node_id = '...'  -- target paper node ID
  AND ge1.edge_type = 'cites'
GROUP BY gn.id, gn.display_name, gn.properties->>'pub_year'
ORDER BY co_citation_count DESC
LIMIT 20;

-- ============================================================
-- Semantic similarity: find papers near a query in embedding space
-- ============================================================
SELECT gn.display_name, gn.properties->>'pub_year' AS year,
       gn.external_ids->>'doi' AS doi,
       ge.embedding <=> '[0.1, 0.2, ...]'::vector AS distance
FROM graph_embeddings ge
JOIN graph_nodes gn ON ge.node_id = gn.id AND gn.node_type = 'work'
WHERE ge.embedding_scope = 'title_abstract'
ORDER BY ge.embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 20;

-- ============================================================
-- Conflict-of-interest check: shared institutions between reviewers and authors
-- ============================================================
SELECT DISTINCT
    reviewer_gn.display_name AS reviewer,
    author_gn.display_name AS author,
    inst_gn.display_name AS shared_institution
FROM graph_nodes reviewer_gn
JOIN graph_edges rev_aff ON rev_aff.source_node_id = reviewer_gn.id
    AND rev_aff.edge_type = 'affiliated_with'
JOIN graph_edges auth_aff ON auth_aff.target_node_id = rev_aff.target_node_id
    AND auth_aff.edge_type = 'affiliated_with'
JOIN graph_nodes author_gn ON author_gn.id = auth_aff.source_node_id
JOIN graph_nodes inst_gn ON inst_gn.id = rev_aff.target_node_id
WHERE reviewer_gn.external_ids->>'orcid' = '...'
  AND author_gn.id IN (
    SELECT ge.target_node_id FROM graph_edges ge
    WHERE ge.source_node_id = '...'  -- manuscript node ID
      AND ge.edge_type = 'authored_by'
  );
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users & Auth | 3 | users, organizations, organization_members |
| Libraries & Collections | 4 | libraries, library_members, collections, collection_items |
| Items | 1 | Relational items with graph_node_id reference |
| Tags | 2 | tags, item_tags |
| Attachments & Annotations | 2 | attachments, annotations |
| Graph Layer | 3 | graph_nodes, graph_edges, graph_embeddings |
| AI | 1 | ai_outputs |
| Systematic Review | 3 | review_projects, review_decisions, data_extractions |
| Reading History | 1 | reading_history |
| **Total** | **~20** | Plus graph layer provides relationship richness without table proliferation |

---

## Key Design Decisions

1. **Dual-layer architecture** — Operational CRUD (add a paper, create a collection, upload a PDF) happens in standard relational tables. Graph queries (citation traversal, co-author networks, bibliographic coupling) happen against `graph_nodes` / `graph_edges`. The `items.graph_node_id` foreign key bridges the two layers.

2. **Generic node/edge tables with JSONB properties** — Rather than separate tables for works, authors, institutions, and concepts, two tables (`graph_nodes` and `graph_edges`) model all entity types and relationship types. The `node_type` and `edge_type` columns differentiate them, and `properties` JSONB stores type-specific attributes. This is the property graph model used by Neo4j and OpenAlex.

3. **CiTO ontology for citation edges** — Citation relationships use the Citation Typing Ontology (CiTO) predicates: `cites`, `supports`, `disputes`, `extends`. This is richer than a simple "cites" boolean and aligns with the Scite-inspired citation classification feature.

4. **Precomputed graph metrics** — `pagerank`, `in_degree`, `out_degree`, and `betweenness` centrality are stored on graph nodes and updated by background jobs. This avoids computing expensive graph metrics on every query and enables instant sorting by "most influential papers."

5. **OpenAlex-compatible external IDs** — Graph nodes store external identifiers (DOI, ORCID, ROR, OpenAlex ID) in a JSONB column, enabling direct import from OpenAlex's bulk data or API. A paper found via OpenAlex can be immediately linked to its graph node without identifier mapping.

6. **PostgreSQL-native graph** — The graph layer is implemented as PostgreSQL tables rather than requiring a separate graph database. While Neo4j or Neptune would offer more powerful graph query languages (Cypher, Gremlin), keeping everything in PostgreSQL simplifies operations, backups, and transactions. The recursive CTE examples above show that PostgreSQL handles 2-3 hop traversals efficiently with proper indexing.

7. **Unique edge constraint** — The unique index on `(source_node_id, target_node_id, edge_type)` prevents duplicate relationships. If Scite reclassifies a citation from "mentioning" to "supporting," the application deletes the old edge and creates a new one (or updates the edge type, depending on the chosen mutation strategy).

8. **Graph embeddings for similarity** — Semantic similarity between papers is computed via embedding vectors stored on graph nodes, enabling "find related papers" queries that go beyond citation-based relatedness. This powers the visual citation graph exploration feature: nodes are positioned by embedding similarity, not just citation links.
