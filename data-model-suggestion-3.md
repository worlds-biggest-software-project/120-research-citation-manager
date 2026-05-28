# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Research & Citation Manager · Created: 2026-05-19

## Philosophy

This model treats every change to the research library as an immutable domain event stored in an append-only event store. The current state of any entity — an item's metadata, a collection's membership, an annotation's text — is derived by replaying events rather than queried from mutable rows. Read-optimized projections (materialized views) serve the application's query needs, while the event store provides a complete, tamper-evident audit trail of every action every user has ever taken.

This architecture is a natural fit for a research & citation manager because academic research demands provenance and reproducibility. Researchers need to answer questions like: "When did I add this paper to my library?", "Who changed this annotation?", "What did my collection look like when I submitted my manuscript on March 15?", "Which AI model generated this synthesis, and from which papers?" Event sourcing answers all of these by design, not as an afterthought.

The CQRS (Command Query Responsibility Segregation) pattern separates write operations (commands that produce events) from read operations (queries against projections). This enables independent scaling: the write path can be a simple append to the event store, while the read path uses denormalized tables optimized for each query pattern (library browsing, citation graph traversal, semantic search).

**Best for:** Deployments requiring complete audit trails, temporal queries ("what was true on date X?"), regulatory compliance, and AI provenance tracking where every synthesis must be traceable to its inputs.

**Trade-offs:**
- (+) Complete, immutable audit trail of every change — built-in, not bolted on
- (+) Temporal queries are trivial: replay events up to any point in time
- (+) AI provenance is first-class: every AI output is an event with model ID, input references, and timestamp
- (+) Event replay enables rebuilding read models with new schemas without data migration
- (+) Natural fit for real-time collaboration and sync (events = changes)
- (-) Higher storage cost: events are never deleted, and read models duplicate data
- (-) Eventual consistency between event store and projections; reads may lag writes
- (-) Debugging requires understanding both events and projections
- (-) Complex queries still need well-designed projections; ad-hoc SQL is harder
- (-) Schema evolution of events requires careful versioning (upcasting)
- (-) Higher implementation complexity; team must understand event sourcing patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CSL-JSON / CSL 1.0.2 | Item metadata events carry CSL-JSON payloads; projections store current CSL-JSON state |
| DOI (ISO 26324) | DOI resolution events record when a DOI was resolved and what metadata was returned |
| ORCID | Author identity events link ORCID iDs to creator records |
| W3C PROV-O (Provenance Ontology) | Event structure aligns with PROV-O concepts: Entity (item), Activity (event), Agent (user/AI) |
| W3C Web Annotation Data Model | Annotation events carry W3C-compliant body/target/selector data |
| Dublin Core (DCMI) | Metadata fields in events align with DC terms |
| ISO/IEC 27001 | Immutable event store supports information security audit requirements |
| GDPR (EU 2016/679) | Event tombstoning (marking events as redacted without deletion) supports right-to-erasure while preserving audit integrity |

---

## Event Store (Source of Truth)

```sql
-- ============================================================
-- EVENT STORE — Immutable, append-only
-- ============================================================

CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,             -- aggregate root ID (item, collection, library, etc.)
    stream_type     TEXT NOT NULL,             -- 'item', 'collection', 'library', 'annotation', 'review'
    event_type      TEXT NOT NULL,             -- 'ItemCreated', 'ItemMetadataUpdated', 'ItemAddedToCollection', etc.
    event_version   INTEGER NOT NULL,          -- version within the stream (optimistic concurrency)
    
    -- Event payload
    data            JSONB NOT NULL,
    -- Example data for ItemCreated:
    -- {
    --   "library_id": "uuid",
    --   "item_type": "article-journal",
    --   "metadata": { <full CSL-JSON> },
    --   "source": "browser_extension",
    --   "source_url": "https://doi.org/10.1126/...",
    --   "identifiers": {"doi": "10.1126/...", "pmid": "17754304"}
    -- }

    -- Event metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "correlation_id": "uuid",
    --   "causation_id": "uuid",
    --   "ip_address": "192.168.1.1",
    --   "user_agent": "Mozilla/5.0...",
    --   "client": "web",
    --   "api_version": "v1"
    -- }

    -- Actor
    actor_type      TEXT NOT NULL,             -- 'user', 'system', 'ai_agent'
    actor_id        UUID,                      -- user ID, or NULL for system events
    actor_model_id  TEXT,                      -- AI model ID if actor_type = 'ai_agent'

    -- Timestamps
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- GDPR support
    is_redacted     BOOLEAN NOT NULL DEFAULT FALSE,

    UNIQUE (stream_id, event_version)
);

-- Append-only: no UPDATE or DELETE triggers (enforced at application level)
-- Partition by month for performance
-- CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_events_stream ON events (stream_id, event_version);
CREATE INDEX idx_events_type ON events (event_type, occurred_at);
CREATE INDEX idx_events_actor ON events (actor_id, occurred_at) WHERE actor_id IS NOT NULL;
CREATE INDEX idx_events_occurred ON events (occurred_at);
CREATE INDEX idx_events_stream_type ON events (stream_type, occurred_at);

-- ============================================================
-- EVENT TYPE CATALOGUE
-- ============================================================
-- Item events:
--   ItemCreated, ItemMetadataUpdated, ItemDeleted, ItemRetracted,
--   ItemIdentifierAdded, ItemIdentifierRemoved,
--   ItemCreatorAdded, ItemCreatorRemoved, ItemCreatorReordered
--
-- Collection events:
--   CollectionCreated, CollectionRenamed, CollectionMoved, CollectionDeleted,
--   ItemAddedToCollection, ItemRemovedFromCollection
--
-- Library events:
--   LibraryCreated, LibraryMemberAdded, LibraryMemberRoleChanged,
--   LibraryMemberRemoved, LibrarySettingsUpdated
--
-- Annotation events:
--   AnnotationCreated, AnnotationUpdated, AnnotationDeleted
--
-- Tag events:
--   TagCreated, TagRenamed, TagDeleted, ItemTagged, ItemUntagged
--
-- Citation events:
--   CitationDiscovered, CitationClassified, CitationReclassified
--
-- AI events:
--   AISummaryGenerated, AISynthesisGenerated, AIGapAnalysisGenerated,
--   AIExtractionPerformed, AIEmbeddingGenerated,
--   SemanticSearchPerformed, LiteratureDiscoveryPerformed
--
-- Review events:
--   ReviewProjectCreated, ReviewDecisionMade, ReviewDecisionChanged,
--   DataExtractionPerformed, DataExtractionCorrected
--
-- Attachment events:
--   AttachmentUploaded, AttachmentDeleted, FulltextExtracted
--
-- System events:
--   RetractionCheckPerformed, DOIResolved, MetadataEnriched,
--   ExternalSyncCompleted
```

## Read Model Projections

```sql
-- ============================================================
-- PROJECTION: Users (updated by UserCreated, UserUpdated events)
-- ============================================================

CREATE TABLE proj_users (
    id              UUID PRIMARY KEY,
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    orcid_id        TEXT,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    preferences     JSONB NOT NULL DEFAULT '{}',
    storage_used_bytes BIGINT NOT NULL DEFAULT 0,
    last_event_version INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- ============================================================
-- PROJECTION: Libraries (updated by Library* events)
-- ============================================================

CREATE TABLE proj_libraries (
    id              UUID PRIMARY KEY,
    owner_user_id   UUID,
    owner_org_id    UUID,
    name            TEXT NOT NULL,
    library_type    TEXT NOT NULL,
    is_public       BOOLEAN NOT NULL DEFAULT FALSE,
    item_count      INTEGER NOT NULL DEFAULT 0,  -- denormalized counter
    last_event_version INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_library_members (
    library_id      UUID NOT NULL REFERENCES proj_libraries(id),
    user_id         UUID NOT NULL REFERENCES proj_users(id),
    role            TEXT NOT NULL,
    joined_at       TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (library_id, user_id)
);

-- ============================================================
-- PROJECTION: Items (updated by Item* events)
-- ============================================================

CREATE TABLE proj_items (
    id              UUID PRIMARY KEY,
    library_id      UUID NOT NULL,
    item_type       TEXT NOT NULL,
    item_key        TEXT NOT NULL,
    
    -- Current CSL-JSON metadata (rebuilt from events)
    metadata        JSONB NOT NULL DEFAULT '{}',
    enrichment      JSONB NOT NULL DEFAULT '{}',
    
    -- Denormalized for fast queries
    doi             TEXT,
    title           TEXT,
    pub_year        INTEGER,
    first_author    TEXT,
    
    -- Identifiers
    identifiers     JSONB NOT NULL DEFAULT '{}',
    -- {"doi": "10.1126/...", "pmid": "17754304", "isbn": null}
    
    -- Creator list (denormalized)
    creators        JSONB NOT NULL DEFAULT '[]',
    -- [{"first": "Eugene", "last": "Garfield", "role": "author", "orcid": "..."}]
    
    is_retracted    BOOLEAN NOT NULL DEFAULT FALSE,
    
    -- Event tracking
    last_event_version INTEGER NOT NULL DEFAULT 0,
    event_count     INTEGER NOT NULL DEFAULT 0, -- total events for this item
    
    date_added      TIMESTAMPTZ NOT NULL,
    date_modified   TIMESTAMPTZ NOT NULL,
    
    UNIQUE (library_id, item_key)
);

CREATE INDEX idx_proj_items_library ON proj_items (library_id);
CREATE INDEX idx_proj_items_doi ON proj_items (doi) WHERE doi IS NOT NULL;
CREATE INDEX idx_proj_items_metadata ON proj_items USING GIN (metadata jsonb_path_ops);
CREATE INDEX idx_proj_items_title_fts ON proj_items USING GIN (to_tsvector('english', COALESCE(title, '')));

-- ============================================================
-- PROJECTION: Collections (updated by Collection* events)
-- ============================================================

CREATE TABLE proj_collections (
    id              UUID PRIMARY KEY,
    library_id      UUID NOT NULL,
    parent_id       UUID,
    name            TEXT NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    item_count      INTEGER NOT NULL DEFAULT 0,
    last_event_version INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_collections_library ON proj_collections (library_id);

CREATE TABLE proj_collection_items (
    collection_id   UUID NOT NULL REFERENCES proj_collections(id),
    item_id         UUID NOT NULL REFERENCES proj_items(id),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    added_at        TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (collection_id, item_id)
);

-- ============================================================
-- PROJECTION: Annotations (updated by Annotation* events)
-- ============================================================

CREATE TABLE proj_annotations (
    id              UUID PRIMARY KEY,
    attachment_id   UUID NOT NULL,
    item_id         UUID NOT NULL,
    user_id         UUID NOT NULL,
    annotation_type TEXT NOT NULL,
    body            JSONB NOT NULL DEFAULT '{}',
    selector        JSONB NOT NULL DEFAULT '{}',
    page_number     INTEGER,
    color           TEXT,
    sort_index      TEXT,
    last_event_version INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_annotations_attachment ON proj_annotations (attachment_id);
CREATE INDEX idx_proj_annotations_user ON proj_annotations (user_id);

-- ============================================================
-- PROJECTION: Citations (updated by Citation* events)
-- ============================================================

CREATE TABLE proj_citations (
    id              UUID PRIMARY KEY,
    citing_item_id  UUID NOT NULL,
    cited_item_id   UUID,
    cited_doi       TEXT,
    citation_context TEXT,
    classification  TEXT,
    confidence      REAL,
    classified_by   TEXT,                      -- 'ai:scite-v3', 'user:uuid', 'system'
    classified_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_citations_citing ON proj_citations (citing_item_id);
CREATE INDEX idx_proj_citations_cited ON proj_citations (cited_item_id) WHERE cited_item_id IS NOT NULL;

-- ============================================================
-- PROJECTION: AI Outputs (updated by AI* events)
-- ============================================================

CREATE TABLE proj_ai_outputs (
    id              UUID PRIMARY KEY,
    target_type     TEXT NOT NULL,
    target_id       UUID NOT NULL,
    output_type     TEXT NOT NULL,
    model_id        TEXT NOT NULL,
    model_version   TEXT NOT NULL,
    content         JSONB NOT NULL,
    input_item_ids  UUID[],                    -- which items were used as input
    input_event_ids UUID[],                    -- which events triggered this output
    token_count     INTEGER,
    feedback_score  INTEGER,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_ai_outputs_target ON proj_ai_outputs (target_type, target_id);

-- ============================================================
-- PROJECTION: Embeddings (updated by AIEmbeddingGenerated events)
-- ============================================================

CREATE TABLE proj_embeddings (
    id              UUID PRIMARY KEY,
    source_type     TEXT NOT NULL,
    source_id       UUID NOT NULL,
    embedding_scope TEXT NOT NULL,
    model_id        TEXT NOT NULL,
    embedding       vector(1536),
    created_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (source_id, embedding_scope, model_id)
);

CREATE INDEX idx_proj_embeddings_vector ON proj_embeddings USING ivfflat (embedding vector_cosine_ops);

-- ============================================================
-- PROJECTION: Tags
-- ============================================================

CREATE TABLE proj_tags (
    id              UUID PRIMARY KEY,
    library_id      UUID NOT NULL,
    name            TEXT NOT NULL,
    tag_type        TEXT NOT NULL DEFAULT 'user',
    color           TEXT,
    item_count      INTEGER NOT NULL DEFAULT 0,
    UNIQUE (library_id, name, tag_type)
);

CREATE TABLE proj_item_tags (
    item_id         UUID NOT NULL REFERENCES proj_items(id),
    tag_id          UUID NOT NULL REFERENCES proj_tags(id),
    PRIMARY KEY (item_id, tag_id)
);

-- ============================================================
-- PROJECTION: Systematic Review
-- ============================================================

CREATE TABLE proj_review_projects (
    id              UUID PRIMARY KEY,
    library_id      UUID NOT NULL,
    name            TEXT NOT NULL,
    protocol        TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    status          TEXT NOT NULL,
    decision_counts JSONB NOT NULL DEFAULT '{}',
    -- {"screening": {"include": 45, "exclude": 120, "maybe": 12}, "full_text": {...}}
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE proj_review_decisions (
    id              UUID PRIMARY KEY,
    review_id       UUID NOT NULL,
    item_id         UUID NOT NULL,
    reviewer_id     UUID NOT NULL,
    stage           TEXT NOT NULL,
    decision        TEXT NOT NULL,
    reason          TEXT,
    decided_at      TIMESTAMPTZ NOT NULL,
    UNIQUE (review_id, item_id, reviewer_id, stage)
);

-- ============================================================
-- PROJECTION: Activity Timeline (for research memory)
-- ============================================================

CREATE TABLE proj_activity_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    event_id        UUID NOT NULL,
    event_type      TEXT NOT NULL,
    target_type     TEXT NOT NULL,
    target_id       UUID NOT NULL,
    summary         TEXT NOT NULL,             -- human-readable: "Added 'Garfield 1972' to collection 'Bibliometrics'"
    occurred_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_activity_user ON proj_activity_timeline (user_id, occurred_at DESC);
CREATE INDEX idx_activity_target ON proj_activity_timeline (target_type, target_id, occurred_at DESC);
```

## Snapshot Store (Performance Optimization)

```sql
-- Periodic snapshots to avoid replaying all events for long-lived aggregates
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    event_version   INTEGER NOT NULL,          -- snapshot taken at this event version
    state           JSONB NOT NULL,            -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, event_version)
);

-- Projection checkpoints: track which events each projection has processed
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Queries

```sql
-- Temporal query: What was in my collection on March 15, 2026?
SELECT e.data->>'item_id' AS item_id, e.event_type
FROM events e
WHERE e.stream_type = 'collection'
  AND e.stream_id = '...'  -- collection ID
  AND e.event_type IN ('ItemAddedToCollection', 'ItemRemovedFromCollection')
  AND e.occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY e.event_version;
-- Application replays these events to reconstruct the collection state at that point

-- Audit trail: Who changed this item's metadata and when?
SELECT e.event_id, e.event_type, e.actor_type, e.actor_id,
       e.data->>'changed_fields' AS changed_fields,
       e.occurred_at
FROM events e
WHERE e.stream_id = '...'  -- item ID
  AND e.stream_type = 'item'
  AND e.event_type IN ('ItemMetadataUpdated', 'ItemCreated')
ORDER BY e.event_version;

-- AI provenance: What inputs produced this synthesis?
SELECT e.event_id, e.event_type,
       e.data->>'model_id' AS model,
       e.data->>'model_version' AS version,
       e.data->'input_item_ids' AS source_papers,
       e.data->>'token_count' AS tokens,
       e.occurred_at
FROM events e
WHERE e.stream_type = 'item'
  AND e.event_type = 'AISynthesisGenerated'
  AND e.stream_id = '...'  -- collection ID
ORDER BY e.occurred_at DESC
LIMIT 1;

-- Research memory: What has the user interacted with recently?
SELECT target_type, target_id, event_type, summary, occurred_at
FROM proj_activity_timeline
WHERE user_id = '...'
ORDER BY occurred_at DESC
LIMIT 50;

-- Standard read query against projections (same as any relational model)
SELECT id, title, first_author, pub_year, doi
FROM proj_items
WHERE library_id = '...'
  AND item_type = 'article-journal'
ORDER BY date_modified DESC
LIMIT 50;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single events table (partitioned by month) |
| Snapshots & Checkpoints | 2 | snapshots, projection_checkpoints |
| Projection: Users | 1 | proj_users |
| Projection: Libraries | 2 | proj_libraries, proj_library_members |
| Projection: Items | 1 | proj_items (with denormalized creators/identifiers) |
| Projection: Collections | 2 | proj_collections, proj_collection_items |
| Projection: Tags | 2 | proj_tags, proj_item_tags |
| Projection: Annotations | 1 | proj_annotations |
| Projection: Citations | 1 | proj_citations |
| Projection: AI | 2 | proj_ai_outputs, proj_embeddings |
| Projection: Review | 2 | proj_review_projects, proj_review_decisions |
| Projection: Activity | 1 | proj_activity_timeline |
| **Total** | **~18** | Plus 1 event store = 19 total; projections are rebuildable |

---

## Key Design Decisions

1. **Single event store table** — All events across all aggregate types live in one table, partitioned by month. The `stream_type` and `stream_id` fields identify the aggregate. This simplifies infrastructure (one table to back up, monitor, and partition) while supporting cross-aggregate queries via the event type index.

2. **Events carry full CSL-JSON payloads** — `ItemCreated` events include the complete CSL-JSON metadata, not just a diff. This makes each event self-contained and enables event replay without needing to fetch earlier events for context. `ItemMetadataUpdated` events carry both old and new values for changed fields.

3. **PROV-O-aligned actor model** — Every event records who (actor_id, actor_type) and what (correlation_id, causation_id) caused it. When `actor_type = 'ai_agent'`, the `actor_model_id` records which AI model generated the change. This maps to W3C PROV-O's Entity/Activity/Agent model.

4. **Projections are disposable** — Every `proj_*` table can be dropped and rebuilt by replaying events from the event store. This means read model schema changes (adding a column, changing an index) require no data migration — just replay. The `projection_checkpoints` table tracks where each projection left off for incremental updates.

5. **GDPR via tombstoning** — Rather than deleting events (which would break the append-only guarantee), the `is_redacted` flag marks events whose personal data has been scrubbed. The event structure remains for audit trail integrity, but PII in the `data` payload is replaced with a redaction notice.

6. **Optimistic concurrency** — The `UNIQUE (stream_id, event_version)` constraint prevents concurrent writes to the same aggregate. If two users edit the same item simultaneously, the second write fails and must retry with the latest version.

7. **Activity timeline projection** — A dedicated projection translates raw events into human-readable activity summaries, powering the "research memory" feature. This projection is optimized for the query "what has this user done recently?" without requiring the application to parse raw event payloads.

8. **Snapshot store for long-lived aggregates** — Items with hundreds of events (heavily annotated papers, frequently updated metadata) use periodic snapshots to avoid replaying the entire event history on every read. Snapshots are an optimization, not a requirement — correctness comes from the event store.
