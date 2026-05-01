# Research & Citation Manager — Feature & Functionality Survey

> Candidate #120 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Zotero | Open-source reference manager with browser connector and word processor plugins | OSS — AGPL-3.0 | zotero.org |
| Mendeley (Elsevier) | Reference manager + PDF annotation + research social network | Proprietary Freemium | mendeley.com |
| EndNote (Clarivate) | Desktop/cloud reference manager with manuscript matching | Proprietary Subscription | endnote.com |
| Paperpile | Browser-based reference manager with Google Docs integration | Proprietary Subscription | paperpile.com |
| ReadCube Papers | Reference manager with smart recommendations and publisher access | Proprietary Subscription | papersapp.com |
| Scite | AI citation analysis: supportive vs. contrasting vs. mentioning citations | Proprietary SaaS | scite.ai |
| Elicit | AI literature review assistant: semantic search, data extraction, synthesis | Proprietary SaaS | elicit.com |
| Semantic Scholar | Free AI-powered academic search with citation graphs | Free (AI2) | semanticscholar.org |
| Connected Papers | Graph visualisation of related papers via citation relationships | Proprietary Freemium | connectedpapers.com |
| Research Rabbit | AI-powered literature discovery with collection-based snowballing | Free | researchrabbitapp.com |

## Feature Analysis by Solution

### Zotero

**Core features**
- Browser connector (Chrome, Firefox, Safari, Edge) capturing metadata, PDFs, and full-text from 1,000+ databases and websites in a single click
- In-library PDF reader with annotation (highlights, sticky notes, image snapshots)
- Word processor plugins for Word, LibreOffice, and Google Docs with live bibliography generation
- Citation style support: CSL-based renderer covering 10,000+ citation styles (APA, MLA, Chicago, Vancouver, etc.)
- Group libraries: shared collections with role-based access (owner, admin, member, read-only)
- Zotero Sync: 300 MB free cloud storage for attachments; paid tiers from $20/year
- RIS, BibTeX, and COinS export for interoperability

**Differentiating features**
- Only major reference manager with fully open-source client and server infrastructure
- Browser connector quality is consistently rated best-in-class; captures metadata from the widest range of sites
- Group libraries with public or private sharing enable collaborative research without institutional licences
- Zotero 8 (2026) introduced a redesigned reader with improved annotation syncing and mobile PDF reader

**UX patterns**
- Three-panel desktop app: collections (left), item list (centre), item detail/reader (right)
- Drag-and-drop organisation into nested collections and sub-collections
- Quick search across all fields; advanced search with saved search criteria
- Right-click context menus for most common actions (Add to Collection, Find Full Text, Duplicate Item)

**Integration points**
- Word, LibreOffice, Google Docs via word processor plugins
- Browser: Chrome, Firefox, Safari, Edge via Zotero Connector
- BibTeX/BibLaTeX export for LaTeX/Overleaf workflows
- REST API (Zotero API) for third-party application integration
- CrossRef DOI lookup for auto-population of metadata
- ORCID, arXiv, PubMed, PubChem, and 100+ database translators

**Known gaps**
- No native AI synthesis, semantic search, or literature review assistance
- Collaborative editing of shared notes requires workarounds; real-time co-editing is absent
- Mobile apps (iOS, Android) are available but significantly less capable than the desktop client
- Full-text search within PDFs requires local indexing and is slower than cloud-native alternatives

**Licence / IP notes**
- AGPL-3.0 (client application). The AGPL-3.0 is the strongest copyleft licence: any modified version of Zotero served over a network (web app or API) must have its source code published under AGPL-3.0. Building a commercial SaaS that embeds or forks Zotero's client code without releasing all modifications is not permissible. Organisations wishing to integrate Zotero features in a proprietary product should use the Zotero API (which does not create copyleft obligations for the calling application) or obtain explicit written permission from the Corporation for Digital Scholarship (Zotero's steward). The CSL library that Zotero uses is licensed separately under Creative Commons Attribution-ShareAlike for styles.

---

### Mendeley (Elsevier)

**Core features**
- Desktop reference manager (Windows, macOS, Linux) and web interface
- PDF annotation: highlights, sticky notes, and free-text comments synced across devices
- Word and LibreOffice plugins for in-document citation insertion and bibliography generation
- Mendeley Data: integrated research data repository for dataset deposit and citation
- Research social network: follow other researchers, join topic groups, discover papers by field
- Institutional access management: libraries can provision seats and monitor usage
- Suggests related papers based on the user's library

**Differentiating features**
- Mendeley Data integration connects reference management to dataset citation in a single workflow
- Institutional access analytics: library administrators can see which journals and resources are accessed most
- Social research network (now significantly reduced after API shutdown) was an early differentiator in community building

**UX patterns**
- Three-panel layout similar to Zotero; consistent with desktop reference manager conventions
- Unified search across library, annotations, and Mendeley's broader academic content index
- "Suggest" panel offering related paper recommendations based on current library contents

**Integration points**
- Word, LibreOffice plugins
- Mendeley API for third-party integrations (limited public access post-2023 API changes)
- BibTeX, RIS, EndNote XML export
- Elsevier journal submission integration (ScienceDirect, Scopus)
- ORCID integration for researcher profile

**Known gaps**
- Elsevier ownership creates privacy concerns: library data (what papers a researcher reads and annotates) informs Elsevier's commercial intelligence
- Social network features have been progressively reduced after the 2022 API shutdown
- No AI synthesis, semantic search, or literature review assistance
- Desktop client has seen slow development relative to competitors since Elsevier acquisition

**Licence / IP notes**
- Proprietary; acquired by Elsevier (RELX Group) for ~$80–100M (2013); free with limited cloud storage; institutional plans available; researcher library data is used by Elsevier for product analytics per privacy policy — institutions should review data governance implications

---

### EndNote (Clarivate)

**Core features**
- Desktop reference manager with cloud sync (EndNote 21 + EndNote Web)
- Manuscript Matcher: recommends journals for a manuscript based on title, abstract, and references
- Smart Groups: auto-population of groups based on saved search criteria
- Reference sharing: read-only or read-write shared libraries between collaborators
- PDF import with automatic metadata retrieval
- Support for 7,000+ citation styles plus custom style editor
- PubMed, Web of Science, and database direct search from within the application

**Differentiating features**
- Manuscript Matcher for journal recommendation is unique among desktop reference managers
- Deepest integration with Web of Science (both Clarivate products): one-click import with full metadata from WoS search results
- Longest-established commercial product (founded 1988); widest adoption in biomedical and life sciences

**UX patterns**
- Desktop-primary (Windows/macOS); web interface has fewer features
- Library panel with configurable column display; sort by author, year, journal, rating
- Cite While You Write: real-time citation insertion in Word without switching applications

**Integration points**
- Word and LibreOffice via Cite While You Write plugin
- Web of Science, PubMed, Scopus, and 4,000+ database connectors
- BibTeX, RIS, MEDLINE export
- API limited to institutional partners; no public REST API

**Known gaps**
- High price (~$275 one-time or ~$170/year subscription); significantly more expensive than alternatives
- Desktop-primary architecture feels dated compared to cloud-native alternatives
- No AI-powered synthesis, semantic search, or literature discovery
- Collaboration features require both users to have a paid licence; no guest access

**Licence / IP notes**
- Proprietary; owned by Clarivate (which acquired ProQuest for $5.3B in 2021); Web of Science and EndNote are both Clarivate products enabling cross-sell; standard per-user licence; institutional volume licensing available

---

### Paperpile

**Core features**
- Browser-based reference manager optimised for Google Workspace (Chrome extension)
- One-click capture from Google Scholar, PubMed, arXiv, Semantic Scholar, and 2,000+ sources
- Native Google Docs integration: inline citation insertion without leaving the browser
- PDF reader with annotation synced to cloud
- Microsoft Word add-in (Windows; macOS in development as of 2026)
- AI paper summary: generates a plain-language abstract summary of any PDF in the library
- BibTeX, RIS, APA, MLA, Chicago export; supports all CSL styles

**Differentiating features**
- Deepest native Google Docs integration of any reference manager; citation insertion is seamless for Google Workspace users
- AI summary (PaperAI) integrated directly in the PDF reader — one click to summarise the current paper
- Fast, responsive browser-native UI; no desktop app download required

**UX patterns**
- Web app with familiar three-panel layout adapted for browser navigation
- Label and folder system for collection organisation
- Bulk import via DOI list, BibTeX file, or PDF drag-and-drop
- In-reader annotation panel that does not obscure the PDF

**Integration points**
- Google Docs native add-on
- Microsoft Word add-in
- Chrome extension for one-click capture
- Overleaf (LaTeX) via BibTeX sync
- REST API (beta) for programmatic access

**Known gaps**
- Microsoft Word support is less mature than Google Docs; macOS Word support lagging
- Group library and collaboration features are less fully developed than Zotero
- AI features are limited to summarisation; no synthesis across multiple papers, semantic search, or literature gap identification

**Licence / IP notes**
- Proprietary; US company; GDPR compliant; library data stored in Google Cloud; standard privacy policy; no institutional licensing model as of 2026

---

### ReadCube Papers

**Core features**
- Smart PDF reader with automatic metadata enrichment and citation linking within the text
- Enhanced PDF: clickable in-text references navigate to full reference record
- Smart Recommendations: daily paper suggestions based on library contents
- Word and Google Docs citation plugins
- List sharing: collaborative shared reading lists for teams and labs
- Publisher access: direct access to licensed content via institutional resolver
- Altmetric score display for each paper

**Differentiating features**
- Enhanced PDF is genuinely unique: every in-text citation in a downloaded PDF becomes clickable within the reader
- Altmetric integration provides real-time social and news impact data alongside bibliometric data
- Smart Recommendations system is more proactive than most competitors; sends a daily email digest

**UX patterns**
- Dark-mode-optional reading interface designed for long sessions
- Papers suggest similar articles at the end of each paper ("Related Articles")
- Label and list organisation similar to Zotero collections

**Integration points**
- Word and Google Docs plugins
- Institutional resolver (OpenURL) for full-text access
- CrossRef, PubMed, arXiv import
- RIS, BibTeX export

**Known gaps**
- No AI synthesis, systematic review tooling, or structured data extraction
- Group collaboration features are less robust than Zotero shared libraries
- Pricing is higher than Paperpile for equivalent storage

**Licence / IP notes**
- Proprietary; Digital Science (Holtzbrinck Publishing Group subsidiary); standard subscription licence; institutional deals available

---

### Scite

**Core features**
- Citation context analysis: classifies each citation as supporting, contrasting, or mentioning the cited claim
- Smart Citations: shows how many papers support or contradict each paper in the user's search results
- Scite Assistant: AI Q&A interface that answers research questions with citations from the literature
- Scite Reference Check: scans a manuscript bibliography and flags retracted or heavily-contested papers
- Dashboard: custom alerts for new citations to papers or authors the researcher follows
- Integration plugin for reference managers (Zotero, Mendeley, Google Scholar)

**Differentiating features**
- Only platform in market that classifies the rhetorical function of every citation at scale (supporting vs. contrasting)
- Reference Check is uniquely valuable for manuscript preparation and peer review
- Scite Assistant grounds AI-generated answers in actual cited papers rather than LLM parametric knowledge, reducing hallucination

**UX patterns**
- Search interface returning papers with Smart Citation counts prominently displayed
- Citation drill-down: click "contrasting" count to read the actual sentences in which the paper was contradicted
- Browser extension showing Smart Citation badges on Google Scholar, PubMed, and other search results

**Integration points**
- Zotero plugin: Smart Citation data appears in Zotero item detail panel
- Mendeley plugin
- Google Scholar overlay via browser extension
- API for institutional and publisher integrations
- CrossRef DOI resolution

**Known gaps**
- Not a full reference manager; does not manage a personal library, generate bibliographies, or insert citations into documents
- Coverage is weighted toward English-language journals indexed by CrossRef; limited for humanities, grey literature, and non-English sources
- Subscription required for full Smart Citation data; free tier is limited

**Licence / IP notes**
- Proprietary; raised $4.6M seed (2021); ~150,000 users by 2024; standard SaaS terms; citation data aggregated from CrossRef under CC licence; institutional API licences available

---

### Elicit

**Core features**
- Semantic literature search: finds relevant papers from a plain-language research question
- Data extraction: reads PDFs and extracts structured data (participant count, outcomes, methods) into a table
- Literature synthesis table: compares findings across multiple papers in a structured format
- Concept taxonomy: groups papers by methodology, population, or finding for systematic review
- PDF upload for analysis of papers not in Elicit's database
- Citation export to Zotero, BibTeX, and RIS

**Differentiating features**
- Automated structured data extraction from papers is the strongest in market; rivals manual extraction for systematic reviews
- Semantic search quality is high for empirical social science and medicine; finds relevant papers keyword search misses
- Table view comparing papers across extracted attributes directly addresses the most time-consuming step in systematic review

**UX patterns**
- Query input box → results list with automated abstract summaries → add to table for extraction
- Column-based extraction table where each column is an attribute the researcher wants to extract
- Paper grouping by concept to identify thematic clusters without reading each paper

**Integration points**
- Export to Zotero, BibTeX, RIS, CSV
- PDF upload for private literature
- API access (beta) for institutional integrations

**Known gaps**
- Coverage is restricted to Semantic Scholar's corpus; papers not indexed may be missed
- Data extraction accuracy varies by field; biomedical and social science perform better than humanities or law
- Not a full reference manager; does not manage a personal library or generate in-document citations
- AI-extracted data requires human verification before use in systematic reviews

**Licence / IP notes**
- Proprietary; raised $9.4M Series A (2023, Mosaic Ventures); GDPR compliant; uploaded PDFs are processed but not retained beyond the session per privacy policy; AI outputs are not guaranteed to be accurate and require researcher validation

---

### Semantic Scholar

**Core features**
- Free academic search engine covering 200M+ papers across all disciplines
- AI-generated paper summaries using TLDR model (one-sentence abstract)
- Citation graph: view papers that cite and are cited by any paper
- Highly Influential Citations filter: identifies citations where a paper changed the direction of the citing work
- Author disambiguation and profile pages with publication lists and citation counts
- Semantic Reader: enhanced PDF reader with citation popups and author info inline

**Differentiating features**
- Free with no paywall; all features accessible without subscription
- Open Academic Graph data is available for download enabling academic research on citation patterns
- Semantic Reader's inline citation preview is the best in-PDF navigation experience among free tools
- APIs are publicly documented and free for academic use up to rate limits

**UX patterns**
- Search result card shows paper, TLDR summary, citation count, and Highly Influential Citations indicator
- Paper page aggregates abstract, citations, references, related papers, and author information in a single view
- Alerts: email notification when a followed author or paper receives a new citation

**Integration points**
- Public REST API for paper search, author lookup, and citation data (free, rate-limited)
- Zotero integration via browser capture
- Bulk dataset download (Open Research Corpus) for institutional analytics
- Connected Papers and Research Rabbit use Semantic Scholar as their underlying data source

**Known gaps**
- Not a reference manager; no library management, annotation, or word processor integration
- Metadata quality varies; newer and preprint papers sometimes have incomplete bibliographic data
- TLDR summaries are one sentence; full synthesis requires Elicit or manual work

**Licence / IP notes**
- Free service run by the Allen Institute for AI (AI2), a non-profit; API terms allow academic use; bulk data available under Open Data Commons Attribution Licence (ODC-By); commercial use of bulk data requires a separate agreement with AI2

---

### Connected Papers

**Core features**
- Graph visualisation of academic papers: nodes are papers, edges are shared citation relationships
- Graph generated from any seed paper via DOI, arXiv ID, or search
- Prior Works and Derivative Works views: see what a paper builds on and what has built on it
- Export graph papers to Zotero, Mendeley, or BibTeX
- Graph history: save and revisit graphs from previous research sessions (paid)
- Concept cluster identification through visual graph density

**Differentiating features**
- Visual exploration of a literature landscape is uniquely intuitive for identifying key nodes and gaps
- Prior Works view surfaces foundational papers that seed-paper authors themselves cited most; faster than manual backward citation chaining
- The visual format is accessible to early-stage researchers unfamiliar with bibliometric analysis

**UX patterns**
- Graph canvas: zoom, pan, click nodes to read abstracts without leaving the graph
- Sidebar listing all papers in the graph with sort by year, citations, or similarity
- Colour coding: nodes coloured by publication year; size by citation count

**Integration points**
- Zotero, Mendeley, RefWorks export
- BibTeX export
- Semantic Scholar as the underlying graph data source

**Known gaps**
- Not a reference manager; no library, annotation, citation insertion, or bibliography generation
- Graph limited to one seed paper at a time; multi-paper graphs not supported
- Five free graphs per month (paid tier removes limits)
- Graph is static at generation time; does not update as new papers are published

**Licence / IP notes**
- Proprietary; Israeli company; free tier with usage limits; Pro plan ~$6/month; Semantic Scholar API data used under academic terms

---

### Research Rabbit

**Core features**
- Collection-based literature discovery: add papers to a collection; platform suggests related works
- Snowballing: automated forward and backward citation chaining from seed papers
- Visualisation: network graph of paper and author relationships within a collection
- Author-based discovery: find all papers by an author and see their collaborator network
- Alerts: notification when a new paper is added to Semantic Scholar that matches a collection
- Export to Zotero (native integration), BibTeX, and RIS

**Differentiating features**
- Collection-first approach (not single-paper-first like Connected Papers) is more suitable for researchers with an existing reading list
- Author network visualisation finds collaborative clusters that purely citation-based graphs miss
- New paper alerts are automated and ongoing; the platform monitors for new publications matching research interests
- Completely free with no usage limits

**UX patterns**
- Collection panel + graph canvas dual view
- Drag papers between collections; graph updates in real time
- Timeline view: sort papers chronologically to trace the evolution of a research area

**Integration points**
- Zotero: native two-way integration (add papers from Research Rabbit directly to a Zotero library)
- BibTeX, RIS export
- Semantic Scholar as the underlying data source

**Known gaps**
- Not a full reference manager; no annotation, word processor plugin, or bibliography generation
- Limited to Semantic Scholar corpus; coverage gaps for humanities and grey literature
- No systematic review tooling (screening, PRISMA workflow) or structured data extraction

**Licence / IP notes**
- Free; venture-backed company; standard privacy policy; no institutional tier; data sourced from Semantic Scholar API

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Import references from DOI, RIS, BibTeX, and direct database connectors (PubMed, arXiv, Web of Science, Scopus)
- In-library PDF storage with annotation (highlights, notes)
- Word processor plugin for citation insertion in Word and/or Google Docs
- Citation style rendering covering major styles (APA, MLA, Chicago, Vancouver) via CSL
- Collection/folder organisation with search across metadata and full text
- BibTeX and RIS export for interoperability

### Differentiating Features
- Supportive vs. contrasting citation classification (Scite — unique in market)
- Automated structured data extraction from papers into tables for systematic review (Elicit)
- Visual citation graph exploration for literature mapping (Connected Papers, Research Rabbit)
- Continuous new-paper alerts for saved research topics (Research Rabbit, Semantic Scholar, ReadCube)
- Manuscript journal matching (EndNote Manuscript Matcher)
- AI paper summarisation at the PDF level (Paperpile PaperAI)
- Enhanced PDF with clickable in-text citations (ReadCube Papers)
- Full AGPL-3.0 open-source client with self-hosting capability (Zotero)

### Underserved Areas / Opportunities
- Multi-paper synthesis: no single product reads an entire library and produces a structured literature review draft with gap identification
- Citation reliability scoring integrated into the reference manager workflow (Scite does this well but is not a reference manager)
- Research project memory: tracking what the researcher has read, argued, and cited across multiple projects and papers over years
- Retraction monitoring with automatic library flagging when a stored paper is retracted or subject to an expression of concern
- Systematic review workflow integration: PRISMA-compliant screening, PICO extraction, and risk-of-bias forms embedded in a reference manager (Rayyan partially fills this but is not a reference manager)
- Grey literature and non-English source coverage is a gap across all AI-powered discovery tools

### AI-Augmentation Candidates
- Semantic literature discovery answering plain-language research questions beyond keyword search
- Automated multi-paper synthesis identifying agreements, contradictions, and methodological gaps across a full library
- Citation reliability and retraction monitoring integrated into in-document citation management
- Research project memory: persistent AI assistant that knows everything the researcher has read and can answer "have I already seen something about X?"
- Auto-formatted bibliography edge-case handling for novel source types (datasets, software, social media, preprints) that break template-based CSL formatters

## Legal & IP Summary

- **Zotero is AGPL-3.0**: the copyleft obligation applies to the network-served application. A commercial product that calls the Zotero REST API is not affected by AGPL; a product that forks or embeds Zotero's client code and serves it over the network must release all modifications under AGPL-3.0. The CSL style library is CC Attribution-ShareAlike; distributing or modifying CSL styles requires attribution and ShareAlike for the styles file itself, not the overall product.
- **Mendeley (Elsevier)**: researcher library data (reading habits, annotations) is processed by Elsevier under its privacy policy; institutions should conduct a data protection impact assessment (DPIA) before mandating Mendeley, particularly in the EU, given Elsevier's commercial use of usage analytics.
- **Semantic Scholar Open Data**: the Open Research Corpus bulk download is available under ODC-By for academic use; commercial use requires a separate licence from AI2. The API is free for academic use; building a commercial product solely dependent on the public API may violate terms of service above rate limits.
- **Connected Papers / Research Rabbit**: both use Semantic Scholar as the underlying data source; their own interfaces and graph algorithms are proprietary; paper metadata is governed by CrossRef and publisher terms.
- **CrossRef Metadata API**: free for polite use with an email address; bulk commercial use requires a CrossRef Plus membership (~$10k+/year); any platform performing millions of DOI resolutions per day should register for Crossref Plus.
- **Publisher full-text access**: accessing full text programmatically from publisher websites (Elsevier, Springer, Wiley) without a text-and-data mining (TDM) agreement likely violates terms of service; platforms building AI synthesis features must obtain TDM licences from publishers or use open-access content only.

## Recommended Feature Scope

**Must-have (MVP)**:
- Browser extension capturing metadata and PDFs from 500+ academic databases (PubMed, arXiv, Semantic Scholar, CrossRef, Google Scholar, institutional proxies)
- Personal library with folder/collection organisation and full-text PDF storage
- In-library PDF reader with annotation (highlights, sticky notes) synced to cloud
- Word processor plugin for citation insertion and bibliography generation in Word and Google Docs
- CSL-based citation style support covering at least APA, MLA, Chicago, Vancouver, IEEE (10,000+ styles via the open CSL repository)
- BibTeX and RIS export for LaTeX/Overleaf interoperability
- Retraction watch integration: automatic flag when a stored paper is retracted or subject to an expression of concern

**Should-have (v1.1)**:
- Semantic literature search from plain-language research question (not just keyword matching)
- AI paper summarisation: one-paragraph plain-language summary for any PDF in the library
- Visual citation graph for exploration of a literature neighbourhood from one or more seed papers
- Supportive vs. contrasting citation classification for papers in the library (informed by Scite's methodology)
- Group/shared library with role-based access (owner, editor, read-only) and comment threads on shared annotations
- Systematic review screening mode (PRISMA-compatible title/abstract screening with include/exclude decisions and inter-rater reliability tracking)

**Nice-to-have (backlog)**:
- Automated multi-paper synthesis: read a collection of papers and draft a structured literature review section identifying agreements, contradictions, and gaps
- Research project memory: persistent AI assistant aware of the researcher's full reading and citation history across projects
- Automated structured data extraction from papers (study design, sample size, outcomes) into a comparison table for systematic review
- Manuscript journal recommender based on title, abstract, and reference list
- Publisher TDM API integration for full-text access to licensed content for AI-assisted analysis
