# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Research & Citation Manager · Created: 2026-05-19

## Philosophy

This model follows the traditional normalized relational approach inspired by Zotero's proven SQLite schema but adapted for PostgreSQL and multi-tenant SaaS deployment. Every concept gets its own table with strict foreign key relationships, and bibliographic metadata is modeled using an Entity-Attribute-Value (EAV) pattern that mirrors how CSL-JSON and Zotero handle the inherent variability of bibliographic item types (a journal article has different fields from a book chapter, a dataset, or a conference paper).

The EAV approach for item metadata is battle-tested: Zotero has used it for 20+ years to support 30+ item types with varying field sets without schema changes. The trade-off is query complexity — retrieving a complete item requires joining across multiple tables — but this is well-understood and can be mitigated with materialized views or application-level caching.

This model is best suited for teams that value data integrity above all else, need complex cross-entity SQL queries (e.g., "find all papers by authors affiliated with institution X that cite retracted papers"), and are comfortable with a higher table count in exchange for strict normalization and referential integrity.

**Best for:** Institutional deployments requiring strict data integrity, complex cross-entity queries, and regulatory compliance with full referential constraints.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Standard SQL queries work naturally; no JSONB operators needed
- (+) Schema is self-documenting; every concept has its own table
- (+) Easy to add new item types and fields without schema migration (EAV)
- (-) High table count (~45 tables) increases schema complexity
- (-) Retrieving a complete bibliographic item requires multiple JOINs across EAV tables
- (-) EAV pattern makes some queries verbose (e.g., filtering by a specific field value)
- (-) Schema migrations needed when adding new relationship types or core entities
- (-) Performance requires careful indexing and potentially materialized views for common access patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CSL-JSON / CSL 1.0.2 | Item types and fields map directly to CSL type/field definitions; `item_types` and `fields` reference tables align with CSL schema |
| DOI (ISO 26324) | Stored as canonical external identifier in `item_identifiers` with type `doi` |
| ORCID | Stored as canonical creator identifier in `creator_identifiers` with type `orcid` |
| ISSN (ISO 3297) | Journal-level identifier stored in `item_identifiers` with type `issn` |
| ISBN (ISO 2108) | Book-level identifier stored in `item_identifiers` with type `isbn` |
| ISNI (ISO 27729) | Non-academic contributor identifier in `creator_identifiers` with type `isni` |
| W3C Web Annotation Data Model | Annotation tables follow W3C body/target/selector structure |
| Dublin Core (DCMI) | Field names align with DC terms where applicable (title, date, publisher, language) |
| OpenURL (NISO Z39.88) | `item_identifiers` supports OpenURL-compatible key-value pairs for link resolver integration |
| BibTeX / RIS | Import/export mapping tables (`bibtex_type_mappings`, `ris_type_mappings`) enable lossless round-tripping |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- USERS & AUTHENTICATION
-- ============================================================

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    orcid_id        TEXT,                    -- ORCID iD (ISO 27729 compatible)
    password_hash   TEXT,                    -- NULL if SSO-only
    auth_provider   TEXT DEFAULT 'local',    -- 'local', 'oidc', 'saml'
    auth_subject    TEXT,                    -- external IdP subject identifier
    storage_used_bytes BIGINT DEFAULT 0,
    storage_limit_bytes BIGINT DEFAULT 314572800, -- 300 MB default
    preferences     JSONB DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_orcid ON users (orcid_id) WHERE orcid_id IS NOT NULL;
CREATE INDEX idx_users_auth ON users (auth_provider, auth_subject) WHERE auth_subject IS NOT NULL;

-- ============================================================
-- ORGANIZATIONS & TENANCY
-- ============================================================

CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    org_type        TEXT NOT NULL DEFAULT 'team',  -- 'institution', 'lab', 'team'
    ror_id          TEXT,                    -- Research Organization Registry ID
    domain          TEXT,                    -- e.g., 'mit.edu' for auto-join
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization_members (
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member', -- 'owner', 'admin', 'member'
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, user_id)
);
```

## Bibliographic Items (EAV Pattern)

```sql
-- ============================================================
-- ITEM TYPE & FIELD DEFINITIONS (Reference Data)
-- ============================================================

CREATE TABLE item_types (
    id              SERIAL PRIMARY KEY,
    type_name       TEXT NOT NULL UNIQUE,       -- e.g., 'journalArticle', 'book', 'conferencePaper'
    csl_type        TEXT NOT NULL,              -- CSL type mapping, e.g., 'article-journal'
    bibtex_type     TEXT,                       -- BibTeX type, e.g., 'article'
    ris_type        TEXT,                       -- RIS type, e.g., 'JOUR'
    display_name    TEXT NOT NULL,
    description     TEXT
);

CREATE TABLE fields (
    id              SERIAL PRIMARY KEY,
    field_name      TEXT NOT NULL UNIQUE,       -- e.g., 'title', 'abstractNote', 'DOI'
    csl_field       TEXT,                       -- CSL field mapping, e.g., 'title'
    data_type       TEXT NOT NULL DEFAULT 'text', -- 'text', 'date', 'integer', 'url'
    display_name    TEXT NOT NULL
);

CREATE TABLE item_type_fields (
    item_type_id    INTEGER NOT NULL REFERENCES item_types(id),
    field_id        INTEGER NOT NULL REFERENCES fields(id),
    order_index     INTEGER NOT NULL DEFAULT 0,
    is_required     BOOLEAN NOT NULL DEFAULT FALSE,
    PRIMARY KEY (item_type_id, field_id)
);

-- ============================================================
-- ITEMS (Core bibliographic records)
-- ============================================================

CREATE TABLE items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_type_id    INTEGER NOT NULL REFERENCES item_types(id),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    parent_item_id  UUID REFERENCES items(id) ON DELETE CASCADE,  -- for notes/attachments
    item_key        TEXT NOT NULL,              -- short alphanumeric key (like Zotero)
    date_added      TIMESTAMPTZ NOT NULL DEFAULT now(),
    date_modified   TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_retracted    BOOLEAN NOT NULL DEFAULT FALSE,
    retraction_doi  TEXT,
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (library_id, item_key)
);

CREATE INDEX idx_items_library ON items (library_id);
CREATE INDEX idx_items_type ON items (item_type_id);
CREATE INDEX idx_items_parent ON items (parent_item_id) WHERE parent_item_id IS NOT NULL;
CREATE INDEX idx_items_retracted ON items (library_id) WHERE is_retracted = TRUE;

-- EAV: Item field values
CREATE TABLE item_data_values (
    id              SERIAL PRIMARY KEY,
    value           TEXT NOT NULL,
    UNIQUE (value)                             -- deduplication of common values
);

CREATE TABLE item_data (
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    field_id        INTEGER NOT NULL REFERENCES fields(id),
    value_id        INTEGER NOT NULL REFERENCES item_data_values(id),
    PRIMARY KEY (item_id, field_id)
);

CREATE INDEX idx_item_data_value ON item_data (value_id);

-- External identifiers (DOI, ISBN, ISSN, PMID, arXiv ID, etc.)
CREATE TABLE item_identifiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    identifier_type TEXT NOT NULL,             -- 'doi', 'isbn', 'issn', 'pmid', 'arxiv', 'openalex'
    identifier_value TEXT NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (item_id, identifier_type, identifier_value)
);

CREATE INDEX idx_item_identifiers_type_value ON item_identifiers (identifier_type, identifier_value);
CREATE INDEX idx_item_identifiers_doi ON item_identifiers (identifier_value) WHERE identifier_type = 'doi';
```

## Creators (Authors, Editors, Translators)

```sql
CREATE TABLE creator_types (
    id              SERIAL PRIMARY KEY,
    type_name       TEXT NOT NULL UNIQUE,       -- 'author', 'editor', 'translator', 'contributor'
    csl_type        TEXT NOT NULL              -- CSL creator type mapping
);

CREATE TABLE creators (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    first_name      TEXT,
    last_name       TEXT NOT NULL,
    display_name    TEXT GENERATED ALWAYS AS (
        CASE WHEN first_name IS NOT NULL THEN first_name || ' ' || last_name
             ELSE last_name END
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_creators_name ON creators (last_name, first_name);

CREATE TABLE creator_identifiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id      UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    identifier_type TEXT NOT NULL,             -- 'orcid', 'isni', 'openalex'
    identifier_value TEXT NOT NULL,
    UNIQUE (creator_id, identifier_type)
);

CREATE INDEX idx_creator_ident_value ON creator_identifiers (identifier_type, identifier_value);

CREATE TABLE item_creators (
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    creator_id      UUID NOT NULL REFERENCES creators(id),
    creator_type_id INTEGER NOT NULL REFERENCES creator_types(id),
    order_index     INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (item_id, creator_id, creator_type_id)
);

CREATE INDEX idx_item_creators_creator ON item_creators (creator_id);
```

## Libraries & Collections

```sql
CREATE TABLE libraries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_user_id   UUID REFERENCES users(id) ON DELETE CASCADE,
    owner_org_id    UUID REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL DEFAULT 'My Library',
    library_type    TEXT NOT NULL DEFAULT 'personal', -- 'personal', 'group'
    is_public       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_library_owner CHECK (
        (owner_user_id IS NOT NULL AND owner_org_id IS NULL) OR
        (owner_user_id IS NULL AND owner_org_id IS NOT NULL)
    )
);

CREATE TABLE library_members (
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'reader', -- 'owner', 'editor', 'reader'
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
CREATE INDEX idx_collections_parent ON collections (parent_id) WHERE parent_id IS NOT NULL;

CREATE TABLE collection_items (
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (collection_id, item_id)
);
```

## Tags & Relations

```sql
CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    tag_type        INTEGER NOT NULL DEFAULT 0, -- 0 = user, 1 = automatic
    color           TEXT,                       -- hex color for UI display
    UNIQUE (library_id, name, tag_type)
);

CREATE INDEX idx_tags_library_name ON tags (library_id, name);

CREATE TABLE item_tags (
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (item_id, tag_id)
);

-- Inter-item relations (e.g., "related", "replaces", "reviews")
CREATE TABLE item_relations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject_item_id UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    predicate       TEXT NOT NULL,             -- 'dc:relation', 'dc:replaces', 'cito:cites'
    object_item_id  UUID REFERENCES items(id) ON DELETE SET NULL,
    object_uri      TEXT,                      -- external URI if not in library
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_item_relations_subject ON item_relations (subject_item_id);
CREATE INDEX idx_item_relations_object ON item_relations (object_item_id) WHERE object_item_id IS NOT NULL;
```

## Annotations & PDF Attachments

```sql
CREATE TABLE attachments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    file_name       TEXT NOT NULL,
    content_type    TEXT NOT NULL,             -- 'application/pdf', 'text/html'
    file_size_bytes BIGINT,
    storage_key     TEXT NOT NULL,             -- object storage path
    md5_hash        TEXT,
    link_mode       TEXT NOT NULL DEFAULT 'stored', -- 'stored', 'linked_url', 'linked_file'
    url             TEXT,                      -- original URL if captured from web
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attachments_item ON attachments (item_id);

-- Annotations following W3C Web Annotation Data Model concepts
CREATE TABLE annotations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attachment_id   UUID NOT NULL REFERENCES attachments(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    annotation_type TEXT NOT NULL,             -- 'highlight', 'note', 'underline', 'image'
    color           TEXT,                      -- hex color
    comment         TEXT,                      -- user's note text (W3C body)
    -- W3C target/selector fields
    page_number     INTEGER,
    position_json   JSONB,                     -- selector coordinates
    -- Example position_json:
    -- {
    --   "type": "FragmentSelector",
    --   "value": "page=3",
    --   "refinedBy": {
    --     "type": "TextPositionSelector",
    --     "start": 412,
    --     "end": 548
    --   }
    -- }
    sort_index      TEXT,                      -- for ordering annotations within a document
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_annotations_attachment ON annotations (attachment_id);
CREATE INDEX idx_annotations_user ON annotations (user_id);
CREATE INDEX idx_annotations_page ON annotations (attachment_id, page_number);
```

## Citation Graph & AI Features

```sql
-- Citation relationships between items
CREATE TABLE citations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citing_item_id  UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    cited_item_id   UUID REFERENCES items(id) ON DELETE SET NULL,
    cited_doi       TEXT,                      -- DOI of cited work if not in library
    cited_title     TEXT,                      -- title fallback for unresolved citations
    citation_context TEXT,                     -- surrounding sentence text
    citation_type   TEXT,                      -- 'supporting', 'contrasting', 'mentioning' (Scite-style)
    confidence      REAL,                      -- AI classification confidence 0.0-1.0
    page_number     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_citations_citing ON citations (citing_item_id);
CREATE INDEX idx_citations_cited ON citations (cited_item_id) WHERE cited_item_id IS NOT NULL;
CREATE INDEX idx_citations_doi ON citations (cited_doi) WHERE cited_doi IS NOT NULL;
CREATE INDEX idx_citations_type ON citations (citation_type);

-- AI-generated summaries and synthesis
CREATE TABLE ai_summaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id         UUID REFERENCES items(id) ON DELETE CASCADE,
    collection_id   UUID REFERENCES collections(id) ON DELETE CASCADE,
    summary_type    TEXT NOT NULL,             -- 'paper_summary', 'collection_synthesis', 'gap_analysis'
    model_id        TEXT NOT NULL,             -- AI model identifier
    model_version   TEXT NOT NULL,
    content         TEXT NOT NULL,
    token_count     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_summary_target CHECK (
        (item_id IS NOT NULL AND collection_id IS NULL) OR
        (item_id IS NULL AND collection_id IS NOT NULL)
    )
);

CREATE INDEX idx_ai_summaries_item ON ai_summaries (item_id) WHERE item_id IS NOT NULL;
CREATE INDEX idx_ai_summaries_collection ON ai_summaries (collection_id) WHERE collection_id IS NOT NULL;

-- Full-text search index
CREATE TABLE item_fulltext (
    item_id         UUID PRIMARY KEY REFERENCES items(id) ON DELETE CASCADE,
    content         TEXT NOT NULL,
    tsvector_content TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', content)) STORED,
    indexed_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_fulltext_search ON item_fulltext USING GIN (tsvector_content);

-- Embedding vectors for semantic search
CREATE TABLE item_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    embedding_type  TEXT NOT NULL,             -- 'title_abstract', 'fulltext', 'user_notes'
    model_id        TEXT NOT NULL,
    embedding       vector(1536),              -- pgvector extension
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (item_id, embedding_type, model_id)
);

CREATE INDEX idx_embeddings_vector ON item_embeddings USING ivfflat (embedding vector_cosine_ops);
```

## Systematic Review Support

```sql
CREATE TABLE review_projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    protocol        TEXT NOT NULL DEFAULT 'prisma', -- 'prisma', 'cochrane', 'custom'
    inclusion_criteria TEXT,
    exclusion_criteria TEXT,
    status          TEXT NOT NULL DEFAULT 'screening', -- 'screening', 'extraction', 'synthesis', 'complete'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE review_decisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review_projects(id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    reviewer_id     UUID NOT NULL REFERENCES users(id),
    stage           TEXT NOT NULL,             -- 'title_abstract', 'full_text'
    decision        TEXT NOT NULL,             -- 'include', 'exclude', 'maybe'
    reason          TEXT,
    decided_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, item_id, reviewer_id, stage)
);

CREATE INDEX idx_review_decisions_review ON review_decisions (review_id, stage);

CREATE TABLE data_extractions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review_projects(id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    extractor_id    UUID NOT NULL REFERENCES users(id),
    field_name      TEXT NOT NULL,             -- 'sample_size', 'study_design', 'outcome_measure'
    field_value     TEXT NOT NULL,
    confidence      TEXT,                      -- 'high', 'medium', 'low'
    extracted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_extractions_review_item ON data_extractions (review_id, item_id);
```

## Saved Searches & Alerts

```sql
CREATE TABLE saved_searches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    search_conditions JSONB NOT NULL,
    -- Example search_conditions:
    -- [
    --   {"field": "title", "operator": "contains", "value": "machine learning"},
    --   {"field": "date", "operator": "isAfter", "value": "2024-01-01"}
    -- ]
    is_alert        BOOLEAN NOT NULL DEFAULT FALSE,
    alert_frequency TEXT,                      -- 'daily', 'weekly'
    last_alerted_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE retraction_watches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    doi             TEXT NOT NULL,
    last_checked_at TIMESTAMPTZ,
    is_retracted    BOOLEAN NOT NULL DEFAULT FALSE,
    retraction_date DATE,
    retraction_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_retraction_watches_doi ON retraction_watches (doi);
```

## Example Queries

```sql
-- Retrieve a complete bibliographic item with all its fields (EAV unpivot)
SELECT i.id, it.type_name, it.csl_type, f.field_name, f.csl_field, idv.value
FROM items i
JOIN item_types it ON i.item_type_id = it.id
JOIN item_data id ON i.id = id.item_id
JOIN fields f ON id.field_id = f.id
JOIN item_data_values idv ON id.value_id = idv.id
WHERE i.id = '...'
ORDER BY f.field_name;

-- Find all items in a collection with their first author
SELECT i.id, idv_title.value AS title, c.last_name AS first_author
FROM collection_items ci
JOIN items i ON ci.item_id = i.id
JOIN item_data id_title ON i.id = id_title.item_id
JOIN fields f_title ON id_title.field_id = f_title.id AND f_title.field_name = 'title'
JOIN item_data_values idv_title ON id_title.value_id = idv_title.id
LEFT JOIN item_creators ic ON i.id = ic.item_id AND ic.order_index = 0
LEFT JOIN creators c ON ic.creator_id = c.id
WHERE ci.collection_id = '...'
ORDER BY ci.sort_order;

-- Recursive CTE to get a collection hierarchy
WITH RECURSIVE collection_tree AS (
    SELECT id, name, parent_id, 0 AS depth
    FROM collections WHERE id = '...'
    UNION ALL
    SELECT c.id, c.name, c.parent_id, ct.depth + 1
    FROM collections c
    JOIN collection_tree ct ON c.parent_id = ct.id
)
SELECT * FROM collection_tree ORDER BY depth, name;

-- Find papers that cite retracted work
SELECT DISTINCT i.id, idv.value AS title
FROM items i
JOIN item_data id ON i.id = id.item_id
JOIN fields f ON id.field_id = f.id AND f.field_name = 'title'
JOIN item_data_values idv ON id.value_id = idv.id
JOIN citations cit ON cit.citing_item_id = i.id
JOIN items cited ON cit.cited_item_id = cited.id
WHERE cited.is_retracted = TRUE;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users & Auth | 2 | users, organizations + junction |
| Organization | 2 | organizations, organization_members |
| Libraries & Collections | 4 | libraries, library_members, collections, collection_items |
| Item Types & Fields (Reference) | 3 | item_types, fields, item_type_fields |
| Items & Metadata (EAV) | 3 | items, item_data, item_data_values |
| Identifiers | 2 | item_identifiers, creator_identifiers |
| Creators | 3 | creators, creator_types, item_creators |
| Tags & Relations | 3 | tags, item_tags, item_relations |
| Attachments & Annotations | 2 | attachments, annotations |
| Citations & AI | 4 | citations, ai_summaries, item_fulltext, item_embeddings |
| Systematic Review | 3 | review_projects, review_decisions, data_extractions |
| Search & Alerts | 2 | saved_searches, retraction_watches |
| **Total** | **~33** | |

---

## Key Design Decisions

1. **EAV for bibliographic metadata** — Following Zotero's proven pattern, item field values are stored in `item_data` + `item_data_values` rather than as columns. This allows adding new item types and fields without schema migrations, which is essential when supporting 30+ CSL item types with varying field sets.

2. **Value deduplication** — `item_data_values` stores each unique value once and references it by ID, reducing storage for common values like journal names and publisher names that appear across thousands of items.

3. **Separate identifier tables** — DOIs, ORCIDs, ISBNs, PMIDs, arXiv IDs, and OpenAlex IDs are stored in dedicated identifier tables rather than as item fields. This enables efficient lookup by any identifier type and supports items/creators having multiple identifiers of different types.

4. **Library-scoped tenancy** — Multi-tenancy is at the library level, not the database or schema level. Personal and group libraries share the same tables with library_id as the partition key. This scales to millions of libraries without schema-per-tenant overhead.

5. **W3C-aligned annotations** — Annotation positioning uses a JSONB column following W3C Web Annotation selector concepts, allowing flexibility for different selector types (text position, fragment, CSS, XPath) without separate tables for each.

6. **Citation classification** — The `citations` table includes Scite-inspired `citation_type` (supporting/contrasting/mentioning) with AI confidence scores, enabling citation reliability analysis as a core feature rather than an afterthought.

7. **pgvector for semantic search** — Embedding vectors stored via the pgvector extension enable semantic similarity search without a separate vector database, keeping the architecture simple for initial deployment.

8. **Recursive collections** — Collections use a simple adjacency list (`parent_id` self-reference) for hierarchy, queryable via recursive CTEs. This is simpler than nested sets or materialized paths and handles the typical collection depth (rarely > 5 levels) efficiently.
