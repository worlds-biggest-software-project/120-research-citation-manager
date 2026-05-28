# Development Plan: Research & Citation Manager

> Project: 120 - Research & Citation Manager
> Generated: 2026-05-25
> Status: Comprehensive phased development plan

---

## Technology Decisions

### Database: PostgreSQL 16+ with Hybrid Relational + JSONB Model (Data Model Suggestion 2)

**Rationale:** Data Model Suggestion 2 (Hybrid Relational + JSONB) is selected as the primary schema approach, with graph layer elements from Suggestion 4 introduced in Phase 8.

- **Why not EAV normalized (Suggestion 1)?** The EAV pattern adds significant query complexity (multi-table JOINs to retrieve a single bibliographic item) and is justified only when item types change frequently at runtime. Since CSL-JSON already defines the canonical item schema, storing it as JSONB eliminates the EAV decomposition/recomposition overhead entirely. The ~33-table count slows MVP velocity.
- **Why not event-sourced (Suggestion 3)?** CQRS/ES adds substantial operational complexity (projection infrastructure, eventual consistency, replay logic, snapshot management) that is unjustified at MVP. An `audit_log` table on the hybrid model provides adequate audit trail for regulatory needs without the architectural overhead. Event sourcing can be retrofitted for specific aggregates (annotations, review decisions) if temporal queries prove essential post-launch.
- **Why Suggestion 2?** The ~19-table schema stores bibliographic metadata as native CSL-JSON in a JSONB column, which is the canonical format already. Import/export is near-zero overhead. Denormalized columns (`doi`, `title`, `pub_year`, `first_author`) cover 90%+ of sort/filter operations. GIN indexes on JSONB enable containment queries for the remaining cases. This matches how modern SaaS platforms (Linear, Notion) handle semi-structured data.
- **Graph layer from Suggestion 4 deferred to Phase 8.** The `graph_nodes` / `graph_edges` tables and CiTO-typed citation edges will be introduced when visual citation graph exploration and co-author network analysis are built. The relational `citations` table handles MVP citation tracking; the graph layer becomes necessary for multi-hop traversals (bibliographic coupling, co-citation analysis, literature neighbourhood visualization).

### Backend: Node.js (TypeScript) with Fastify

**Rationale:**
- TypeScript provides type safety across the stack, critical for a data-intensive application where CSL-JSON transformations, citation style rendering, and metadata enrichment must preserve data integrity.
- Fastify outperforms Express by 2-3x on JSON serialization benchmarks, important for endpoints that return large library listings with enrichment data.
- Native JSON Schema validation in Fastify aligns with CSL-JSON validation at the API boundary.
- OpenAPI 3.1 spec generation via `@fastify/swagger` satisfies the standards.md requirement for published API documentation.
- Prisma ORM for type-safe database access with migration management; raw SQL for complex aggregation queries and pgvector similarity search.
- Bull (BullMQ) for background job queues: PDF text extraction, metadata enrichment from CrossRef/OpenAlex, embedding generation, retraction checking.

### Frontend: Next.js 15 (App Router) with React 19

**Rationale:**
- Server Components reduce client-side JavaScript for library browsing views that are primarily read-heavy.
- App Router's layout nesting maps naturally to the library > collection > item > PDF reader navigation hierarchy.
- React Server Actions simplify form mutations (add item, create collection, update annotation) without dedicated API endpoints for every form.
- Tailwind CSS + shadcn/ui for consistent, accessible UI components.
- @tanstack/react-table for data-dense library item lists with sortable columns, filtering, and bulk actions.
- PDF.js (Mozilla) for the in-library PDF reader; Fabric.js or Rough.js for annotation overlay rendering.
- D3.js or vis.js for citation graph visualization (Phase 8).

### AI Layer: LangChain.js with Claude API

**Rationale:**
- Claude (Sonnet / Opus models) for semantic literature search interpretation, paper summarisation, multi-paper synthesis, and citation classification. Claude's long context window (200K tokens) handles full-text papers and large collection contexts in a single query.
- LangChain.js for structured tool-calling (the AI agent queries the library database via defined tools rather than generating raw SQL), retrieval-augmented generation for synthesis, and prompt templating.
- pgvector for embedding storage and similarity search, avoiding a separate vector database at MVP.
- Embedding model: Voyage AI `voyage-3` or OpenAI `text-embedding-3-large` (1536 dimensions) for title+abstract embeddings.
- MCP Server exposure per standards.md recommendation, enabling external AI tools to query a researcher's library.

### Citation Engine: citeproc-rs (Rust/WASM)

**Rationale:**
- citeproc-rs is the Rust implementation of the CSL 1.0.2 specification, compilable to WebAssembly for browser-side rendering and usable natively on the server.
- Supports all 10,000+ CSL styles from the open CSL repository without custom formatting logic.
- WASM compilation enables real-time citation preview in the word processor plugin and browser extension without server round-trips.
- citeproc-js (JavaScript) as a fallback/compatibility layer for edge cases where citeproc-rs coverage gaps exist.

### Authentication: NextAuth.js v5 with OIDC/SAML

**Rationale:**
- NextAuth.js v5 supports OAuth 2.0 / OpenID Connect natively for Google, GitHub, and ORCID login.
- SAML 2.0 proxy via identity provider federation (Shibboleth, InCommon, eduGAIN) for institutional SSO, which is the dominant authentication mechanism in higher education.
- ORCID OAuth integration enables researcher identity verification and automatic ORCID iD population on user profiles.
- Row-Level Security (RLS) policies on PostgreSQL enforce library-level access isolation at the database layer.

### Object Storage: S3-Compatible (MinIO self-hosted / AWS S3 managed)

**Rationale:**
- PDF attachments are the largest storage consumer; object storage is the standard approach for large binary files.
- MinIO for self-hosted deployments (Docker Compose), AWS S3 or Cloudflare R2 for managed cloud.
- Presigned URLs for direct browser upload/download, reducing API server load.
- Storage quota enforcement at the database level (users.storage_used_bytes) with S3 lifecycle policies for retention.

### Testing: Vitest + Playwright + pgTAP

**Rationale:**
- Vitest for unit and integration tests (CSL-JSON transformations, BibTeX/RIS import/export, metadata enrichment logic, AI tool definitions).
- Playwright for end-to-end tests (library browsing, PDF annotation, citation insertion workflow, browser extension capture simulation).
- pgTAP for database-level tests (RLS policies, JSONB constraint validation, full-text search indexing, embedding similarity queries).

### Deployment: Docker Compose (self-hosted) + Vercel (managed cloud)

**Rationale:**
- Docker Compose for the self-hosted deployment mode (institutional data sovereignty requirement from README).
- Vercel for the managed cloud offering: Next.js is natively optimised for Vercel, edge functions handle API routing, and Vercel Postgres provides managed PostgreSQL with pgvector.
- CI/CD via GitHub Actions with migration checks, type checking, linting, and test execution on every PR.

---

## Project Structure

```
research-citation-manager/
  apps/
    web/                        # Next.js 15 frontend
      src/
        app/                    # App Router pages and layouts
          (auth)/               # Login, SSO, ORCID callback routes
          (library)/            # Authenticated layout
            library/            # Personal library views
            collections/        # Collection management
            items/[id]/         # Item detail, metadata editor
            reader/[id]/        # PDF reader with annotations
            search/             # Library search and semantic search
            discover/           # AI literature discovery
            synthesis/          # Multi-paper synthesis interface
            reviews/            # Systematic review projects
            graph/              # Citation graph explorer (Phase 8)
            settings/           # User preferences, API keys, storage
        components/
          ui/                   # shadcn/ui base components
          library/              # Library-specific components
          reader/               # PDF reader and annotation components
          citation/             # Citation rendering, style picker
          search/               # Search interface components
          ai/                   # AI synthesis, summary display
          graph/                # Citation graph visualization (Phase 8)
        lib/
          api/                  # API client functions
          hooks/                # React hooks
          csl/                  # CSL style utilities, citeproc-rs WASM bindings
          utils/                # Formatters, validators
    api/                        # Fastify backend API
      src/
        routes/                 # REST API route modules
          libraries/
          collections/
          items/
          attachments/
          annotations/
          citations/
          search/
          ai/
          reviews/
          export/
          admin/
        services/               # Business logic layer
          import/               # BibTeX, RIS, CSL-JSON importers
          export/               # BibTeX, RIS, CSL-JSON exporters
          metadata/             # CrossRef, OpenAlex, Semantic Scholar enrichment
          pdf/                  # PDF text extraction, parsing
          retraction/           # Retraction Watch monitoring
          citation-engine/      # citeproc-rs server-side rendering
        models/                 # Prisma-generated types + custom types
        plugins/                # Fastify plugins (auth, RLS, audit, rate-limit)
        jobs/                   # Background job definitions (BullMQ)
          enrich-metadata.ts
          extract-pdf-text.ts
          generate-embeddings.ts
          check-retractions.ts
          sync-external-apis.ts
        ai/                     # AI agent tools and prompts
          tools/                # LangChain tool definitions
          prompts/              # Prompt templates (summarise, synthesise, classify)
          mcp/                  # MCP server implementation
        validators/             # JSON Schema validators for CSL-JSON, JSONB columns
    browser-extension/          # Chrome/Firefox extension (Phase 3)
      src/
        background/             # Service worker
        content/                # Content scripts for metadata extraction
        popup/                  # Extension popup UI
        translators/            # Site-specific metadata extractors
    word-plugin/                # Word/Google Docs plugin (Phase 4)
  packages/
    db/                         # Prisma schema, migrations, seeds
    csl/                        # CSL style processing, citeproc-rs WASM
    shared/                     # Shared types, constants, utilities
    bibtex-parser/              # BibTeX import/export
    ris-parser/                 # RIS import/export
    eslint-config/              # Shared ESLint configuration
    tsconfig/                   # Shared TypeScript configuration
  docker/
    docker-compose.yml          # Self-hosted deployment
    Dockerfile.api
    Dockerfile.web
    Dockerfile.worker           # Background job worker
  docs/
    api/                        # Generated OpenAPI docs
    architecture/               # Architecture decision records
  tests/
    e2e/                        # Playwright E2E tests
    db/                         # pgTAP database tests
```

---

## Phase Dependency Graph

```
Phase 1: Foundation & Auth
    |
    v
Phase 2: Library Management & Item CRUD
    |
    +-----------------------------------+
    |                                   |
    v                                   v
Phase 3: Browser Extension           Phase 4: Citation Engine &
  & Metadata Capture                   Word Processor Plugin
    |                                   |
    +-----------------------------------+
    |
    v
Phase 5: PDF Reader & Annotations
    |
    v
Phase 6: AI Paper Summarisation & Semantic Search
    |
    +-----------------------------------+
    |                                   |
    v                                   v
Phase 7: Citation Analysis           Phase 8: Citation Graph
  & Retraction Monitoring              & Literature Discovery
    |                                   |
    +-----------------------------------+
    |
    v
Phase 9: Systematic Review Workflow
    |
    v
Phase 10: Multi-Paper Synthesis & Research Memory
    |
    v
Phase 11: Group Libraries & Collaboration
    |
    v
Phase 12: MCP Server, Integrations & Enterprise Features
```

**Critical path:** Phases 1 -> 2 -> 3+4 -> 5 -> 6 -> 10

---

## Phase 1: Foundation & Authentication

**Duration:** 3 weeks
**Depends on:** Nothing (starting point)

### Definition of Done
- PostgreSQL database provisioned with user, organization, and library tables; RLS policies active
- User can register, sign in via email/password and Google OIDC, and sign in via ORCID OAuth
- Library-scoped RLS enforces that users only see their own library data
- API server returns 401 for unauthenticated requests and 403 for unauthorized requests
- Audit log captures all authentication events
- CI pipeline runs type checking, linting, unit tests, and database tests on every PR

### Task 1.1: Database Foundation & Schema

**What:** Create the PostgreSQL database schema for users, organizations, organization_members, libraries, and library_members tables. Enable pgvector and pg_trgm extensions. Implement Row-Level Security policies for library-level isolation. Set up Prisma ORM with migration management.

**Design:**

```sql
-- packages/db/migrations/001_extensions.sql

CREATE EXTENSION IF NOT EXISTS "pgvector";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Core tables following Data Model Suggestion 2 schema
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
    storage_limit_bytes BIGINT NOT NULL DEFAULT 314572800,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

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

-- RLS policies
ALTER TABLE libraries ENABLE ROW LEVEL SECURITY;

CREATE POLICY library_owner_access ON libraries
  USING (owner_user_id = current_setting('app.current_user_id')::UUID);

CREATE POLICY library_member_access ON libraries
  USING (id IN (
    SELECT library_id FROM library_members
    WHERE user_id = current_setting('app.current_user_id')::UUID
  ));

CREATE OR REPLACE FUNCTION set_user_context(user_id UUID)
RETURNS VOID AS $$
BEGIN
  PERFORM set_config('app.current_user_id', user_id::TEXT, true);
END;
$$ LANGUAGE plpgsql;
```

```typescript
// apps/api/src/plugins/auth-context.ts

import { FastifyPluginAsync } from 'fastify';
import fp from 'fastify-plugin';

const authContextPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.addHook('preHandler', async (request) => {
    const userId = request.user?.id;
    if (userId) {
      await fastify.prisma.$executeRaw`SELECT set_user_context(${userId}::UUID)`;
    }
  });
};

export default fp(authContextPlugin);
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 1.1.1 | Create user with valid email and display name | Integration | User created, UUID primary key generated, default storage limit set |
| 1.1.2 | Create user with duplicate email | Integration | Unique constraint violation, 409 Conflict returned |
| 1.1.3 | Auto-create personal library on user registration | Integration | Library created with owner_user_id set, type 'personal' |
| 1.1.4 | Query libraries with RLS for User A | pgTAP | Only User A's libraries returned; User B's libraries invisible |
| 1.1.5 | Query libraries without user context set | pgTAP | Empty result set (RLS blocks all rows) |
| 1.1.6 | Prisma migration applies cleanly to empty database | Integration | All tables, indexes, RLS policies, and extensions created |
| 1.1.7 | pgvector extension is available and supports vector(1536) | pgTAP | CREATE TABLE with vector column succeeds |

### Task 1.2: Authentication (NextAuth.js v5 + ORCID)

**What:** Implement authentication with email/password registration, Google OIDC login, and ORCID OAuth login. Generate JWT access tokens for API requests. Auto-populate ORCID iD on user profile when authenticating via ORCID.

**Design:**

```typescript
// apps/web/src/app/api/auth/[...nextauth]/route.ts

import NextAuth from 'next-auth';
import GoogleProvider from 'next-auth/providers/google';
import CredentialsProvider from 'next-auth/providers/credentials';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from '@/lib/prisma';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: { strategy: 'jwt', maxAge: 24 * 60 * 60 },
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    {
      id: 'orcid',
      name: 'ORCID',
      type: 'oauth',
      authorization: {
        url: 'https://orcid.org/oauth/authorize',
        params: { scope: '/authenticate' },
      },
      token: 'https://orcid.org/oauth/token',
      userinfo: 'https://pub.orcid.org/v3.0/{orcid}/person',
      clientId: process.env.ORCID_CLIENT_ID!,
      clientSecret: process.env.ORCID_CLIENT_SECRET!,
      profile(profile) {
        return {
          id: profile.orcid,
          name: `${profile.name?.['given-names']?.value} ${profile.name?.['family-name']?.value}`,
          email: profile.emails?.email?.[0]?.email,
          orcidId: profile.orcid,
        };
      },
    },
  ],
  callbacks: {
    async jwt({ token, user, account }) {
      if (user) {
        token.userId = user.id;
        if (account?.provider === 'orcid') {
          token.orcidId = (user as any).orcidId;
          await prisma.user.update({
            where: { id: user.id },
            data: { orcid_id: (user as any).orcidId },
          });
        }
      }
      return token;
    },
    async session({ session, token }) {
      session.user.id = token.userId as string;
      session.user.orcidId = token.orcidId as string | undefined;
      return session;
    },
  },
});
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 1.2.1 | Register with email/password | E2E | User created, session established, redirected to library |
| 1.2.2 | Login with valid email/password | E2E | JWT issued, session cookie set, 200 response |
| 1.2.3 | Login with invalid password | E2E | 401 Unauthorized, no session created |
| 1.2.4 | Google OIDC login flow | E2E | User created or matched, session established |
| 1.2.5 | ORCID OAuth login flow | Integration | User created, orcid_id populated on user record |
| 1.2.6 | API request without Authorization header | Integration | 401 Unauthorized |
| 1.2.7 | API request with expired JWT | Integration | 401 Unauthorized with 'token_expired' error code |
| 1.2.8 | Auto-create personal library on first login | Integration | Library with type='personal' created for new user |

### Task 1.3: CI/CD Pipeline

**What:** Set up GitHub Actions CI pipeline with TypeScript type checking, ESLint, Vitest unit tests, pgTAP database tests, and Prisma migration verification. Configure Docker Compose for local development.

**Design:**

```yaml
# .github/workflows/ci.yml

name: CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: citation_test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/citation_test
      - run: npm run typecheck
      - run: npm run lint
      - run: npm run test:unit
      - run: npm run test:db
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/citation_test
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 1.3.1 | CI pipeline runs on PR to main | Workflow | All jobs pass: typecheck, lint, test:unit, test:db |
| 1.3.2 | docker-compose up starts all services | Manual | PostgreSQL, API, Web, Worker containers start and are healthy |
| 1.3.3 | Prisma migrate deploy succeeds against fresh database | CI | All migrations applied, schema matches expected state |

---

## Phase 2: Library Management & Item CRUD

**Duration:** 4 weeks
**Depends on:** Phase 1

### Definition of Done
- User can create, read, update, and delete bibliographic items in their library
- Items are stored as CSL-JSON in the JSONB `metadata` column with denormalized sort columns
- BibTeX and RIS import creates items with correctly mapped CSL-JSON metadata
- BibTeX, RIS, and CSL-JSON export produces standards-compliant output
- Collections support nested hierarchy (unlimited depth) with items assignable to multiple collections
- Tags are library-scoped with colour support
- Full-text search works across item titles, authors, and metadata fields
- DOI lookup auto-populates metadata via CrossRef API

### Task 2.1: Items Table & CSL-JSON CRUD

**What:** Implement the items table with JSONB metadata storage, denormalized columns, and GIN indexes. Build CRUD API endpoints for creating, reading, updating, and deleting items. Implement CSL-JSON validation using JSON Schema.

**Design:**

```typescript
// apps/api/src/routes/items/create.ts

import { FastifyPluginAsync } from 'fastify';
import { randomBytes } from 'crypto';
import { validateCSLJSON } from '@citation/shared/validators';

const createItemRoute: FastifyPluginAsync = async (fastify) => {
  fastify.post<{
    Body: {
      library_id: string;
      item_type: string;
      metadata: Record<string, unknown>;
    };
  }>('/items', {
    schema: {
      body: {
        type: 'object',
        required: ['library_id', 'item_type', 'metadata'],
        properties: {
          library_id: { type: 'string', format: 'uuid' },
          item_type: { type: 'string' },
          metadata: { type: 'object' },
        },
      },
    },
    handler: async (request, reply) => {
      const { library_id, item_type, metadata } = request.body;

      // Validate CSL-JSON structure
      const validation = validateCSLJSON(metadata);
      if (!validation.valid) {
        return reply.code(400).send({ error: 'Invalid CSL-JSON', details: validation.errors });
      }

      // Generate short alphanumeric key (Zotero-style)
      const item_key = randomBytes(4).toString('base64url').substring(0, 8).toUpperCase();

      // Extract denormalized fields
      const doi = (metadata as any).DOI || null;
      const title = (metadata as any).title || null;
      const pub_year = (metadata as any).issued?.['date-parts']?.[0]?.[0] || null;
      const first_author = (metadata as any).author?.[0]?.family || null;

      const item = await fastify.prisma.item.create({
        data: {
          library_id,
          item_type,
          item_key,
          metadata: metadata as any,
          doi,
          title,
          pub_year,
          first_author,
        },
      });

      // Queue background enrichment job
      await fastify.jobQueue.add('enrich-metadata', {
        item_id: item.id,
        doi: item.doi,
      });

      return reply.code(201).send(item);
    },
  });
};
```

```typescript
// packages/shared/src/validators/csl-json.ts

import Ajv from 'ajv';
import cslSchema from '@citation/csl/csl-data.json';

const ajv = new Ajv({ allErrors: true });
const validate = ajv.compile(cslSchema);

export function validateCSLJSON(metadata: unknown): { valid: boolean; errors?: any[] } {
  const valid = validate(metadata);
  return { valid: !!valid, errors: validate.errors ?? undefined };
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 2.1.1 | Create item with valid CSL-JSON for journal article | Integration | Item created, denormalized fields (doi, title, pub_year, first_author) populated |
| 2.1.2 | Create item with invalid CSL-JSON (missing type) | Integration | 400 Bad Request with validation errors |
| 2.1.3 | Read item returns full CSL-JSON metadata | Integration | metadata JSONB field returned intact, round-trips with input |
| 2.1.4 | Update item metadata preserves item_key | Integration | metadata updated, item_key unchanged, denormalized fields re-extracted |
| 2.1.5 | Delete item removes from database | Integration | Item deleted, 404 on subsequent GET |
| 2.1.6 | JSONB containment query finds item by author family name | Integration | `metadata @> '{"author":[{"family":"Garfield"}]}'` returns matching item |
| 2.1.7 | GIN index is used for JSONB containment query | pgTAP | EXPLAIN shows Index Scan using idx_items_metadata |
| 2.1.8 | Denormalized pub_year allows efficient range filtering | Integration | `WHERE pub_year BETWEEN 2020 AND 2026` uses B-tree index |

### Task 2.2: BibTeX & RIS Import/Export

**What:** Build import parsers for BibTeX and RIS formats that convert to CSL-JSON items. Build export functions that convert CSL-JSON items to BibTeX, RIS, and raw CSL-JSON. Support bulk import of files with 1000+ entries.

**Design:**

```typescript
// packages/bibtex-parser/src/index.ts

import { parse as parseBibtex } from '@retorquere/bibtex-parser';

interface CSLItem {
  type: string;
  title?: string;
  author?: Array<{ family: string; given?: string }>;
  DOI?: string;
  [key: string]: unknown;
}

const BIBTEX_TO_CSL_TYPE: Record<string, string> = {
  article: 'article-journal',
  book: 'book',
  inproceedings: 'paper-conference',
  inbook: 'chapter',
  phdthesis: 'thesis',
  mastersthesis: 'thesis',
  techreport: 'report',
  misc: 'article',
  unpublished: 'manuscript',
};

export function bibtexToCSL(bibtexString: string): CSLItem[] {
  const parsed = parseBibtex(bibtexString);
  return parsed.entries.map((entry) => {
    const cslType = BIBTEX_TO_CSL_TYPE[entry.type.toLowerCase()] || 'article';
    const authors = (entry.fields.author || []).map((a: any) => ({
      family: a.lastName,
      given: a.firstName,
    }));

    const item: CSLItem = {
      type: cslType,
      title: entry.fields.title?.[0],
      author: authors,
      'container-title': entry.fields.journal?.[0] || entry.fields.booktitle?.[0],
      volume: entry.fields.volume?.[0],
      issue: entry.fields.number?.[0],
      page: entry.fields.pages?.[0],
      DOI: entry.fields.doi?.[0],
      ISBN: entry.fields.isbn?.[0],
      ISSN: entry.fields.issn?.[0],
      abstract: entry.fields.abstract?.[0],
      publisher: entry.fields.publisher?.[0],
    };

    if (entry.fields.year) {
      item.issued = { 'date-parts': [[parseInt(entry.fields.year[0], 10)]] };
    }

    return item;
  });
}

export function cslToBibtex(items: CSLItem[]): string {
  return items.map((item, i) => {
    const type = Object.entries(BIBTEX_TO_CSL_TYPE).find(
      ([, v]) => v === item.type
    )?.[0] || 'misc';
    const key = item.DOI?.replace(/[^a-zA-Z0-9]/g, '') || `item${i}`;
    const fields: string[] = [];

    if (item.title) fields.push(`  title = {${item.title}}`);
    if (item.author) {
      fields.push(`  author = {${item.author.map(
        (a) => `${a.family}, ${a.given || ''}`
      ).join(' and ')}}`);
    }
    if (item['container-title']) fields.push(`  journal = {${item['container-title']}}`);
    if (item.volume) fields.push(`  volume = {${item.volume}}`);
    if (item.page) fields.push(`  pages = {${item.page}}`);
    if (item.DOI) fields.push(`  doi = {${item.DOI}}`);
    if (item.issued?.['date-parts']?.[0]?.[0]) {
      fields.push(`  year = {${item.issued['date-parts'][0][0]}}`);
    }

    return `@${type}{${key},\n${fields.join(',\n')}\n}`;
  }).join('\n\n');
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 2.2.1 | Import valid BibTeX file with 5 entries | Unit | 5 CSLItem objects returned with correct type mappings |
| 2.2.2 | BibTeX @article maps to CSL 'article-journal' | Unit | item.type === 'article-journal' |
| 2.2.3 | BibTeX author parsing handles "Last, First and Last2, First2" | Unit | Two author objects with correct family/given fields |
| 2.2.4 | Import RIS file with TY-JOUR entry | Unit | CSLItem with type 'article-journal' and mapped fields |
| 2.2.5 | Export CSL-JSON to BibTeX round-trips key fields | Unit | Parse exported BibTeX, compare title/author/DOI to original |
| 2.2.6 | Export CSL-JSON to RIS produces valid RIS format | Unit | Output starts with 'TY  -', ends with 'ER  -', fields are tagged |
| 2.2.7 | Bulk import of 1000-entry BibTeX file completes in <5s | Performance | Parsed and inserted within timeout |
| 2.2.8 | Import handles missing fields gracefully (no DOI, no abstract) | Unit | Item created with null/undefined optional fields |

### Task 2.3: Collections & Tags

**What:** Implement nested collection hierarchy with adjacency list and materialized path. Implement library-scoped tags with colour support. Support assigning items to multiple collections and multiple tags.

**Design:**

```typescript
// apps/api/src/routes/collections/create.ts

import { FastifyPluginAsync } from 'fastify';

const collectionsRoute: FastifyPluginAsync = async (fastify) => {
  // Create collection
  fastify.post<{
    Body: { library_id: string; name: string; parent_id?: string };
  }>('/collections', {
    handler: async (request, reply) => {
      const { library_id, name, parent_id } = request.body;

      let path = '';
      if (parent_id) {
        const parent = await fastify.prisma.collection.findUnique({
          where: { id: parent_id },
        });
        if (!parent) return reply.code(404).send({ error: 'Parent collection not found' });
        path = `${parent.path}/${parent_id}`;
      }

      const collection = await fastify.prisma.collection.create({
        data: {
          library_id,
          name,
          parent_id: parent_id || null,
          path,
        },
      });

      return reply.code(201).send(collection);
    },
  });

  // Get collection tree for a library
  fastify.get<{ Params: { libraryId: string } }>(
    '/libraries/:libraryId/collections',
    {
      handler: async (request) => {
        const collections = await fastify.prisma.collection.findMany({
          where: { library_id: request.params.libraryId },
          orderBy: [{ path: 'asc' }, { sort_order: 'asc' }],
          include: { _count: { select: { collection_items: true } } },
        });
        return collections;
      },
    }
  );

  // Add item to collection
  fastify.post<{
    Params: { collectionId: string };
    Body: { item_id: string };
  }>('/collections/:collectionId/items', {
    handler: async (request, reply) => {
      await fastify.prisma.collectionItem.create({
        data: {
          collection_id: request.params.collectionId,
          item_id: request.body.item_id,
        },
      });
      return reply.code(201).send({ ok: true });
    },
  });
};
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 2.3.1 | Create root collection | Integration | Collection created with empty path |
| 2.3.2 | Create nested sub-collection | Integration | Collection created with path containing parent ID |
| 2.3.3 | Get collection tree returns hierarchical ordering | Integration | Collections ordered by path then sort_order |
| 2.3.4 | Add item to collection | Integration | collection_items junction record created |
| 2.3.5 | Add same item to two different collections | Integration | Two junction records created; item appears in both |
| 2.3.6 | Delete collection cascades to collection_items | Integration | Junction records removed; items themselves preserved |
| 2.3.7 | Create tag with colour | Integration | Tag created with library_id, name, and hex colour |
| 2.3.8 | Assign tag to item; query items by tag | Integration | item_tags junction created; filter by tag returns correct items |
| 2.3.9 | Duplicate tag name in same library rejected | Integration | Unique constraint violation (library_id, name, tag_type) |

### Task 2.4: DOI Lookup & CrossRef Enrichment

**What:** Implement DOI-based metadata lookup via the CrossRef REST API. When a user enters a DOI, auto-populate the item's CSL-JSON metadata from CrossRef. Implement as both an on-demand API endpoint and a background enrichment job.

**Design:**

```typescript
// apps/api/src/services/metadata/crossref.ts

const CROSSREF_API = 'https://api.crossref.org/works';
const POLITE_MAILTO = 'api@citation-manager.dev'; // CrossRef Polite Pool

interface CrossRefWork {
  DOI: string;
  title: string[];
  author?: Array<{ family: string; given?: string; ORCID?: string }>;
  'container-title'?: string[];
  volume?: string;
  issue?: string;
  page?: string;
  issued?: { 'date-parts': number[][] };
  type: string;
  ISSN?: string[];
  abstract?: string;
  publisher?: string;
}

export async function lookupDOI(doi: string): Promise<Record<string, unknown> | null> {
  const response = await fetch(`${CROSSREF_API}/${encodeURIComponent(doi)}`, {
    headers: {
      'User-Agent': `ResearchCitationManager/1.0 (mailto:${POLITE_MAILTO})`,
    },
  });

  if (!response.ok) return null;

  const data = await response.json();
  const work: CrossRefWork = data.message;

  // Map CrossRef type to CSL type
  const typeMap: Record<string, string> = {
    'journal-article': 'article-journal',
    'book-chapter': 'chapter',
    'proceedings-article': 'paper-conference',
    'posted-content': 'article', // preprints
  };

  return {
    type: typeMap[work.type] || work.type,
    title: work.title?.[0],
    author: work.author?.map((a) => ({
      family: a.family,
      given: a.given,
      ...(a.ORCID ? { ORCID: a.ORCID } : {}),
    })),
    'container-title': work['container-title']?.[0],
    volume: work.volume,
    issue: work.issue,
    page: work.page,
    DOI: work.DOI,
    ISSN: work.ISSN?.[0],
    issued: work.issued,
    abstract: work.abstract,
    publisher: work.publisher,
  };
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 2.4.1 | Lookup valid DOI (10.1126/science.178.4060.471) | Integration | Returns CSL-JSON with title, author, journal, year |
| 2.4.2 | Lookup invalid DOI returns null | Integration | null returned, no error thrown |
| 2.4.3 | CrossRef User-Agent includes mailto for Polite Pool | Unit | Request header contains mailto: parameter |
| 2.4.4 | CrossRef type 'journal-article' maps to CSL 'article-journal' | Unit | Correct type mapping |
| 2.4.5 | Author ORCID from CrossRef preserved in CSL output | Unit | ORCID field present on author object |
| 2.4.6 | Background enrichment job populates enrichment JSONB | Integration | Item's enrichment column updated with CrossRef data |
| 2.4.7 | Rate limiting respects CrossRef Polite Pool guidelines | Unit | Requests throttled to <50/second |

---

## Phase 3: Browser Extension & Metadata Capture

**Duration:** 4 weeks
**Depends on:** Phase 2

### Definition of Done
- Chrome extension captures bibliographic metadata from PubMed, arXiv, Semantic Scholar, Google Scholar, and CrossRef/DOI landing pages
- One-click save adds the item to the user's library with correct CSL-JSON metadata
- Extension popup shows recent items and allows collection assignment
- PDF download and attachment to saved item (when user has access)
- Firefox extension build from same codebase via WebExtension API

### Task 3.1: Extension Architecture & Authentication

**What:** Build the browser extension scaffold with a service worker background script, content scripts for metadata extraction, and a popup UI. Implement authentication flow that connects the extension to the user's API session.

**Design:**

```typescript
// apps/browser-extension/src/background/index.ts

chrome.action.onClicked.addListener(async (tab) => {
  if (!tab.id || !tab.url) return;

  // Send message to content script to extract metadata
  const metadata = await chrome.tabs.sendMessage(tab.id, {
    type: 'EXTRACT_METADATA',
  });

  if (metadata) {
    // Save to library via API
    const token = await getStoredToken();
    const response = await fetch(`${API_BASE}/items`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`,
      },
      body: JSON.stringify({
        library_id: await getDefaultLibraryId(),
        item_type: metadata.type,
        metadata,
      }),
    });

    if (response.ok) {
      chrome.action.setBadgeText({ text: '✓', tabId: tab.id });
      setTimeout(() => chrome.action.setBadgeText({ text: '', tabId: tab.id }), 2000);
    }
  }
});
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 3.1.1 | Extension popup shows login form when not authenticated | E2E | Login form rendered, API key input visible |
| 3.1.2 | Extension stores API token securely in chrome.storage | Unit | Token stored in encrypted storage, not localStorage |
| 3.1.3 | Extension sends Authorization header on all API requests | Unit | Bearer token included in request headers |
| 3.1.4 | Extension popup shows recent items after authentication | E2E | Last 5 saved items displayed with titles |

### Task 3.2: Site-Specific Metadata Translators

**What:** Build content script translators for major academic databases that extract structured metadata from the page DOM. Each translator produces CSL-JSON from the site's HTML metadata (meta tags, JSON-LD, COinS, Highwire Press tags).

**Design:**

```typescript
// apps/browser-extension/src/content/translators/base.ts

export interface MetadataExtractor {
  matches(url: string): boolean;
  extract(document: Document): Record<string, unknown> | null;
}

// apps/browser-extension/src/content/translators/pubmed.ts

export const pubmedExtractor: MetadataExtractor = {
  matches(url: string): boolean {
    return /pubmed\.ncbi\.nlm\.nih\.gov\/\d+/.test(url);
  },

  extract(document: Document): Record<string, unknown> | null {
    const doi = document.querySelector('meta[name="citation_doi"]')?.getAttribute('content');
    const title = document.querySelector('meta[name="citation_title"]')?.getAttribute('content');
    const journal = document.querySelector('meta[name="citation_journal_title"]')?.getAttribute('content');
    const authors = Array.from(
      document.querySelectorAll('meta[name="citation_author"]')
    ).map((el) => {
      const name = el.getAttribute('content') || '';
      const parts = name.split(', ');
      return { family: parts[0], given: parts[1] };
    });
    const date = document.querySelector('meta[name="citation_date"]')?.getAttribute('content');
    const pmid = document.querySelector('meta[name="citation_pmid"]')?.getAttribute('content');

    if (!title) return null;

    return {
      type: 'article-journal',
      title,
      author: authors,
      'container-title': journal,
      DOI: doi,
      PMID: pmid,
      issued: date ? { 'date-parts': [[parseInt(date.split('/')[0], 10)]] } : undefined,
    };
  },
};
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 3.2.1 | PubMed article page extracts DOI, title, authors, journal | Unit | CSL-JSON with all fields populated from meta tags |
| 3.2.2 | arXiv abstract page extracts arXiv ID, title, authors | Unit | CSL-JSON with type='article', arXiv ID in identifiers |
| 3.2.3 | Semantic Scholar paper page extracts DOI and Semantic Scholar ID | Unit | CSL-JSON with enrichment data |
| 3.2.4 | Google Scholar result page extracts title and partial metadata | Unit | CSL-JSON with title; DOI may be absent |
| 3.2.5 | DOI landing page (doi.org) extracts metadata via Highwire Press tags | Unit | CSL-JSON from publisher landing page meta tags |
| 3.2.6 | Page with no recognizable metadata returns null | Unit | null returned, no error |
| 3.2.7 | JSON-LD ScholarlyArticle extraction from publisher page | Unit | CSL-JSON mapped from Schema.org vocabulary |

### Task 3.3: PDF Capture & Attachment

**What:** When the user saves a page that links to a PDF (or is a PDF), download the PDF and attach it to the item in the library. Upload to S3-compatible storage via presigned URL.

**Design:**

```typescript
// apps/browser-extension/src/background/pdf-capture.ts

export async function capturePDF(
  pdfUrl: string,
  itemId: string,
  token: string
): Promise<void> {
  // 1. Request presigned upload URL from API
  const uploadResp = await fetch(`${API_BASE}/attachments/upload-url`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify({
      item_id: itemId,
      file_name: pdfUrl.split('/').pop() || 'paper.pdf',
      content_type: 'application/pdf',
    }),
  });

  const { upload_url, attachment_id } = await uploadResp.json();

  // 2. Download PDF
  const pdfResponse = await fetch(pdfUrl);
  const pdfBlob = await pdfResponse.blob();

  // 3. Upload to S3 via presigned URL
  await fetch(upload_url, {
    method: 'PUT',
    body: pdfBlob,
    headers: { 'Content-Type': 'application/pdf' },
  });

  // 4. Confirm upload to API (triggers text extraction job)
  await fetch(`${API_BASE}/attachments/${attachment_id}/confirm`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${token}` },
  });
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 3.3.1 | Save PubMed article with open-access PDF link | E2E | Item created, PDF downloaded and attached |
| 3.3.2 | Save arXiv paper captures PDF from /pdf/ URL | E2E | PDF downloaded from arXiv, attached to item |
| 3.3.3 | PDF upload updates user's storage_used_bytes | Integration | storage_used_bytes incremented by file size |
| 3.3.4 | Upload exceeding storage limit returns 413 | Integration | 413 Payload Too Large, upload rejected |
| 3.3.5 | PDF text extraction job triggered on upload confirmation | Integration | Job queued in BullMQ; fulltext column populated after processing |

---

## Phase 4: Citation Engine & Word Processor Plugin

**Duration:** 4 weeks
**Depends on:** Phase 2

### Definition of Done
- CSL-based citation rendering produces formatted citations and bibliographies in APA, MLA, Chicago, Vancouver, and IEEE styles
- Word add-in inserts formatted citations and generates a bibliography in Microsoft Word documents
- Google Docs add-on inserts formatted citations and generates a bibliography in Google Docs
- Bibliography updates when citation style is changed
- BibTeX export from selected citations for LaTeX/Overleaf workflows

### Task 4.1: CSL Rendering Engine

**What:** Integrate citeproc-rs (compiled to WASM) for client-side citation rendering and citeproc-js for server-side rendering. Load CSL style definitions from the open CSL repository. Implement citation formatting for in-text citations and bibliography entries.

**Design:**

```typescript
// packages/csl/src/engine.ts

import init, { CslProcessor } from 'citeproc-rs-wasm';

let processor: CslProcessor | null = null;

export async function initCiteprocEngine(styleXml: string, locale: string = 'en-US') {
  await init();
  processor = new CslProcessor(styleXml, locale);
}

export function formatCitation(
  items: Array<{ id: string; [key: string]: unknown }>,
  citationItems: Array<{ id: string; locator?: string; label?: string }>
): string {
  if (!processor) throw new Error('Citeproc engine not initialized');

  // Register items
  for (const item of items) {
    processor.insertReference(JSON.stringify(item));
  }

  // Format citation cluster
  const citation = processor.processCitationCluster(
    JSON.stringify({
      citationItems: citationItems.map((ci) => ({
        id: ci.id,
        locator: ci.locator,
        label: ci.label || 'page',
      })),
    })
  );

  return citation;
}

export function formatBibliography(
  items: Array<{ id: string; [key: string]: unknown }>
): string[] {
  if (!processor) throw new Error('Citeproc engine not initialized');

  for (const item of items) {
    processor.insertReference(JSON.stringify(item));
  }

  const [params, entries] = processor.makeBibliography();
  return entries;
}

// Style loader with caching
const styleCache = new Map<string, string>();

export async function loadStyle(styleId: string): Promise<string> {
  if (styleCache.has(styleId)) return styleCache.get(styleId)!;

  const response = await fetch(
    `https://raw.githubusercontent.com/citation-style-language/styles/master/${styleId}.csl`
  );
  const xml = await response.text();
  styleCache.set(styleId, xml);
  return xml;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 4.1.1 | APA 7th edition in-text citation for single author | Unit | "(Garfield, 1972)" |
| 4.1.2 | APA 7th bibliography entry for journal article | Unit | "Garfield, E. (1972). Citation analysis... *Science*, *178*(4060), 471-479." |
| 4.1.3 | MLA 9th edition in-text citation | Unit | "(Garfield 471)" |
| 4.1.4 | Chicago author-date citation | Unit | "(Garfield 1972, 471)" |
| 4.1.5 | Vancouver numbered citation | Unit | "[1]" with corresponding numbered bibliography |
| 4.1.6 | IEEE citation format | Unit | "[1]" with IEEE-formatted bibliography entry |
| 4.1.7 | Multiple authors (>3) triggers "et al." in APA | Unit | "(Smith et al., 2024)" |
| 4.1.8 | Citation with page locator | Unit | "(Garfield, 1972, p. 473)" |
| 4.1.9 | Style switching re-renders all citations | Unit | Change from APA to MLA updates all formatted strings |
| 4.1.10 | Load CSL style from GitHub repository | Integration | XML retrieved, parsed, and usable by citeproc engine |

### Task 4.2: Microsoft Word Add-in

**What:** Build a Word Web Add-in (Office.js) that connects to the user's library, allows citation search and insertion, and generates a bibliography at the end of the document. Citations are stored as content controls with metadata linking back to library item IDs.

**Design:**

```typescript
// apps/word-plugin/src/taskpane/citation-inserter.ts

import { formatCitation, formatBibliography, loadStyle } from '@citation/csl';

export async function insertCitation(
  itemIds: string[],
  style: string = 'apa-7th-edition'
): Promise<void> {
  await Word.run(async (context) => {
    const selection = context.document.getSelection();

    // Fetch items from API
    const items = await fetchLibraryItems(itemIds);

    // Format citation
    const styleXml = await loadStyle(style);
    await initCiteprocEngine(styleXml);
    const citationText = formatCitation(
      items,
      itemIds.map((id) => ({ id }))
    );

    // Insert as content control with metadata
    const contentControl = selection.insertText(citationText, Word.InsertLocation.replace);
    const cc = contentControl.insertContentControl();
    cc.tag = JSON.stringify({ type: 'citation', itemIds, style });
    cc.appearance = Word.ContentControlAppearance.hidden;

    await context.sync();
  });
}

export async function generateBibliography(style: string = 'apa-7th-edition'): Promise<void> {
  await Word.run(async (context) => {
    // Collect all citation content controls
    const contentControls = context.document.contentControls;
    contentControls.load('tag');
    await context.sync();

    const allItemIds = new Set<string>();
    for (const cc of contentControls.items) {
      try {
        const tag = JSON.parse(cc.tag);
        if (tag.type === 'citation') {
          tag.itemIds.forEach((id: string) => allItemIds.add(id));
        }
      } catch { /* skip non-citation controls */ }
    }

    // Fetch all cited items and generate bibliography
    const items = await fetchLibraryItems([...allItemIds]);
    const styleXml = await loadStyle(style);
    await initCiteprocEngine(styleXml);
    const bibEntries = formatBibliography(items);

    // Insert bibliography at end of document
    const body = context.document.body;
    body.insertParagraph('References', Word.InsertLocation.end).style = 'Heading 1';
    for (const entry of bibEntries) {
      body.insertParagraph(entry, Word.InsertLocation.end);
    }

    await context.sync();
  });
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 4.2.1 | Insert single citation into Word document | E2E | Citation text appears at cursor; content control created with metadata |
| 4.2.2 | Insert citation with page locator | E2E | "(Garfield, 1972, p. 473)" inserted |
| 4.2.3 | Generate bibliography from all cited items | E2E | "References" heading + formatted entries appended at end |
| 4.2.4 | Change citation style re-renders all citations and bibliography | E2E | All content controls updated with new style formatting |
| 4.2.5 | Search library items from within Word add-in taskpane | E2E | Search results appear, clicking inserts citation |
| 4.2.6 | Remove citation removes content control and updates bibliography | E2E | Bibliography regenerated without removed citation |

### Task 4.3: Google Docs Add-on

**What:** Build a Google Docs add-on using Google Apps Script that mirrors the Word add-in functionality: citation search, insertion, and bibliography generation.

**Design:**

```typescript
// apps/word-plugin/src/google-docs/sidebar.ts
// (Google Apps Script transpiled from TypeScript)

function onOpen() {
  DocumentApp.getUi()
    .createAddonMenu()
    .addItem('Insert Citation', 'showCitationSidebar')
    .addItem('Generate Bibliography', 'generateBibliography')
    .addItem('Change Style', 'showStylePicker')
    .addToUi();
}

function showCitationSidebar() {
  const html = HtmlService.createHtmlOutputFromFile('citation-search')
    .setTitle('Insert Citation')
    .setWidth(350);
  DocumentApp.getUi().showSidebar(html);
}

function insertCitationAtCursor(itemIds: string[], formattedText: string) {
  const cursor = DocumentApp.getActiveDocument().getCursor();
  if (!cursor) {
    DocumentApp.getUi().alert('Place your cursor where you want the citation.');
    return;
  }

  const element = cursor.insertText(formattedText);
  // Store citation metadata as named range
  const doc = DocumentApp.getActiveDocument();
  doc.addNamedRange(`citation:${JSON.stringify(itemIds)}`, doc.newRange().addElement(element).build());
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 4.3.1 | Open add-on sidebar in Google Docs | E2E | Citation search sidebar appears |
| 4.3.2 | Search and insert citation | E2E | Formatted citation text inserted at cursor |
| 4.3.3 | Generate bibliography in Google Docs | E2E | Bibliography appended at end of document |
| 4.3.4 | Named ranges store citation metadata | Integration | Citation item IDs recoverable from named ranges |

---

## Phase 5: PDF Reader & Annotations

**Duration:** 4 weeks
**Depends on:** Phase 3, Phase 4

### Definition of Done
- In-library PDF reader renders PDFs using PDF.js with page navigation, zoom, and scroll
- Users can create highlight annotations (with colour selection), sticky notes, and text notes
- Annotations are synced to the database following W3C Web Annotation Data Model selectors
- Annotations are visible across devices and in shared libraries
- Full-text PDF content is extracted and indexed for search

### Task 5.1: PDF.js Reader Component

**What:** Build a React PDF reader component using PDF.js that renders PDFs from S3 storage, supports page navigation, zoom controls, text selection, and a scroll-synced page indicator.

**Design:**

```typescript
// apps/web/src/components/reader/PDFViewer.tsx

'use client';

import { useEffect, useRef, useState } from 'react';
import * as pdfjsLib from 'pdfjs-dist';
import { AnnotationLayer } from './AnnotationLayer';

pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.js';

interface PDFViewerProps {
  url: string;
  attachmentId: string;
  annotations: Annotation[];
  onAnnotationCreate: (annotation: NewAnnotation) => void;
}

export function PDFViewer({ url, attachmentId, annotations, onAnnotationCreate }: PDFViewerProps) {
  const containerRef = useRef<HTMLDivElement>(null);
  const [pdfDoc, setPdfDoc] = useState<pdfjsLib.PDFDocumentProxy | null>(null);
  const [currentPage, setCurrentPage] = useState(1);
  const [scale, setScale] = useState(1.2);
  const [selectionMode, setSelectionMode] = useState<'cursor' | 'highlight' | 'note'>('cursor');

  useEffect(() => {
    const loadPdf = async () => {
      const doc = await pdfjsLib.getDocument(url).promise;
      setPdfDoc(doc);
    };
    loadPdf();
  }, [url]);

  const handleTextSelection = () => {
    if (selectionMode !== 'highlight') return;
    const selection = window.getSelection();
    if (!selection || selection.isCollapsed) return;

    const range = selection.getRangeAt(0);
    const text = selection.toString();

    onAnnotationCreate({
      attachment_id: attachmentId,
      annotation_type: 'highlight',
      body: { type: 'TextualBody', value: '', format: 'text/plain', purpose: 'highlighting' },
      selector: {
        type: 'TextQuoteSelector',
        exact: text,
        prefix: text.substring(0, 30),
        suffix: text.substring(text.length - 30),
      },
      page_number: currentPage,
      color: '#FFFF00',
    });
  };

  return (
    <div className="flex flex-col h-full">
      <div className="flex items-center gap-2 p-2 border-b">
        <button onClick={() => setScale((s) => s - 0.2)}>-</button>
        <span>{Math.round(scale * 100)}%</span>
        <button onClick={() => setScale((s) => s + 0.2)}>+</button>
        <span className="ml-4">Page {currentPage} / {pdfDoc?.numPages || '...'}</span>
        <div className="ml-auto flex gap-1">
          <button onClick={() => setSelectionMode('cursor')} data-active={selectionMode === 'cursor'}>Cursor</button>
          <button onClick={() => setSelectionMode('highlight')} data-active={selectionMode === 'highlight'}>Highlight</button>
          <button onClick={() => setSelectionMode('note')} data-active={selectionMode === 'note'}>Note</button>
        </div>
      </div>
      <div ref={containerRef} className="flex-1 overflow-auto" onMouseUp={handleTextSelection}>
        {pdfDoc && Array.from({ length: pdfDoc.numPages }, (_, i) => (
          <PDFPage key={i} doc={pdfDoc} pageNumber={i + 1} scale={scale}>
            <AnnotationLayer
              annotations={annotations.filter((a) => a.page_number === i + 1)}
              onAnnotationCreate={onAnnotationCreate}
            />
          </PDFPage>
        ))}
      </div>
    </div>
  );
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 5.1.1 | Load and render a multi-page PDF | E2E | All pages rendered, page count displayed |
| 5.1.2 | Zoom in/out changes scale | E2E | PDF re-renders at new scale; text remains crisp |
| 5.1.3 | Scroll updates current page indicator | E2E | Page number updates as user scrolls |
| 5.1.4 | Text selection in cursor mode does not create annotation | E2E | No annotation created; text selected normally |
| 5.1.5 | Text selection in highlight mode creates annotation | E2E | Highlight annotation created with TextQuoteSelector |

### Task 5.2: Annotation CRUD & W3C Compliance

**What:** Implement annotation creation, reading, updating, and deletion. Store annotations with W3C Web Annotation Data Model body/selector JSONB columns. Sync annotations across devices via the API.

**Design:**

```typescript
// apps/api/src/routes/annotations/index.ts

import { FastifyPluginAsync } from 'fastify';

const annotationsRoute: FastifyPluginAsync = async (fastify) => {
  // Create annotation
  fastify.post<{
    Body: {
      attachment_id: string;
      annotation_type: string;
      body: Record<string, unknown>;
      selector: Record<string, unknown>;
      page_number?: number;
      color?: string;
    };
  }>('/annotations', {
    handler: async (request, reply) => {
      const annotation = await fastify.prisma.annotation.create({
        data: {
          ...request.body,
          user_id: request.user!.id,
          sort_index: generateSortIndex(request.body.page_number, request.body.selector),
        },
      });

      return reply.code(201).send(annotation);
    },
  });

  // Get annotations for an attachment
  fastify.get<{ Params: { attachmentId: string } }>(
    '/attachments/:attachmentId/annotations',
    {
      handler: async (request) => {
        return fastify.prisma.annotation.findMany({
          where: { attachment_id: request.params.attachmentId },
          orderBy: { sort_index: 'asc' },
          include: { user: { select: { display_name: true } } },
        });
      },
    }
  );
};

function generateSortIndex(page?: number, selector?: Record<string, unknown>): string {
  const p = String(page || 0).padStart(6, '0');
  const pos = selector?.start ? String(selector.start).padStart(10, '0') : '0000000000';
  return `${p}|${pos}`;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 5.2.1 | Create highlight annotation with TextQuoteSelector | Integration | Annotation stored with W3C-compliant selector JSONB |
| 5.2.2 | Create note annotation with TextualBody | Integration | Annotation stored with body containing note text |
| 5.2.3 | Annotations ordered by sort_index (page, then position) | Integration | Page 1 annotations before page 2; within page, ordered by position |
| 5.2.4 | Update annotation comment text | Integration | body.value updated; other fields unchanged |
| 5.2.5 | Delete annotation | Integration | Annotation removed; 404 on subsequent GET |
| 5.2.6 | Annotations visible to library members with read access | Integration | Shared library member can see annotations from all users |

### Task 5.3: PDF Text Extraction & Indexing

**What:** Build a background job that extracts text from uploaded PDFs using pdf-parse (Node.js) or Apache Tika. Store extracted text in the attachments.fulltext column and update the tsvector index for full-text search.

**Design:**

```typescript
// apps/api/src/jobs/extract-pdf-text.ts

import { Job } from 'bullmq';
import pdfParse from 'pdf-parse';
import { prisma } from '@/lib/prisma';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';

const s3 = new S3Client({ region: process.env.AWS_REGION });

export async function extractPdfText(job: Job<{ attachment_id: string }>) {
  const attachment = await prisma.attachment.findUnique({
    where: { id: job.data.attachment_id },
  });

  if (!attachment || attachment.content_type !== 'application/pdf') return;

  // Download PDF from S3
  const s3Obj = await s3.send(new GetObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: attachment.storage_key,
  }));

  const buffer = Buffer.from(await s3Obj.Body!.transformToByteArray());
  const parsed = await pdfParse(buffer);

  // Update attachment with extracted text (tsvector auto-generated by STORED column)
  await prisma.attachment.update({
    where: { id: attachment.id },
    data: { fulltext: parsed.text },
  });

  // Queue embedding generation
  await job.queue.add('generate-embeddings', {
    item_id: attachment.item_id,
    text: parsed.text.substring(0, 8000), // first 8K chars for embedding
  });
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 5.3.1 | Extract text from a standard academic PDF | Integration | fulltext column populated with readable text |
| 5.3.2 | Full-text search finds item by content keyword | Integration | `WHERE fulltext_ts @@ plainto_tsquery('bibliometrics')` returns matching attachment |
| 5.3.3 | Scanned PDF (image-based) returns empty text | Integration | fulltext set to empty string; no error thrown |
| 5.3.4 | Embedding generation job queued after text extraction | Integration | Job appears in BullMQ queue with truncated text |
| 5.3.5 | Large PDF (500 pages) extracts without timeout | Performance | Extraction completes within 60s timeout |

---

## Phase 6: AI Paper Summarisation & Semantic Search

**Duration:** 4 weeks
**Depends on:** Phase 5

### Definition of Done
- AI generates a one-paragraph plain-language summary for any PDF in the library
- Semantic search accepts a natural-language research question and returns relevant papers ranked by embedding similarity
- Embedding vectors stored in pgvector for all items with abstracts or full-text
- Search results display relevance scores and highlight matching context

### Task 6.1: Embedding Generation Pipeline

**What:** Build a background job that generates embedding vectors for library items using their title, abstract, and optionally full-text. Store embeddings in pgvector. Implement incremental updates when item metadata changes.

**Design:**

```typescript
// apps/api/src/jobs/generate-embeddings.ts

import { Job } from 'bullmq';
import { prisma } from '@/lib/prisma';
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

export async function generateEmbeddings(
  job: Job<{ item_id: string; text?: string }>
) {
  const item = await prisma.item.findUnique({ where: { id: job.data.item_id } });
  if (!item) return;

  const metadata = item.metadata as Record<string, unknown>;
  const titleAbstract = [metadata.title, metadata.abstract]
    .filter(Boolean)
    .join('\n\n');

  if (!titleAbstract) return;

  // Generate embedding via Voyage AI or OpenAI
  const embeddingResponse = await fetch('https://api.voyageai.com/v1/embeddings', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.VOYAGE_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'voyage-3',
      input: [titleAbstract],
      input_type: 'document',
    }),
  });

  const { data } = await embeddingResponse.json();
  const embedding = data[0].embedding;

  // Upsert embedding in pgvector
  await prisma.$executeRaw`
    INSERT INTO embeddings (id, source_type, source_id, embedding_scope, model_id, embedding, created_at)
    VALUES (gen_random_uuid(), 'item', ${item.id}::UUID, 'title_abstract', 'voyage-3',
            ${JSON.stringify(embedding)}::vector, now())
    ON CONFLICT (source_id, embedding_scope, model_id)
    DO UPDATE SET embedding = ${JSON.stringify(embedding)}::vector, created_at = now()
  `;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 6.1.1 | Generate embedding for item with title and abstract | Integration | Embedding row created in embeddings table, vector has 1536 dimensions |
| 6.1.2 | Upsert embedding on metadata update | Integration | Existing embedding replaced; no duplicate rows |
| 6.1.3 | Skip embedding for item with no title or abstract | Integration | No embedding created; job completes without error |
| 6.1.4 | Batch embedding generation for 100 items | Performance | Completes within rate limits, all embeddings stored |

### Task 6.2: Semantic Search API

**What:** Build a search endpoint that accepts a natural-language query, converts it to an embedding vector, and queries pgvector for the most similar items. Combine with full-text search for hybrid results.

**Design:**

```typescript
// apps/api/src/routes/search/semantic.ts

import { FastifyPluginAsync } from 'fastify';

const semanticSearchRoute: FastifyPluginAsync = async (fastify) => {
  fastify.post<{
    Body: { query: string; library_id: string; limit?: number };
  }>('/search/semantic', {
    handler: async (request) => {
      const { query, library_id, limit = 20 } = request.body;

      // 1. Generate query embedding
      const queryEmbedding = await generateQueryEmbedding(query);

      // 2. Vector similarity search via pgvector
      const results = await fastify.prisma.$queryRaw`
        SELECT i.id, i.title, i.first_author, i.pub_year, i.doi,
               i.metadata->>'abstract' AS abstract,
               e.embedding <=> ${JSON.stringify(queryEmbedding)}::vector AS distance
        FROM embeddings e
        JOIN items i ON e.source_id = i.id AND e.source_type = 'item'
        WHERE i.library_id = ${library_id}::UUID
          AND e.embedding_scope = 'title_abstract'
        ORDER BY e.embedding <=> ${JSON.stringify(queryEmbedding)}::vector
        LIMIT ${limit}
      `;

      // 3. Compute relevance score (1 - distance for cosine)
      return (results as any[]).map((r) => ({
        ...r,
        relevance_score: Math.round((1 - r.distance) * 100) / 100,
      }));
    },
  });
};

async function generateQueryEmbedding(query: string): Promise<number[]> {
  const response = await fetch('https://api.voyageai.com/v1/embeddings', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.VOYAGE_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: 'voyage-3',
      input: [query],
      input_type: 'query',
    }),
  });
  const { data } = await response.json();
  return data[0].embedding;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 6.2.1 | Semantic search for "citation analysis methods" | Integration | Returns papers about citation analysis ranked by relevance |
| 6.2.2 | Semantic search returns relevance scores between 0 and 1 | Integration | All scores in [0, 1] range, sorted descending |
| 6.2.3 | Semantic search scoped to specific library | Integration | Only items from specified library_id returned |
| 6.2.4 | Empty library returns empty results | Integration | Empty array, no error |
| 6.2.5 | Cross-disciplinary query finds semantically related papers | Integration | Query about "impact metrics" finds bibliometrics papers even without keyword match |

### Task 6.3: AI Paper Summarisation

**What:** Build an API endpoint and UI component that generates a plain-language summary of any paper in the library using Claude. Use the paper's abstract, full-text (if available), and metadata as context. Store generated summaries in the ai_outputs table.

**Design:**

```typescript
// apps/api/src/routes/ai/summarise.ts

import { FastifyPluginAsync } from 'fastify';
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

const summariseRoute: FastifyPluginAsync = async (fastify) => {
  fastify.post<{
    Body: { item_id: string };
  }>('/ai/summarise', {
    handler: async (request, reply) => {
      const item = await fastify.prisma.item.findUnique({
        where: { id: request.body.item_id },
        include: { attachments: { where: { content_type: 'application/pdf' } } },
      });

      if (!item) return reply.code(404).send({ error: 'Item not found' });

      const metadata = item.metadata as Record<string, unknown>;
      const fulltext = item.attachments[0]?.fulltext;

      const context = [
        `Title: ${metadata.title}`,
        metadata.author
          ? `Authors: ${(metadata.author as any[]).map((a) => `${a.given} ${a.family}`).join(', ')}`
          : null,
        metadata.abstract ? `Abstract: ${metadata.abstract}` : null,
        fulltext ? `Full text (first 50,000 chars): ${fulltext.substring(0, 50000)}` : null,
      ].filter(Boolean).join('\n\n');

      const response = await anthropic.messages.create({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 500,
        messages: [
          {
            role: 'user',
            content: `You are an expert research assistant. Summarise the following academic paper in one clear paragraph (150-200 words) that a graduate student could understand. Focus on the key finding, methodology, and significance. Do not use jargon without explanation.\n\n${context}`,
          },
        ],
      });

      const summary = response.content[0].type === 'text' ? response.content[0].text : '';

      // Store in ai_outputs
      const output = await fastify.prisma.aiOutput.create({
        data: {
          target_type: 'item',
          target_id: item.id,
          output_type: 'summary',
          model_id: 'claude-sonnet-4-20250514',
          model_version: 'claude-sonnet-4-20250514',
          content: { summary, format: 'plain_text' },
          token_count: response.usage.output_tokens,
        },
      });

      return { summary, output_id: output.id };
    },
  });
};
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 6.3.1 | Summarise paper with abstract only | Integration | Returns 150-200 word summary in plain language |
| 6.3.2 | Summarise paper with full-text PDF | Integration | Summary incorporates details beyond abstract |
| 6.3.3 | Summary stored in ai_outputs table | Integration | ai_output row with target_type='item', output_type='summary' |
| 6.3.4 | Re-summarise replaces existing summary | Integration | New ai_output created; UI shows latest |
| 6.3.5 | Item with no abstract or full-text | Integration | 400 Bad Request with "insufficient content" message |
| 6.3.6 | Token count recorded in ai_outputs | Integration | token_count > 0, matches API response |

---

## Phase 7: Citation Analysis & Retraction Monitoring

**Duration:** 3 weeks
**Depends on:** Phase 6

### Definition of Done
- Citation classification labels each citation as supporting, contrasting, or mentioning
- Retraction Watch integration automatically flags retracted papers in the library
- Citation reliability indicators visible on item detail view
- Background monitoring checks for new retractions on a scheduled basis

### Task 7.1: Citation Classification

**What:** Implement AI-powered classification of citation context as supporting, contrasting, or mentioning (inspired by Scite's methodology). Extract citation contexts from PDF full-text and classify them using Claude.

**Design:**

```typescript
// apps/api/src/services/citation-classifier.ts

import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

interface CitationContext {
  cited_doi: string;
  context_text: string;
  page_number: number;
}

interface CitationClassification {
  cited_doi: string;
  classification: 'supporting' | 'contrasting' | 'mentioning';
  confidence: number;
  context_text: string;
}

export async function classifyCitations(
  contexts: CitationContext[]
): Promise<CitationClassification[]> {
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 2000,
    messages: [
      {
        role: 'user',
        content: `Classify each citation context below as one of:
- "supporting": the citing paper agrees with, builds on, or provides evidence for the cited work
- "contrasting": the citing paper disagrees with, contradicts, or presents opposing evidence to the cited work
- "mentioning": the citing paper references the cited work without clear agreement or disagreement

Return JSON array with: { "index": <number>, "classification": "<type>", "confidence": <0.0-1.0> }

Citation contexts:
${contexts.map((c, i) => `[${i}] "${c.context_text}"`).join('\n')}`,
      },
    ],
  });

  const text = response.content[0].type === 'text' ? response.content[0].text : '[]';
  const classifications = JSON.parse(text);

  return classifications.map((c: any) => ({
    cited_doi: contexts[c.index].cited_doi,
    classification: c.classification,
    confidence: c.confidence,
    context_text: contexts[c.index].context_text,
  }));
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 7.1.1 | Classify "consistent with findings by Smith (2020)" | Unit | classification='supporting', confidence > 0.8 |
| 7.1.2 | Classify "contradicts the results reported by Jones (2019)" | Unit | classification='contrasting', confidence > 0.8 |
| 7.1.3 | Classify "as described by Lee et al. (2021)" | Unit | classification='mentioning', confidence > 0.7 |
| 7.1.4 | Batch classification of 10 citation contexts | Integration | All 10 classified with valid types and confidence scores |
| 7.1.5 | Classification results stored in citations table | Integration | citation rows with classification and confidence populated |

### Task 7.2: Retraction Watch Integration

**What:** Implement a background job that checks the Retraction Watch database and CrossRef for retraction notices. Flag retracted papers in the library and notify the user. Run on a weekly schedule.

**Design:**

```typescript
// apps/api/src/jobs/check-retractions.ts

import { Job } from 'bullmq';
import { prisma } from '@/lib/prisma';

export async function checkRetractions(job: Job) {
  // Get all items with DOIs that haven't been checked recently
  const items = await prisma.item.findMany({
    where: {
      doi: { not: null },
      is_retracted: false,
    },
    select: { id: true, doi: true, library_id: true },
  });

  for (const item of items) {
    // Check CrossRef for retraction status
    const response = await fetch(
      `https://api.crossref.org/works/${encodeURIComponent(item.doi!)}`,
      {
        headers: {
          'User-Agent': 'ResearchCitationManager/1.0 (mailto:api@citation-manager.dev)',
        },
      }
    );

    if (!response.ok) continue;

    const data = await response.json();
    const work = data.message;

    // Check for retraction
    if (work['update-to']?.some((u: any) => u.type === 'retraction') ||
        work['is-retracted'] === true) {
      await prisma.item.update({
        where: { id: item.id },
        data: {
          is_retracted: true,
          enrichment: {
            ...(item as any).enrichment,
            retraction: {
              detected_at: new Date().toISOString(),
              source: 'crossref',
              retraction_doi: work['update-to']?.find((u: any) => u.type === 'retraction')?.DOI,
            },
          },
        },
      });

      // TODO: Send notification to library owner
    }

    // Rate limit: 50 req/s for CrossRef Polite Pool
    await new Promise((r) => setTimeout(r, 25));
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 7.2.1 | Known retracted DOI detected by CrossRef check | Integration | is_retracted set to true, enrichment.retraction populated |
| 7.2.2 | Non-retracted DOI remains unflagged | Integration | is_retracted remains false |
| 7.2.3 | Retracted item displays warning in UI | E2E | Red "RETRACTED" badge visible on item detail view |
| 7.2.4 | Weekly cron job executes without error | Integration | Job completes, processes all items with DOIs |
| 7.2.5 | Rate limiting respects CrossRef guidelines | Unit | Requests throttled to <50/second |

---

## Phase 8: Citation Graph & Literature Discovery

**Duration:** 5 weeks
**Depends on:** Phase 7

### Definition of Done
- Graph layer (graph_nodes, graph_edges) stores citation network from library items and external sources
- Visual citation graph renders an interactive node-link diagram from seed papers
- 2-hop citation neighbourhood exploration (papers that cite and are cited by seed papers)
- OpenAlex/Semantic Scholar integration populates graph with external citation data
- Co-citation and bibliographic coupling analysis available as computed graph queries

### Task 8.1: Graph Layer Schema & Sync

**What:** Create the graph_nodes and graph_edges tables from Data Model Suggestion 4. Build a sync pipeline that mirrors library items into graph nodes and populates citation edges from OpenAlex and Semantic Scholar APIs.

**Design:**

```sql
-- packages/db/migrations/008_graph_layer.sql

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       TEXT NOT NULL,  -- 'work', 'author', 'source', 'institution', 'concept'
    external_ids    JSONB NOT NULL DEFAULT '{}',
    properties      JSONB NOT NULL DEFAULT '{}',
    display_name    TEXT,
    pagerank        REAL,
    in_degree       INTEGER DEFAULT 0,
    out_degree      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes (node_type);
CREATE INDEX idx_graph_nodes_external_ids ON graph_nodes USING GIN (external_ids jsonb_path_ops);
CREATE INDEX idx_graph_nodes_name_fts ON graph_nodes USING GIN (to_tsvector('english', COALESCE(display_name, '')));

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL,  -- 'cites', 'supports', 'disputes', 'authored_by', etc.
    properties      JSONB NOT NULL DEFAULT '{}',
    weight          REAL DEFAULT 1.0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges (source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_node_id, edge_type);
CREATE UNIQUE INDEX idx_graph_edges_unique ON graph_edges (source_node_id, target_node_id, edge_type);

-- Add graph_node_id reference to items table
ALTER TABLE items ADD COLUMN graph_node_id UUID REFERENCES graph_nodes(id);
CREATE INDEX idx_items_graph_node ON items (graph_node_id) WHERE graph_node_id IS NOT NULL;
```

```typescript
// apps/api/src/services/graph/sync-from-openalex.ts

import { prisma } from '@/lib/prisma';

export async function syncItemToGraph(itemId: string): Promise<string> {
  const item = await prisma.item.findUnique({ where: { id: itemId } });
  if (!item?.doi) throw new Error('Item has no DOI');

  // 1. Create or find graph node for this work
  let node = await prisma.graphNode.findFirst({
    where: { external_ids: { path: ['doi'], equals: item.doi } },
  });

  if (!node) {
    node = await prisma.graphNode.create({
      data: {
        node_type: 'work',
        external_ids: { doi: item.doi },
        properties: {
          title: item.title,
          pub_year: item.pub_year,
          type: item.item_type,
        },
        display_name: item.title,
      },
    });
  }

  // Link item to graph node
  await prisma.item.update({
    where: { id: itemId },
    data: { graph_node_id: node.id },
  });

  // 2. Fetch citation data from OpenAlex
  const oaResponse = await fetch(
    `https://api.openalex.org/works/doi:${item.doi}?mailto=api@citation-manager.dev`
  );

  if (oaResponse.ok) {
    const oaWork = await oaResponse.json();

    // Create edges for referenced works
    for (const ref of oaWork.referenced_works || []) {
      const refDoi = ref.replace('https://openalex.org/', '');
      let refNode = await prisma.graphNode.findFirst({
        where: { external_ids: { path: ['openalex'], equals: refDoi } },
      });

      if (!refNode) {
        refNode = await prisma.graphNode.create({
          data: {
            node_type: 'work',
            external_ids: { openalex: refDoi },
            display_name: refDoi,
          },
        });
      }

      await prisma.graphEdge.upsert({
        where: {
          source_node_id_target_node_id_edge_type: {
            source_node_id: node.id,
            target_node_id: refNode.id,
            edge_type: 'cites',
          },
        },
        create: {
          source_node_id: node.id,
          target_node_id: refNode.id,
          edge_type: 'cites',
        },
        update: {},
      });
    }

    // Update degree counts
    await prisma.$executeRaw`
      UPDATE graph_nodes SET
        out_degree = (SELECT COUNT(*) FROM graph_edges WHERE source_node_id = ${node.id}),
        in_degree = (SELECT COUNT(*) FROM graph_edges WHERE target_node_id = ${node.id})
      WHERE id = ${node.id}
    `;
  }

  return node.id;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 8.1.1 | Sync item with DOI creates graph node | Integration | graph_node created, items.graph_node_id set |
| 8.1.2 | OpenAlex citation data creates edges | Integration | graph_edges created with edge_type='cites' |
| 8.1.3 | Duplicate sync does not create duplicate nodes or edges | Integration | Upsert logic prevents duplicates |
| 8.1.4 | Degree counts updated after sync | Integration | in_degree and out_degree match actual edge counts |
| 8.1.5 | Item without DOI raises error | Integration | Error thrown with "Item has no DOI" message |

### Task 8.2: Citation Graph Visualization

**What:** Build an interactive citation graph visualization using D3.js force-directed layout. Nodes represent papers (coloured by year, sized by citation count). Edges represent citation relationships (typed by CiTO classification). Support pan, zoom, click-to-expand, and click-to-view-item.

**Design:**

```typescript
// apps/web/src/components/graph/CitationGraph.tsx

'use client';

import { useEffect, useRef, useState } from 'react';
import * as d3 from 'd3';

interface GraphNode {
  id: string;
  display_name: string;
  pub_year: number;
  in_degree: number;
  node_type: 'seed' | 'cited' | 'citing';
}

interface GraphEdge {
  source: string;
  target: string;
  edge_type: string;
}

interface CitationGraphProps {
  seedItemId: string;
  hops?: number;
}

export function CitationGraph({ seedItemId, hops = 2 }: CitationGraphProps) {
  const svgRef = useRef<SVGSVGElement>(null);
  const [graphData, setGraphData] = useState<{ nodes: GraphNode[]; edges: GraphEdge[] } | null>(null);

  useEffect(() => {
    fetch(`/api/graph/neighbourhood?item_id=${seedItemId}&hops=${hops}`)
      .then((r) => r.json())
      .then(setGraphData);
  }, [seedItemId, hops]);

  useEffect(() => {
    if (!graphData || !svgRef.current) return;

    const svg = d3.select(svgRef.current);
    const width = svgRef.current.clientWidth;
    const height = svgRef.current.clientHeight;

    // Year-based colour scale
    const colorScale = d3.scaleSequential(d3.interpolateViridis)
      .domain([1960, 2026]);

    // Node size by citation count
    const sizeScale = d3.scaleSqrt()
      .domain([0, d3.max(graphData.nodes, (n) => n.in_degree) || 100])
      .range([4, 20]);

    const simulation = d3.forceSimulation(graphData.nodes as any)
      .force('link', d3.forceLink(graphData.edges).id((d: any) => d.id).distance(80))
      .force('charge', d3.forceManyBody().strength(-200))
      .force('center', d3.forceCenter(width / 2, height / 2));

    // Render edges
    const link = svg.selectAll('.link')
      .data(graphData.edges)
      .join('line')
      .attr('class', 'link')
      .attr('stroke', (d) => d.edge_type === 'disputes' ? '#ef4444' : '#94a3b8')
      .attr('stroke-width', 1);

    // Render nodes
    const node = svg.selectAll('.node')
      .data(graphData.nodes)
      .join('circle')
      .attr('class', 'node')
      .attr('r', (d) => sizeScale(d.in_degree))
      .attr('fill', (d) => colorScale(d.pub_year))
      .attr('stroke', (d) => d.node_type === 'seed' ? '#f59e0b' : 'none')
      .attr('stroke-width', (d) => d.node_type === 'seed' ? 3 : 0)
      .call(d3.drag() as any);

    simulation.on('tick', () => {
      link
        .attr('x1', (d: any) => d.source.x)
        .attr('y1', (d: any) => d.source.y)
        .attr('x2', (d: any) => d.target.x)
        .attr('y2', (d: any) => d.target.y);
      node
        .attr('cx', (d: any) => d.x)
        .attr('cy', (d: any) => d.y);
    });
  }, [graphData]);

  return <svg ref={svgRef} className="w-full h-[600px] border rounded" />;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 8.2.1 | Graph renders seed paper as highlighted node | E2E | Seed node has amber stroke, centered in graph |
| 8.2.2 | 2-hop neighbourhood shows cited and citing papers | E2E | Multiple nodes visible connected by edges |
| 8.2.3 | Contrasting citation edges rendered in red | E2E | Edges with edge_type='disputes' are red |
| 8.2.4 | Click node shows paper details in sidebar | E2E | Title, authors, year, DOI displayed |
| 8.2.5 | Zoom and pan work via mouse/trackpad | E2E | Graph is navigable at different zoom levels |
| 8.2.6 | Node size scales with citation count | E2E | Highly cited papers have larger nodes |
| 8.2.7 | Node colour represents publication year | E2E | Recent papers are different colour from older papers |

### Task 8.3: Literature Discovery via Semantic Scholar

**What:** Integrate Semantic Scholar API for automated literature discovery. Given a seed paper or collection, find related work not yet in the library via Semantic Scholar's recommendation API. Display recommendations with the option to add directly to library.

**Design:**

```typescript
// apps/api/src/services/discovery/semantic-scholar.ts

const S2_API = 'https://api.semanticscholar.org/graph/v1';

export async function findRelatedPapers(
  doi: string,
  limit: number = 20
): Promise<Array<Record<string, unknown>>> {
  // Get paper ID from DOI
  const paperResp = await fetch(`${S2_API}/paper/DOI:${doi}?fields=paperId`);
  if (!paperResp.ok) return [];
  const paper = await paperResp.json();

  // Get recommendations
  const recResp = await fetch(
    `https://api.semanticscholar.org/recommendations/v1/papers/forpaper/${paper.paperId}?limit=${limit}&fields=title,authors,year,externalIds,abstract,citationCount`
  );
  if (!recResp.ok) return [];
  const { recommendedPapers } = await recResp.json();

  return recommendedPapers.map((p: any) => ({
    type: 'article-journal',
    title: p.title,
    author: p.authors?.map((a: any) => ({ family: a.name.split(' ').pop(), given: a.name.split(' ').slice(0, -1).join(' ') })),
    DOI: p.externalIds?.DOI,
    issued: p.year ? { 'date-parts': [[p.year]] } : undefined,
    abstract: p.abstract,
    _semantic_scholar_id: p.paperId,
    _citation_count: p.citationCount,
  }));
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 8.3.1 | Discover related papers for a known DOI | Integration | Returns 10-20 related papers with titles and authors |
| 8.3.2 | Recommendations exclude papers already in library | Integration | Library items filtered out of results |
| 8.3.3 | Add recommended paper to library | E2E | Paper added as new item with CSL-JSON metadata |
| 8.3.4 | Invalid DOI returns empty recommendations | Integration | Empty array, no error |

---

## Phase 9: Systematic Review Workflow

**Duration:** 4 weeks
**Depends on:** Phase 8

### Definition of Done
- Users can create a systematic review project with PRISMA-compatible protocol
- Title/abstract screening with include/exclude/maybe decisions per reviewer
- Full-text screening stage with the same decision workflow
- Inter-rater reliability (Cohen's kappa) calculated and displayed
- Configurable data extraction fields with per-reviewer extraction
- PRISMA flow diagram auto-generated from review decisions

### Task 9.1: Review Project Management

**What:** Implement review project CRUD with configurable inclusion/exclusion criteria, screening stages, and extraction field definitions stored in JSONB config.

**Design:**

```typescript
// apps/api/src/routes/reviews/create.ts

import { FastifyPluginAsync } from 'fastify';

const reviewsRoute: FastifyPluginAsync = async (fastify) => {
  fastify.post<{
    Body: {
      library_id: string;
      name: string;
      protocol: 'prisma' | 'cochrane' | 'custom';
      config: {
        inclusion_criteria: string[];
        exclusion_criteria: string[];
        extraction_fields: Array<{
          name: string;
          type: 'text' | 'integer' | 'number' | 'enum';
          options?: string[];
        }>;
        screening_stages: string[];
        min_reviewers_per_stage: number;
      };
    };
  }>('/reviews', {
    handler: async (request, reply) => {
      const review = await fastify.prisma.reviewProject.create({
        data: {
          library_id: request.body.library_id,
          name: request.body.name,
          protocol: request.body.protocol,
          config: request.body.config as any,
          status: 'screening',
        },
      });

      return reply.code(201).send(review);
    },
  });
};
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 9.1.1 | Create PRISMA review project with criteria | Integration | Review created with config JSONB containing criteria and fields |
| 9.1.2 | Review project status transitions (screening -> extraction -> synthesis -> complete) | Integration | Status updates allowed in correct order only |
| 9.1.3 | Review project lists items pending screening | Integration | Items not yet decided by current reviewer returned |

### Task 9.2: Screening & Decision Workflow

**What:** Build the screening interface where reviewers make include/exclude/maybe decisions on each item. Track decisions per reviewer per stage. Calculate Cohen's kappa for inter-rater reliability when multiple reviewers are assigned.

**Design:**

```typescript
// apps/api/src/services/review/reliability.ts

export function calculateCohensKappa(
  decisions1: string[],
  decisions2: string[]
): number {
  if (decisions1.length !== decisions2.length || decisions1.length === 0) return 0;

  const n = decisions1.length;
  let agreements = 0;

  const categories = [...new Set([...decisions1, ...decisions2])];
  const freq1: Record<string, number> = {};
  const freq2: Record<string, number> = {};

  for (const cat of categories) {
    freq1[cat] = 0;
    freq2[cat] = 0;
  }

  for (let i = 0; i < n; i++) {
    if (decisions1[i] === decisions2[i]) agreements++;
    freq1[decisions1[i]]++;
    freq2[decisions2[i]]++;
  }

  const po = agreements / n; // observed agreement
  let pe = 0; // expected agreement
  for (const cat of categories) {
    pe += (freq1[cat] / n) * (freq2[cat] / n);
  }

  if (pe === 1) return 1;
  return (po - pe) / (1 - pe);
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 9.2.1 | Submit screening decision (include) | Integration | review_decision created with stage, decision, reviewer_id |
| 9.2.2 | Same reviewer cannot decide same item twice in same stage | Integration | Unique constraint violation |
| 9.2.3 | Cohen's kappa = 1.0 for perfect agreement | Unit | Both reviewers agree on all items -> kappa = 1.0 |
| 9.2.4 | Cohen's kappa = 0.0 for chance-level agreement | Unit | kappa near 0 for random decisions |
| 9.2.5 | PRISMA flow diagram data generated from decisions | Integration | Counts returned: identified, screened, included, excluded by reason |
| 9.2.6 | Conflict resolution interface shows disagreements | E2E | Items where reviewers disagree highlighted for resolution |

### Task 9.3: Data Extraction

**What:** Build configurable data extraction interface where reviewers extract structured data from included papers. Extraction fields defined in review_project.config. Support AI-assisted extraction with human verification.

**Design:**

```typescript
// apps/api/src/routes/reviews/extract.ts

import { FastifyPluginAsync } from 'fastify';

const extractionRoute: FastifyPluginAsync = async (fastify) => {
  fastify.post<{
    Params: { reviewId: string; itemId: string };
    Body: { extracted_data: Record<string, unknown>; confidence: string };
  }>('/reviews/:reviewId/items/:itemId/extract', {
    handler: async (request, reply) => {
      const extraction = await fastify.prisma.dataExtraction.upsert({
        where: {
          review_id_item_id_extractor_id: {
            review_id: request.params.reviewId,
            item_id: request.params.itemId,
            extractor_id: request.user!.id,
          },
        },
        create: {
          review_id: request.params.reviewId,
          item_id: request.params.itemId,
          extractor_id: request.user!.id,
          extracted_data: request.body.extracted_data as any,
          confidence: request.body.confidence,
        },
        update: {
          extracted_data: request.body.extracted_data as any,
          confidence: request.body.confidence,
        },
      });

      return reply.send(extraction);
    },
  });
};
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 9.3.1 | Extract data with configured fields (sample_size, study_design) | Integration | Extraction stored with JSONB matching config schema |
| 9.3.2 | AI-assisted extraction pre-fills fields from PDF | Integration | Extracted data auto-populated, marked for human review |
| 9.3.3 | Export extractions as CSV | Integration | CSV with columns matching extraction_fields, one row per item |
| 9.3.4 | Compare extractions from two reviewers | Integration | Side-by-side view showing agreements and disagreements |

---

## Phase 10: Multi-Paper Synthesis & Research Memory

**Duration:** 5 weeks
**Depends on:** Phase 9

### Definition of Done
- AI reads a collection of papers and produces a structured synthesis (agreements, contradictions, gaps)
- Synthesis output stored with provenance (which papers, which model, when)
- Research memory tracks reading history and answers "have I read something about X?"
- Literature review draft generation from a collection with inline citations

### Task 10.1: Multi-Paper Synthesis Engine

**What:** Build an AI synthesis pipeline that reads abstracts and full-text from a collection of papers, identifies agreements, contradictions, and methodological gaps, and produces a structured synthesis report with citations.

**Design:**

```typescript
// apps/api/src/ai/tools/synthesise-collection.ts

import Anthropic from '@anthropic-ai/sdk';
import { prisma } from '@/lib/prisma';

const anthropic = new Anthropic();

interface SynthesisResult {
  summary: string;
  agreements: string[];
  contradictions: string[];
  gaps: string[];
  methodology_breakdown: Record<string, number>;
  sources_used: string[];
}

export async function synthesiseCollection(collectionId: string): Promise<SynthesisResult> {
  // 1. Load all items in the collection with their metadata and full-text
  const items = await prisma.collectionItem.findMany({
    where: { collection_id: collectionId },
    include: {
      item: {
        include: {
          attachments: { where: { content_type: 'application/pdf' }, select: { fulltext: true } },
        },
      },
    },
  });

  // 2. Prepare paper contexts for the AI
  const paperContexts = items.map((ci, i) => {
    const meta = ci.item.metadata as Record<string, unknown>;
    const fulltext = ci.item.attachments[0]?.fulltext;
    return `[Paper ${i + 1}] ID: ${ci.item.id}
Title: ${meta.title}
Authors: ${(meta.author as any[])?.map((a: any) => `${a.given} ${a.family}`).join(', ') || 'Unknown'}
Year: ${ci.item.pub_year}
Abstract: ${meta.abstract || 'Not available'}
${fulltext ? `Key text (first 3000 chars): ${fulltext.substring(0, 3000)}` : ''}`;
  }).join('\n\n---\n\n');

  // 3. Generate synthesis
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4000,
    messages: [
      {
        role: 'user',
        content: `You are an expert research synthesiser. Analyse the following ${items.length} academic papers and produce a structured synthesis.

Return a JSON object with:
- "summary": A 2-3 paragraph overview of the collective findings
- "agreements": Array of statements where multiple papers agree (cite by Paper number)
- "contradictions": Array of statements where papers contradict each other (cite both sides)
- "gaps": Array of research questions or areas that no paper in the collection addresses
- "methodology_breakdown": Object counting how many papers use each methodology type

Papers:
${paperContexts}`,
      },
    ],
  });

  const text = response.content[0].type === 'text' ? response.content[0].text : '{}';
  const synthesis: SynthesisResult = {
    ...JSON.parse(text),
    sources_used: items.map((ci) => ci.item.id),
  };

  // 4. Store synthesis in ai_outputs
  await prisma.aiOutput.create({
    data: {
      target_type: 'collection',
      target_id: collectionId,
      output_type: 'synthesis',
      model_id: 'claude-sonnet-4-20250514',
      model_version: 'claude-sonnet-4-20250514',
      content: synthesis as any,
      token_count: response.usage.output_tokens,
    },
  });

  return synthesis;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 10.1.1 | Synthesise collection of 5 papers | Integration | Returns synthesis with summary, agreements, contradictions, gaps |
| 10.1.2 | Synthesis cites specific papers by reference | Integration | Paper numbers in agreements/contradictions match input papers |
| 10.1.3 | Synthesis stored in ai_outputs with sources_used | Integration | ai_output row references all 5 item IDs |
| 10.1.4 | Synthesis of 20+ papers handles context window | Integration | Truncation/chunking produces coherent output within limits |
| 10.1.5 | Empty collection returns error | Integration | 400 Bad Request with "collection has no items" message |

### Task 10.2: Research Memory

**What:** Build a research memory system that tracks what the user has read, annotated, and cited. Expose an AI-powered query interface that answers questions like "have I already read something about X?" by searching reading history and library embeddings.

**Design:**

```typescript
// apps/api/src/routes/ai/memory.ts

import { FastifyPluginAsync } from 'fastify';
import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic();

const memoryRoute: FastifyPluginAsync = async (fastify) => {
  fastify.post<{
    Body: { query: string; library_id: string };
  }>('/ai/memory', {
    handler: async (request) => {
      const { query, library_id } = request.body;
      const userId = request.user!.id;

      // 1. Search reading history
      const recentlyRead = await fastify.prisma.readingHistory.findMany({
        where: { user_id: userId },
        orderBy: { created_at: 'desc' },
        take: 50,
        include: { item: { select: { title: true, metadata: true } } },
      });

      // 2. Semantic search across library
      const queryEmbedding = await generateQueryEmbedding(query);
      const semanticResults = await fastify.prisma.$queryRaw`
        SELECT i.id, i.title, i.first_author, i.pub_year,
               e.embedding <=> ${JSON.stringify(queryEmbedding)}::vector AS distance
        FROM embeddings e
        JOIN items i ON e.source_id = i.id AND e.source_type = 'item'
        WHERE i.library_id = ${library_id}::UUID
          AND e.embedding_scope = 'title_abstract'
        ORDER BY e.embedding <=> ${JSON.stringify(queryEmbedding)}::vector
        LIMIT 10
      `;

      // 3. Ask Claude to synthesise the memory response
      const response = await anthropic.messages.create({
        model: 'claude-sonnet-4-20250514',
        max_tokens: 1000,
        messages: [
          {
            role: 'user',
            content: `The researcher asks: "${query}"

Their reading history (most recent):
${recentlyRead.map((r) => `- ${r.action}: "${(r.item.metadata as any).title}" (${r.created_at.toISOString().split('T')[0]})`).join('\n')}

Most semantically relevant papers in their library:
${(semanticResults as any[]).map((r) => `- "${r.title}" by ${r.first_author} (${r.pub_year}) [relevance: ${(1 - r.distance).toFixed(2)}]`).join('\n')}

Answer their question based on what they have read and what is in their library. If they have read something relevant, mention it. If not, say so and suggest what to search for.`,
          },
        ],
      });

      const answer = response.content[0].type === 'text' ? response.content[0].text : '';

      return {
        answer,
        related_items: semanticResults,
        recently_read: recentlyRead.slice(0, 5),
      };
    },
  });
};
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 10.2.1 | "Have I read anything about citation analysis?" | Integration | Returns relevant papers from library with reading timestamps |
| 10.2.2 | Query about topic not in library | Integration | Response says "nothing found" and suggests search terms |
| 10.2.3 | Reading history records item opens | Integration | reading_history row created with action='opened' |
| 10.2.4 | Reading history records time spent | Integration | duration_seconds populated on reading completion |
| 10.2.5 | Memory search combines reading history with semantic search | Integration | Both recently read and semantically similar items included |

### Task 10.3: Literature Review Draft Generation

**What:** Generate a structured literature review draft from a collection of papers, with inline citations in the selected citation style. Output as Markdown with CSL-style citation keys that can be converted to formatted citations.

**Design:**

```typescript
// apps/api/src/ai/tools/generate-review-draft.ts

export async function generateLiteratureReviewDraft(
  collectionId: string,
  style: string = 'apa-7th-edition',
  focusQuestion?: string
): Promise<{ markdown: string; citations_used: string[] }> {
  const items = await loadCollectionItems(collectionId);

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 8000,
    messages: [
      {
        role: 'user',
        content: `Write a literature review draft based on the following ${items.length} papers.
${focusQuestion ? `Focus question: ${focusQuestion}` : ''}

Use inline citations in the format [@Paper1_ID], [@Paper2_ID] etc.
Structure the review with sections: Introduction, Thematic Analysis, Methodological Approaches, Gaps and Future Directions, Conclusion.

Papers:
${items.map((item) => formatPaperContext(item)).join('\n\n---\n\n')}`,
      },
    ],
  });

  const markdown = response.content[0].type === 'text' ? response.content[0].text : '';
  const citationKeys = [...markdown.matchAll(/@(\w+)/g)].map((m) => m[1]);

  return { markdown, citations_used: citationKeys };
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 10.3.1 | Generate review draft from 10 papers | Integration | Markdown output with sections, inline citations, 2000+ words |
| 10.3.2 | Inline citations reference actual papers from collection | Integration | All citation keys map to real item IDs |
| 10.3.3 | Generated draft includes gap identification section | Integration | "Gaps and Future Directions" section present and substantive |
| 10.3.4 | Focus question constrains the review scope | Integration | Review addresses the specific question, not just general summary |

---

## Phase 11: Group Libraries & Collaboration

**Duration:** 3 weeks
**Depends on:** Phase 10

### Definition of Done
- Users can create group/shared libraries with role-based access (owner, editor, reader)
- Invite members by email or ORCID iD
- Comment threads on annotations visible to all library members
- Activity feed shows recent changes by collaborators
- Conflict resolution for concurrent edits to the same item

### Task 11.1: Group Library Management

**What:** Implement group library creation, member invitation, and role-based access control. Extend RLS policies to allow library_members access to shared libraries.

**Design:**

```typescript
// apps/api/src/routes/libraries/members.ts

import { FastifyPluginAsync } from 'fastify';

const libraryMembersRoute: FastifyPluginAsync = async (fastify) => {
  // Invite member to library
  fastify.post<{
    Params: { libraryId: string };
    Body: { email: string; role: 'editor' | 'reader' };
  }>('/libraries/:libraryId/members', {
    handler: async (request, reply) => {
      // Verify requester is owner or admin
      const library = await fastify.prisma.library.findUnique({
        where: { id: request.params.libraryId },
      });

      if (library?.owner_user_id !== request.user!.id) {
        return reply.code(403).send({ error: 'Only library owner can invite members' });
      }

      // Find user by email
      const invitee = await fastify.prisma.user.findUnique({
        where: { email: request.body.email },
      });

      if (!invitee) {
        // TODO: Send invitation email for unregistered user
        return reply.code(404).send({ error: 'User not found. Invitation email sent.' });
      }

      const membership = await fastify.prisma.libraryMember.create({
        data: {
          library_id: request.params.libraryId,
          user_id: invitee.id,
          role: request.body.role,
        },
      });

      return reply.code(201).send(membership);
    },
  });
};
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 11.1.1 | Create group library with type='group' | Integration | Library created with owner_org_id or owner_user_id |
| 11.1.2 | Invite member by email | Integration | library_member created with specified role |
| 11.1.3 | Reader role can view items but not edit | Integration | GET succeeds, PUT returns 403 |
| 11.1.4 | Editor role can add and modify items | Integration | POST and PUT succeed |
| 11.1.5 | Non-member cannot access group library | Integration | 403 Forbidden returned |
| 11.1.6 | Owner can remove member | Integration | library_member deleted, member loses access |

### Task 11.2: Annotation Comments & Activity Feed

**What:** Add comment threads on annotations so library members can discuss highlighted passages. Build an activity feed showing recent changes by collaborators.

**Design:**

```typescript
// apps/api/src/routes/annotations/comments.ts

import { FastifyPluginAsync } from 'fastify';

const commentsRoute: FastifyPluginAsync = async (fastify) => {
  fastify.post<{
    Params: { annotationId: string };
    Body: { text: string };
  }>('/annotations/:annotationId/comments', {
    handler: async (request, reply) => {
      const comment = await fastify.prisma.annotationComment.create({
        data: {
          annotation_id: request.params.annotationId,
          user_id: request.user!.id,
          text: request.body.text,
        },
      });

      return reply.code(201).send(comment);
    },
  });
};
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 11.2.1 | Add comment to annotation | Integration | Comment created with user attribution and timestamp |
| 11.2.2 | Activity feed shows recent additions by collaborators | E2E | Feed displays "User X added 'Paper Y' 2 hours ago" |
| 11.2.3 | Activity feed excludes own actions (optional filter) | E2E | Toggle to show/hide own activity |
| 11.2.4 | Comment thread renders chronologically | E2E | Comments ordered by created_at ascending |

---

## Phase 12: MCP Server, Integrations & Enterprise Features

**Duration:** 4 weeks
**Depends on:** Phase 11

### Definition of Done
- MCP server exposes library search, item retrieval, and synthesis as tools consumable by Claude, ChatGPT, and other AI assistants
- SAML 2.0 SSO for institutional authentication (Shibboleth, InCommon)
- Institutional proxy support for full-text access via OpenURL link resolvers
- Usage analytics dashboard for library administrators
- Self-hosted Docker Compose deployment documented and tested
- API rate limiting and quota enforcement

### Task 12.1: MCP Server Implementation

**What:** Implement a Model Context Protocol server that exposes the citation manager's capabilities as tools: search library, get item metadata, summarise paper, synthesise collection, check retraction status.

**Design:**

```typescript
// apps/api/src/ai/mcp/server.ts

import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { z } from 'zod';

const server = new McpServer({
  name: 'research-citation-manager',
  version: '1.0.0',
});

// Tool: Search library
server.tool(
  'search_library',
  'Search the researcher\'s citation library by keyword or semantic query',
  {
    query: z.string().describe('Search query (natural language or keywords)'),
    library_id: z.string().uuid().describe('Library to search'),
    search_type: z.enum(['keyword', 'semantic']).default('semantic'),
    limit: z.number().max(50).default(10),
  },
  async ({ query, library_id, search_type, limit }) => {
    const results = search_type === 'semantic'
      ? await semanticSearch(query, library_id, limit)
      : await keywordSearch(query, library_id, limit);

    return {
      content: [{
        type: 'text',
        text: JSON.stringify(results, null, 2),
      }],
    };
  }
);

// Tool: Get item details
server.tool(
  'get_item',
  'Get full bibliographic metadata for a specific library item',
  {
    item_id: z.string().uuid(),
  },
  async ({ item_id }) => {
    const item = await getItem(item_id);
    return {
      content: [{
        type: 'text',
        text: JSON.stringify(item, null, 2),
      }],
    };
  }
);

// Tool: Cite items (format bibliography)
server.tool(
  'format_citation',
  'Format one or more items as a citation in a specified style',
  {
    item_ids: z.array(z.string().uuid()),
    style: z.string().default('apa-7th-edition'),
  },
  async ({ item_ids, style }) => {
    const formatted = await formatCitations(item_ids, style);
    return {
      content: [{
        type: 'text',
        text: formatted,
      }],
    };
  }
);

// Tool: Synthesise collection
server.tool(
  'synthesise_collection',
  'Generate a structured synthesis of papers in a collection',
  {
    collection_id: z.string().uuid(),
  },
  async ({ collection_id }) => {
    const synthesis = await synthesiseCollection(collection_id);
    return {
      content: [{
        type: 'text',
        text: JSON.stringify(synthesis, null, 2),
      }],
    };
  }
);
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 12.1.1 | MCP search_library tool returns results | Integration | JSON array of matching items with metadata |
| 12.1.2 | MCP get_item tool returns full CSL-JSON | Integration | Complete item metadata including enrichment |
| 12.1.3 | MCP format_citation tool produces APA citation | Integration | Correctly formatted APA bibliography entry |
| 12.1.4 | MCP synthesise_collection tool generates synthesis | Integration | Structured synthesis with agreements, contradictions, gaps |
| 12.1.5 | MCP server responds to tool listing request | Integration | All tools listed with descriptions and parameter schemas |
| 12.1.6 | Unauthorized MCP request rejected | Integration | Error response for invalid credentials |

### Task 12.2: Institutional SSO & OpenURL

**What:** Implement SAML 2.0 authentication for institutional SSO (Shibboleth, InCommon, eduGAIN). Integrate OpenURL link resolvers for institutional full-text access.

**Design:**

```typescript
// apps/api/src/plugins/saml.ts

import { FastifyPluginAsync } from 'fastify';
import { SAML } from '@node-saml/node-saml';

const samlPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.get('/auth/saml/metadata', async (request, reply) => {
    const saml = getSAMLInstance(request.query.institution as string);
    const metadata = saml.generateServiceProviderMetadata(null, null);
    reply.type('application/xml').send(metadata);
  });

  fastify.post('/auth/saml/callback', async (request, reply) => {
    const saml = getSAMLInstance(request.body.RelayState);
    const profile = await saml.validatePostResponseAsync(request.body);

    // Match or create user from SAML attributes
    const user = await findOrCreateFromSAML(profile);
    const token = generateJWT(user);

    return reply.redirect(`/auth/callback?token=${token}`);
  });
};

// OpenURL link resolver integration
export function buildOpenURL(
  resolverBase: string,
  item: Record<string, unknown>
): string {
  const params = new URLSearchParams();
  params.set('url_ver', 'Z39.88-2004');
  params.set('rft_val_fmt', 'info:ofi/fmt:kev:mtx:journal');
  if (item.DOI) params.set('rft_id', `info:doi/${item.DOI}`);
  if (item.ISSN) params.set('rft.issn', item.ISSN as string);
  if (item.title) params.set('rft.atitle', item.title as string);
  return `${resolverBase}?${params.toString()}`;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 12.2.1 | SAML metadata endpoint returns valid XML | Integration | SP metadata with correct entity ID and ACS URL |
| 12.2.2 | SAML login flow creates user from assertion | E2E | User created with institutional attributes |
| 12.2.3 | OpenURL generates valid link resolver URL | Unit | URL contains Z39.88-2004 parameters with DOI and ISSN |
| 12.2.4 | OpenURL link resolves to full-text at institution | Manual | Clicking link accesses article via institutional subscription |

### Task 12.3: Self-Hosted Deployment & Admin

**What:** Finalize Docker Compose deployment configuration for self-hosted installations. Build admin dashboard for institutional usage analytics, user management, and storage quota monitoring.

**Design:**

```yaml
# docker/docker-compose.yml

services:
  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: citation_manager
      POSTGRES_USER: citation
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - '5432:5432'

  api:
    build:
      context: ..
      dockerfile: docker/Dockerfile.api
    environment:
      DATABASE_URL: postgresql://citation:${DB_PASSWORD}@db:5432/citation_manager
      S3_ENDPOINT: http://minio:9000
      S3_BUCKET: citations
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    depends_on:
      - db
      - minio
    ports:
      - '3001:3001'

  web:
    build:
      context: ..
      dockerfile: docker/Dockerfile.web
    environment:
      NEXT_PUBLIC_API_URL: http://api:3001
    depends_on:
      - api
    ports:
      - '3000:3000'

  worker:
    build:
      context: ..
      dockerfile: docker/Dockerfile.worker
    environment:
      DATABASE_URL: postgresql://citation:${DB_PASSWORD}@db:5432/citation_manager
      REDIS_URL: redis://redis:6379
    depends_on:
      - db
      - redis

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${S3_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${S3_SECRET_KEY}
    volumes:
      - minio_data:/data
    ports:
      - '9000:9000'
      - '9001:9001'

volumes:
  pgdata:
  minio_data:
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 12.3.1 | docker-compose up starts all 6 services | Manual | All containers healthy within 60 seconds |
| 12.3.2 | Web UI accessible at localhost:3000 | Manual | Login page renders correctly |
| 12.3.3 | API health check returns 200 | Integration | GET /health returns { status: 'ok' } |
| 12.3.4 | Admin dashboard shows user count and storage usage | E2E | Dashboard displays aggregate metrics |
| 12.3.5 | API rate limiting returns 429 on excess | Integration | 429 Too Many Requests after exceeding limit |
| 12.3.6 | Database migrations run on fresh deployment | Integration | All tables created via docker-compose exec api prisma migrate deploy |

---

## Summary

| Phase | Duration | Key Deliverables |
|-------|----------|------------------|
| 1. Foundation & Auth | 3 weeks | PostgreSQL + RLS, user auth (email/Google/ORCID), CI/CD |
| 2. Library Management | 4 weeks | Item CRUD (CSL-JSON), BibTeX/RIS import/export, collections, tags, DOI lookup |
| 3. Browser Extension | 4 weeks | Chrome/Firefox extension, metadata capture from 5+ sites, PDF download |
| 4. Citation Engine | 4 weeks | citeproc-rs WASM, Word add-in, Google Docs add-on, 10K+ styles |
| 5. PDF Reader | 4 weeks | PDF.js viewer, highlight/note annotations (W3C model), text extraction |
| 6. AI Summarisation & Search | 4 weeks | Embedding pipeline, semantic search (pgvector), paper summaries |
| 7. Citation Analysis | 3 weeks | Supporting/contrasting classification, retraction monitoring |
| 8. Citation Graph | 5 weeks | Graph layer (nodes/edges), D3.js visualization, Semantic Scholar discovery |
| 9. Systematic Review | 4 weeks | PRISMA workflow, screening, inter-rater reliability, data extraction |
| 10. Synthesis & Memory | 5 weeks | Multi-paper synthesis, research memory, literature review drafts |
| 11. Collaboration | 3 weeks | Group libraries, RBAC, annotation comments, activity feed |
| 12. MCP & Enterprise | 4 weeks | MCP server, SAML SSO, OpenURL, Docker deployment, admin |
| **Total** | **~47 weeks** | |
