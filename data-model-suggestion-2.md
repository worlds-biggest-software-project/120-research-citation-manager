# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Research & Citation Manager · Created: 2026-05-19

## Philosophy

This model uses PostgreSQL's JSONB columns to store the inherently variable parts of bibliographic metadata — item fields, annotation selectors, and AI outputs — while keeping core relational structure for entities that benefit from foreign keys, indexes, and joins (users, libraries, collections, citations). Instead of the EAV pattern used by Zotero, each item stores its complete CSL-JSON metadata in a single JSONB column, which can be queried with GIN indexes and JSONB operators.

This approach is used by modern SaaS platforms that need schema flexibility without abandoning relational integrity. The CSL-JSON standard is itself a JSON schema, making JSONB storage a natural fit: the canonical representation of a bibliographic item is already JSON, so storing it as JSONB avoids the impedance mismatch of decomposing it into EAV tables. PostgreSQL's JSONB is binary-parsed, indexable, and supports containment queries (`@>`), path queries (`->>`), and JSON Schema validation via CHECK constraints or triggers.

This model is ideal for rapid MVP development, teams comfortable with JSONB operators, and deployments where bibliographic metadata varies widely (e.g., supporting grey literature, datasets, software, social media posts) without requiring schema migrations for each new field.

**Best for:** Rapid development teams building an MVP that needs schema flexibility for diverse item types while maintaining relational integrity for core entities.

**Trade-offs:**
- (+) Complete bibliographic item retrieved in a single row — no multi-table JOINs
- (+) Adding new fields or item types requires no schema migration
- (+) CSL-JSON stored natively; import/export is near-zero overhead
- (+) Fewer tables (~22) reduces schema complexity
- (+) JSONB GIN indexes enable efficient containment and path queries
- (-) JSONB queries use different syntax than standard SQL; learning curve for the team
- (-) No foreign key constraints within JSONB; referential integrity for nested data is application-enforced
- (-) Reporting and analytics queries on JSONB fields can be slower than on indexed columns
- (-) JSONB updates rewrite the entire document; frequent field-level updates are less efficient
- (-) Schema validation requires application logic or CHECK constraints rather than column-level types

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CSL-JSON / CSL 1.0.2 | Item metadata stored directly as CSL-JSON in `items.metadata` JSONB column; the canonical format IS the storage format |
| DOI (ISO 26324) | Extracted from JSONB `metadata->>'DOI'` and indexed separately for fast lookup |
| ORCID | Stored in creator objects within JSONB and in `users.orcid_id` for authenticated users |
| ISSN (ISO 3297) / ISBN (ISO 2108) | Stored within JSONB metadata; extracted indexes enable lookup |
| W3C Web Annotation Data Model | Annotations store W3C-compliant selector JSON in `annotations.selector` JSONB column |
| Dublin Core (DCMI) | CSL-JSON field names align with DC terms; JSONB queries use CSL field paths |
| BibTeX / RIS | Direct mapping from CSL-JSON; conversion functions operate on the JSONB column |
| OpenAlex Entity Model | External enrichment data (OpenAlex work/author IDs, concepts, topics) stored in `items.enrichment` JSONB |
| Schema.org / ScholarlyArticle | `metadata` JSONB can include Schema.org-compatible fields for web export |

---

## Users & Organizations

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    orcid_id        TEXT,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    auth_subject    TEXT,
    password_hash   TEXT,
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example preferences:
    -- {
    --   "default_citation_style": "apa-7th",
    --   "theme": "dark",
    --   "default_library_view": "table",
    --   "email_alerts": true,
    --   "language": "en"
    -- }
    storage_used_bytes BIGINT NOT NULL DEFAULT 0,
    storage_limit_bytes BIGINT NOT NULL DEFAULT 314572800,
    last_login_at   TIMESTAMPTZ,
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
```

## Libraries, Collections & Items

```sql
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
    description     TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    path            TEXT,                      -- materialized path, e.g., '/root-id/parent-id/this-id'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_collections_library ON collections (library_id);
CREATE INDEX idx_collections_path ON collections (path text_pattern_ops);

CREATE TABLE collection_items (
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (collection_id, item_id)
);

-- ============================================================
-- ITEMS — JSONB-First Bibliographic Records
-- ============================================================

CREATE TABLE items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    item_type       TEXT NOT NULL,             -- CSL type: 'article-journal', 'book', 'chapter', etc.
    item_key        TEXT NOT NULL,

    -- Complete CSL-JSON metadata in a single column
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata for a journal article:
    -- {
    --   "type": "article-journal",
    --   "title": "Citation analysis as a tool in journal evaluation",
    --   "author": [
    --     {"family": "Garfield", "given": "Eugene", "ORCID": "0000-0001-..."}
    --   ],
    --   "container-title": "Science",
    --   "volume": "178",
    --   "issue": "4060",
    --   "page": "471-479",
    --   "issued": {"date-parts": [[1972]]},
    --   "DOI": "10.1126/science.178.4060.471",
    --   "ISSN": "0036-8075",
    --   "PMID": "17754304",
    --   "abstract": "...",
    --   "language": "en",
    --   "publisher": "AAAS"
    -- }

    -- Enrichment data from external APIs (OpenAlex, Semantic Scholar, Scite)
    enrichment      JSONB NOT NULL DEFAULT '{}',
    -- Example enrichment:
    -- {
    --   "openalex_id": "W2023271753",
    --   "semantic_scholar_id": "abc123",
    --   "citation_count": 4523,
    --   "influential_citation_count": 312,
    --   "concepts": [
    --     {"id": "C41008148", "display_name": "Bibliometrics", "score": 0.92}
    --   ],
    --   "open_access": {"is_oa": true, "oa_url": "https://..."},
    --   "scite": {
    --     "supporting": 2100,
    --     "contrasting": 45,
    --     "mentioning": 2378
    --   },
    --   "altmetric_score": 128.5,
    --   "retraction": null
    -- }

    -- Denormalized fields for fast filtering/sorting
    doi             TEXT,                      -- extracted from metadata->>'DOI'
    title           TEXT,                      -- extracted from metadata->>'title'
    pub_year        INTEGER,                   -- extracted from metadata->'issued'->'date-parts'
    first_author    TEXT,                      -- extracted from metadata->'author'->0->>'family'

    is_retracted    BOOLEAN NOT NULL DEFAULT FALSE,
    version         INTEGER NOT NULL DEFAULT 1,
    date_added      TIMESTAMPTZ NOT NULL DEFAULT now(),
    date_modified   TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (library_id, item_key)
);

-- Structural indexes
CREATE INDEX idx_items_library ON items (library_id);
CREATE INDEX idx_items_type ON items (library_id, item_type);
CREATE INDEX idx_items_doi ON items (doi) WHERE doi IS NOT NULL;
CREATE INDEX idx_items_year ON items (library_id, pub_year);
CREATE INDEX idx_items_retracted ON items (library_id) WHERE is_retracted = TRUE;

-- GIN index on JSONB metadata for containment queries
CREATE INDEX idx_items_metadata ON items USING GIN (metadata jsonb_path_ops);

-- GIN index on enrichment for filtering by concepts, OA status, etc.
CREATE INDEX idx_items_enrichment ON items USING GIN (enrichment jsonb_path_ops);

-- Full-text search on title (denormalized column + tsvector)
CREATE INDEX idx_items_title_fts ON items USING GIN (to_tsvector('english', COALESCE(title, '')));
```

## Tags

```sql
CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    tag_type        TEXT NOT NULL DEFAULT 'user', -- 'user', 'automatic', 'ai'
    color           TEXT,
    UNIQUE (library_id, name, tag_type)
);

CREATE TABLE item_tags (
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (item_id, tag_id)
);
```

## Attachments & Annotations

```sql
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
    fulltext        TEXT,                      -- extracted PDF text for search
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

    -- W3C Web Annotation body
    body            JSONB NOT NULL DEFAULT '{}',
    -- Example body:
    -- {
    --   "type": "TextualBody",
    --   "value": "This contradicts Smith 2019 findings",
    --   "format": "text/plain",
    --   "purpose": "commenting"
    -- }

    -- W3C Web Annotation target with selector
    selector        JSONB NOT NULL DEFAULT '{}',
    -- Example selector:
    -- {
    --   "type": "TextQuoteSelector",
    --   "exact": "the results demonstrate a significant correlation",
    --   "prefix": "In conclusion, ",
    --   "suffix": " between the two variables."
    -- }

    page_number     INTEGER,
    color           TEXT,
    sort_index      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_annotations_attachment ON annotations (attachment_id);
CREATE INDEX idx_annotations_user ON annotations (user_id);
```

## Citations & Citation Analysis

```sql
CREATE TABLE citations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    citing_item_id  UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    cited_item_id   UUID REFERENCES items(id) ON DELETE SET NULL,
    cited_doi       TEXT,
    citation_context TEXT,
    classification  TEXT,                      -- 'supporting', 'contrasting', 'mentioning'
    confidence      REAL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "section": "introduction",
    --   "page": 3,
    --   "citation_number": 12,
    --   "is_self_citation": false,
    --   "classification_model": "scite-v3",
    --   "classification_timestamp": "2026-05-19T10:30:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_citations_citing ON citations (citing_item_id);
CREATE INDEX idx_citations_cited ON citations (cited_item_id) WHERE cited_item_id IS NOT NULL;
CREATE INDEX idx_citations_doi ON citations (cited_doi) WHERE cited_doi IS NOT NULL;
CREATE INDEX idx_citations_class ON citations (classification);
```

## AI Features & Semantic Search

```sql
CREATE TABLE ai_outputs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_type     TEXT NOT NULL,             -- 'item', 'collection', 'review_project'
    target_id       UUID NOT NULL,
    output_type     TEXT NOT NULL,             -- 'summary', 'synthesis', 'gap_analysis', 'extraction'
    model_id        TEXT NOT NULL,
    model_version   TEXT NOT NULL,

    content         JSONB NOT NULL,
    -- Example content for a synthesis:
    -- {
    --   "format": "structured",
    --   "summary": "These 12 papers address...",
    --   "agreements": ["All papers agree that...", "..."],
    --   "contradictions": ["Smith 2020 contradicts Jones 2019 on...", "..."],
    --   "gaps": ["No paper addresses the relationship between...", "..."],
    --   "methodology_breakdown": {
    --     "rct": 3,
    --     "cohort": 5,
    --     "case_study": 4
    --   },
    --   "sources_used": ["uuid1", "uuid2", "..."]
    -- }

    token_count     INTEGER,
    feedback_score  INTEGER,                   -- user rating 1-5
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_outputs_target ON ai_outputs (target_type, target_id);
CREATE INDEX idx_ai_outputs_type ON ai_outputs (output_type);

-- Semantic search embeddings (pgvector)
CREATE TABLE embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type     TEXT NOT NULL,             -- 'item', 'annotation', 'note'
    source_id       UUID NOT NULL,
    embedding_scope TEXT NOT NULL,             -- 'title_abstract', 'fulltext', 'notes'
    model_id        TEXT NOT NULL,
    embedding       vector(1536),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, embedding_scope, model_id)
);

CREATE INDEX idx_embeddings_vector ON embeddings USING ivfflat (embedding vector_cosine_ops);

-- Research memory: tracks what the user has read and interacted with
CREATE TABLE reading_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    action          TEXT NOT NULL,             -- 'opened', 'read', 'annotated', 'cited', 'shared'
    duration_seconds INTEGER,                  -- time spent reading
    progress        REAL,                      -- 0.0 to 1.0 read progress
    context         JSONB DEFAULT '{}',        -- what collection, review project, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reading_history_user ON reading_history (user_id, created_at DESC);
CREATE INDEX idx_reading_history_item ON reading_history (item_id);
```

## Systematic Review

```sql
CREATE TABLE review_projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    library_id      UUID NOT NULL REFERENCES libraries(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    protocol        TEXT NOT NULL DEFAULT 'prisma',
    
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "inclusion_criteria": [
    --     "Published 2020-2026",
    --     "English language",
    --     "RCT or cohort study"
    --   ],
    --   "exclusion_criteria": [
    --     "Case reports",
    --     "Non-human subjects"
    --   ],
    --   "extraction_fields": [
    --     {"name": "sample_size", "type": "integer"},
    --     {"name": "study_design", "type": "enum", "options": ["rct", "cohort", "case-control"]},
    --     {"name": "primary_outcome", "type": "text"},
    --     {"name": "effect_size", "type": "number"},
    --     {"name": "confidence_interval", "type": "text"}
    --   ],
    --   "screening_stages": ["title_abstract", "full_text"],
    --   "min_reviewers_per_stage": 2
    -- }

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

-- Data extraction uses JSONB to match the configurable extraction_fields
CREATE TABLE data_extractions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    review_id       UUID NOT NULL REFERENCES review_projects(id) ON DELETE CASCADE,
    item_id         UUID NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    extractor_id    UUID NOT NULL REFERENCES users(id),
    extracted_data  JSONB NOT NULL DEFAULT '{}',
    -- Example extracted_data:
    -- {
    --   "sample_size": 450,
    --   "study_design": "rct",
    --   "primary_outcome": "mortality at 30 days",
    --   "effect_size": 0.72,
    --   "confidence_interval": "0.55-0.94"
    -- }
    confidence      TEXT DEFAULT 'medium',
    extracted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (review_id, item_id, extractor_id)
);

CREATE INDEX idx_review_decisions_review ON review_decisions (review_id, stage);
CREATE INDEX idx_extractions_review ON data_extractions (review_id);
```

## Example Queries

```sql
-- Find all journal articles in a library about "machine learning" published after 2022
SELECT id, title, first_author, pub_year, doi,
       metadata->>'container-title' AS journal,
       enrichment->'scite'->>'supporting' AS supporting_citations
FROM items
WHERE library_id = '...'
  AND item_type = 'article-journal'
  AND pub_year > 2022
  AND to_tsvector('english', COALESCE(title, '')) @@ plainto_tsquery('machine learning');

-- JSONB containment query: find items by a specific author
SELECT id, title, doi
FROM items
WHERE library_id = '...'
  AND metadata @> '{"author": [{"family": "Garfield"}]}';

-- Find items with high contrasting citation counts (using enrichment JSONB)
SELECT id, title, doi,
       (enrichment->'scite'->>'contrasting')::int AS contrasting_count
FROM items
WHERE library_id = '...'
  AND (enrichment->'scite'->>'contrasting')::int > 10
ORDER BY contrasting_count DESC;

-- Semantic search: find papers similar to a query embedding
SELECT i.id, i.title, i.doi,
       e.embedding <=> '[0.1, 0.2, ...]'::vector AS distance
FROM embeddings e
JOIN items i ON e.source_id = i.id AND e.source_type = 'item'
WHERE e.embedding_scope = 'title_abstract'
ORDER BY e.embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 20;

-- Export a collection as CSL-JSON (nearly free — metadata IS CSL-JSON)
SELECT jsonb_agg(i.metadata || jsonb_build_object('id', i.item_key))
FROM collection_items ci
JOIN items i ON ci.item_id = i.id
WHERE ci.collection_id = '...';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Users & Auth | 3 | users, organizations, organization_members |
| Libraries & Collections | 4 | libraries, library_members, collections, collection_items |
| Items | 1 | Single table with JSONB metadata + enrichment |
| Tags | 2 | tags, item_tags |
| Attachments & Annotations | 2 | attachments, annotations (with JSONB body/selector) |
| Citations | 1 | citations (with JSONB metadata) |
| AI & Search | 3 | ai_outputs, embeddings, reading_history |
| Systematic Review | 3 | review_projects, review_decisions, data_extractions |
| **Total** | **~19** | Significantly fewer than normalized EAV approach |

---

## Key Design Decisions

1. **CSL-JSON as storage format** — Bibliographic metadata is stored as-is in CSL-JSON format within the `metadata` JSONB column. This eliminates the EAV decomposition/recomposition overhead and makes import/export from BibTeX, RIS, and other formats a single-step transformation to/from CSL-JSON.

2. **Denormalized columns for hot paths** — `doi`, `title`, `pub_year`, and `first_author` are extracted from JSONB into real columns with B-tree indexes. These cover 90%+ of sort/filter operations without touching JSONB, while the full metadata remains in JSONB for rich queries.

3. **Enrichment as separate JSONB column** — External data from OpenAlex, Semantic Scholar, Scite, and Altmetric is stored in a dedicated `enrichment` JSONB column rather than mixed into CSL-JSON metadata. This keeps the canonical bibliographic record clean and makes re-enrichment a simple column update.

4. **JSONB for configurable extraction schemas** — Systematic review extraction fields are defined in `review_projects.config` JSONB and extracted values stored in `data_extractions.extracted_data` JSONB. This allows each review project to define its own extraction schema without table modifications.

5. **W3C annotation model in JSONB** — Annotation body and selector are stored as JSONB following W3C Web Annotation Data Model structure, supporting multiple selector types (TextQuoteSelector, TextPositionSelector, FragmentSelector) without separate tables for each.

6. **Reading history for research memory** — The `reading_history` table captures user interactions with papers (opened, read, annotated, cited) with timing data, enabling the "research project memory" feature that answers questions like "have I already read something about X?"

7. **Materialized path for collections** — Collections use both adjacency list (`parent_id`) and materialized path (`path`) for hierarchy. The materialized path enables efficient subtree queries (`WHERE path LIKE '/root/%'`) without recursive CTEs, at the cost of path maintenance on moves.

8. **Single items table** — All item types (journal articles, books, datasets, software, preprints) live in one table differentiated by `item_type`. The JSONB metadata column handles field variation naturally. This is a deliberate contrast to the EAV model's three-table decomposition.
