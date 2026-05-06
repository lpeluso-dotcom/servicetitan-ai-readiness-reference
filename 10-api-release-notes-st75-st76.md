# 10 — ServiceTitan API Release Notes — ST-75.1 through ST-76.3

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `10-api-release-notes-st75-st76.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Integration architects, API consumers, and AI tooling teams that read or write ServiceTitan data via the V2 API.  
> **Source:** `developer.servicetitan.io/release-notes`  
> **Period covered:** November 2025 to March 2026

This file is a consolidated, plain-language reference for every API change shipped across the four most recent ServiceTitan platform releases at time of writing. Each release section reproduces the change log from the ServiceTitan Developer Portal and adds an integration-impact analysis describing what the change enables, who should care, and how it fits into a modern home-services automation stack (Make.com, Cloudflare Workers, BI dashboards, voice AI, etc.).

**Legend:** `N` = New endpoint · `E` = Existing endpoint (modified)

Releases covered:
- **ST-75.1** — November 12, 2025
- **ST-76** — January 26, 2026
- **ST-76.1** — February 16, 2026
- **ST-76.2** — March 4, 2026
- **ST-76.3** — March 16, 2026

Two additional releases shipped during this window — ST-75.2 (November 20, 2025) and ST-75.3 (December 16, 2025) — and are not covered here. Subscribe to the ServiceTitan Developer Portal email list to receive future release announcements.

---

## 1. Cross-Release Themes

Four themes emerge across this six-month period:

1. **Deep technician and employee data enrichment.** ST-76.1 dramatically expanded the Settings API surface for both Employees and Technicians, exposing fields that previously required UI-only access — hourly rate, payroll profile, manager hierarchy, mobile phone, skills, positions, start/termination dates, and more. Combined with new payroll-enabled flags in ST-75.1, this moves ServiceTitan toward being a fully API-driveable HRIS source-of-truth.

2. **New Location Findings API as a first-class structured opportunity record.** ST-76.1 introduced an entirely new namespace — `/location-findings` — with full CRUD plus attachment management. This formalizes what many shops have been tracking informally as "tech recommendations" or "future work needed" notes. Findings are now first-class API objects with urgency levels, recommended solutions, linked equipment, and linked estimates/jobs.

3. **Accounting workflow automation, especially around credit memos and bank deposits.** ST-75.1 added the Bank Deposits API (list, retrieve transactions, mark-as-exported). ST-76.1 added Credit Memo update, custom fields, splits, and mark-as-exported endpoints. Both are clear signals that ServiceTitan is investing in making the export-to-GL handoff fully API-driven for accounting integrations.

4. **Equipment Types are now API-writable for the first time.** ST-76.1 promoted Equipment Types from read-only metadata to a full CRUD resource. Combined with new `equipmentTypeId` and `status` fields on Installed Equipment create/update operations, integrators can now provision their own equipment taxonomy programmatically.

For background on the OAuth + tenant model these endpoints sit on, see `01-tenant-architecture.md` §5. For pricebook-specific endpoint behavior, see `02-pricebook-reference.md` §3–4.

---

## 2. Release ST-75.1 — November 12, 2025

### 2.1 Change Log

| N/E | Endpoint | Change Summary | New Fields | New Filters |
|-----|----------|----------------|------------|-------------|
| E | `GET {tenant}/export/business-units`<br>`GET {tenant}/business-units`<br>`GET {tenant}/business-units/{id}` | Coordinate information added to Business Unit endpoints. | `Address.Latitude`, `Address.Longitude`, `Address.IsManualCoordinates`, `Address.IsMilitary` | N/A |
| N | `POST {tenant}/technician-ratings`<br>`GET {tenant}/technician-ratings`<br>`GET {tenant}/export/technician-ratings` | New endpoints to set technician ratings, query technician ratings, and export. | All new | All new |
| N | `GET {tenant}/bank-deposits`<br>`GET {tenant}/bank-deposits/{id}/transactions`<br>`POST {tenant}/bank-deposits/markasexported` | New endpoints to retrieve bank deposits, retrieve bank deposit transactions, and mark deposits as exported. | All new | All new |
| E | `GET {tenant}/payments` | New field on each payment specifies the deposit associated with the payment. New filter to search by deposit ID. | `depositId` | `depositIds` |
| E | `PATCH {tenant}/payments/{id}` | New field allows applying a payment to an invoice based on invoice number rather than invoice ID. | `splits.invoiceNumber` | N/A |
| E | `PUT {tenant}/employees/{employee}/payroll-settings`<br>`GET {tenant}/employees/{employee}/payroll-settings`<br>`PUT {tenant}/technicians/{technician}/payroll-settings`<br>`GET {tenant}/technicians/{technician}/payroll-settings` | New field indicates whether an employee or technician currently has payroll enabled. | `IsIncludedInPayroll` | N/A |
| E | `GET {tenant}/activities`<br>`GET {tenant}/activities/{id}` | New field identifies the ActivityType of an Activity. | `ActivityTypeId` | N/A |
| E | `GET {tenant}/activities`<br>`GET {tenant}/activities/{id}` | GPS coordinates at start and end of an Activity. | `StartCoordinate`, `EndCoordinate` | N/A |
| E | `GET {tenant}/memberships`<br>`GET {tenant}/membership-types`<br>`GET {tenant}/recurring-services`<br>`GET {tenant}/recurring-service-types` | Added standard `sort` filter to membership family endpoints. | N/A | `sort` |
| E | `GET {tenant}/recurring-service-events` | Added standard date and sort filters. | N/A | `modifiedOnOrAfter`, `modifiedBefore`, `sort` |
| E | `POST {tenant}/customers` | New fields allow setting tax exempt status and payment term at customer creation time. | `TaxExempt`, `PaymentTermId` | N/A |
| N | `POST {tenant}/locations/{id}/preferredtechnician/{preferredTechnicianId}`<br>`GET {tenant}/locations/{id}/preferredtechnician` | New endpoints to set and retrieve the preferred technician on a given location. | `TechnicianId` | N/A |
| E | `POST {tenant}/jobs/{job}/timesheets`<br>`PUT {tenant}/jobs/{job}/timesheets/{id}` | When Flexible Timekeeping is enabled, legacy timesheet Create/Update endpoints throw an error: *"Timesheet action cannot be approved because flexible timekeeping is enabled."* | N/A | N/A |

### 2.2 Integration Impact Analysis

**High-impact:** Bank Deposits API and Technician Ratings API are both brand-new namespaces.

The Bank Deposits API closes a long-standing gap in accounting export workflows. Previously, integrators had to reconstruct deposit groupings from individual payment records. Now there is a canonical deposit record, transactions can be retrieved per deposit, and the `markasexported` endpoint provides a clean idempotency signal for downstream GL systems (QuickBooks, Sage Intacct, NetSuite, etc.). The corresponding `depositId` field added to the Payments endpoint allows existing payment-sync pipelines to enrich payment records with their deposit grouping without a second API roundtrip.

The Technician Ratings API formalizes an opportunity area that most shops handle through CSR follow-up calls or third-party survey platforms. With POST, GET, and Export endpoints all shipping simultaneously, integrators can now write rating data into ServiceTitan from external survey tools (NiceJob, Listen360, post-call surveys from voice AI platforms, etc.) and then read those ratings back through the Export endpoint for performance dashboards.

**Medium-impact for HRIS integrations:** The new `IsIncludedInPayroll` flag on both Employee and Technician payroll-settings endpoints is the first reliable API signal for whether someone is an active payroll participant versus a deactivated/historical record. Combined with start/termination date fields shipping later in ST-76.1, this enables fully automated payroll-export filtering.

**Medium-impact for capacity and dispatch analytics:** Activity GPS coordinates at start and end (`StartCoordinate`, `EndCoordinate`) finally allow API consumers to reconstruct technician travel paths and on-site time without a separate GPS provider integration. This is significant for drive-time analysis, capacity-utilization studies, and after-action investigations of dispatch incidents.

**Workflow change to be aware of:** Tenants with Flexible Timekeeping enabled will see legacy job-timesheet POST/PUT endpoints fail with the documented error message. Integrations writing timesheet data must detect this configuration and route through the Flexible Timekeeping endpoints instead.

**Standard filter rollout:** The addition of `sort`, `modifiedOnOrAfter`, and `modifiedBefore` to the Memberships family is a quiet but important upgrade. Previously, incremental sync of memberships, recurring services, and recurring service events required either full export or client-side filtering. With these standard filters in place, every membership-family resource now supports the same incremental-sync pattern as Jobs, Customers, and Invoices.

**Customer creation enrichment:** Setting `TaxExempt` and `PaymentTermId` at customer-creation time eliminates a common two-step pattern where integrators created a customer, then immediately PATCHed it to set the correct payment terms. Particularly relevant for commercial-customer onboarding flows.

**Location preferred technician:** A small but high-leverage addition for tenants using preferred-tech routing. Previously this was a UI-only setting; now external scheduling systems and voice AI can write this preference based on past job history or customer requests.

**Business Unit coordinates:** The added latitude/longitude fields enable mapping BUs onto a service-territory map programmatically. Useful for territory analysis, BU-level service area visualization on dashboards, and routing optimization that needs to know each BU's home base.

---

## 3. Release ST-76 — January 26, 2026

### 3.1 Change Log

| N/E | Endpoint | Change Summary | New Fields | New Filters |
|-----|----------|----------------|------------|-------------|
| E | `GET {tenant}/appointments`<br>`GET {tenant}/appointments/{id}` | New `active` field and query parameter added to GET Appointments transactional endpoints. | `active` | `active` |

### 3.2 Integration Impact Analysis

ST-76 was an unusually small release — a single endpoint pair received an `active` filter and field. This brings the Appointments resource into alignment with the rest of the platform, which has long supported `active=true|false` filtering on most collection endpoints.

**Practical impact:** Appointment-sync jobs that needed to filter out canceled or deleted appointments previously had to do this client-side after retrieval. Now the filter happens server-side, reducing payload size and improving sync performance — particularly for tenants with high appointment volume or long-running historical exports.

For existing integrations: no breaking change. The `active` field is additive and the default behavior of the GET endpoints is unchanged. New integrations should incorporate `active=true` into incremental sync logic to skip canceled appointments unless they are explicitly required for reporting.

---

## 4. Release ST-76.1 — February 16, 2026

### 4.1 Change Log

| N/E | Endpoint | Change Summary | New Fields | New Filters |
|-----|----------|----------------|------------|-------------|
| N | `GET {tenant}/export/estimates` | Standardized estimates export endpoint. The old `/estimates/export` is now deprecated but both will continue to work. | N/A | N/A |
| E | `GET {tenant}/employees`<br>`GET {tenant}/employees/{id}`<br>`GET {tenant}/export/employees` | Substantially expanded employee response payload. | `overtimeProfileId`, `phone`, `agentId`, `hourlyRate`, `overtimeMode`, `managerId`, `firstName`, `lastName`, `home`, `startDate`, `terminationDate` | N/A |
| E | `GET {tenant}/technicians`<br>`GET {tenant}/technicians/{id}`<br>`GET {tenant}/export/technicians` | Substantially expanded technician response payload. | `appointmentId`, `bio`, `commissionRate`, `currentValue`, `firstName`, `lastName`, `hourlyRate`, `jobId`, `location`, `memo`, `mobilePhone`, `outboundCallerId`, `payrollId`, `payrollProfileId`, `shiftEnd`, `shiftStart`, `soldByRate`, `startDate`, `status`, `positions`, `skills` | N/A |
| E | `GET {tenant}/export/membership-types`<br>`GET {tenant}/membership-types`<br>`GET {tenant}/membership-types/{id}` | Tag type IDs added to membership type endpoints. Returned by default in export and by-ID; opt-in for the list endpoint. | `TagTypeIds` | `IncludeTags` |
| N | `GET {tenant}/technician-skills`<br>`GET {tenant}/technician-skills/{id}`<br>`POST {tenant}/technician-skills`<br>`PATCH {tenant}/technician-skills`<br>`DELETE {tenant}/technician-skills/{id}` | New endpoints for managing technician skills as a first-class resource. | `Id`, `Name`, `Active`, `CreatedOn`, `ModifiedOn` | N/A |
| E | `POST {tenant}/installed-equipment`<br>`PATCH {tenant}/installed-equipment/{id}` | EquipmentTypeId and Status can be specified at create/update time. | `EquipmentTypeId`, `Status` | N/A |
| N | `POST {tenant}/equipment-types`<br>`PATCH {tenant}/equipment-types/{id}` | New endpoints for creation and updates of Equipment Types. | `Id`, `Name`, `Active`, `CreatedOn`, `ModifiedOn` | N/A |
| N | `GET {tenant}/equipment-types`<br>`GET {tenant}/equipment-types/{id}` | New endpoints for retrieving Equipment Types. | All new | N/A |
| E | `GET {tenant}/trucks`<br>`GET {tenant}/warehouses` | New optional `InventoryTemplateId` field returned. | `InventoryTemplateId` | N/A |
| E | `GET {tenant}/adjustments`<br>`GET {tenant}/transfers`<br>`GET {tenant}/export/adjustments`<br>`GET {tenant}/export/transfers` | Cost fields added to inventory movement responses. | `Cost`, `TotalCost` | N/A |
| N | `GET {tenant}/inventory-templates` | New endpoint returns Inventory Template details with full filtering. | `Id`, `Name`, `Active`, `Type`, `CreatedOn`, `ModifiedOn`, `IsConsignment`, `LocationIds`, `Items` | `Ids`, `Page`, `PageSize`, `IncludeTotal`, `Active`, `CreatedBefore`, `CreatedOnOrAfter`, `ModifiedBefore`, `ModifiedOnOrAfter`, `Sort` |
| N | `GET {tenant}/location-findings` | New endpoint to retrieve Location Findings (list view). | `id`, `locationId`, `name`, `description`, `recommendedSolution`, `internalNotes`, `installedEquipmentId`, `urgencyLevel`, `sourceJobId`, `archived`, `createdBy`, `modifiedBy`, `createdOn`, `modifiedOn` | `ids`, `createdBefore`, `createdOnOrAfter`, `modifiedBefore`, `modifiedOnOrAfter`, `active`, `locationId`, `sourceJobId`, `installedEquipmentId` |
| N | `GET {tenant}/location-findings/{id}` | Detail view of a Location Finding. Adds Status, Attachments, LinkedEstimateIds, LinkedJobIds. | All list fields plus `status`, `attachments[]`, `linkedEstimateIds`, `linkedJobIds` | N/A |
| N | `POST {tenant}/location-findings` | Create a new Location Finding. | `locationId`, `name`, `desc`, `urgencyLevel`, `recommendedSolution`, `internalNotes`, `installedEquipmentId`, `sourceJobId`, `attachments[]` | N/A |
| N | `PATCH {tenant}/location-findings/{id}` | Update a Location Finding (attachments managed via separate endpoints). | `name`, `description`, `recommendedSolution`, `internalNotes`, `installedEquipmentId`, `urgencyLevel`, `sourceJobId`, `archived`, `active` | N/A |
| N | `GET {tenant}/location-findings/{id}/attachments` | Retrieve list of attachments associated with a finding. | `id`, `fileName`, `url`, `type` | `ids` |
| N | `POST {tenant}/location-findings/{id}/attachments` | Associate a new attachment with a finding. | `fileName`, `url`, `type` | N/A |
| N | `GET {tenant}/location-findings/{id}/attachments/{attachmentId}` | Retrieve a specific attachment. | `id`, `fileName`, `url`, `type` | N/A |
| N | `PATCH {tenant}/location-findings/{id}/attachments/{attachmentId}` | Update a specific attachment. | `fileName`, `url`, `type` | N/A |
| N | `DELETE {tenant}/location-findings/{id}/attachments/{attachmentId}` | Delete a specific attachment. | N/A | N/A |
| N | `POST {tenant}/assets`<br>`GET {tenant}/assets` | Upload and download asset files to be associated with location findings. | All new (form file upload) | N/A |
| N | `PATCH {tenant}/credit-memos/custom-fields` | Update custom fields for specified credit memos. | N/A | N/A |
| N | `GET {tenant}/credit-memos/custom-fields` | Retrieve all custom fields for a credit memo. | N/A | N/A |
| E | `GET {tenant}/credit-memos` | `syncStatus` field added to response. | `syncStatus` | N/A |
| N | `PATCH {tenant}/credit-memos/{id}` | Update credit memo fields. | `active`, `summary`, `date`, `businessUnitId` | N/A |
| N | `POST {tenant}/credit-memos/{id}/splits` | Add new credit memo splits. Returns all splits (existing + new). | `splits[{invoiceId, amount}]` | N/A |
| N | `POST {tenant}/credit-memos/markasexported` | Mark credit memo as exported, assign external ID and message. | `ExternalId`, `externalMessage` | N/A |

### 4.2 Integration Impact Analysis

ST-76.1 is the largest release in this period and the most consequential for integration architects. Five major shifts are worth calling out.

**1. Location Findings is a brand-new first-class namespace.** The introduction of structured Location Findings — with full CRUD, attachments, asset uploads, links to source jobs, links to estimates, links to installed equipment, and explicit urgency levels — is one of the most significant API additions in years. This formalizes what shops have historically tracked as freeform tech notes, photos in the customer file, or unstructured "future work" tags. With Findings now structured:

- Voice AI agents that perform follow-up calls can read findings programmatically and reference them by name with the customer.
- Scheduled-maintenance dispatch logic can query findings by `urgencyLevel` and surface deferred work that needs to convert to estimates.
- Dashboards can compute "deferred revenue opportunity" KPIs by aggregating open findings and their associated recommended solutions.
- Marketing automation can trigger personalized email follow-ups when a finding is created with a particular urgency level (e.g., "your technician noted a critical safety issue").

**2. Equipment Types are now fully API-writable.** Before ST-76.1, Equipment Types were UI-managed metadata. Now integrators can create, retrieve, update, and list them through the API. Combined with the new `EquipmentTypeId` and `Status` fields on Installed Equipment create/update operations, this enables programmatic equipment-taxonomy management. Use cases include: bulk-importing equipment types from a manufacturer catalog; auto-creating equipment types when a previously unseen equipment family appears on a job; deactivating obsolete equipment types as part of a pricebook rebuild.

**3. Massive expansion of Settings → Employees and Technicians payload.** The ST-76.1 changes to `/employees` and `/technicians` are the largest single-release expansion of those endpoints in recent memory. Notable additions:

- **HRIS-relevant fields**: `firstName`, `lastName`, `phone`, `mobilePhone`, `home` (address), `startDate`, `terminationDate`, `managerId`, `agentId` — these collectively allow integrators to treat ServiceTitan's employee/technician records as a primary or secondary source-of-truth for downstream HRIS, payroll, and identity systems.
- **Payroll-relevant fields**: `hourlyRate`, `overtimeProfileId`, `overtimeMode`, `commissionRate`, `payrollProfileId`, `payrollId`, `soldByRate` — these unlock fully API-driven payroll integrations and pay-rate change auditing.
- **Operational fields on Technicians**: `bio`, `memo`, `outboundCallerId`, `shiftStart`, `shiftEnd`, `currentValue`, `appointmentId`, `jobId`, `location` (current GPS), `status`, `positions`, `skills` — these are the building blocks for live dispatch dashboards, technician performance scoring, and AI-powered routing recommendations.

The `outboundCallerId` field in particular is significant for shops running outbound voice campaigns — it eliminates a common manual lookup step.

**4. Technician Skills as a first-class resource.** Previously, skills appeared only as a derived field on technician records (and only ambiguously). With dedicated Technician Skills CRUD endpoints (`POST`, `PATCH`, `GET`, `DELETE`), skills can now be managed as standalone records and assigned to technicians programmatically. This is foundational for skills-based dispatch routing, certification tracking, and matching technicians to job types based on required competencies.

**5. Credit memo workflow automation.** ST-76.1 transformed the Credit Memos resource from a read-mostly endpoint into a full accounting workflow primitive:

- `PATCH /credit-memos/{id}` allows updating active status, summary, date, and business unit on existing credit memos.
- `POST /credit-memos/{id}/splits` allows attaching credit memo amounts to specific invoices via splits — critical for accurate AR reconciliation.
- `POST /credit-memos/markasexported` provides the same idempotency signal pattern as `bank-deposits/markasexported` from ST-75.1.
- The new `syncStatus` field surfaces export state without needing to maintain it in the integration's own database.
- Custom fields on credit memos are now both readable and writable, allowing integrators to flag credit memos with arbitrary metadata for downstream routing or reporting.

**Smaller but useful additions:**

- The `inventory-templates` endpoint plus `InventoryTemplateId` on Trucks and Warehouses enables programmatic management of truck-stock and warehouse-stock templates — useful for new-truck onboarding automations and consignment inventory flows.
- Inventory adjustments and transfers now return `Cost` and `TotalCost`, eliminating a common pattern of joining adjustments back to current item costs in client code.
- The standardized `/export/estimates` endpoint replaces the older `/estimates/export` path. Existing integrations should plan a migration before the deprecated path is eventually removed.
- Membership Types now support tag retrieval via the `IncludeTags` query parameter and `TagTypeIds` field — relevant for any segmentation or filtering logic that depends on membership-type tags.

---

## 5. Release ST-76.2 — March 4, 2026

### 5.1 Change Log

| N/E | Endpoint | Change Summary | New Fields | New Filters |
|-----|----------|----------------|------------|-------------|
| E | `GET {tenant}/technicians`<br>`GET {tenant}/technicians/{id}`<br>`GET {tenant}/export/technicians` | New field returns the type of license the technician carries. Values: `NonManagedTech`, `ManagedTech`, `ManagedInstaller`. | `LicenseType` | `LicenseType` |
| E | `GET {tenant}/invoices`<br>`GET {tenant}/export/invoices`<br>`GET {tenant}/export/invoice-items` | New boolean field indicates whether an invoice item is marked as chargeable. | `isChargeable` | N/A |

### 5.2 Integration Impact Analysis

ST-76.2 is a small but pointed release with two changes that will quietly improve the accuracy of common reporting workflows.

**`LicenseType` on Technicians.** Until this release, integrators inferred technician license type from the `IsManagedTech` boolean and other flags scattered across the technician record. The explicit enum (`NonManagedTech`, `ManagedTech`, `ManagedInstaller`) provides a clean single-field signal that can drive license-aware reporting, capacity planning by license type, and seat-count auditing for license cost allocation. The matching `LicenseType` filter allows targeted retrieval of, for example, all `ManagedInstaller` technicians for install-team-specific dashboards.

**`isChargeable` on Invoice Items.** This is a small but meaningful field for revenue-recognition accuracy. Many shops include warranty work, callback line items, or no-charge demonstration items on invoices for documentation purposes — but those line items shouldn't count toward revenue or technician commission calculations. Before ST-76.2, integrators had to infer chargeability from line-item amounts (zero-dollar lines), pricebook flags, or business-unit configuration. The explicit `isChargeable` flag eliminates that guesswork and produces cleaner revenue reports, more accurate commission calculations, and better marketing attribution numbers.

The fact that `isChargeable` ships on both the transactional list endpoint and both export endpoints (`/export/invoices` and `/export/invoice-items`) means data-warehouse pipelines should add the field to their schemas in their next migration. Existing pipelines won't break — the field is additive — but new analyses should adopt it immediately.

---

## 6. Release ST-76.3 — March 16, 2026

### 6.1 Change Log

| N/E | Endpoint | Change Summary | New Fields | New Filters |
|-----|----------|----------------|------------|-------------|
| N | `GET {tenant}/business-units/intacct` | New endpoint to query Sage Intacct dimension mappings assigned to Business Units. Previously required a manual SQL export by internal support staff. | N/A | N/A |
| E | `GET {tenant}/jobs`<br>`GET {tenant}/jobs/{id}`<br>`GET {tenant}/export/jobs`<br>`PATCH {tenant}/jobs/{id}` | Auto-dispatch boolean added to jobs request/response data. | `IsAutoDispatched` | N/A |
| N | `GET {tenant}/jobs/hold-reasons` | Ability to retrieve Job Hold reasons associated with a job. | N/A | N/A |

### 6.2 Integration Impact Analysis

ST-76.3 was a small, focused release with three changes spanning Settings, Job Planning, and Dispatch.

**`IsAutoDispatched` on Jobs.** This field finally makes auto-dispatch state visible through the API. For shops running Dispatch Pro / Adaptive Capacity Planning (ServiceTitan's AI-powered scheduling optimization), this flag distinguishes between jobs that the system placed automatically versus jobs that a human dispatcher manually scheduled or removed from the auto-dispatch pool. Reporting use cases include:

- Computing the percentage of bookings that successfully auto-dispatched without dispatcher intervention.
- Trending dispatcher override behavior — what proportion of jobs are pulled out of auto-dispatch and why.
- Identifying capacity-planning gaps: if a high percentage of jobs are *not* auto-dispatched in a particular business unit, that's a signal that auto-dispatch rules need tuning or capacity windows are misconfigured.
- Per-dispatcher KPIs: which dispatchers tend to override auto-dispatch most often, and is that correlated with on-time arrival rates or customer satisfaction?

The fact that `IsAutoDispatched` is included on the PATCH endpoint as well means integrators can programmatically pull a job out of auto-dispatch (e.g., when an external scheduling override system needs to take control) — though this should be done sparingly and with care, as it bypasses ServiceTitan's optimization layer. For background on Dispatch Pro and the AI-tools layer, see `08-csr-scoring-and-ai-tools.md` §1.

**Job Hold Reasons endpoint.** A small but useful addition. Previously, the *fact* that a job was on hold was visible (job status), but the *reason* was buried in dispatch notes. The new `GET /jobs/hold-reasons` endpoint exposes structured hold-reason data, enabling reports like "top reasons jobs went on hold this month" and automated workflows like "when a job is held for 'parts on order' for more than 48 hours, escalate to inventory team." For shops that have configured a controlled vocabulary of hold reasons, this becomes a cleaner data source than parsing free-text dispatch notes.

**Sage Intacct BU mappings endpoint.** Niche but meaningful for tenants on Sage Intacct. The note in the release log explicitly states this previously required a manual SQL export by ServiceTitan support staff — now it's a self-service API call. Any integration that needs to map ServiceTitan business units to Intacct dimensions for GL posting can now retrieve the mapping table directly rather than maintaining a hand-curated lookup. This is particularly relevant for any integration that posts journal entries, invoices, or vendor bills from ServiceTitan into Intacct.

---

## 7. Cumulative Field Additions — Quick Reference

For integrators planning a schema migration, the following fields became available across all four releases combined. Group by domain:

### 7.1 Settings — Employees
`overtimeProfileId`, `phone`, `agentId`, `hourlyRate`, `overtimeMode`, `managerId`, `firstName`, `lastName`, `home`, `startDate`, `terminationDate`, `IsIncludedInPayroll`

### 7.2 Settings — Technicians
`appointmentId`, `bio`, `commissionRate`, `currentValue`, `firstName`, `lastName`, `hourlyRate`, `jobId`, `location`, `memo`, `mobilePhone`, `outboundCallerId`, `payrollId`, `payrollProfileId`, `shiftEnd`, `shiftStart`, `soldByRate`, `startDate`, `status`, `positions`, `skills`, `IsIncludedInPayroll`, `LicenseType`

### 7.3 Settings — Business Units
`Address.Latitude`, `Address.Longitude`, `Address.IsManualCoordinates`, `Address.IsMilitary`

### 7.4 Dispatch — Jobs
`IsAutoDispatched`

### 7.5 Dispatch — Appointments
`active`

### 7.6 Accounting — Payments
`depositId` (plus `depositIds` filter), `splits.invoiceNumber` (PATCH)

### 7.7 Accounting — Invoice Items
`isChargeable`

### 7.8 Accounting — Credit Memos
`syncStatus` (plus full update, splits, custom fields, mark-as-exported endpoints)

### 7.9 CRM — Customers
`TaxExempt`, `PaymentTermId` (at create time)

### 7.10 Memberships — Membership Types
`TagTypeIds` (with `IncludeTags` filter)

### 7.11 Equipment Systems — Installed Equipment
`EquipmentTypeId`, `Status` (at create/update)

### 7.12 Inventory — Trucks, Warehouses
`InventoryTemplateId`

### 7.13 Inventory — Adjustments, Transfers
`Cost`, `TotalCost`

### 7.14 Timesheets — Activities
`ActivityTypeId`, `StartCoordinate`, `EndCoordinate`

---

## 8. Cumulative New Endpoints — Quick Reference

The following endpoints did not exist before ST-75.1. Plan integration coverage accordingly:

### 8.1 Customer Interactions
- `POST /technician-ratings`
- `GET /technician-ratings`
- `GET /export/technician-ratings`

### 8.2 Accounting — Bank Deposits
- `GET /bank-deposits`
- `GET /bank-deposits/{id}/transactions`
- `POST /bank-deposits/markasexported`

### 8.3 Accounting — Credit Memos (workflow operations)
- `PATCH /credit-memos/{id}`
- `POST /credit-memos/{id}/splits`
- `POST /credit-memos/markasexported`
- `GET /credit-memos/custom-fields`
- `PATCH /credit-memos/custom-fields`

### 8.4 CRM — Locations
- `POST /locations/{id}/preferredtechnician/{preferredTechnicianId}`
- `GET /locations/{id}/preferredtechnician`

### 8.5 Dispatch — Technician Skills
- `GET /technician-skills`
- `GET /technician-skills/{id}`
- `POST /technician-skills`
- `PATCH /technician-skills`
- `DELETE /technician-skills/{id}`

### 8.6 Equipment Systems
- `GET /equipment-types`
- `GET /equipment-types/{id}`
- `POST /equipment-types`
- `PATCH /equipment-types/{id}`

### 8.7 Inventory
- `GET /inventory-templates`

### 8.8 Findings (entirely new namespace)
- `GET /location-findings`
- `GET /location-findings/{id}`
- `POST /location-findings`
- `PATCH /location-findings/{id}`
- `GET /location-findings/{id}/attachments`
- `POST /location-findings/{id}/attachments`
- `GET /location-findings/{id}/attachments/{attachmentId}`
- `PATCH /location-findings/{id}/attachments/{attachmentId}`
- `DELETE /location-findings/{id}/attachments/{attachmentId}`
- `POST /assets`
- `GET /assets`

### 8.9 Job Planning — Jobs
- `GET /jobs/hold-reasons`

### 8.10 Settings
- `GET /business-units/intacct`

### 8.11 SalesTech
- `GET /export/estimates` (standardized replacement for `/estimates/export`)

---

## 9. Migration & Adoption Recommendations

For integration teams planning their next sprint, the following adoption priorities make sense:

### 9.1 Adopt immediately (clear, high-value, low-risk)

1. Add `isChargeable` to invoice-item ingestion pipelines and any revenue/commission calculations.
2. Add `IsAutoDispatched` to job-sync pipelines and dispatch performance dashboards.
3. Migrate any code referencing `/estimates/export` to `/export/estimates` to align with the new standardized path before the old path is eventually retired.
4. Replace any inferred-license-type logic with the explicit `LicenseType` field on Technicians.

### 9.2 Adopt deliberately (high-value but requires schema work)

5. Expand Employee and Technician schema in your data warehouse to capture the ST-76.1 field additions. Particularly important for any HRIS, payroll, or identity-management integration.
6. Add the Bank Deposits sync to accounting export pipelines if you currently reconstruct deposit groupings from individual payment records.
7. Add Credit Memo update and splits flows to accounting integrations that currently treat credit memos as read-only.
8. Build Location Findings ingestion if your organization has an existing "deferred work" or "tech recommendations" tracking system that could be replaced or augmented by the structured findings record.

### 9.3 Evaluate (depends on tenant configuration)

9. Technician Skills CRUD — adopt if you maintain technician skills outside ServiceTitan and want to push the canonical list back into the platform.
10. Equipment Types CRUD — adopt if you're rebuilding pricebook or equipment taxonomy and want to do it programmatically. See `02-pricebook-reference.md` §10.
11. Inventory Templates — adopt if you provision new trucks or warehouses through automation and want to apply standardized stocking templates.
12. Sage Intacct BU mappings — adopt only if your tenant uses Sage Intacct for GL.

### 9.4 Detect and handle (configuration-sensitive)

13. If your tenant has Flexible Timekeeping enabled and you write timesheets via API, ensure your integration handles the new error condition on legacy job-timesheet endpoints and routes through the Flexible Timekeeping endpoints instead.

---

## 10. Closing Notes

The pace of API expansion across these four releases — particularly ST-76.1 — signals that ServiceTitan continues to invest heavily in API surface as a strategic platform capability rather than a bolt-on feature. The patterns are clear: structured records replacing freeform notes (Findings), full CRUD replacing read-only metadata (Equipment Types, Technician Skills), workflow automation replacing manual exports (Bank Deposits and Credit Memo mark-as-exported), and deep enrichment of payloads to reduce N+1 query patterns (the Employee and Technician field expansions).

For integration architects, the practical implication is that integrations built today should assume continued rapid API growth and design for additive schema evolution. Pipelines that depend on rigid schemas will require frequent maintenance; pipelines that store records as JSON and project specific fields at query time will adapt more gracefully.

Subscribe to release notifications at the ServiceTitan Developer Portal to receive future release announcements as they ship.

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Source: developer.servicetitan.io/release-notes — releases ST-75.1 through ST-76.3. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
