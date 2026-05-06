# 00 — Overview, AI-Readiness Principles, and Naming Conventions

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `00-overview-and-naming.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Anyone using an LLM to reason about ServiceTitan configuration, audits, integration, or AI tool readiness.

This file establishes the foundation for every other reference in the series: why ServiceTitan tenants must be configured as if their data will feed an AI (because it will), and the naming/lifecycle conventions that make that possible.

---

## 1. The AI-Readiness Principle

ServiceTitan is feature-rich and unforgiving. Every poorly named tag, every duplicate form, every untagged tracking number, every orphan pricebook item degrades the quality of the data that downstream systems depend on. Those downstream systems include:

- **Titan Intelligence (TI)** — ServiceTitan's native predictive analytics
- **Dispatch Pro (DP)** — the AI dispatch optimizer
- **Adaptive Capacity (AdCap)** — the rules-based capacity engine
- **AI CSR scoring platforms** (Lace AI, etc.) — post-call coaching
- **External BI dashboards** — Plecto, Power BI, Looker, Tableau
- **Voice agents** — ST-native or third-party (Lace, Retell, etc.)
- **Marketing attribution** — Marketing Pro, Scorpion Convert, Google Ads
- **Custom RAG / LLM workflows** — anything that ingests pricebook, job, customer, or call data

The oldest principle in computing applies in full: **garbage in, hallucinated dashboards out**.

### 1.1 The three properties of an AI-ready tenant

1. **Deterministic naming.** Every entity (job type, tag, form, campaign, tracking number, AdCap rule, pricebook code) follows a parseable, tokenized convention. A regex or LLM can extract trade, business unit, intent, and version from the name alone.

2. **No orphan records.** Every tracking number is mapped to a campaign. Every job is mapped to a job type with skills, tags, and a sold threshold. Every pricebook item lives in exactly one — or, where intentionally configured, two — categories. Deactivated items do not leak into reports.

3. **Stable cardinality.** When a category, tag, business unit, or campaign is added, deprecated, or merged, it is done through an explicit migration rather than a silent edit. Historical values are deactivated, never deleted, so time-series analytics survive.

When all three hold, ServiceTitan's AI tools generate accurate forecasts. Until they hold, every model is fighting noise.

### 1.2 Headline operating principles

These apply to any ServiceTitan tenant pursuing AI-readiness. Each is expanded in later sections of this reference series.

- **Deactivate, don't delete.** Historical reporting depends on it. This applies to job types, tags, pricebook items, forms, tracking numbers, AdCap rules, and campaigns.
- **Rename via versioned suffix** (e.g., `v081524` style — month/day/year) so audit trails survive when rules or templates evolve.
- **One configuration workstream at a time.** Parallel browser-automation sessions thrash the ServiceTitan UI and produce silent partial writes.
- **Pricebook export is positional, not random-access.** When parsing the Categories sheet, iterate top-to-bottom with running state — never read individual rows in isolation. (See `02-pricebook-reference.md` §6.)
- **AdCap rules and Scheduling Pro day-of-week settings are independent systems.** Day-of-week restrictions must be set in **both** schedulers; AdCap can say "100% available" while Scheduling Pro is closed. (See `03-dispatch-and-capacity.md` §5.)
- **The AdCap Adoption report is not the same as Capacity Hours.** They are separate data needs. The Scheduling Utilization report tracks whether dispatchers used AdCap, NOT whether the tenant was busy. (See `03-dispatch-and-capacity.md` §3.7.)
- **Multi-category items use array syntax** in the API: `"categories": [{"id": 123}, {"id": 456}]`.
- **Tag standardization comes before job-type assignment.** Clean the tag dictionary first, then map to job types in a single pass.
- **Asterisk-prefixed items are canonical.** `*Default Service` is active; `DNU-Default Service` is legacy. (See §3.2 below.)

---

## 2. Tenant Architecture — Quick Reference

A full architecture deep dive lives in `01-tenant-architecture.md`. The minimum viable mental model:

| Layer | What lives here |
|---|---|
| **Tenant** | The numeric tenant identifier; everything else is scoped to it. |
| **Business Units (BUs)** | Trade × Residential/Commercial × Mode (e.g., HVAC Service Residential). Every job type binds to exactly one BU. |
| **BU Groups** | Higher abstraction wrapping multiple BUs for cross-trade reporting and AdCap rule scope. |
| **Configuration domain** | Job types, forms, tags, skills, custom fields, AdCap rules, campaigns, tracking numbers. The cleanup target. |
| **Operational data** | Customers, locations, jobs, invoices, equipment. Generated continuously; quality is downstream of configuration. |
| **Pricebook** | Services, materials, equipment, configurable equipment, estimate templates. Independent system; see file `02`. |

**Rule of thumb:** Almost every audit and cleanup workstream targets the **configuration domain** — because configuration is what determines how operational data gets classified, and configuration changes propagate to history when done correctly.

---

## 3. Naming Conventions

### 3.1 Why naming is the highest-leverage cleanliness target

In a tenant with thousands of records — pricebook items, tracking numbers, job types, forms, AdCap rules — a deterministic naming convention is what makes the data parseable by both humans and AI. A name that encodes trade, intent, scope, and version is self-documenting and survives RAG chunking. A name like `*HVAC Service | 8AM-12PM | 90% | v081524` tells you the trade, window, capacity, and version in 35 characters; a name like `Morning HVAC Rule` tells you nothing.

### 3.2 The asterisk / DNU convention

A two-state convention for any list-style entity (job types, forms, tags, equipment templates, campaigns):

- **`*Name`** — canonical, active, official. The asterisk is the marker for "this is the one to use." It also sorts to the top of any picker.
- **`DNU-Name`** — Do Not Use. Legacy item kept for historical reporting. Deactivated where possible. Renamed to the asterisk convention if migrated, or fully deactivated.

The visual prefix sorts canonical items to the top of any picker, making the right choice the easy choice for techs and CSRs.

### 3.3 The pipe-delimited token convention

For entities with multiple orthogonal dimensions (channel × BU × campaign × variant × version), use pipes as field separators:

```
[Dimension1] | [Dimension2] | [Dimension3] | [Version]
```

**Tracking number example:**
```
LSA | HVAC Residential | Google | v031226
```

**AdCap strategic rule example:**
```
HVAC Service | 8AM-12PM | 90% | v081524
```

**Form example:**
```
*Maintenance Checklist | HVAC | v062524
```

**Why pipes:** They survive most CSV/Excel exports, they're not used in normal English text (so they're an unambiguous separator), and they're trivial to split on programmatically.

### 3.4 The versioned-suffix pattern

Use `v[MMDDYY]` as the trailing token. Example: `v081524` = August 15, 2024.

When rules or templates evolve, **create a new version with an incremented suffix and deactivate the old one** rather than editing in place. This:

- Preserves the audit trail: you can always answer "what threshold was active in March?"
- Survives most-restrictive-wins evaluation cleanly: when you re-version, you deactivate the old rule and the new one becomes the only matching rule.
- Makes superseded rules instantly identifiable in audits.

Avoid `vYYYYMMDD` — it sorts the same as `vMMDDYY` for cross-year work but is two characters longer and humans read it slower.

### 3.5 Pricebook code grammar

A typical pricebook code structure encodes trade, item type, and a numeric or descriptive specifier:

```
[TRADE]-[TYPE]-[SPECIFIER]
```

| Pattern | Example | Meaning |
|---|---|---|
| `EL-RES-ST` | Electrical Residential Standard Diagnostic | Trade-Audience-Item |
| `HV-COM-ST` | HVAC Commercial Standard Diagnostic | Same pattern |
| `WHNG-1701` | Water Heater, Natural Gas, Model 1701 | Trade-Subtype-Identifier |
| `GEN-GAS-XX` | Generator, Gas, Variant XX | Modular kit pattern |

Codes are short, uppercase, hyphen-delimited. The trade prefix should be **two letters** when possible (`EL`, `HV`, `PL`, `GN`); only go to three when ambiguity forces it.

**Codes are immutable.** Never reuse a deactivated code for a different item — you create permanent semantic ambiguity in historical line items.

### 3.6 Item name convention — search-first ordering

Pricebook items appear in a tech's search bar on the mobile app. Names should put the **highest-discriminative tokens first** because techs search by typing the first few characters.

**Bad:** `Standard Diagnostic - HVAC - Residential`  
**Better:** `HV-RES-ST | Diagnostic | Residential | $85`  
**Best:** `HVAC Diagnostic - Residential ($85)` — followed by the SKU code in description

Front-load the trade and item type. Money goes at the end.

### 3.7 The asterisk and DNU lifecycle in practice

Daily lifecycle of an item under this convention:

1. **Created** — given a temp name during cleanup work; flagged for canonicalization.
2. **Canonicalized** — renamed to `*Name`. Marked active. Bound to its BU/category/etc.
3. **Superseded** — replaced by a new canonical (`*Name v2`); old one renamed to `DNU-Name`.
4. **Deactivated** — `DNU-Name` is deactivated once historical reporting cycles complete (typically 13+ months).
5. **Never deleted.** The record remains in the database forever to preserve historical line items.

---

## 4. The Four Pillars of Cleanliness

Every audit workstream targets one of four pillars. They are independent but ordered: cleaning Pillar 1 before Pillar 2 saves you rework.

| Pillar | Scope | Reference |
|---|---|---|
| **1. Pricebook** | Services, materials, equipment, categories, templates, configurable equipment | `02-pricebook-reference.md` |
| **2. Marketing & Tracking** | Tracking numbers, campaigns, attribution, DNI pools, IVRs | `06-phone-marketing-attribution.md` |
| **3. Dispatch & Capacity** | AdCap, Scheduling Pro, Dispatch Pro, strategic rules, shifts | `03-dispatch-and-capacity.md` |
| **4. Forms & Job Types** | Job types, forms, tags, skills, custom fields | `04-job-types-forms-tags.md` |

Cross-cutting concerns (CRM data architecture, anti-patterns, audit methodology) live in their own files.

---

## 5. The Universal Cleanup Pattern

Every audit follows the same four-phase shape:

1. **Inventory** — export the current state (Excel, JSON, screenshots).
2. **Classify** — group records into action buckets (correct / rename / re-categorize / deactivate).
3. **Plan** — produce an import-ready file or API payload that effects the changes.
4. **Execute** — apply via API, browser automation, or manual UI work.

Every phase produces a durable artifact. The plan should be a single import-ready Excel and an execution Markdown — both regenerable from the inventory if needed. Specific playbooks per pillar live in `09-anti-patterns-and-failure-modes.md` §3.

---

## 6. The Integration Surface

A typical multi-trade tenant exposes or consumes the following integration classes. Full details in `01-tenant-architecture.md` §5.

| Integration class | Purpose | Typical implementation |
|---|---|---|
| Pricebook API (v2) | Read/write services, materials, equipment, categories | REST API, often fronted by an edge proxy |
| Dispatch / Capacity API | Adaptive availability, capacity hours | REST; behavior varies by ACP vs. AdCap |
| Browser automation | UI navigation for unsupported configuration tasks | Puppeteer, Playwright, Claude in Chrome |
| Voice agent platform | Inbound AI call answering | ST-native Voice Agent or third-party (Lace AI, Retell) |
| AI CSR scoring | Post-call scoring against a rubric | Lace AI, DialedIn, Convoso |
| Marketing data sources | Lead attribution, SEO, listings | Scorpion, Yext via Marketing Pro, Google Ads, Local Service Ads (LSA) |
| Phone/messaging platform | Forward chains, IVR, SMS | Podium, RingCentral, etc. |
| Dashboard platform | KPI reporting | Plecto, Power BI, Looker, Tableau |
| Automation platform | Workflow orchestration | Make.com, Zapier, n8n |
| iPaaS / MCP tools | Programmatic tool calls for LLMs | Custom MCP servers, Make.com MCP |

---

## 7. Reading the Rest of This Series

Every file in this series is structured the same way:

- **Front matter** — author, scope, audience.
- **Section 1** — the core architectural model for the topic.
- **Sections 2–N** — specific behavior, schemas, configurations, gotchas.
- **Final section** — error catalog or known-failure-modes table where applicable.
- **Cross-references** — full section numbers (e.g., "see `02-pricebook-reference.md` §4.3") so chunked retrieval still resolves.

Each file is self-contained: you can drop just the one you need into an LLM and it will work standalone. You can also drop the whole set into a Project / Notebook / RAG store and they will compose.

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
