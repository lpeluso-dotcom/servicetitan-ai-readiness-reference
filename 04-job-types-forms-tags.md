# 04 — Job Types, Forms, Tags, Skills, and Custom Fields

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `04-job-types-forms-tags.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Operations leaders, integration engineers, and AI tooling that classifies, routes, or analyzes job data.

This file covers the five interlocking configuration entities that determine how every job, customer, and location gets classified in ServiceTitan: **Job Types**, **Forms**, **Tags**, **Skills**, and **Custom Fields**. These five together form the **Configuration domain** — the highest-leverage cleanup target in the tenant after the Pricebook (see `00-overview-and-naming.md` §4).

For Form 2.0 JSON authoring at the schema level (the import format, conditional logic, the micro-rule constraint, error catalog), see `05-form-2-0-json-reference.md`.

---

## 1. The Five-Entity Configuration Model

| Entity | Purpose | Where it appears |
|---|---|---|
| **Job Types** | The taxonomy of work | CSR booking, Dispatch Board, reports, Scheduling Pro |
| **Forms** | Structured data capture during/after a job | Field Pro mobile, dispatchable, can be triggered |
| **Tags** | Classification flags applied to customers, locations, jobs | Filtering, segmentation, automation, dispatch board |
| **Skills** | Tech competency declarations | Dispatch Pro routing, AdCap availability |
| **Custom Fields** | Structured data on jobs and customers | Reporting, AI extraction |

**Cleaning these in the right order matters.** Tag standardization comes before job-type assignment. Job types must be stable before form triggers can be assigned. Custom fields are last because they tighten the funnel of what got captured by the steps above.

---

## 2. Job Types — Architecture and Downstream Effects

Job types are the primary classification unit for all field service work. Every job must be assigned a job type at creation. The job type then propagates through the entire workflow and drives:

- **Dispatch Pro routing** — only job types with DP enabled receive optimization
- **AdCap rule targeting** — all strategic rules filter by job type
- **Scheduling Pro categorization** — the SP widget maps job types to customer-facing categories
- **Skills-based tech matching** — job type skill requirements gate DP tech selection
- **Revenue reporting** — drives department and trade segmentation
- **Marketing attribution** — campaign-to-job-type mapping for lead source analysis
- **Form triggers** — automated form attachment depends on job type
- **Sold threshold** — conversion rate calculation varies by job type

Changes to job types propagate forward to all new jobs but **do not retroactively alter completed jobs**. This makes job type renaming and configuration changes low-risk for historical data, but careful planning is still required because dashboards and saved reports may reference job-type names.

### 2.1 Complete field reference

| Field | Type | Required | Downstream impact |
|---|---|---|---|
| Name | String | Yes | Dispatch board display, reports, SP, form triggers |
| Prefix | `*` (active) / `DNU-` (deprecated) | No | Visual identification in lists; sort order |
| Business Unit | Single-select | Yes | Revenue segmentation, BU Group capacity calculations |
| Priority | Normal / High / Urgent | Yes | Dispatch board sort order, notification triggers |
| Duration (hours) | Decimal | Yes | Dispatch Pro schedule optimization |
| Sold Threshold ($) | Decimal | Yes | Conversion rate denominator |
| Job Class | Service / Install / Sales | Yes | Broad reporting categorization |
| Tags | Multi-select | No | Dispatch board visual ID, filtering |
| Skills | Multi-select | No | DP tech matching — missing skills disable DP routing |
| Dispatch Pro Enabled | Boolean | No | Must be ON for DP optimization |
| DP Auto API | Boolean | No | Enables automated DP without dispatcher confirmation |
| Recurring Service Required | Boolean | No | Forces recurring enrollment on completion |
| Invoice Signature Required | Boolean | No | Requires customer signature before close |
| No Charge / Unconvertible | Boolean | No | Marks non-revenue job types |
| Summary / Description | Text | No | CSR-facing intake guidance, appointment notes |
| Custom Fields | Configured per field | No | Job-type-specific data collection |

### 2.2 The API vs UI gap

Some job-type fields are exposed via API; many are not. Here's the split based on what comes back from the export.

**Fields the API exposes:**
```
id, name, active, business_units, priority, default_estimate_sold_action,
duration_seconds, duration_pretty, sold_threshold, class, tags, skills,
summary, no_charge, is_smart_dispatched, enforce_recurring_service_event_selection,
invoice_signatures_required, external_data, modified_on, created_on
```

**Fields the API does NOT expose (UI-only, must scrape):**

| Field | Where it lives in the UI |
|---|---|
| `default_services` | Default Services tab — table with Name, Description, Qty, Price |
| `custom_fields` | Bottom of Details tab — and ALSO shown directly on the Job Types list page (column 5) |
| `payroll` | Payroll tab — gated by feature flag `App.Features.EnableCustomPayrollFields` (often off) |
| `auto_dispatch_pro_api` | Sub-checkbox "Automatically enable Dispatch Pro for jobs in API calls" — only appears when "Enable Dispatch Pro" is checked |
| `project_label` | Dropdown on Details tab |
| `invoice_summary_required` | Checkbox "Require an invoice summary..." on Details tab |

**Free win:** `custom_fields` is visible on the list page — capture all rows in one DOM scrape without opening any modal. (See §2.7.)

### 2.3 UI structure

- Path: **Settings (gear) → Operations → Job Types**
- URL: `https://go.servicetitan.com/#/Settings/job-types`
- Edit URL pattern: `/#/Settings/job-types/Edit/{id}` — but **direct URL navigation does not load the modal data**. You must click Edit on the row.

Three tabs in the Edit modal: **Details / Default Services / Payroll**. Active tab gets `.active` class on `<a class="btn">`. The Custom Fields editor sits at the bottom of the Details tab (not its own tab).

**Useful DOM hooks for browser automation:**
- List rows: `table tbody tr`, cells `[select, name, priority, duration, custom_fields, active, edit-button]`
- Edit button per row: `tr.querySelector('.e2e-edit')`
- Edit form: `form.form-horizontal`
- Default Services table: identifiable by headers containing both "Qty" and "Price"
- Tab panels: `.tab-pane`, the active one has `.active` and is the only visible one
- Payroll panel reveals the gating flag in its `data-bind` attribute

### 2.4 Sold threshold — conversion rate integrity

The Sold Threshold defines the minimum invoice amount for a job to count as "sold" in conversion-rate reporting. **One of the most frequently misconfigured fields and a major source of inflated conversion-rate reporting.**

Recommended threshold by job type category:

| Category | Recommended Threshold | Rationale |
|---|---|---|
| Repair / Service | Standard service fee (e.g., $86) | Every repair visit should generate at least one billable service fee |
| Maintenance (pre-paid SA) | $0.00 | Revenue recognized at SA sale, not at visit |
| Install | Minimum install labor ($200–$500+) | Below-threshold installs indicate incomplete invoicing |
| Sales / Estimate | $0.00 | Revenue tracked on follow-up job |
| Diagnostic / No-charge | $0.00 | Explicitly non-revenue |

Setting a repair job type's threshold to $0.00 means every dispatched tech who arrives and closes the job — including warranty callbacks, no-charge diagnostics, customer-declined-estimate visits — counts as "sold". This can inflate conversion rates by 15–30% in typical operations.

### 2.5 Duration accuracy and Dispatch Pro impact

DP uses Duration to calculate a tech's remaining capacity. A tech with 4 hours remaining can be considered for a 3-hour job but not a 5-hour job.

| Direction of error | DP impact |
|---|---|
| Duration too short | DP over-assigns; stacking jobs creates schedule overruns |
| Duration too long | DP under-assigns; visible capacity DP considers occupied goes unused |

Duration calibration should be reviewed quarterly using actual completion-time data from the **Average Job Duration by Job Type** report.

### 2.6 The CSR Summary template

The Summary/Description field is displayed to CSRs during booking intake. **Empty Summary forces CSRs to interpret the job-type name alone**, leading to incorrect job type selection, missing intake info, and inconsistent routing.

**Summary template structure:**
```
[Trade] [job category] for [residential/commercial] customers.
[Scope description — 1-2 sentences on what this job type covers.]
[Key intake questions or special conditions.]
```

A more structured pattern that pre-fills the booking notes:

```
[Customer reports]: ____
[Equipment/system involved]: ____
[Symptoms / when started]: ____
[Member?]: Y/N
[How heard about us]: ____
[Special access / gate code / pets]: ____
```

This is one of the highest-ROI cleanup items — pays for itself in the first week as CSR booking notes become consistently structured.

### 2.7 Default services — the diagnostic-fee mapping pattern

Each job type can have N services that auto-attach to every new job of that type. Each line is `(Name, Description, Qty, Price)`. Price defaults from the pricebook unless overridden.

**Diagnostic-fee mapping pattern** (drop diagnostics into service-call job types only — by trade × residential/commercial):

| Trade | Commercial | Residential | After-Hours Res |
|---|---|---|---|
| Electrical | EL-COM-ST ($115) | EL-RES-ST ($85) | — |
| HVAC | HV-COM-ST ($115) | HV-RES-ST ($85) | HV-RES-AH ($110) |
| Plumbing | PL-COM-ST ($115) | PL-RES-ST ($85) | — |
| Generator | GN-COM-ST ($115) | GN-RES-ST ($85) | — |

**Skip rules (no diagnostic added):**
- Install / replacement — quoted up front
- Maintenance / inspection / clean & check — flat-rate or contract
- Sales leads — free estimates
- Pre-quoted repairs (`*Repair` jobs) — diagnostic baked into the quote
- Closeout / CIC / Estimate Follow Up — admin workflows

### 2.8 Programmatic patterns for job-type extraction

**Capture custom_fields from the list page** (no modal needed):

```javascript
const rows = document.querySelectorAll('table tbody tr');
const out = Array.from(rows).map(tr => {
  const c = tr.querySelectorAll('td');
  return {
    name: c[1].innerText.trim(),
    custom_fields: c[4].innerText.trim().replace(/\s*,\s*/g, '; ')
  };
});
```

**Per-job extractor loop (for fields requiring modal):**

Per job, six actions: click Edit → wait 2s → read details + open Default Services tab → wait 1s → read services + click Cancel → wait 1s. Group ~3 jobs per browser-batch call.

```javascript
window.__clickEdit = idx =>
  document.querySelectorAll('table tbody tr')[idx].querySelector('.e2e-edit').click();

window.__readExtras = () => {
  const findCB = re =>
    Array.from(document.querySelectorAll('label'))
      .find(l => re.test(l.innerText.trim()))
      ?.querySelector('input[type=checkbox]');
  const enableDP = findCB(/^Enable Dispatch Pro$/i);
  const apiDP = findCB(/Automatically enable Dispatch Pro for jobs in API/i);
  const invSum = findCB(/^Require an invoice summary/i);
  return {
    auto_dispatch_pro_api: apiDP ? (apiDP.checked ? 'Y' : 'N') : 'N',
    invoice_summary_required: invSum?.checked ? 'Y' : 'N'
  };
};

window.__readServices = () => {
  const tables = document.querySelectorAll('table');
  let t = null;
  tables.forEach(tt => {
    if (Array.from(tt.querySelectorAll('th')).map(th=>th.innerText.trim()).includes('Qty'))
      t = tt;
  });
  if (!t) return 'none';
  const rows = Array.from(t.querySelectorAll('tbody tr')).filter(tr => {
    const inp = tr.querySelector('td:first-child input');
    return inp ? inp.value !== '' : tr.querySelector('td')?.innerText.trim().length > 0;
  });
  if (!rows.length) return 'none';
  return rows.map(tr => {
    const c = tr.querySelectorAll('td');
    const name = c[0].innerText.trim() || c[0].querySelector('input')?.value;
    const qty = c[2].querySelector('input')?.value || c[2].innerText.trim();
    const price = c[3].querySelector('input')?.value || c[3].innerText.trim();
    return `${name} (${qty} x ${price})`;
  }).join('; ');
};
```

**Feature-flag trick:** when a UI section looks empty or wrong, check the panel's `data-bind="visible: window.App.Features.<Flag>"` attribute. ServiceTitan gates many sections behind these flags — if it's off, the section renders empty rather than throwing.

**Scaling notes:**
- ~50 jobs × 6 actions = ~300 tool calls; ~17 batches of 3 jobs each. Wall-clock ~5 min total.
- Chrome extensions drop occasionally mid-batch — persist state in `window.__results` so you can resume from the next index.
- Always dump partial results to disk periodically; `window` state is lost on reload.

### 2.9 Bulk writes (e.g., applying default services)

Same modal approach as reads, but per job:

1. Click Edit on row.
2. Click Default Services tab.
3. Click Add row → search input opens.
4. Type pricebook code (e.g., `EL-COM-ST`) → wait for autocomplete.
5. Click the matching item — verify it's the unique match (typos = wrong item silently picked).
6. Confirm Qty=1 and Price reflects pricebook (don't override).
7. Click **Save** (not Cancel).
8. Verify success toast / closed modal before moving on.

**Always pilot one before bulk.** Failure modes:
- Pricebook code typo → wrong item selected.
- Account suspension → save blocked silently.
- Modal closes without saving (network blip) → row not persisted.

---

## 3. Forms

For Form 2.0 JSON schema and conditional-logic authoring, see `05-form-2-0-json-reference.md`. This section covers Forms as a **configuration entity** — types, triggers, naming, audit.

### 3.1 Form types and workflow position

| Form Type | Where it appears | Who completes | Typical use |
|---|---|---|---|
| Service Form | Tech mobile app, job record | Field tech | Post-visit documentation, system inspection findings |
| Audit Form | ST web UI | Office staff | Quality review, job closure audits |
| Vehicle Inspection | Fleet record | Tech or driver | Pre/post-shift vehicle condition |
| Custom Form | Configurable | Configurable | Any workflow requirement |

### 3.2 Form trigger architecture

Form triggers automate form attachment to jobs. Without triggers, forms must be manually added to each job — creating inconsistent documentation.

| Trigger Type | Activation | Configuration location |
|---|---|---|
| Job Type | Form attaches when a job of specified type is created | Job Type settings or Form settings |
| Status | Form appears when job reaches a specified status (e.g., Complete) | Form trigger configuration |
| Manual | No automatic attachment | Form is available for manual add |

> **Critical dependency chain:** form trigger assignment by job type **requires that the job type be in its final, stable state**. If job types are renamed, merged, or replaced after triggers are assigned, triggers may break or point to obsolete job types. **Job Types audit must precede Form Triggers audit.**

### 3.3 Naming convention

Recommended UPPERCASE naming for forms in multi-trade operations:

```
[TRADE] - [FORM TYPE] - [SPECIFICITY]
```

Examples:
```
HVAC - SERVICE REPORT - RESIDENTIAL
HVAC - MAINTENANCE INSPECTION - COMMERCIAL
PLUMBING - WATER HEATER INSTALL - CHECKLIST
ELECTRICAL - SAFETY INSPECTION - RESIDENTIAL
ALL TRADES - CUSTOMER SATISFACTION SURVEY
FLEET - PRE-SHIFT VEHICLE INSPECTION
```

**Why UPPERCASE:** visually distinct in lists, sorts predictably, less susceptible to duplicate creation. Users are less likely to create "HVAC - Service Report" alongside "HVAC - SERVICE REPORT" than they are to create "hvac service report" alongside "HVAC Service Report".

### 3.4 Form lifecycle audit protocol

Form sprawl creates maintenance burden and tech confusion (multiple forms for the same purpose appearing on mobile). Quarterly audit:

1. **Export forms with usage data** — submission count, last submission date, job-type trigger assignments.
2. **Zero-submission forms** — candidates for deactivation if unused over the audit period.
3. **Near-duplicate identification** — forms with same/similar names triggering on same job types.
4. **Untriggered forms** — forms with no trigger assignment that are neither intentionally manual nor should be automated.
5. **Apply naming convention** — rename all active forms to standard.
6. **Deactivate confirmed duplicates** — deactivate the lower-usage duplicate; retain the higher-usage one.
7. **Never delete forms with historical submissions** — past jobs reference the form record.

---

## 4. Tags

### 4.1 The two purposes of tags

1. **Dispatch board visual identification** — colored chips on job cards, enabling dispatchers to identify trade, urgency, and category at a glance without reading the full job-type name.
2. **Reporting and filtering** — filtered views in reports and dashboards.

### 4.2 Tag taxonomy

| Category | Examples |
|---|---|
| Trade tags | `HVAC`, `PLUMBING`, `ELECTRICAL`, `GENERATOR` |
| Equipment tags | `WATER HEATER`, `FURNACE`, `CONDENSER`, `MINI-SPLIT`, `PANEL` |
| Customer tags | `MEMBER`, `COMMERCIAL`, `PROPERTY MGMT`, `WARRANTY`, `VIP` |
| Operational tags | `RECALL`, `WARRANTY CALLBACK`, `PARTS ORDERED`, `INSURANCE CLAIM` |
| Marketing-source tags | `LSA`, `PPC`, `ORGANIC`, `REFERRAL`, `YEXT` |

### 4.3 Governance principles

- One primary trade tag per job type (`HVAC`, `PLUMBING`, `ELECTRICAL`, `GENERATOR`).
- Secondary function tags where useful (`INSTALL`, `EMERGENCY`, `MAINTENANCE`).
- **Customer/property-level tags** (`DO NOT SERVICE`, `MEMBERSHIP TIER`) belong on the customer/location record, **NOT on the job type**.
- Deactivate unused tags — do not delete — to preserve historical dispatch board records.

### 4.4 The standardization-first principle

> **Critical sequencing rule:** clean the tag dictionary **first**, then map tags to job types in a single pass. Doing it the other way around (assigning tags before standardizing the dictionary) produces redundant rework when the dictionary later changes.

Tag types accumulate duplicates over time (e.g., "HVAC" and "HVAC Service" and "HVAC Repair" existing as separate tags when one would suffice). Audit the tag-type list to consolidate duplicates and establish one canonical tag per category before any per-job-type assignment.

---

## 5. Skills

Skills are the join between a job type's required-competency declaration and a technician's profile. **Both Dispatch Pro and AdCap availability calculations honor skills.**

### 5.1 Common skill conventions

- **`[Trade] Service Novice`** / **`[Trade] Service Pro`** / **`[Trade] Service Master`** — competency tier.
- **`[Equipment] Equipped`** — has the physical equipment for the job type (e.g., `Drain Auger Equipped`, `Camera Inspection Equipped`).
- **`[Specialty] Certified`** — vendor or factory certification (e.g., `Trane Certified`, `Generac Certified`).

### 5.2 The skill-matching gate

In any job-type cleanup, every job type is reviewed for required skills. **Field Pro will not let a tech be dispatched to a job whose required skills they do not hold** (or, if soft-enforced, the dispatcher sees a warning).

A job type with Dispatch Pro enabled but **no Skills assigned** cannot receive optimized routing because DP has no skill criteria to match against tech capabilities. DP will either default to geographic routing only or fail to make assignments.

---

## 6. Custom Fields — The AI-Readiness Goldmine

Custom Fields on job types are how unstructured CSR booking notes become **structured, query-able data**. Every named field on a job type becomes a column in reports.

### 6.1 Standard custom fields seen across multi-trade tenants

- `Why are you using this job type` (operational hygiene)
- `BDR Service Type` (business-development reporting)
- `How Are You Planning to Pay?` (cash, credit, financing, insurance)
- `Project or Property Type` (residential, commercial, multi-unit, new construction)
- `Where is the clog?` (drain-specific)
- `Equipment Type` (per trade — panel, sub-panel, outlet, fixture for electrical)
- `Maintenance Type` (PM cycle, contract type)
- `Describe the Issue` (free-text discovery — use sparingly)

### 6.2 The Golden Rule for custom fields

> **Pick lists over free text wherever possible.**
>
> Pick lists are AI-friendly and produce clean dimensional data. Free text becomes regex hell at analysis time.

Every free-text custom field is a future cleanup project. Every pick-list field is a future column in a clean dashboard.

### 6.3 The DNU → asterisk migration pattern

Job types prefixed with `DNU-` either:
- Get renamed to `*Name` if still used.
- Get deactivated entirely if a `*` equivalent already exists.

Never delete — historical reporting depends on the records.

---

## 7. Cleanup Sequencing

The cleanup order matters. Doing these out of order creates redundant rework.

```
Step 1: Tag Type Dictionary cleanup
  ↓ (before any job-type assignment)
Step 2: Skills Dictionary review and tech profile updates
  ↓ (independent — can run in parallel with Tags)
Step 3: Job Types audit and cleanup
  ↓ (must be stable before triggers)
Step 4: Form triggers / Form audit
  ↓ (depends on stable job types)
Step 5: Custom fields review and pick-list standardization
  ↓ (final tightening pass)
```

Each step exposes findings that feed the next. Tag cleanup commonly surfaces 15–20% redundant tags. Job type cleanup surfaces deactivation candidates and Sold Threshold defaults. Form audits surface duplicates and zero-submission candidates. Custom field cleanup is where you turn free-text into pick-lists.

---

## 8. The Data-Readiness Checklist

Before turning on any AI tool — Dispatch Pro, Titan Intelligence, Adaptive Capacity, or external AI CSR scoring — run this checklist:

- [ ] Job types follow asterisk convention; DNU items deactivated.
- [ ] Every job type has a BU, skills, sold threshold, summary template, and custom fields configured.
- [ ] Tag dictionary cleaned and tags mapped to job types.
- [ ] Skills dictionary reflects current technician competencies.
- [ ] Every tech has a current shift, BU binding, and skill profile.
- [ ] AdCap Strategic Rules follow the naming grammar with active versions only.
- [ ] Tracking numbers have campaign tags (orphans deactivated or assigned).
- [ ] Forms are deduplicated, named to convention, and triggered where applicable.
- [ ] Pricebook items live in correct categories; multi-category items documented.
- [ ] Phone IVRs have keys configured and after-hours routing verified.

A tenant that passes this checklist is genuinely AI-ready. Until it passes, every AI tool is fighting noise.

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
