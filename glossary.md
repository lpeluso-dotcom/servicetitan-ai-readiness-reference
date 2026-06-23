# Glossary — ServiceTitan AI-Readiness Reference

> **Part of:** ServiceTitan AI-Readiness Reference
> **File:** `glossary.md`
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company
> **Status:** Public reference — community-sourced findings
> **Audience:** Anyone (human or LLM) reading the rest of the series and needing a quick disambiguation.

This is the unified glossary across all files in the AI-Readiness Reference. Where a term has a deep treatment in one of the topical files, the file number and section anchor are noted so chunked retrieval still resolves.

---

## A

**ACP — Adjustable Capacity Planning.** ServiceTitan's legacy capacity model. A simpler shift-count-based system without rule logic. Mutually exclusive with AdCap as the Scheduling Pro capacity source. Confirm with the ST CSM whether your tenant's Dispatch / Capacity API surfaces ACP or AdCap data — they are not the same. *(See `03-dispatch-and-capacity.md` §2.2.)*

**AdCap — Adaptive Capacity.** ServiceTitan's rule-based dynamic capacity management engine. Sits between raw technician shift data and the customer-facing booking interfaces (CSR Get Adaptive Availability tool, Scheduling Pro widget when in Adaptive Capacity mode). Evaluates Strategic Rules; most-restrictive-wins. *(See `03-dispatch-and-capacity.md` §1–3.)*

**AdCap Adoption** vs. **Capacity Hours.** Two distinct metrics that sound similar and are commonly conflated. AdCap Adoption (sourced from the Scheduling Utilization report) measures whether dispatchers are using AdCap. Capacity Hours (sourced from the Dispatch / Capacity API) measures actual booked-vs-available hours. Dashboards must use the right source for the question being asked. *(See `03-dispatch-and-capacity.md` §7.)*

**Adaptive Login.** A two-step login flow introduced in ST-77.3 for the ServiceTitan web application. The tenant identity step precedes the auth-method selection step, replacing the previous single-page credential form. This does not affect OAuth 2.0 machine-to-machine API authentication, but breaks browser automation scripts that use single-pass credential fill. Migrate to session-reuse (persisting browser session state across runs) as the workaround. See `09-anti-patterns-and-failure-modes.md` §2.19.

**AFUE — Annual Fuel Utilization Efficiency.** Gas furnace efficiency rating.

**AH — After Hours.** A common job-type and shift modifier (e.g., `HV-RES-AH` = HVAC residential after-hours diagnostic).

**AH Tech row.** A row on the Dispatch Board updated daily before the on-call window starts, listing the on-call tech's name, phone, and trade. The reference for the answering service when after-hours calls come in.

**Application Key (`ak1.` prefix).** One of the three credentials issued when registering an application in the ServiceTitan Developer Portal. Sent on every API call as the `ST-App-Key` header. Distinct from Client ID and Client Secret. *(See `01-tenant-architecture.md` §5.)*

**Arrival Window.** A time range offered to customers for technician arrival (e.g., 8 AM–12 PM). The primary unit of scheduling display in both the CSR workflow and Scheduling Pro.

**Asterisk convention (`*Name`).** The naming prefix used for canonical, active, official records. Visual prefix sorts canonical items to the top of any picker. Counterpart to `DNU-`. *(See `00-overview-and-naming.md` §3.2.)*

**Atlas.** ServiceTitan's native natural-language Strategic Rule builder for Adaptive Capacity. Integrated directly into ST as of ST-77.3 (previously a third-party tool). Accepts a plain-English rule description (e.g., *"Reduce HVAC capacity by 20% in July"*) and generates the rule configuration. Output is a standard strategic rule subject to normal most-restrictive-wins evaluation. Always review before activating; use the `vMMDDYY` versioned suffix. See `03-dispatch-and-capacity.md` §2.6.

**Audit Trail.** ServiceTitan's structural log of mutations. Modern Audit Trail uses immutable Record Numbers (not Record IDs), supports up to 500 rows per query.

---

## B

**Bank Deposits API.** A new endpoint family in ST-75.1 (`GET /bank-deposits`, `GET /bank-deposits/{id}/transactions`, `POST /bank-deposits/markasexported`). Closes a long-standing gap in accounting export workflows. *(See `10-api-release-notes-st75-st76.md` §2.)*

**BDR — Business Development Representative.** Common as a custom-field dimension on job types.

**`BillingScheduleType: Custom`.** A new enum value added to Service Agreements in ST-77.1. Allows fully custom billing schedules on membership-backed service agreements beyond the predefined `Monthly`, `Quarterly`, and `Annual` tiers.

**BTU — British Thermal Unit.** Heating/cooling capacity unit.

**BU — Business Unit.** The structural partition that ties dispatch capacity, AdCap rules, dashboards, scheduled reports, and revenue rollups together. Typically `Trade × Residential/Commercial × Service-Mode` (e.g., HVAC Service Residential). Every job type binds to exactly one BU. **Renaming a BU is destructive to historical reporting** — never do it without an explicit migration plan. *(See `01-tenant-architecture.md` §2.)*

**BU Group.** A higher abstraction wrapping multiple BUs for cross-trade reporting and AdCap rule scope.

**`BundleValidationError`.** A common error from Make.com or similar iPaaS write proxies on Pricebook writes. Usually means the Make scenario's input bundle validation rejects the payload format before it reaches the ST API, OR the ST-specific Make connector silently drops the request body. Workaround: use a generic HTTP module with direct OAuth, or front the API with a Cloudflare Worker. *(See `09-anti-patterns-and-failure-modes.md` §2.3.)*

---

## C

**Capacity Threshold.** The percentage of booked vs. available capacity at which an AdCap strategic rule activates. Typical values: 60% (tight), 90% (standard), 110% (intentional over-book for sales contexts).

**Categories sheet (Pricebook export).** A sheet in the ST pricebook XLSX export that uses **positional encoding** for hierarchy. Each row populates only the deepest non-empty `CategoryN` column it reaches; parent levels are implied by the row above. Reading rows in isolation produces false orphan classifications. *(See `02-pricebook-reference.md` §6.)*

**Category Type.** The classification of a pricebook category — `Services`, `Materials`, or `Equipment`. Set on category creation; cannot be silently changed.

**CIC — Customer Information Center.** Common name for live-agent answering services in trades.

**Client ID / Client Secret.** Two of the three credentials issued in the ServiceTitan Developer Portal. Used in the OAuth 2.0 client credentials grant flow. *(See `01-tenant-architecture.md` §5.)*

**CLV — Customer Lifetime Value.** A scoring output from Titan Intelligence. Reliability depends on clean job-type and customer-history data.

**Configurable Equipment.** A pricebook entity type for equipment with variants (size, fuel, efficiency tier). One logical equipment item with picklists for variant axes, rather than a separate SKU per variant. Requires three things: an underlying equipment-type definition, the parent service it sells through, and the variant equipment records linked to it. *(See `02-pricebook-reference.md` §5.)*

**Configuration domain.** ServiceTitan's data layer that determines how operational data is classified — job types, forms, tags, skills, custom fields, BUs, zones, AdCap rules, marketing campaigns, tracking numbers. Almost every audit and cleanup workstream targets this domain.

**Contact Hub.** ServiceTitan's centralized, polymorphic Contact model. Contact records exist autonomously and link via M:N association objects to Customer and Location records. Updating a contact's details cascades automatically to every linked record. *(See `07-crm-data-model-and-deduplication.md` §4.)*

**Cowork (CW1–CW8).** A common pattern for organizing ST cleanup work — structured, numbered working sessions with explicit dependencies. One workstream at a time; parallel runs cause Chrome-extension thrash and silent partial writes. *(See `09-anti-patterns-and-failure-modes.md` §4.)*

**CSM — Customer Success Manager (ServiceTitan's).** ST's primary support contact for tenant-level questions.

**CSR — Customer Service Representative.** Inbound call taker. The 10-step inbound rubric for AI CSR scoring is in `08-csr-scoring-and-ai-tools.md` §3.

**Custom Fields.** Tenant-defined extensions on Customers, Locations, Jobs, Equipment. Returned in API JSON responses as a nested array of `{typeId, name, value}` objects under the `customFields` property. ETL pipelines must pivot these into flat columns. **Pick lists over free text** is the universal rule — pick lists become AI-friendly dimensional data; free text becomes regex hell. *(See `04-job-types-forms-tags.md` §6.)*

**`customFieldsUpdateMode`.** Write parameter on job-type PATCH and POST (ST-77.2). Controls whether the `customFieldTypeIds` array replaces or merges existing custom field type assignments on the job type. Values: `Replace` (default — array overwrites); `Merge` (array contents are added without removing existing). Use `Merge` when adding a field type to a job type in isolation. *(See `04-job-types-forms-tags.md` §2.1.)*

---

## D

**Data Export API.** ServiceTitan API endpoints under the `/export/` path that return complete unfiltered snapshots including active, inactive, and merged records. Used for warehousing and dedup. Distinct from the Transactional APIs (which filter and return small targeted payloads).

**Data Monster.** A tenant whose configuration is so clean and structured that AI predictive analytics can accurately forecast capacity, revenue, and growth without fighting against noise. The thesis behind this entire reference series. *(See `00-overview-and-naming.md` §1.)*

**Default services.** The set of services that auto-attach to every new job of a given Job Type. Not exposed by the API — must be scraped from the UI Job Type editor's Default Services tab. *(See `04-job-types-forms-tags.md` §2.7.)*

**Dispatch Pro (DP).** ServiceTitan's AI-driven dispatch optimization engine. Assigns techs based on revenue potential, tech skills, location, traffic, and current load. Per-job-type toggle (`Enable Dispatch Pro` boolean). Default ON for reactive service and repair — disabling it on those types erases the AI dispatching wedge on the highest-volume call types. *(See `03-dispatch-and-capacity.md` §6 and `09-anti-patterns-and-failure-modes.md` §1 row 5.)*

**DID — Direct Inward Dialing.** A phone number that connects callers directly to a specific person or group, bypassing a main IVR. The ST tracking-number → phone-platform-DID → staff-phone chain preserves attribution and call recording. Direct forwarding from ST tracking number to personal mobile breaks the chain. *(See `06-phone-marketing-attribution.md` §2.)*

**DNI — Dynamic Number Insertion.** A technique where the phone number displayed on a website rotates based on the visitor's traffic source, enabling per-source attribution. Toll-free DNI pools are the legitimate exception to "no orphan tracking numbers." *(See `06-phone-marketing-attribution.md` §3.5.)*

**DNU — Do Not Use.** Naming prefix for legacy items kept for historical reporting. Counterpart to the asterisk convention. `DNU-`-prefixed items should be deactivated; `*`-prefixed items are canonical/active. *(See `00-overview-and-naming.md` §3.2.)*

**Do Not Service flag.** A boolean on Customer records that gates dispatching. Critical to evaluate during a customer merge — restrictive inheritance applies (the survivor inherits the flag if any merged record carried it).

**`doNotMail` / `doNotService`.** Customer-level operational restriction flags. See **Do Not Service flag**.

**Dual-role outbound SMS conflict.** A common configuration error: a tracking number configured as both the system-wide outbound SMS sender AND as an inbound tracking number with no campaign assigned. Inbound calls or texts on this number have ambiguous attribution. *(See `06-phone-marketing-attribution.md` §3.4.)*

**`dwyerBigBoardReport`.** A boolean on the FormData object in Form 2.0 JSON. Typically `false`. Legacy display flag.

---

## E

**Equipment Types.** A pricebook taxonomy that classifies equipment by category (`AC System`, `Furnace`, `Tank Water Heater`, `Generator`, etc.). Promoted to fully API-writable in ST-76.1 — previously UI-only metadata. *(See `02-pricebook-reference.md` §4.3, §5 and `10-api-release-notes-st75-st76.md` §4.)*

**Estimate Templates.** Pre-bundled option sets (often Good/Better/Best/Ultra) that techs present in the Sales tab on Field Pro. Each tier references the same equipment SKU but bundles different accessories — enables clean tier-mix reporting. *(See `02-pricebook-reference.md` §13.)*

**`externalData` / `externalId`.** An array on customer/location/equipment records used to store third-party unique identifier keys from legacy systems or concurrent platforms (Salesforce, ERPs). The strongest dedup vector — a matching `externalId` is near-certain proof of duplication. *(See `07-crm-data-model-and-deduplication.md` §5.3.)*

---

## F

**Field Pro.** ServiceTitan's mobile tech app — the primary interface techs use on jobs. As of ST-77, job-site **recording** is handled by a separate standalone **Field Pro** app (distinct from Field Mobile). Both apps must coexist on the tech's device for recordings to be captured and linked to the job. See `08-csr-scoring-and-ai-tools.md` §1.5.

**`fieldType`.** A property on Form 2.0 unit objects that determines the rendering type. Confirmed values: `Text`, `Dropdown`, `Checkbox`, `Picture` (NOT `Photo`), `Stoplight`, `Number`, `Date`. Using an invalid value (`Photo`, etc.) causes a `400 ValidationException` on import. *(See `05-form-2-0-json-reference.md` §6.)*

**Findings — see Location Findings.**

**Form 2.0.** ServiceTitan's client-side rendered dynamic form engine. Activated by `"IsForm20": true` at the JSON document root. Supports conditional logic via a rules engine. Imported via `Settings → Operations → Forms → Import`. *(See file `05-form-2-0-json-reference.md`.)*

**`FormData`.** The top-level object inside the Form 2.0 JSON envelope containing form metadata and the nested `definition`. *(See `05-form-2-0-json-reference.md` §3.)*

**`FormTriggers`.** An array at the Form 2.0 root level supporting automatic form dispatch based on ServiceTitan events. Typically empty `[]` — trigger configuration is managed through the UI after import.

---

## G

**GAA — Get Adaptive Availability.** The CSR-facing tool inside ServiceTitan that queries AdCap and displays available slots for booking. One of the two channels (along with Scheduling Pro in Adaptive Capacity mode) that AdCap rules constrain.

**GP% — Gross Profit Percent.**

**GPM — Gallons Per Minute.** Tankless water heater throughput rating.

---

## H

**`hasConditionalLogic`.** A boolean on `definition.definition` in Form 2.0 JSON. Must be `true` for conditional logic rules to execute at runtime. Setting it to `false` (or omitting it) produces a runtime silent failure — the form imports fine but rules never fire. *(See `05-form-2-0-json-reference.md` §8 and `09-anti-patterns-and-failure-modes.md` §2.12.)*

**Haversine formula.** The standard mathematical formula for computing great-circle distance between two latitude/longitude pairs. Used in spatial deduplication of Location records. Threshold of 3–5 meters reliably classifies two records as occupying the same physical site. *(See `07-crm-data-model-and-deduplication.md` §7.1.)*

---

## I

**IPaaS — Integration Platform-as-a-Service.** Make.com, Zapier, n8n, etc. Used for orchestration across multiple systems.

**`isChargeable`.** Boolean field on invoice items, added in ST-76.2. Indicates whether the line item should count toward revenue and commission calculations. Eliminates inference logic from zero-dollar lines or pricebook flags.

**`IsAutoDispatched`.** Boolean field on jobs, added in ST-76.3. Distinguishes between jobs the system auto-placed and jobs a human dispatcher manually scheduled or removed from the auto-dispatch pool.

**`IsForm20`.** Top-level boolean in Form 2.0 JSON. Must be `true` to activate the Form 2.0 rendering engine. Without it, ST attempts to render via the legacy engine and conditional logic is silently ignored.

**`IsIncludedInPayroll`.** Boolean field on Employee and Technician payroll-settings endpoints, added in ST-75.1. The first reliable API signal for whether someone is an active payroll participant.

**`IsLocked`.** Boolean at the Form 2.0 root. When `true`, the form cannot be edited in the UI. Set to `false` for editable imports.

**IVR — Interactive Voice Response.** An automated phone menu system that routes callers based on keypresses or voice commands. **A common critical failure mode is an IVR with zero menu keys configured** — the menu plays, the caller presses any key, nothing happens, the call drops. *(See `06-phone-marketing-attribution.md` §8.1.)*

---

## J

**Jaro-Winkler distance.** A string-distance metric optimized for matching human names. Assigns heavier weights to prefix matches (typographical errors most often appear at the trailing end of names). A score > 0.92 with shared geographic association is the standard threshold for auto-merging contact duplicates. *(See `07-crm-data-model-and-deduplication.md` §7.2.)*

**Job Type.** The taxonomy of work in ServiceTitan. Drives CSR booking flow, Dispatch Board, skills routing, sold-threshold reporting, Dispatch Pro enablement, default services. Every job type binds to exactly one BU. Typical multi-trade tenant runs 30–50 active job types. *(See `04-job-types-forms-tags.md` §2.)*

---

## L

**Lace AI.** AI CSR scoring and (emerging) voice-agent platform. Common in trades.

**Lead Time.** In AdCap, the minimum advance booking time (in minutes) before a slot can be booked. Example: Lead Time 120 on a 6 PM slot = customer must book by 4 PM. Common pattern for after-hours on-call slots.

**License Type.** Field on Technicians added in ST-76.2. Values: `NonManagedTech`, `ManagedTech`, `ManagedInstaller`. Drives license-aware reporting and seat-count auditing.

**Location Findings.** A new first-class API namespace introduced in ST-76.1 (`/location-findings`). Formalizes structured "future work needed" / "tech recommendations" tracking. Supports CRUD plus attachments, urgency levels, links to source jobs, estimates, and installed equipment. *(See `10-api-release-notes-st75-st76.md` §4.)*

**Local Service Ads (LSA).** Google's Local Services Ads — Google Guaranteed program. A common high-ROI marketing channel for trades.

---

## M

**`manufactured_on`.** A date field on installed equipment records (ST-77), writable via the `/equipmentsystems/v2/` API. Captures the manufacture date of the installed unit, enabling warranty tracking and equipment aging calculations without custom fields. See `07-crm-data-model-and-deduplication.md` §11.

**Marketing Pro.** ServiceTitan's marketing automation module. Hosts Pro Campaigns (initiatives with goals, automation rules, customer enrollment) alongside Tracking Campaigns (linked to ST tracking phone numbers). *(See `06-phone-marketing-attribution.md` §10.)*

**MCP — Model Context Protocol.** Standard for connecting LLMs to tools.

**`mergedToId`.** A tombstone pointer field on Customer and Location records. When a record is merged into another, this field is stamped with the surviving record's ID; the duplicate's `active` flag flips to `false`; foreign keys on child records are re-parented. Records with `mergedToId > 0` are dead nodes; dedup pipelines must prune them before processing. PATCHing this field directly via the API does NOT trigger the cascading re-parenting — that's a system-protected internal operation. *(See `07-crm-data-model-and-deduplication.md` §6.)*

**Micro-Rule.** The Form 2.0 rules-engine constraint: maximum 2 actions per rule. More than 2 actions silently breaks the form (it imports successfully but the preview throws "Something went wrong"). Rules with N actions must be split into ⌈N/2⌉ separate rules with the same conditions. *(See `05-form-2-0-json-reference.md` §9.)*

**Mode per Day.** A Dispatch Pro configuration setting introduced in ST-77.3. Allows the Assist/Auto × Full/Light operating mode to be set per calendar day rather than globally. Common pattern: keep the current day in Assist (protect intraday dispatcher decisions), set upcoming days to Auto Light or Auto Full. See `03-dispatch-and-capacity.md` §6.7.

**Most-Restrictive-Wins.** AdCap's rule-evaluation principle: when multiple rules match a slot, the rule producing the lowest available capacity threshold governs. No explicit rule priority ranking exists. Rules layer like CSS specificity, but with "most-restrictive" instead of "most-specific." *(See `03-dispatch-and-capacity.md` §3.2.)*

---

## N

**Network Group.** A designated subset of Tenant IDs that share a single Client ID and Secret Key pair. Used by enterprise organizations managing multiple discrete BUs or franchises through the Enterprise Hub.

**NG — Natural Gas.**

**No charge.** A boolean on Job Types indicating the type does not generate revenue (warranty/recall jobs, etc.).

---

## O

**OAuth 2.0 Client Credentials.** ServiceTitan's authentication protocol for V2 API access. Three credentials: Application Key (`ak1.` prefix), Client ID, Client Secret. Token endpoint: `https://auth.servicetitan.io/connect/token`. Tokens last 16 hours; consumers must cache and refresh before expiry. *(See `01-tenant-architecture.md` §5.)*

**Orphaned tracking number.** A tracking number with no campaign assignment. Accepts inbound calls but contributes zero data to source attribution. In tenants with dozens of orphaned numbers, can make 20–40% of inbound revenue unattributable to source. *(See `06-phone-marketing-attribution.md` §3.2.)*

**`outboundCallerId`.** Field on Technicians added in ST-76.1. Significant for shops running outbound voice campaigns — eliminates a common manual lookup step.

**`ownerTypes`.** Array on Form 2.0 `FormData` controlling which ServiceTitan record types the form can be attached to. Confirmed values: `Job`, `Call`, `Equipment`, `Customer`, `Location`.

---

## P

**PCG.** Common test-account suffix; also Performance Coaching Group (a typical category of technicians excluded from AdCap calculations).

**Pipe-delimited token convention.** Naming convention for entities with multiple orthogonal dimensions. Format: `[Dimension1] | [Dimension2] | [Dimension3] | [Version]`. Used across tracking numbers, AdCap rules, marketing campaigns. *(See `00-overview-and-naming.md` §3.3.)*

**Plecto.** Dashboard platform commonly used in trades.

**PM — Preventive Maintenance.**

**Podium.** Phone + messaging platform commonly used in trades.

**Pricebook.** ServiceTitan's catalogue of services, materials, equipment, and configurable equipment. Drives every customer-facing financial transaction. The highest-leverage cleanliness target in the tenant. *(See `02-pricebook-reference.md`.)*

**`primaryVendor.primarySubAccount`.** GL ledger sub-code on equipment / material records that vendor invoices post against.

**Pro Campaign (Marketing Pro).** Active marketing initiatives with goals, spend tracking, automation rules, and customer enrollment. Distinct from Tracking Campaigns (which are passively linked to ST tracking numbers). Both appear in the same Marketing Pro campaign list — naming convention is critical to disambiguation.

---

## R

**RAG — Retrieval-Augmented Generation.** The AI pattern this entire reference series is optimized for. Each section is sized as a coherent retrievable chunk; cross-references use full section numbers so context survives chunking.

**Recirc — Recirculation pump.** A water heater accessory.

**Reconciliation report (deduplication).** The structured output from an external dedup engine identifying duplicate Customer pairs (`Primary Target: Customer ID 1001, Duplicate to Remove: Customer ID 1002`). This is the safe handoff mechanism — administrators or UI-layer scripts then run merges through the platform's native merge interface. Direct PATCH of `mergedToId` does not work. *(See `07-crm-data-model-and-deduplication.md` §6.4.)*

**Restrictive inheritance protocol.** The dedup rule for opt-out and TCPA flags during contact merges: if any single duplicate contact has a `false` value for an outreach-preference flag, the surviving record permanently inherits that restriction. Critical for TCPA compliance.

---

## S

**Scheduling Pro (SP).** ServiceTitan's customer-facing online self-service booking widget. Configured in WorkConnect (`workconnect.servicetitan.com`). Each deployment is a separate scheduler with independent settings. **SP and AdCap are independent systems for day-of-week and time-range restrictions** — AdCap can say "available" while SP is closed, and vice versa. *(See `03-dispatch-and-capacity.md` §5.)*

**Scheduling Utilization report.** A ServiceTitan report that tracks **AdCap tool adoption** (how often dispatchers used AdCap vs. manual dispatch), NOT capacity utilization. Common dashboard mistake to use this as a "capacity hours" metric.

**SEER — Seasonal Energy Efficiency Ratio.** Cooling efficiency rating.

**Service Form.** A Form 2.0 form that appears on the tech mobile app in the job workflow. The most common form type.

**Shift Type.** Classification of a technician's work schedule in ST: `Scheduled` (regular hours), `On-Call`, `Non-Managed`. Used as a filter condition in AdCap strategic rules.

**SKU — Stock Keeping Unit.** Pricebook item code. Stable; once on a customer invoice, never reused with different meaning.

**`smartFieldTag`.** A property on Text and Date fields in Form 2.0 that enables auto-population from ServiceTitan record data (job number, customer name, service address, technician name, etc.).

**Sold Threshold.** The minimum invoice amount for a job to count as "sold" in ST conversion rate reporting. Critical for sold-rate reporting; common configuration target during job-types cleanup. *(See `04-job-types-forms-tags.md` §2.4.)*

**SOP — Standard Operating Procedure.**

**ST — ServiceTitan.**

**StaticElement (Form 2.0).** A `unitType` (not a `fieldType`) that renders HTML content. No interactive component, no data collection. Used for section headers, instructions, warnings.

**Stoplight (Form 2.0 fieldType).** A three-state toggle with color coding: Green (Pass), Yellow (Monitor), Red (Fail). Best used as an output/assessment field, not a logic trigger.

**Strategic Rule (AdCap).** An individual AdCap configuration record defining conditions and a capacity action threshold. Rules are evaluated independently; most-restrictive wins. Naming convention: `[Trade | Job Group] | [Arrival Window] | [Capacity %] | [Version]`. *(See `03-dispatch-and-capacity.md` §3.4.)*

**`syncStatus`.** Field added to Credit Memos in ST-76.1. Surfaces export state without needing to maintain it in the integration's own database.

---

## T

**T&P Valve — Temperature & Pressure relief valve.** Water heater component.

**TBR — To Be Routed / To Be Reviewed.** Placeholder tag.

**TCPA — Telephone Consumer Protection Act.** US law governing opt-out compliance for marketing SMS and automated calls. Schema implications during contact merges: any opt-out flag on any duplicate must be inherited by the surviving record. *(See `07-crm-data-model-and-deduplication.md` §8.2.)*

**Technician Exclusions.** Global AdCap setting that removes specific technicians from all capacity calculations. Affects both GAA and Scheduling Pro availability display. Typical exclusion candidates: owner/operator records used administratively, PCG technicians dispatched outside standard workflow, internal QA / inspection resources, managers with tech profiles for mobile app access.

**Tenant.** ServiceTitan's account / multi-tenant boundary. Every API call must include the Tenant ID in the URL path. Native primary keys are guaranteed unique only within tenant boundaries — external warehouses must include `tenant_id` in composite primary keys to prevent collisions. *(See `01-tenant-architecture.md` §1.)*

**Time Range.** An AdCap strategic rule field restricting when the rule is active for booking by wall-clock time (e.g., 08:00–17:00). Controls booking cutoffs in the GAA and SP-in-AdCap-mode channels. **Does NOT affect Scheduling Pro independently** — SP day-of-week and time-block restrictions must be configured separately.

**Time-stamped lifecycle (Field Pro).** The five green-button progression: Dispatched → On My Way → Arrived → Working → Done. Each tap stamps a timestamp that feeds dispatch, the customer invoice, dashboards, AI tools, and forecasting. Skipped steps produce permanently bad timestamp data.

**TitanAdvisor.** ServiceTitan's built-in platform health-score system. Grades configuration completeness across key modules. Useful as a north-star metric for cleanup work — often surfaces 5–10 missing scheduled reports that materially improve operational visibility.

**Titan Intelligence (TI).** ServiceTitan's growing AI suite. Capabilities include revenue forecasting by BU, customer lifetime value scoring, churn risk flags, equipment-replacement opportunity detection, lead-source ROI modeling. Outputs are only as good as the upstream taxonomies. *(See `08-csr-scoring-and-ai-tools.md` §1.1.)*

**Tombstone — see `mergedToId`.**

**Toggle (Form 2.0).** A confirmed action type for Form 2.0 rules. Modes: `On` (show field) and `Off` (hide field). NOT a `fieldType` — `Toggle` cannot be used as a `fieldType` value.

**Tracking Campaign.** A Marketing Pro campaign linked to a ServiceTitan tracking phone number. Attribution is automatic — when a call comes through the tracking number, the resulting job is assigned the linked campaign. Distinct from Pro Campaigns. *(See `06-phone-marketing-attribution.md` §10.1.)*

**Tracking Number.** A ServiceTitan-managed phone number used for marketing attribution. Routes calls into the phone platform while recording the call and tagging it with the linked campaign. Orphaned tracking numbers (no campaign tag) are direct attribution debt.

**Transactional API.** ServiceTitan API endpoints supporting GET/POST/PATCH/DELETE on small targeted payloads. Distinct from the `/export/` Data Export API (which returns full unfiltered snapshots).

---

## U

**`useStaticPrices`.** A pricebook field on Services. **Plural with a final `s`** — the singular `useStaticPrice` is silently ignored. Tri-state: `null` = dynamic-priced (markup engine produces the invoice price); `true` = static-priced (a settable `price` field is honored); `false` = static-priced with explicit override behavior. *(See `02-pricebook-reference.md` §7.1.)*

**UUID (Form 2.0).** Every unit (root section, field, section, static element) requires a unique `id` containing a valid UUID. Format `[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}`. Only lowercase hex characters. Common invalid pattern: prefixes like `g`, `h`, `n`, `p` (parsed as `0` and trigger `400 ValidationException`). *(See `05-form-2-0-json-reference.md` §7.)*

---

## V

**Versioned suffix (`v081524`).** The naming convention for any record that may be revised over time. Format: `v` followed by `MMDDYY` (e.g., `v081524` = August 15, 2024). Applied to AdCap Strategic Rules, Estimate Templates, Form templates, occasionally Pricebook items. *(See `00-overview-and-naming.md` §3.4.)*

---

## W

**WH — Water Heater.**

**`whenMatch`.** A property at the Form 2.0 rule root. Value `"All"` = all conditions must be true (AND logic). `"Any"` has not been confirmed from live exports. Required on every rule object.

**WorkConnect.** The separate ServiceTitan administration platform (`workconnect.servicetitan.com`) where Scheduling Pro schedulers are configured and managed. Distinct from the main `go.servicetitan.com` interface.

---

## Y

**Yext.** Listings management platform; common Marketing Pro data source.

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
