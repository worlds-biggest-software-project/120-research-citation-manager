# Research & Citation Manager

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source reference manager that unifies academic library management, semantic literature discovery, and multi-paper synthesis in a single workflow.

Research & Citation Manager is a reference, annotation, and literature-review platform for graduate students, faculty, systematic reviewers, and research-driven analysts. It combines the library management strengths of tools like Zotero with AI capabilities pioneered by Scite and Elicit — semantic search, citation reliability classification, and structured synthesis — so researchers can move from literature discovery to bibliography in one place.

---

## Why Research & Citation Manager?

- Established managers (Zotero, Mendeley, EndNote) have no native AI synthesis, semantic search, or literature review assistance, leaving researchers to stitch together separate tools for discovery, analysis, and citation.
- AI-first tools that solve discrete problems well — Scite for citation context, Elicit for data extraction, Connected Papers and Research Rabbit for graph exploration — are not reference managers, so they cannot generate bibliographies, manage personal libraries, or insert citations into documents.
- Commercial incumbents are expensive (EndNote ~$170/year, institutional RefWorks at $10k–100k+/year) or raise governance concerns: Mendeley library data informs Elsevier's commercial analytics under its privacy policy.
- AI-augmented citation tools are growing faster than the broader market — ~30% of citation management tools incorporated AI features in 2024 versus 15% the prior year — but no single product unifies library management with full AI synthesis and citation reliability scoring.
- Zotero's AGPL-3.0 client cannot be embedded in proprietary SaaS without copyleft obligations, leaving room for a new open-source codebase designed from the start for AI workflows and self-hosting.

---

## Key Features

### Library Capture & Management

- Browser extension capturing metadata and PDFs from 500+ academic databases (PubMed, arXiv, Semantic Scholar, CrossRef, Google Scholar, institutional proxies)
- Personal library with folder/collection organisation and full-text PDF storage
- In-library PDF reader with annotation (highlights, sticky notes) synced to cloud
- Retraction Watch integration: automatic flag when a stored paper is retracted or subject to an expression of concern
- Group/shared library with role-based access (owner, editor, read-only) and comment threads on shared annotations

### Citation & Bibliography Generation

- Word processor plugin for citation insertion and bibliography generation in Word and Google Docs
- CSL-based citation style support covering at least APA, MLA, Chicago, Vancouver, and IEEE (10,000+ styles via the open CSL repository)
- BibTeX and RIS export for LaTeX/Overleaf interoperability
- Auto-formatted bibliography handling for novel source types (datasets, software, social media, preprints) that break template-based CSL formatters

### AI Literature Discovery & Synthesis

- Semantic literature search from a plain-language research question, beyond keyword matching
- AI paper summarisation: one-paragraph plain-language summary for any PDF in the library
- Visual citation graph for exploration of a literature neighbourhood from one or more seed papers
- Supportive vs. contrasting citation classification for papers in the library, informed by Scite's methodology
- Automated multi-paper synthesis: read a collection and draft a structured literature review identifying agreements, contradictions, and gaps

### Systematic Review & Research Memory

- Systematic review screening mode: PRISMA-compatible title/abstract screening with include/exclude decisions and inter-rater reliability tracking
- Automated structured data extraction from papers (study design, sample size, outcomes) into a comparison table
- Manuscript journal recommender based on title, abstract, and reference list
- Research project memory: persistent AI assistant aware of the researcher's full reading and citation history across projects

---

## AI-Native Advantage

Unlike incumbent reference managers, Research & Citation Manager treats AI as a first-class layer rather than an add-on. Semantic search interprets research intent expressed in natural language and surfaces cross-disciplinary work that keyword search misses. Multi-paper synthesis reads an entire collection, extracts claims, compares methodologies, and identifies contradictions — collapsing weeks of manual literature review into hours. Citation reliability scoring flags retracted or heavily-contested papers before they are relied upon, and a persistent research memory answers questions like "have I already read something about X?" across years of work.

---

## Tech Stack & Deployment

The project is designed around open standards: BibTeX and RIS for interoperability, DOI / ISO 26324 and CrossRef metadata for canonical resolution, CSL for citation style rendering, OpenURL for institutional link resolution, and ORCID for author identity. Expected deployment modes include self-hosted (institution or lab), managed cloud, and hybrid configurations where library data stays on-premises while AI inference runs against a hosted gateway. Browser extensions, Word and Google Docs plugins, and a public REST API are core surfaces, with Semantic Scholar and CrossRef as primary upstream metadata sources.

---

## Market Context

The global reference management tools market was valued at approximately $400 million in 2024 and is projected to reach $670–760 million by 2033 (CAGR ~7.5–7.9%), with roughly 42 million active users across major platforms and 68% of academic institutions using digital reference tools. Pricing spans free open-source (Zotero) through consumer subscriptions ($3–20/month), AI literature add-ons ($10–30/month), and institutional licences ($10k–100k+/year). Primary buyers are graduate students and PhD researchers, faculty conducting systematic reviews, university library systems, clinical evidence-synthesis teams, and rigorous publishing organisations.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
