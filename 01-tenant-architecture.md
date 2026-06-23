# 01 — Tenant Architecture

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `01-tenant-architecture.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Operators, integration engineers, and AI-tool implementers needing the structural model of a ServiceTitan tenant.

This file is the structural map of a ServiceTitan tenant: tenant identity, Business Units (BUs), the four core entity domains, the integration surface, and the segmentation model that drives both phone routing and pricing.

---

## 1. Tenant Identity and URL Patterns

Every ServiceTitan tenant has a numeric tenant identifier. The tenant is accessed via the `go.servicetitan.com` web application. Stable URL fragments useful for navigation, automation, and documentation:

| Path | Purpose |
|---|---|
| `https://go.servicetitan.com/#/Settings/...` | Settings root |
| `https://go.servicetitan.com/#/Settings/job-types` | Job Types list |
| `https://go.servicetitan.com/#/Settings/job-types/Edit/{jobTypeId}` | Specific job type editor |
| `https://go.servicetitan.com/#/Settings/forms` | Forms list |
| `https://go.servicetitan.com/#/Settings/AdaptiveCapacitySettings` | Adaptive Capacity Basic Settings |
| `https://go.servicetitan.com/#/Settings/AdaptiveCapacitySettings/advanced-settings` | AdCap Advanced |
| `https://go.servicetitan.com/#/new/marketing/standalone/campaigns/campaign-manager/all/pro` | Marketing Pro Campaign Manager |

Stable URLs are useful for documentation, audit checklists, and targeting browser automation tools. **However, deep-link navigation can fail in extended automation sessions**: reconnecting via the Settings root and clicking through is the reliable fallback. Job Type Edit URLs in particular do not load modal data when navigated to directly — you must click Edit on the row.

### 1.1 Multi-tenancy and Network Groups

ServiceTitan operates on a **multi-tenant database architecture**. API requests must explicitly declare the target Tenant ID within the URL path. ServiceTitan-issued primary keys are guaranteed unique only within the boundaries of a specific tenant — when warehousing data from multiple tenants, the warehouse schema must implement a composite primary key incorporating `tenant_id` to prevent collisions.

For enterprise organizations managing multiple discrete business units or franchises through the **Enterprise Hub**, administrators configure API credentials across **Network Groups**. A Network Group is a designated subset of Tenant IDs that share a single Client ID and Secret Key pair.

### 1.2 Integration Environment

ServiceTitan provides an **Integration Environment** — a sandbox cloned from production data, used for development and rigorous testing of integrations. The sandbox host is `api-integration.servicetitan.io` (with corresponding `auth-integration.servicetitan.io` for OAuth).

**Important:** The integration sandbox is **not provisioned by default for every tenant**. Production credentials return `invalid_client` against the integration auth endpoint unless a separate sandbox app is registered in the Developer Portal. If sandbox testing is in your roadmap, register the sandbox app early and add `ST_INT_*` credential slots to your environment from day one.

---

## 2. Business Units (BUs)

Business Units are the structural partition that ties dispatch capacity, AdCap rules, dashboards, scheduled reports, and revenue rollups together. They are the most important configuration object in a tenant — get them wrong and most downstream analytics break.

### 2.1 The standard BU pattern

A typical multi-trade tenant uses BUs that combine **trade × residential/commercial × service-mode**:

- HVAC Service Residential
- HVAC Service Commercial
- HVAC Sales
- HVAC Maintenance
- HVAC Install
- Plumbing Service Residential
- Plumbing Service Commercial
- Plumbing Install
- Electrical Service Residential
- Electrical Service Commercial
- Electrical Install
- Generator Service
- Generator Install

The exact set varies; the principle is consistent: **each combination of trade × audience × mode that needs separate tracking gets its own BU**.

### 2.2 Cardinality rules

| Rule | Effect |
|---|---|
| Every job type binds to exactly **one** BU | A job type cannot span two BUs. If it needs to, it's actually two job types. |
| Every technician is associated with **one or more** BUs | Techs typically bind to all BUs in their trade (`HVAC Service Residential`, `HVAC Service Commercial`, `HVAC Maintenance`, `HVAC Install`). |
| Pricebook items have a "Default Business Unit" but are **not BU-scoped** | Pricebook is global to the tenant; the BU on the item is informational. |
| Dispatch capacity, AdCap rules, and reports scope by **BU or BU Group** | This is the primary axis along which most operational dashboards slice. |

### 2.3 BU Groups

BU Groups are a higher abstraction wrapping multiple BUs for cross-trade reporting and AdCap rule scope. Common pattern: one BU Group per trade, encompassing all that trade's BUs (Service Residential, Service Commercial, Maintenance, Install, Sales).

Honoring BU Groups over BUs is a global toggle in AdCap Basic Settings. Multi-trade operations with BU Groups configured should turn this ON; the rest of the toggles in Basic Settings should typically be OFF (controlled per-rule via strategic rule conditions).

### 2.4 The renaming hazard

**Renaming a BU is destructive to historical reports.** The old BU name disappears from filters, dashboards break, and historical job-counts can no longer be sliced by the prior BU. Never rename — instead:

1. Create a new BU with the corrected name.
2. Migrate active job types and technicians to the new BU.
3. Deactivate the old BU.
4. Document the migration date so dashboards can be reconstructed if needed.

---

## 3. The Four Core Entity Domains

A ServiceTitan tenant's data model has four core entity domains. Almost every audit and cleanup workstream targets the **Configuration domain** — because that's where the inputs live that determine how Customer, Job, and Pricebook data get classified.

| Domain | What lives here | Mostly read or write? |
|---|---|---|
| **Customer & Location** | Account, location, equipment installed, membership status, prior jobs | Mostly continuous write from operations; cleanup tools mostly read |
| **Job & Invoice** | Job type, BU, schedule, tech assignment, line items, invoice, payment | Mostly continuous write from operations |
| **Pricebook** | Services, materials, equipment, configurable equipment, estimate templates | Periodic write from cleanup; continuous read from operations |
| **Configuration / Settings** | Job types, forms, tags, skills, custom fields, BUs, zones, AdCap rules, marketing campaigns, tracking numbers | Periodic batch write from cleanup; continuous read from operations |

---

## 4. Pricebook-Side Entity Types

ServiceTitan distinguishes between four pricebook-side entity types. Full schema details in `02-pricebook-reference.md` §4. The minimum viable model:

| Entity | What it represents | Examples |
|---|---|---|
| **Services** | Sellable line items (labor + bundled materials/equipment) | "Drain Auger Up to 75 ft", "Capacitor Replacement" |
| **Materials** | Small consumables, parts | Drip pans, expansion tanks, isolation valves, fasteners |
| **Equipment** | Major equipment with serial numbers and warranty | Water heaters, furnaces, condensers, electrical panels, generators |
| **Configurable Equipment** | Equipment with variants (size, fuel, efficiency tier) | One logical "40-gallon water heater" with picklists for Rheem/State/Rinnai variants |

**Estimate Templates** are a separate concept — pre-bundled option sets (often Good / Better / Best / Ultra tiers) that techs present in the Sales tab on Field Pro. Each tier references the same equipment SKU but bundles different accessories — this is what enables clean reporting on tier-mix and gross-profit-by-tier.

---

## 5. The Integration Surface

A typical multi-trade tenant exposes or consumes the following integration classes. Each is a separate vendor relationship and a separate cleanup target.

| Integration class | Purpose | Typical implementation |
|---|---|---|
| **Pricebook API (v2)** | Read/write services, materials, equipment, categories | REST API, often fronted by an edge proxy (Cloudflare Worker, etc.) |
| **Dispatch / Capacity API** | Get adaptive availability, capacity hours | REST; behavior varies by Adjustable Capacity Planning (ACP, legacy) vs. Adaptive Capacity (AdCap, current) |
| **Browser automation** | Live UI navigation for tasks the API doesn't expose | Puppeteer, Playwright, Claude in Chrome, etc. |
| **Voice agent platform** | Inbound AI call answering | ServiceTitan-native Voice Agent or third-party (Lace AI, Retell, etc.) |
| **AI CSR scoring** | Post-call scoring of CSR adherence to playbook | Lace AI, DialedIn, Convoso, etc. |
| **Marketing data sources** | Lead attribution, SEO, listings | Scorpion (and Scorpion Convert), Yext via Marketing Pro, Google Ads, Local Service Ads (LSA) |
| **Phone / messaging platform** | Forward chains, IVR, SMS | Podium, RingCentral, etc. |
| **Dashboard platform** | KPI reporting | Plecto, Power BI, Looker, Tableau |
| **Automation platform** | Workflow orchestration | Make.com, Zapier, n8n |
| **iPaaS / MCP tools** | Programmatic tool calls for LLMs | Custom MCP servers, Make.com MCP |

### 5.1 The "transactional vs. export" API split

ServiceTitan APIs differentiate strictly between two endpoint classes:

**Transactional APIs** operate on standard REST principles, supporting GET, PUT, POST, PATCH, and DELETE. They are lightweight conduits for moving small payloads — real-time CRUD on individual records, micro-adjustments, single-item updates.

**Data Export APIs** are identified by the `/export/` path component. They return a complete unfiltered snapshot of the database — including active, inactive, and deleted records — designed for warehousing.

For deduplication or warehousing workflows, ingesting the **complete historical state** (active + inactive + merged) is mandatory. Failing to extract deactivated or historically merged records leads to algorithmic blind spots and accidental resurrection of duplicate entities. See `07-crm-data-model-and-deduplication.md` §6 for the `mergedToId` tombstone pattern that depends on this.

### 5.2 Authentication

Authentication uses **OAuth 2.0** with a machine-to-machine client credentials grant flow. Setup yields three values:

- **Application Key** (`ak1.` prefix)
- **Client ID**
- **Client Secret**

The token endpoint is `https://auth.servicetitan.io/connect/token` (production) or `https://auth-integration.servicetitan.io/connect/token` (sandbox).

Tokens last 16 hours. **Cache them in-memory per process**, not in distributed storage — eventually-consistent caches return stale tokens during rotation, which is a worse failure mode than a few hundred milliseconds of OAuth round-trip on a cold start.

Secrets should never live in source. Standard storage mechanisms (cloud secret managers, environment variables loaded at runtime, Wrangler secrets for Cloudflare Workers) are all acceptable. Rotate any time a secret could have leaked; the migration from one secret prefix (`cs2.*`) to a new one (`cs5.*`) is invisible to consumers as long as the old token is allowed to expire naturally.

> **Adaptive Login (ST-77.3):** ST-77.3 introduced a two-step login flow in the ServiceTitan web application. This does not affect OAuth API authentication (machine-to-machine), but breaks browser automation scripts that use credential-fill login. If your integration includes browser-side automation targeting `go.servicetitan.com`, migrate to session-reuse (persist browser session state across runs) rather than fresh-login per run. See `09-anti-patterns-and-failure-modes.md` §2.19.

---

## 6. Customer Segmentation Drives Routing

A common segmentation pattern that drives both phone routing and pricebook selection:

| Segment | Routing pattern | Pricing |
|---|---|---|
| Residential, new customer | Web booking → live agent → after-hours voice agent | Standard residential pricebook |
| Residential, existing | Same as above; member pricing applied if active | Standard residential pricebook (with member discount) |
| Commercial | Direct lines that bypass the new-customer queue | Commercial pricebook (separate categories) |
| Property management | Dedicated direct line for multi-unit coordination | Commercial; multi-unit terms |
| Trade referral partner | Partner-specific tracking number → IVR → CSR queue | Standard, with referral attribution |

**The architectural lesson:** routing destination is determined by the **tracking number the customer dialed**, not by the customer record. This means tracking-number hygiene (`06-phone-marketing-attribution.md`) is the foundation of correct attribution. A customer who calls the wrong number gets the wrong routing, and the call shows up in the wrong attribution bucket.

---

## 7. Standard URLs and Reference Endpoints

Quick-reference list of stable endpoints and URL paths used throughout this series.

### 7.1 API base paths

```
Production:     https://api.servicetitan.io/{namespace}/v{n}/tenant/{tenantId}/...
Sandbox:        https://api-integration.servicetitan.io/{namespace}/v{n}/tenant/{tenantId}/...

OAuth (prod):   https://auth.servicetitan.io/connect/token
OAuth (sand):   https://auth-integration.servicetitan.io/connect/token
```

### 7.2 Common API namespaces

| Namespace | Purpose | Reference |
|---|---|---|
| `crm` | Customers, locations, contacts | `07-crm-data-model-and-deduplication.md` |
| `pricebook` | Services, materials, equipment, categories | `02-pricebook-reference.md` |
| `jpm` | Job Project Management — jobs, projects, holds | `04-job-types-forms-tags.md` |
| `dispatch` | Capacity, technician assignments | `03-dispatch-and-capacity.md` |
| `marketing` | Campaigns, tracking numbers | `06-phone-marketing-attribution.md` |
| `salestech` | Estimate templates, proposal templates, proposal types | `02-pricebook-reference.md` §3.5, `11-api-release-notes-st77.md` |
| `accounting` | Invoices, payments, deposits, credit memos | See `10-api-release-notes-st75-st76.md` |
| `forms` | Form definitions and submissions | `05-form-2-0-json-reference.md` |
| `settings` | Tenant settings — employees, technicians, business units | See `10-api-release-notes-st75-st76.md` |
| `reporting` | Saved-report execution | See `10-api-release-notes-st75-st76.md` §3 |

### 7.3 The Developer Portal

`https://developer.servicetitan.io/` — the source of truth for endpoint documentation, release notes, and the OAuth app registration UI. Every consumer of ServiceTitan APIs should be on the email list for release announcements; the API surface evolves rapidly (see `10-api-release-notes-st75-st76.md`).

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
