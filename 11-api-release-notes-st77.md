# 11 — ServiceTitan API Release Notes — ST-77 through ST-77.3

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `11-api-release-notes-st77.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Integration architects, API consumers, and AI tooling teams that read or write ServiceTitan data via the V2 API.  
> **Source:** `developer.servicetitan.io/release-notes`  
> **Period covered:** April 2026 to spring 2026

This file is a consolidated, plain-language reference for the API and platform changes shipped across the ST-77 release family. Each section reproduces the change log and adds integration-impact analysis describing what the change enables, who should care, and how it fits into a modern home-services automation stack. For the preceding release period (ST-75.1 through ST-76.3), see `10-api-release-notes-st75-st76.md`.

**Legend:** `N` = New endpoint · `E` = Existing endpoint (modified) · `P` = Platform behavior / configuration change (no API endpoint change)

Releases covered:
- **ST-77** — April 16, 2026
- **ST-77.1** — May 7, 2026
- **ST-77.2** — May 15, 2026
- **ST-77.3** — Spring 2026

---

## 1. Cross-Release Themes

Four themes emerge across the ST-77 family:

1. **Equipment API maturation.** ST-77 delivered confirmed writable fields on installed equipment (`manufacturerName`, `installedOn`, `equipmentTypeId`) and new native aging/warranty fields (`manufactured_on`, `predicted_replacement_*`). ST-77.1 added four new `jpm` endpoints for reading and writing equipment directly on jobs. Together these move the equipment record from a mostly static snapshot to a fully API-driveable service record that supports warranty tracking, lifecycle alerts, and membership-tier targeting.

2. **Estimate and proposal templates have a public API for the first time.** ST-77.2 introduced the `salestech` namespace with full CRUD for estimate templates and proposal templates. Prior to ST-77.2, any template management required browser automation. The new surface also carries an important behavioral gotcha: `items[]` on PATCH is full-replacement, and `modifiedOn` does not advance after item changes — both relevant for sync pipelines.

3. **Dispatch Pro mode and automation deepens.** ST-77.3 introduced per-day mode configuration (Assist/Auto × Full/Light), after-hours auto-dispatch, and multi-appointment optimization. The Mode per Day feature in particular changes how integrations should reason about DP state — it is no longer a single boolean, but a per-day configuration.

4. **Automation surfaces require session-reuse after Adaptive Login (ST-77.3).** The new two-step login flow in the web application does not affect OAuth 2.0 machine-to-machine authentication, but does break browser automation scripts that use credential-fill login. Session-reuse is now the required approach for any automation that targets `go.servicetitan.com`.

For background on the OAuth + tenant model, see `01-tenant-architecture.md` §5. For pricebook-specific endpoint behavior, see `02-pricebook-reference.md` §3–4.

---

## 2. Release ST-77 — April 16, 2026

### 2.1 Change Log

| N/E | Endpoint | Change Summary | New Fields | New Filters |
|-----|----------|----------------|------------|-------------|
| E | `GET {tenant}/installed-equipment/{id}`<br>`PATCH {tenant}/installed-equipment/{id}` | Confirmed writable fields for manufacturer, install date, and equipment type. `typeId` is still silently ignored on PATCH — use `equipmentTypeId`. | `manufacturerName` (write), `installedOn` (write, ISO date), `equipmentTypeId` (write) | N/A |
| E | `GET {tenant}/installed-equipment`<br>`GET {tenant}/installed-equipment/{id}` | New read-only aging and warranty fields on installed equipment response. `type` field now returns correct string (previously returned `[object Object]` in some ST versions). | `manufactured_on`, `predicted_replacement_date`, `predicted_replacement_year`, `type` (fixed) | N/A |
| P | Field Pro app | Job-site recording separated into a standalone Field Pro app (distinct from Field Mobile). Both apps must coexist on the device for recordings to link to jobs. Field Mobile App (FMA): estimate edit/sell/duplicate now available on completed jobs; Mobile Calendar added. | N/A | N/A |

### 2.2 Integration Impact Analysis

**Equipment API — writable fields:** The confirmed write surface on installed equipment (`manufacturerName`, `installedOn`, `equipmentTypeId`) closes a gap that previously required UI-only data entry. Integrations that collect equipment data at intake (customer portal, voice AI, CRM import) can now write directly to the installed equipment record without browser automation.

**Equipment API — `typeId` still silently fails:** Attempting to PATCH `typeId` returns 200 OK with no change. The correct writable surface is `equipmentTypeId`. This mirrors the existing behavior documented in `02-pricebook-reference.md` §7.2 — confirm which field name you are sending. See `09-anti-patterns-and-failure-modes.md` §2.16 for the general "write returns 200 but change doesn't appear" diagnostic.

**Equipment aging fields:** The new `manufactured_on` and `predicted_replacement_*` fields on installed equipment records enable native warranty tracking and aging-based outreach without custom fields or external calculations. Pipelines syncing installed equipment should add these fields in the next schema migration. Existing records may not populate these fields until they are touched (updated or re-synced with an ST-77+ tenant).

**Field Pro standalone recording:** Any tenant relying on automated job-site recording should verify that both Field Mobile and Field Pro are deployed to every tech's device. Recording will fail silently on devices with only Field Mobile. This is an ops-side deployment task, not an API change, but it has downstream implications for integrations that consume recording URLs or summaries from job records.

---

## 3. Release ST-77.1 — May 7, 2026

### 3.1 Change Log

| N/E | Endpoint | Change Summary | New Fields | New Filters |
|-----|----------|----------------|------------|-------------|
| N | `GET {tenant}/jobs/{id}/equipment` | Retrieve equipment items linked to a job. | `id`, `name`, `installedEquipmentId`, `serialNumber`, `warrantyStart`, `warrantyEnd` | N/A |
| N | `POST {tenant}/jobs/{id}/equipment` | Link equipment to a job. | `installedEquipmentId`, `serialNumber`, `name` | N/A |
| N | `DELETE {tenant}/jobs/{id}/equipment` | Bulk-remove equipment links from a job. | N/A | `ids` |
| N | `DELETE {tenant}/jobs/{id}/equipment/{equipmentId}` | Remove a single equipment link from a job. | N/A | N/A |
| N | `POST {tenant}/appointments/{id}/summaries` | Generate or attach a structured summary to an appointment. | `summary`, `summaryOfWork`, `appointmentId` | N/A |
| E | `GET {tenant}/jobs`<br>`GET {tenant}/jobs/{id}`<br>`GET {tenant}/export/jobs` | New `summaryOfWork` field on job responses. | `summaryOfWork` | `equipmentIds` |
| E | `GET {tenant}/appointments`<br>`GET {tenant}/appointments/{id}` | New `appointmentSummaries` array on appointment responses. | `appointmentSummaries` | N/A |
| E | `GET {tenant}/memberships`<br>`GET {tenant}/membership-types/{id}` | New `BillingScheduleType` enum value. | `BillingScheduleType: Custom` | N/A |

### 3.2 Integration Impact Analysis

**Job equipment CRUD endpoints:** ST-77.1 introduces four new `jpm` endpoints that manage the link between jobs and equipment records. This enables integrations to associate specific installed equipment with a job at booking time (e.g., "this repair call is for the furnace installed in 2019"), update equipment linkages based on tech findings, and bulk-remove incorrect links. The `equipmentIds` filter on the jobs list endpoint allows filtering jobs to only those linked to specific equipment — useful for "find all jobs associated with this installed unit" queries.

**Appointment summaries:** The new `POST /appointments/{id}/summaries` endpoint and the `appointmentSummaries` array on appointment responses formalize the structured summary layer that was previously populated only by internal ST mechanisms or AI post-call workflows. Integrations that generate appointment summaries from call recordings, tech debrief forms, or external AI tools can now write structured summaries directly to the appointment record rather than pushing to free-text notes.

**`summaryOfWork` on jobs:** The new `summaryOfWork` field on the job record provides a dedicated structured field for post-job work descriptions, distinct from the general `summary` field. Relevant for any integration that generates job summaries from form submissions, tech debrief, or AI analysis — write to `summaryOfWork` rather than overwriting `summary`.

**`BillingScheduleType: Custom` on Service Agreements:** This new enum value allows fully custom billing schedules on memberships that don't fit the standard `Monthly`, `Quarterly`, or `Annual` tiers. Relevant for tenants with negotiated commercial membership contracts. Downstream data pipelines that branch on `BillingScheduleType` should add handling for `Custom` to avoid treating it as unknown/invalid.

---

## 4. Release ST-77.2 — May 15, 2026

### 4.1 Change Log

| N/E | Endpoint | Change Summary | New Fields | New Filters |
|-----|----------|----------------|------------|-------------|
| N | `GET {tenant}/estimate-templates`<br>`GET {tenant}/estimate-templates/{id}` | Retrieve estimate templates. | `id`, `name`, `active`, `items[]`, `modifiedOn`, `createdOn` | `ids`, `active`, `modifiedOnOrAfter`, `page`, `pageSize` |
| N | `POST {tenant}/estimate-templates` | Create a new estimate template. | `name`, `active`, `items[]` | N/A |
| N | `PATCH {tenant}/estimate-templates/{id}` | Update an estimate template. **`items[]` is full-replacement.** `modifiedOn` does not advance after item changes. | `name`, `active`, `items[]` | N/A |
| N | `DELETE {tenant}/estimate-templates/{id}` | Soft-delete (sets `active = false`). `modifiedOn` advances on soft-delete but not on item PATCH. | N/A | N/A |
| N | `GET {tenant}/proposal-templates`<br>`GET {tenant}/proposal-templates/{id}` | Retrieve proposal templates. Same schema as estimate templates. | `id`, `name`, `active`, `items[]`, `modifiedOn`, `createdOn` | `ids`, `active`, `modifiedOnOrAfter`, `page`, `pageSize` |
| N | `POST {tenant}/proposal-templates` | Create a new proposal template. | `name`, `active`, `items[]` | N/A |
| N | `PATCH {tenant}/proposal-templates/{id}` | Update a proposal template. **Same `items[]` full-replacement gotcha.** | `name`, `active`, `items[]` | N/A |
| N | `DELETE {tenant}/proposal-templates/{id}` | Soft-delete. | N/A | N/A |
| N | `GET {tenant}/proposal-types` | Read-only reference list of proposal types. | `id`, `name`, `active` | N/A |
| E | `GET {tenant}/job-types`<br>`GET {tenant}/job-types/{id}`<br>`PATCH {tenant}/job-types/{id}` | New custom field type ID fields on job types. | `customFieldTypeIds` (read) | N/A |
| E | `PATCH {tenant}/job-types/{id}`<br>`POST {tenant}/job-types` | New write parameter for custom field assignment behavior. | `customFieldsUpdateMode` (`Replace` / `Merge`) | N/A |

All `salestech` endpoints use the base URL `https://api.servicetitan.io/salestech/v2/tenant/{tenantId}/` and require the same `Authorization: Bearer` + `ST-App-Key` headers as pricebook endpoints.

### 4.2 Integration Impact Analysis

**Estimate and proposal template CRUD — the most significant new surface in ST-77.2:** Prior to this release, template management required browser automation. The `salestech/v2/` namespace provides full public CRUD for estimate templates, proposal templates, and read-only access to proposal types. This enables:

- Programmatic template creation and updates (sync from an external product catalog, bulk-update pricing).
- Template lifecycle management (activate/deactivate via PATCH or soft-delete via DELETE).
- Reporting on template content without UI scraping.

**Critical `items[]` gotcha — read this before building your sync:** The `items[]` array on template PATCH is **full-replacement**, not delta-append. Any item absent from the payload is deleted from the template. There is no add-only or delta mode. Always send the complete current `items[]` array when updating a template.

Second gotcha: `modifiedOn` on the template record **does not advance** after `items[]` is changed. It advances only when the template itself is soft-deleted (DELETE endpoint, `active → false`). Consequence: any sync pipeline gating on `modifiedOnOrAfter` will silently miss item-level changes — the timestamp looks unchanged even after items are added, removed, or reordered. Force a full backfill after any template-item edit. See `02-pricebook-reference.md` §7.15 and `09-anti-patterns-and-failure-modes.md` §1 row 36.

**`customFieldTypeIds` and `customFieldsUpdateMode` on Job Types:** Two new fields on the Job Types API enable programmatic management of custom field assignments. `customFieldsUpdateMode` controls whether a write replaces or merges the existing assignment set — use `Merge` when adding a single custom field type without disturbing existing assignments. Use `Replace` (default) when the payload is the definitive complete set.

---

## 5. Release ST-77.3 — Spring 2026

### 5.1 Change Log

| N/E | Endpoint | Change Summary | New Fields | New Filters |
|-----|----------|----------------|------------|-------------|
| P | Dispatch Pro settings | Mode per Day configuration available under Settings → Dispatch Pro → Mode per Day. Each calendar day can be set to Assist/Auto × Full/Light independently. | N/A | N/A |
| P | Dispatch Pro — after-hours | After-hours auto-dispatch: voice booking event → DP finds on-call tech → auto-assignment without dispatcher intervention. | N/A | N/A |
| P | Dispatch Pro — multi-appointment | Return visits and parts-runs now eligible for DP optimization routing (previously excluded). | N/A | N/A |
| P | Adaptive Capacity — Atlas | Atlas natural-language Strategic Rule builder is now a native ST feature. Accepts plain-English rule descriptions; produces standard strategic rules. | N/A | N/A |
| P | Adaptive Capacity — Max Drive Time | Max Drive Time Threshold is now configurable per Business Unit (in addition to tenant-level setting). | N/A | N/A |
| P | FMA — closeout | Book Follow-up appointment and Leave Unassigned are available as closeout options inside FMA. | N/A | N/A |
| P | Authentication — Adaptive Login | Two-step identity → auth-method login flow on `go.servicetitan.com`. Does not affect OAuth 2.0 API. Breaks credential-fill browser automation. | N/A | N/A |
| P | Reporting | Reputation Revenue Reporting: new reporting surface linking reputation/review data to revenue outcomes. | N/A | N/A |

### 5.2 Integration Impact Analysis

ST-77.3 is primarily a configuration and automation platform release. It does not add new REST API endpoints, but every item above has operational implications for systems built on top of ServiceTitan.

**Dispatch Pro Mode per Day:** The shift from a global DP mode to per-day configuration changes how automation should read and set DP operating state. If you have any script or dashboard that reads DP's current mode and branches on Assist vs. Auto, update it to read per-day mode rather than a single global value. The recommended pattern for tenants transitioning to Auto mode: keep today in Assist (protect already-scheduled jobs), set tomorrow and beyond to Auto Light, step up to Auto Full as confidence grows.

**After-hours auto-dispatch:** Before enabling, audit the current after-hours escalation chain. Enabling this feature on a tenant that also uses a third-party answering service or on-call platform for after-hours dispatch will cause both systems to fire simultaneously, resulting in duplicate dispatch events on the same on-call shift. Disable or reconfigure the redundant routing path before activating. See `03-dispatch-and-capacity.md` §6.8.

**Multi-appointment optimization:** Return visits and parts-runs are now eligible for DP optimization. Prerequisite: job types for these workflows must have `Enable Dispatch Pro = ON` and required skills configured. DP optimization is only possible for job types it can see — job types without DP enabled remain manually dispatched. See `03-dispatch-and-capacity.md` §6.9.

**Atlas native rule builder:** Atlas is now a native ST feature, not a third-party integration. Rule output is a standard strategic rule subject to standard most-restrictive-wins evaluation. Always review Atlas-generated rules before activating — Atlas does not prevent naming conflicts, Day of Week gaps, or double-stacking against existing rules. Use the `vMMDDYY` versioned suffix on all Atlas-generated rules. See `03-dispatch-and-capacity.md` §2.6.

**Adaptive Login:** The new two-step login flow on `go.servicetitan.com` does not affect OAuth 2.0 client-credentials authentication (machine-to-machine API calls are unaffected). However, any browser automation script that logs in by filling credentials on the ST web app will break at the intermediate identity/auth-method selection step. Migrate those scripts to session-reuse: persist the browser session state (cookies, local storage) between runs and re-authenticate only when the session expires. See `09-anti-patterns-and-failure-modes.md` §2.19 and `01-tenant-architecture.md` §5.2.

**Reputation Revenue Reporting:** New reporting surface. No API endpoint at this time; accessible through the ST reporting UI. Plan integration once an export endpoint ships.

---

## 6. Cumulative Field Additions — Quick Reference (ST-77 family)

For integrators planning a schema migration, the following fields became available across all four ST-77 releases:

### 6.1 Equipment Systems — Installed Equipment
`manufacturerName` (write), `installedOn` (write), `equipmentTypeId` (write), `manufactured_on` (read), `predicted_replacement_date` (read), `predicted_replacement_year` (read), `type` (fixed to return string, not object)

### 6.2 Dispatch — Jobs
`summaryOfWork` (read/write), `equipmentIds` (filter)

### 6.3 Dispatch — Appointments
`appointmentSummaries` (read)

### 6.4 Memberships — Membership Types
`BillingScheduleType: Custom` (new enum value)

### 6.5 Configuration — Job Types
`customFieldTypeIds` (read), `customFieldsUpdateMode` (write parameter)

---

## 7. Cumulative New Endpoints — Quick Reference (ST-77 family)

The following endpoints did not exist before ST-77. Plan integration coverage accordingly:

### 7.1 Dispatch — Job Equipment (jpm namespace)
- `GET /jpm/v2/tenant/{tid}/jobs/{id}/equipment`
- `POST /jpm/v2/tenant/{tid}/jobs/{id}/equipment`
- `DELETE /jpm/v2/tenant/{tid}/jobs/{id}/equipment` (bulk, with `ids` filter)
- `DELETE /jpm/v2/tenant/{tid}/jobs/{id}/equipment/{equipmentId}` (single)

### 7.2 Dispatch — Appointment Summaries (jpm namespace)
- `POST /jpm/v2/tenant/{tid}/appointments/{id}/summaries`

### 7.3 Salestech — Estimate Templates (entirely new namespace)
- `GET /salestech/v2/tenant/{tid}/estimate-templates`
- `GET /salestech/v2/tenant/{tid}/estimate-templates/{id}`
- `POST /salestech/v2/tenant/{tid}/estimate-templates`
- `PATCH /salestech/v2/tenant/{tid}/estimate-templates/{id}`
- `DELETE /salestech/v2/tenant/{tid}/estimate-templates/{id}`

### 7.4 Salestech — Proposal Templates (salestech namespace)
- `GET /salestech/v2/tenant/{tid}/proposal-templates`
- `GET /salestech/v2/tenant/{tid}/proposal-templates/{id}`
- `POST /salestech/v2/tenant/{tid}/proposal-templates`
- `PATCH /salestech/v2/tenant/{tid}/proposal-templates/{id}`
- `DELETE /salestech/v2/tenant/{tid}/proposal-templates/{id}`

### 7.5 Salestech — Proposal Types (read-only reference)
- `GET /salestech/v2/tenant/{tid}/proposal-types`

---

## 8. Migration & Adoption Recommendations

### 8.1 Adopt immediately (clear, high-value, low-risk)

1. **Add `manufactured_on` and `predicted_replacement_*` to installed-equipment schema.** These fields populate on touch after ST-77. Low friction, high value for lifecycle reporting.
2. **Switch `typeId` writes to `equipmentTypeId`.** If any integration attempts to PATCH `typeId` on installed equipment, rename to `equipmentTypeId`. The silent 200 OK failure mode from `typeId` is not new, but confirming the fix is now more urgent given the expanded equipment write surface.
3. **Add `BillingScheduleType: Custom` handling.** Any pipeline that branches on billing schedule type should handle `Custom` to avoid unknown-value errors.
4. **Add `summaryOfWork` to job schema.** The new dedicated field for post-job summaries replaces the pattern of writing to free-text `summary`. Schema migration is additive.

### 8.2 Adopt deliberately (high-value but requires design work)

5. **Evaluate the salestech template CRUD API.** If your integration currently manages estimate templates or proposal templates via browser automation, migrate to the `salestech/v2/` endpoints. Before migrating: read §4.2 carefully — the `items[]` full-replacement behavior and `modifiedOn` non-advancement are both non-obvious.
6. **Audit after-hours dispatch chain before enabling after-hours auto-dispatch.** Do not activate without verifying that your existing after-hours escalation chain won't create duplicate dispatch events. See §5.2 and `03-dispatch-and-capacity.md` §6.8.
7. **Migrate credential-fill browser automation to session-reuse.** Adaptive Login makes this required, not optional, for any script targeting `go.servicetitan.com`. Session-reuse is a more robust architecture regardless — plan the migration now rather than after the first failure in production.

### 8.3 Evaluate (depends on tenant configuration)

8. **Mode per Day dashboarding.** If you have a DP-mode dashboard, update it to read per-day state rather than global state.
9. **Atlas-generated rules.** Add to your strategic-rules audit protocol: explicitly check for Atlas-generated rules that lack the `vMMDDYY` versioned suffix or have Day of Week fields left blank (which means all days including weekends). See `03-dispatch-and-capacity.md` §3.7.
10. **Job equipment endpoints.** Adopt if your integration associates equipment to jobs at booking time or needs to audit which equipment was serviced on a given job.

---

## 9. Closing Notes

The ST-77 family continues the API maturation arc visible in ST-75 and ST-76: structured records replacing freeform data (estimate templates, appointment summaries), write surfaces replacing read-only metadata (installed equipment fields), and workflow automation replacing UI-only operations (template management, after-hours dispatch). The ST-77.3 platform changes — Mode per Day, Atlas as native, Adaptive Login — signal that ServiceTitan is now treating the Dispatch Pro and Adaptive Capacity configuration layers as first-class operational surfaces, not just setup wizards.

For the preceding release period, see `10-api-release-notes-st75-st76.md`. Subscribe to release notifications at the ServiceTitan Developer Portal to receive future announcements as they ship.

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Source: developer.servicetitan.io/release-notes — releases ST-77 through ST-77.3. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
