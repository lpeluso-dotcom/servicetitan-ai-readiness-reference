# ServiceTitan AI-Readiness Reference

A series of dense, LLM-optimized technical references for ServiceTitan administrators, integration engineers, and AI/automation teams running multi-trade home-services tenants (HVAC, Plumbing, Electrical, Generators).

These documents are not blog posts and not user-facing documentation. They are **drop-in context files** designed to be ingested by an LLM (Claude Projects, NotebookLM, custom GPTs, RAG pipelines, MCP-attached corpora) so that the model can reason about ServiceTitan configuration, API behavior, data architecture, and the gotchas that don't appear in the official product docs.

Each file is self-contained: every acronym is expanded on first use within its section, every section has a stable anchor, and cross-references use full section numbers so chunked retrieval still resolves correctly.

---

## Files

| # | File | Topic |
|---|------|-------|
| 00 | [00-overview-and-naming.md](./00-overview-and-naming.md) | The AI-readiness thesis, naming conventions, lifecycle discipline (deactivate-vs-delete), the asterisk/DNU pattern |
| 01 | [01-tenant-architecture.md](./01-tenant-architecture.md) | Tenant identity, Business Units, the four core entity domains, integration surface, customer segmentation |
| 02 | [02-pricebook-reference.md](./02-pricebook-reference.md) | Pricebook API, object schemas, dynamic pricing, configurable equipment, categories, the export-format trap, gotchas catalogue |
| 03 | [03-dispatch-and-capacity.md](./03-dispatch-and-capacity.md) | Adaptive Capacity (AdCap), Scheduling Pro, Dispatch Pro, strategic rules, most-restrictive-wins, the AdCap/SP scope boundary |
| 04 | [04-job-types-forms-tags.md](./04-job-types-forms-tags.md) | Job Types data model (API vs UI), default services, forms architecture, tags, skills, custom fields |
| 05 | [05-form-2-0-json-reference.md](./05-form-2-0-json-reference.md) | Complete Form 2.0 JSON schema reference — envelope, definition, conditional logic, micro-rule constraint, error catalog |
| 06 | [06-phone-marketing-attribution.md](./06-phone-marketing-attribution.md) | Phone number architecture, IVR, tracking number taxonomy, Marketing Pro campaigns, attribution patterns |
| 07 | [07-crm-data-model-and-deduplication.md](./07-crm-data-model-and-deduplication.md) | Customer/Location/Contact entity schemas, the polymorphic Contact Hub, `mergedToId` tombstones, deduplication algorithms |
| 08 | [08-csr-scoring-and-ai-tools.md](./08-csr-scoring-and-ai-tools.md) | The 10-step CSR rubric, AI scoring platforms, Titan Intelligence, Dispatch Pro, AdCap data dependencies |
| 09 | [09-anti-patterns-and-failure-modes.md](./09-anti-patterns-and-failure-modes.md) | The "don't do" catalogue, common failure modes, workarounds, audit playbooks |
| 10 | [10-api-release-notes-st75-st76.md](./10-api-release-notes-st75-st76.md) | API change log and integration impact analysis, ST-75.1 through ST-76.3 |
| — | [glossary.md](./glossary.md) | Unified glossary of ServiceTitan terms used across all documents |

---

## How to use these files with an LLM

**Claude Projects / Claude.ai:**
1. Create a new Project.
2. Upload the files you need as Project Knowledge. The full set is ~700KB; chunk if your tier limits the upload size.
3. Tell the model: *"Use the attached ServiceTitan AI-readiness references for any question about ServiceTitan configuration, API behavior, or operations. Cite the file number and section anchor when you do."*

**NotebookLM:**
1. Create a new notebook.
2. Add each `.md` as a source. NotebookLM handles markdown cleanly.
3. The cross-references (`see §4.2`) resolve correctly because every section has a stable anchor.

**Custom GPT / OpenAI Assistants:**
1. Upload the files as the Knowledge corpus.
2. Set instructions: *"For ServiceTitan questions, search the attached references first. Quote the section number when citing."*

**RAG / vector store ingestion:**
- Chunk on `## ` (level-2) headers. Each section is sized to be a coherent retrievable unit (typically 200–800 tokens).
- The leading metadata block in each file (author, scope, etc.) should travel with every chunk as document-level metadata.

**MCP / agentic tool corpora:**
- Each file works as a standalone "skill" reference. Drop into your agent's context window when the conversation touches the relevant ServiceTitan domain.

---

## Conventions used across all files

- **Section anchors** use the pattern `# N. Title` and `## N.M Subtitle`. Cross-references use full section numbers (e.g., "see §4.2 in `02-pricebook-reference.md`").
- **Acronyms** are expanded on first use within each section. AdCap, BU, DID, DNI, GAA, IVR, LSA, PPC, TI — all expanded first time they appear in any section.
- **Code-style values** (`like_this`) are used for stable identifiers, payloads, and parseable tokens (UUIDs, JSON keys, API paths).
- **Tables** are used for any enumerable data (field references, error catalogs, rule configurations) so the structure is preserved through chunking.
- **Worked examples** that reference a specific tenant ("QSC tenant snapshot") are marked as illustrative; the patterns generalize.
- **Deactivate, never delete** is the universal lifecycle rule. It applies to job types, tags, pricebook items, forms, tracking numbers, campaigns, and AdCap rules. Historical reporting depends on it.

---

## Provenance and attribution

These references are derived from direct platform configuration, API interaction, live tenant exports, import-validation responses, and behavioral observation across multi-trade ServiceTitan tenants. No portion is copied from official ServiceTitan documentation; where official docs cover a topic, that is noted and the reference does not duplicate them.

**Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
**Contact:** Open an issue or pull request in the repository.  **License:** This work is published as a community technical resource. Use it, fork it, extend it. Attribution appreciated; contributions welcome via pull request.

This is a living set of references. More files will be added as new platform behavior is observed, validated, and written up. The goal: every dollar of effort spent reverse-engineering ServiceTitan should benefit the next operator who comes after.

---

*Not affiliated with ServiceTitan, Inc. All findings are community-sourced and validated against live tenants.*
