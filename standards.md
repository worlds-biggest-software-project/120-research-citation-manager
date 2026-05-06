# Standards & API Reference

> Project: Research & Citation Manager · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

- **ISO 26324:2022 — Digital Object Identifier (DOI) System**
  - URL: https://www.iso.org/standard/81599.html
  - Defines the syntax, description, and resolution functional components of the DOI system; foundation for persistent identification of journal articles, datasets, and other research outputs.

- **ISO 690:2021 — Information and documentation — Guidelines for bibliographic references and citations to information resources**
  - URL: https://www.iso.org/standard/72642.html
  - International standard governing how bibliographic references should be constructed; influential alongside CSL-based style implementations.

- **ISO 27729:2012 — International Standard Name Identifier (ISNI)**
  - URL: https://www.iso.org/standard/44292.html
  - Identifier for the public identities of contributors to media content; complements ORCID for non-academic contributors.

- **ISO 3297:2020 — International Standard Serial Number (ISSN)**
  - URL: https://www.iso.org/standard/73846.html
  - Eight-digit serial identifier for periodical publications; required metadata for journal-level citation records.

- **ISO 2108:2017 — International Standard Book Number (ISBN)**
  - URL: https://www.iso.org/standard/65483.html
  - Identifier scheme for monographic publications; core field in book/chapter citations.

- **ISO/IEC 27001:2022 — Information security management systems**
  - URL: https://www.iso.org/standard/27001
  - Relevant for institutional deployments handling unpublished manuscripts, grant data, and personally identifiable researcher information.

### W3C & IETF Standards

- **W3C PROV-O — Provenance Ontology**
  - URL: https://www.w3.org/TR/prov-o/
  - OWL2 ontology for expressing provenance information; useful for tracking how a citation entered the library, who edited it, and which AI agent generated a synthesis.

- **W3C Web Annotation Data Model**
  - URL: https://www.w3.org/TR/annotation-model/
  - Standard data model for annotations on web/PDF resources; enables interoperable highlights and notes across reference managers.

- **W3C Linked Data Platform 1.0**
  - URL: https://www.w3.org/TR/ldp/
  - Specification for HTTP-based read/write Linked Data; relevant for exposing libraries as queryable RDF.

- **RFC 9110 — HTTP Semantics**
  - URL: https://datatracker.ietf.org/doc/html/rfc9110
  - Foundational HTTP semantics used by all reference manager APIs.

- **RFC 8288 — Web Linking**
  - URL: https://datatracker.ietf.org/doc/html/rfc8288
  - Defines `Link` header and link relations; used by CrossRef, DataCite, and OpenURL link resolvers.

- **RFC 7519 — JSON Web Token (JWT)**
  - URL: https://datatracker.ietf.org/doc/html/rfc7519
  - Token format used by most modern citation APIs for authentication.

- **RFC 6749 — OAuth 2.0 Authorization Framework**
  - URL: https://datatracker.ietf.org/doc/html/rfc6749
  - Used by Mendeley, Zotero, ORCID, and CrossRef Event Data APIs.

- **NISO Z39.88-2004 — OpenURL Framework for Context-Sensitive Services**
  - URL: https://www.niso.org/publications/z3988-2004-r2010
  - Standard library link resolver protocol; required for connecting citation records to institutional full-text holdings.

### Data Model & API Specifications

- **Citation Style Language (CSL) 1.0.2**
  - URL: https://docs.citationstyles.org/en/stable/specification.html
  - Open XML-based language defining citation/bibliography rendering rules; powers 10,000+ citation styles across Zotero, Mendeley, Paperpile.

- **CSL-JSON — Item schema**
  - URL: https://github.com/citation-style-language/schema
  - Canonical JSON representation for bibliographic items consumed by CSL processors (citeproc-js, citeproc-rs).

- **BibTeX / BibLaTeX**
  - URL: https://www.bibtex.org/Format/ ; https://ctan.org/pkg/biblatex
  - De facto plain-text bibliography format in the LaTeX ecosystem; required import/export format.

- **RIS Format Specification**
  - URL: https://en.wikipedia.org/wiki/RIS_(file_format) (canonical Clarivate spec: https://web.archive.org/web/refman_ris/)
  - Tagged plain-text format supported by Web of Science, Scopus, IEEE Xplore, PubMed; near-universal interchange format.

- **MODS — Metadata Object Description Schema (Library of Congress)**
  - URL: https://www.loc.gov/standards/mods/
  - XML schema for bibliographic data; richer than Dublin Core, simpler than full MARC.

- **Dublin Core Metadata Terms (DCMI)**
  - URL: https://www.dublincore.org/specifications/dublin-core/dcmi-terms/
  - Lightweight metadata vocabulary widely supported in repository and OAI-PMH exchange.

- **OAI-PMH 2.0 — Open Archives Initiative Protocol for Metadata Harvesting**
  - URL: https://www.openarchives.org/OAI/openarchivesprotocol.html
  - Standard for harvesting metadata records from repositories such as arXiv, PubMed Central, institutional repositories.

- **MARC 21 / MARCXML**
  - URL: https://www.loc.gov/marc/
  - Library cataloguing standard relevant for integration with institutional library systems.

- **OpenAPI Specification 3.1**
  - URL: https://spec.openapis.org/oas/v3.1.0
  - Standard for describing REST APIs; the format CrossRef, DataCite, and most modern citation services publish documentation in.

- **JSON Schema 2020-12**
  - URL: https://json-schema.org/specification.html
  - Used to validate CSL-JSON, CrossRef metadata, and DataCite metadata.

- **Schema.org / ScholarlyArticle**
  - URL: https://schema.org/ScholarlyArticle
  - Vocabulary used in publisher landing pages enabling structured extraction of citation metadata.

- **JATS — Journal Article Tag Suite (NISO Z39.96)**
  - URL: https://jats.nlm.nih.gov/
  - XML model for journal articles; underpins PubMed Central and most publisher full-text APIs.

### Security & Authentication Standards

- **OAuth 2.0 (RFC 6749) and OAuth 2.1 draft**
  - URL: https://oauth.net/2.1/
  - Required to access user-scoped APIs from Zotero, Mendeley, ORCID, Google Drive integrations.

- **OpenID Connect 1.0**
  - URL: https://openid.net/specs/openid-connect-core-1_0.html
  - Used by ORCID and many institutional SSO deployments via SAML/OIDC bridges.

- **SAML 2.0**
  - URL: https://docs.oasis-open.org/security/saml/v2.0/
  - Predominant federated SSO protocol in higher education (Shibboleth, InCommon, eduGAIN).

- **OWASP Application Security Verification Standard (ASVS) 4.0**
  - URL: https://owasp.org/www-project-application-security-verification-standard/
  - Security verification baseline for web applications handling research data.

- **NIST SP 800-63-3 — Digital Identity Guidelines**
  - URL: https://pages.nist.gov/800-63-3/
  - Reference for assurance levels relevant to research integrity workflows.

- **GDPR (Regulation EU 2016/679)**
  - URL: https://gdpr.eu/
  - Applies to EU researcher accounts, shared libraries, annotations; relevant for any cloud-hosted reference manager.

### MCP Server Specifications

- **Model Context Protocol (MCP) Specification**
  - URL: https://spec.modelcontextprotocol.io/
  - Open protocol enabling AI assistants to interact with external tools and data; an AI-native citation manager could expose its library, search, and synthesis capabilities as an MCP server consumable by Claude, ChatGPT, and other agentic clients.

- **MCP Reference Servers (Anthropic)**
  - URL: https://github.com/modelcontextprotocol/servers
  - Reference implementations including filesystem, fetch, and database servers — useful patterns for a citation MCP server.

## Similar Products — Developer Documentation & APIs

### Zotero

- **Description:** Open-source reference manager with browser connector, group libraries, and word processor plugins.
- **API Documentation:** https://www.zotero.org/support/dev/web_api/v3/start
- **SDKs/Libraries:** Pyzotero (Python) https://github.com/urschrei/pyzotero ; libZotero (PHP); community JS clients
- **Developer Guide:** https://www.zotero.org/support/dev/start
- **Standards:** REST/JSON, Atom; CSL-JSON for items; BibTeX/RIS export
- **Authentication:** API Key (user-generated) and OAuth 1.0a

### Mendeley (Elsevier)

- **Description:** Reference manager with PDF annotation, Word integration, and research social features.
- **API Documentation:** https://dev.mendeley.com/
- **SDKs/Libraries:** Python SDK (community); JavaScript SDK (https://github.com/Mendeley/mendeley-javascript-sdk)
- **Developer Guide:** https://dev.mendeley.com/code/index.html
- **Standards:** REST/JSON, OpenAPI-described
- **Authentication:** OAuth 2.0

### CrossRef

- **Description:** Primary DOI registration agency for scholarly content; provides metadata for ~140 million records.
- **API Documentation:** https://api.crossref.org/swagger-ui/index.html ; https://www.crossref.org/documentation/retrieve-metadata/rest-api/
- **SDKs/Libraries:** habanero (Python) https://github.com/sckott/habanero ; rcrossref (R); crossref-commons (JS)
- **Developer Guide:** https://www.crossref.org/documentation/retrieve-metadata/
- **Standards:** REST/JSON, JSON-LD, OpenAPI
- **Authentication:** Public (Polite Pool requires `mailto`); Plus service uses API token

### DataCite

- **Description:** DOI registration and metadata service for research datasets, software, and other non-article outputs.
- **API Documentation:** https://support.datacite.org/reference/introduction
- **SDKs/Libraries:** datacite (Python) https://github.com/inveniosoftware/datacite ; commons clients
- **Developer Guide:** https://support.datacite.org/docs/api
- **Standards:** REST/JSON:API; DataCite Metadata Schema 4.5 (XML)
- **Authentication:** HTTP Basic for write; public reads

### ORCID

- **Description:** Open researcher identifier and authentication system.
- **API Documentation:** https://info.orcid.org/documentation/api-tutorials/
- **SDKs/Libraries:** Multiple community SDKs (Python, Java, JS); official Postman collections
- **Developer Guide:** https://info.orcid.org/documentation/integration-guide/
- **Standards:** REST/JSON and XML; OpenID Connect Provider; OAuth 2.0
- **Authentication:** OAuth 2.0 (Public, Member APIs)

### Semantic Scholar (Allen Institute for AI)

- **Description:** Free AI-powered academic search engine with citation graph and paper embeddings.
- **API Documentation:** https://api.semanticscholar.org/api-docs/
- **SDKs/Libraries:** semanticscholar (Python) https://github.com/danielnsilva/semanticscholar
- **Developer Guide:** https://www.semanticscholar.org/product/api
- **Standards:** REST/JSON, OpenAPI 3
- **Authentication:** Public; API Key for higher rate limits

### OpenAlex

- **Description:** Free open catalogue of scholarly papers, authors, institutions, concepts, venues; CC0 successor to Microsoft Academic Graph.
- **API Documentation:** https://docs.openalex.org/
- **SDKs/Libraries:** pyalex (Python) https://github.com/J535D165/pyalex ; openalexR
- **Developer Guide:** https://docs.openalex.org/how-to-use-the-api/api-overview
- **Standards:** REST/JSON
- **Authentication:** Public (Polite Pool via `mailto`); Premium API for higher throughput

### PubMed / NCBI E-utilities

- **Description:** US National Library of Medicine API for biomedical literature search and retrieval.
- **API Documentation:** https://www.ncbi.nlm.nih.gov/books/NBK25501/
- **SDKs/Libraries:** Biopython Entrez module; pubmed-parser (Python)
- **Developer Guide:** https://www.ncbi.nlm.nih.gov/books/NBK25497/
- **Standards:** REST; XML and JSON responses; JATS XML for full text
- **Authentication:** Public; API Key for higher rate limits

### arXiv

- **Description:** Open-access preprint repository covering physics, math, CS, biology, economics.
- **API Documentation:** https://info.arxiv.org/help/api/index.html
- **SDKs/Libraries:** arxiv (Python) https://github.com/lukasschwab/arxiv.py
- **Developer Guide:** https://info.arxiv.org/help/api/user-manual.html
- **Standards:** REST returning Atom XML; OAI-PMH endpoint also available
- **Authentication:** Public

### Scite.ai

- **Description:** AI-powered citation analysis showing supporting, mentioning, and contrasting citations.
- **API Documentation:** https://api.scite.ai/docs
- **SDKs/Libraries:** Community Python wrappers
- **Developer Guide:** https://help.scite.ai/en-us/category/api-1aiacng/
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** API Key (Bearer token)

### Paperpile

- **Description:** Browser-based reference manager with Google Docs integration and AI summarization.
- **API Documentation:** Limited public API; primarily integrates via Google Docs Add-on
- **SDKs/Libraries:** N/A (closed)
- **Developer Guide:** https://paperpile.com/h/api/
- **Standards:** BibTeX/CSL-JSON export
- **Authentication:** Google OAuth

### CSL Citeproc (citeproc-js / citeproc-rs)

- **Description:** Open-source reference implementations for rendering citations from CSL styles.
- **API Documentation:** https://citeproc-js.readthedocs.io/ ; https://github.com/zotero/citeproc-rs
- **SDKs/Libraries:** citeproc-js (JavaScript), citeproc-rs (Rust + WASM bindings)
- **Developer Guide:** https://citeproc-js.readthedocs.io/en/latest/running.html
- **Standards:** CSL 1.0.2, CSL-JSON
- **Authentication:** N/A (library)

### Unpaywall

- **Description:** Open database of free, legal full-text scholarly article links resolved by DOI.
- **API Documentation:** https://unpaywall.org/products/api
- **SDKs/Libraries:** unpywall (Python); roadoi (R)
- **Developer Guide:** https://unpaywall.org/data
- **Standards:** REST/JSON
- **Authentication:** Public (requires `email` query parameter)

## Notes

- The bibliographic ecosystem is unusually well-served by open standards (CSL, BibTeX, RIS, OAI-PMH, DOI, ORCID); a new AI-native citation manager has minimal lock-in risk if it adopts CSL-JSON as its canonical internal item model.
- The MCP server angle is an emerging opportunity: no major reference manager yet exposes an MCP-compliant server, which would allow AI assistants to query a researcher's library, request synthesis, or cite from it directly within other agentic workflows.
- Rate limits and Polite Pool conventions (CrossRef, OpenAlex, Unpaywall) require an identifying email/User-Agent in API calls — important to design into client architecture from day one.
- Full-text mining beyond abstracts depends on publisher-specific APIs (Elsevier TDM, Springer Nature, Wiley TDM) that have heterogeneous licensing terms; these were not catalogued here and warrant a follow-up investigation if downstream features rely on full-text access.
