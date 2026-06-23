# 03 — Dispatch and Capacity

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `03-dispatch-and-capacity.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Operations leaders, dispatch managers, and integration engineers configuring Adaptive Capacity (AdCap), Scheduling Pro (SP), and Dispatch Pro (DP).

This file covers ServiceTitan's three independent layers that together govern when a customer can book and how a tech is assigned: **Adaptive Capacity** (the capacity math), **Scheduling Pro** (the customer-facing online widget), and **Dispatch Pro** (the AI dispatch optimizer).

The most important thing in this file is **§5 — the AdCap vs. Scheduling Pro scope boundary**. It is the most frequently misunderstood part of the platform.

---

## 1. The Three Independent Layers

Acronyms expanded:
- **AdCap** = Adaptive Capacity (the rule-based capacity engine)
- **ACP** = Adjustable Capacity Planning (legacy predecessor to AdCap)
- **SP** = Scheduling Pro (online booking widget)
- **DP** = Dispatch Pro (AI dispatch optimizer)
- **GAA** = Get Adaptive Availability (the CSR-facing tool that queries AdCap)
- **BU** = Business Unit
- **TI** = Titan Intelligence (ST's predictive analytics suite)

| Layer | Owns | Surfaces it controls |
|---|---|---|
| **AdCap** | Capacity math — what slots are available given booked load + tech shifts + rules | CSR Get Adaptive Availability tool; SP widget *availability data* (when SP is in Adaptive Capacity mode) |
| **Scheduling Pro** | The customer-facing online widget — its display, weekend blocks, buffers | The widget itself, its temporal restrictions, its marketing attribution |
| **Dispatch Pro** | AI tech assignment for a confirmed job | The dispatch board's recommended tech assignment per job |

These are independent systems. They share inputs (technician shifts, job types, BUs, skills) but they enforce different rules and they each have their own configuration surface.

**The most consequential implication:** turning on a restriction in one layer does not turn it on in the others. AdCap can say "no Saturday bookings" while Scheduling Pro happily takes Saturday bookings online — because Scheduling Pro's weekend block is configured separately. (See §5.)

---

## 2. Adaptive Capacity (AdCap)

### 2.1 What AdCap is and what it controls

AdCap is ServiceTitan's rule-based dynamic capacity management engine. It evaluates all configured strategic rules against the current booking state and returns a set of available or unavailable time slots.

Rules are not chained or prioritized in sequence; they are evaluated independently and the **most-restrictive result wins** — the rule producing the lowest available capacity threshold governs the displayed slot.

**AdCap strategic rules only affect two booking channels:**

1. The CSR-facing **Get Adaptive Availability** tool within ServiceTitan.
2. The **Scheduling Pro** widget when configured in Adaptive Capacity mode (availability data only — see §5).

**AdCap rules do NOT constrain:**

- Dispatchers manually creating or editing jobs from the dispatch board.
- Any backend API job creation that bypasses the booking workflow.
- Job reassignment or rescheduling from the dispatch board.

This boundary is absolute. Setting Time Range to 08:00–17:00 blocks the CSR from booking after 5 PM through GAA, but a dispatcher opening the dispatch board and manually adding a job at 6 PM is completely unrestricted regardless of any AdCap configuration.

### 2.2 ACP vs. AdCap (the legacy distinction)

AdCap is architecturally distinct from its predecessor, **Adjustable Capacity Planning (ACP)**. The two systems are mutually exclusive as the primary capacity driver for Scheduling Pro. ACP uses a simpler shift-count-based model without rule logic. **AdCap is the current recommended system** for multi-trade field service operations.

Important for integrations: when reading capacity data via the Dispatch / Capacity API, confirm with the ServiceTitan CSM that the API returns Adaptive Capacity data (not just legacy ACP data) for your tenant. Some endpoints behave differently depending on which system is the primary capacity source.

### 2.3 Basic Settings

Located at **Settings → Titan Intelligence → Adaptive Capacity → Basic Settings**.

| Toggle | Default | Effect when enabled |
|---|---|---|
| Include Non-Managed Technicians' Capacity | OFF | Non-BU-Group techs appear in capacity math |
| Include On Call Technician Shifts | OFF | On-call shift hours counted as available capacity |
| Include Zones in Availability Calculation | OFF | Geographic zone filtering applied to capacity display |
| Include Business Units in Availability Calculation | OFF | BU-level capacity pooling enabled |
| Honor Business Unit Groups over Business Units | OFF | BU Group assignments take precedence over individual BU settings |

For multi-trade operations with BU Groups configured, the recommended state is: **first four toggles OFF (controlled per-rule via strategic rule conditions), fifth toggle ON.**

### 2.6 Atlas natural-language rule builder (ST-77.3)

**Atlas** is now a native ServiceTitan feature in Adaptive Capacity (no longer a third-party tool). It accepts a natural-language rule description and generates the strategic rule configuration. Example: *"Reduce HVAC Maintenance capacity by 30% in July"* produces a rule with the correct threshold, job-type scope, and activation dates.

The output is a standard strategic rule — same most-restrictive-wins evaluation, same versioning discipline (`vMMDDYY` suffix), same Day of Week requirement. **Always review an Atlas-generated rule before activating it.** Atlas does not prevent naming conflicts, expired date ranges, or double-stacking against existing rules.

### 2.7 Max Drive Time Threshold per BU (ST-77.3)

Dispatch Pro's Max Drive Time Threshold (the maximum drive time DP will accept when assigning a tech to a job) is now configurable per Business Unit, in addition to the tenant-level setting. Useful when BUs have materially different service-area sizes — HVAC Commercial might accept 45-minute drives while HVAC Residential caps at 25 minutes. Configure per-BU under the BU's DP settings rather than relying on the tenant-level default.

The "Include On Call Technician Shifts" toggle is particularly important: leaving it OFF means on-call shift capacity is excluded from the global calculation baseline. On-call availability is then introduced exclusively through strategic rules with `Shift Type = On-Call`, providing fine-grained control over which job types and windows can use on-call techs.

### 2.4 Technician Exclusions

The Technician Exclusions multi-select field in Basic Settings accepts technician records whose shifts and assigned work will be **completely excluded** from AdCap capacity calculations and from GAA results.

Typical exclusion candidates:
- Owner / operator technician records used for administrative purposes.
- Performance Coaching Group (PCG) technicians dispatched through external channels.
- Internal QA or inspection resources not dispatched via standard workflow.
- Managers who hold technician records for mobile app access but do not run service calls.

Excluded technicians remain fully functional in ServiceTitan — they appear on the dispatch board, can receive manual job assignments. Exclusion is scoped only to AdCap capacity math.

### 2.5 Advanced Settings

#### 2.5.1 Availability Threshold

A buffer (in minutes) between consecutive jobs and at the start of arrival windows. Accounts for drive time, setup, and transition overhead not captured in job duration alone.

- **Standard value:** 15 minutes.
- **Scope:** Global. Applies to all rules, all job types, all arrival windows uniformly. There is no per-rule or per-job-type override.
- **Distinct from Lead Time:** Availability Threshold buffers between bookings; Lead Time controls minimum advance booking relative to slot start. (See §2.7.)

Increase for operations with large service areas where drive-time variability is high.

#### 2.5.2 Warning for Exceeding Natural Capacity

Natural capacity = the system's baseline calculation of how many jobs a tech or BU group can handle given scheduled hours, job durations, and the availability threshold. Strategic rules can permit booking *beyond* natural capacity (intentional overbooking).

| Toggle | Standard | Effect |
|---|---|---|
| Display Exceeding Natural Capacity Warning | ON | CSRs see an alert when booking into rule-permitted overbooking |
| Allow CSR to Override the Warning | ON | The warning is informational, not a hard block |

Turning OFF Allow Override creates a hard ceiling at natural capacity regardless of strategic rules — which **defeats the purpose of rules that intentionally exceed it** (e.g., 110% on Sales rules during heavy seasons).

#### 2.5.3 Tag Selections

Allows specific ServiceTitan tag types to become AdCap-aware. Tags listed here can be referenced as conditions in strategic rules, enabling tag-based capacity filtering.

**Use case:** A tag type "New Customer" applied to customer records used to hold a portion of each window's capacity for new customers. Without the tag in Advanced Settings, it cannot be referenced in rule conditions.

**Cost:** Tag-based rules add complexity and require consistent, accurate tag application. Miscategorized tags will silently produce incorrect capacity outcomes. Use sparingly.

---

## 3. Strategic Rules

### 3.1 Rule data model

Each strategic rule is an independent configuration record evaluated against every booking attempt. Rules do not chain, inherit, or modify each other.

| Field | Type | Required | Description |
|---|---|---|---|
| Rule Name | String | Yes | Unique display name. Must be human-readable and version-tagged (§3.4). |
| Status | Enum: Active/Inactive | Yes | Only Active rules are evaluated. |
| Action Threshold | Integer (%) | Yes | Capacity utilization percentage at which the rule activates (e.g., 90 = rule fires when 90% booked). |
| Shift Type | Enum | No | Filters to a specific shift type: Scheduled, On-Call, Non-Managed. Empty = all shift types. |
| Time Range | Time–Time (HH:MM–HH:MM) | No | Wall-clock window during which the rule is active for booking. (Distinct from Lead Time — see §3.6.) |
| Day of Week | Multi-select (Mon–Sun) | No | **Empty field = all 7 days, including weekends.** (See §3.7.) |
| Lead Time | Integer (minutes) | No | Minimum advance booking time relative to slot start. |
| Activation Period | Date range or "Ongoing" | Yes | Calendar period during which the rule is Active. Expired ranges leave rules in Active status but they do not fire. |
| Job Types | Multi-select | Yes | Job types the rule governs. Rules with no job types assigned have no effect. |
| Service Area / Zones | Multi-select | No | Geographic zone scoping. |
| Business Units | Multi-select | No | Restricts to specific BUs. |

### 3.2 The most-restrictive-wins evaluation model

When a CSR or SP session requests availability for a slot, AdCap evaluates all Active rules whose conditions match (job type, shift type, zone, day, time). If multiple match:

- Each calculates the effective available capacity for that slot.
- The rule producing the **lowest** available capacity wins.
- There is no explicit rule priority ranking — restrictiveness is determined by threshold percentage and time-range configuration.

**Practical implication:** if an HVAC Service rule sets 90% threshold (blocking after 90% booked) and an HVAC Maintenance rule sets 30% threshold (blocking after 30% booked) for the same window, and both match a maintenance booking request, the 30% threshold wins. This is the mechanism that enables seasonal demand management — lower the maintenance threshold to force maintenance jobs out of high-demand windows while leaving service jobs unaffected.

### 3.3 Rules layer like CSS specificity, but with "most-restrictive" instead of "most-specific"

- Authoring multiple overlapping rules is safe — the tightest one wins.
- A specific 60% cap on Plumbing Install will override a general 90% cap on Plumbing Service.
- This enables progressive tightening without rule deletion.

### 3.4 Rule naming convention — versioned hierarchical standard

```
[Trade | Job Group] | [Arrival Window] | [Capacity %] | v[MMDDYY]
```

**Examples:**
```
HVAC Service | 8-9 AM | 90% | v031226
HVAC Maintenance | 9-12 PM | 90% | v031226
HVAC Service | 3-6 PM | 90% | v031226
Plumbing Capacity | 8-9 AM | 66% | v031226
Plumbing Capacity | 6-7 PM On-Call | 100% | v031226
Electrical Install | All Day | 90% | v031226
Electrical Sales | All Day | 110% | v031226
```

**Convention rationale:**
- **Trade | Job Group** — enables rapid visual grouping and API-friendly filtering. Separating Service from Maintenance creates independent tuning levers.
- **Arrival Window** — makes the temporal scope immediately visible without opening the rule.
- **Capacity %** — visible at a glance during audit.
- **vMMDDYY** — when rules are rebuilt seasonally, the new version makes superseded rules instantly identifiable. v031226 = March 12, 2026.

**Lifecycle discipline:** when rebuilding a rule set, **create new rules with an incremented version suffix and deactivate (do not delete) the old rules.** Preserves the audit trail of what thresholds were active during any past period.

### 3.5 Modular trade architecture

#### HVAC — Service/Maintenance split pattern

The architectural principle: **separate rules per job group per arrival window** to enable independent threshold management without cross-contaminating related categories.

Standard HVAC rule structure (8 rules for 4 arrival windows):

| Rule Name | Shift Type | Threshold | Time Range |
|---|---|---|---|
| HVAC Service \| 8-9 AM \| 90% | Scheduled | 90% | — |
| HVAC Maintenance \| 8-9 AM \| 90% | Scheduled | 90% | — |
| HVAC Service \| 9-12 PM \| 90% | Scheduled | 90% | — |
| HVAC Maintenance \| 9-12 PM \| 90% | Scheduled | 90% | — |
| HVAC Service \| 12-3 PM \| 90% | Scheduled | 90% | — |
| HVAC Maintenance \| 12-3 PM \| 90% | Scheduled | 90% | — |
| HVAC Service \| 3-6 PM \| 90% | Scheduled | 90% | 08:00–17:00 |
| HVAC Maintenance \| 3-6 PM \| 90% | Scheduled | 90% | 08:00–17:00 |

**Seasonal adjustment mechanism:** as cooling/heating season intensifies and emergency demand rises, the Maintenance threshold for individual windows is lowered independently:

```
HVAC Maintenance | 9-12 PM: threshold 90% → 30%
```

This forces maintenance jobs out of the 9 AM–12 PM window after 30% booked, reserving 70% for service/emergency, without touching the HVAC Service rule.

**HVAC Maintenance step-down example (cooling season):**

| Period | Cap % | Rationale |
|---|---|---|
| Spring shoulder | 90% | Repair demand low; maintenance can fill |
| Cooling-season ramp (April) | ~60% | Repair calls rising; protect repair capacity |
| Cooling peak (May–June) | ~35% | Maintenance must yield to break-fix demand |

**Build the step-down rules in advance** with future activation dates so they activate on time without rushed reconfiguration.

#### Plumbing — combined job group pattern

Plumbing typically does not require Service/Maintenance splitting. A single Plumbing Capacity rule per arrival window:

| Rule Name | Shift Type | Threshold | Time Range | Lead Time |
|---|---|---|---|---|
| Plumbing Capacity \| 8-9 AM | Scheduled | 66% | — | — |
| Plumbing Capacity \| 9-12 PM | Scheduled | 95% | — | — |
| Plumbing Capacity \| 3-6 PM | Scheduled | 95% | 08:00–17:00 | — |
| Plumbing Capacity \| 6-7 PM On-Call | On-Call | 100% | — | 120 min |

Notes:
- 8-9 AM at 66%: conservative early-window threshold; lower threshold creates breathing room for same-day service-call insertion.
- 3-6 PM Time Range 08:00–17:00: enforces a 5 PM booking cutoff. After 5 PM, the 3-6 PM window disappears from GAA regardless of remaining capacity.
- 6-7 PM On-Call Lead Time 120 min: enforces a 4 PM booking cutoff by requiring booking at least 2 hours before slot start.

#### Electrical — sales/install/service split pattern

```
Electrical Service & Maintenance | 12-3 PM | 90% | v031226
Electrical Service & Maintenance | 3-6 PM | 95% | v031226
Electrical On-Call | 6-7 PM | 100% | v031226
Electrical Install | All Day | 90% | v031226
Electrical Sales | All Day | 110% | v031226
```

The 110% threshold on Sales is intentional overbooking — sales/estimate appointments have higher no-show/cancellation rates, so booking beyond natural capacity maintains sell-through efficiency.

### 3.6 Time Range vs. Lead Time

These two fields are frequently confused. They solve different problems.

**Time Range (HH:MM – HH:MM):**
- The wall-clock window during which the **rule is active for booking**.
- References the *current time when the booking attempt is made*, not the slot time.
- Example: Time Range 08:00–17:00 means the rule can only be used to book slots while the clock reads 8 AM–5 PM. At 5:01 PM the rule ceases to be active and its governed slots disappear from availability.
- Use case: enforce end-of-business-day booking cutoffs.

**Lead Time (integer minutes):**
- Minimum advance booking time **relative to the specific slot start time**.
- Example: Lead Time 120 on a rule covering 6-7 PM means the 6 PM slot requires booking at least 120 minutes before 6 PM = 4 PM cutoff.
- The cutoff varies by slot — a 120-min lead time on a 7 PM slot cuts off at 5 PM.
- Use case: enforce advance booking requirements on specific slots.

**Decision matrix:**

| Requirement | Mechanism | Configuration |
|---|---|---|
| "Cannot book any slot in this window after 5 PM" | Time Range | 08:00–17:00 |
| "Must book this slot at least 2 hours in advance" | Lead Time | 120 minutes |
| "Cannot book morning slots after noon" | Time Range | 08:00–12:00 |
| "On-call slot must be booked by 4 PM" | Lead Time | 120 min (on a 6 PM slot) |

> **Trap:** Using Lead Time to enforce a wall-clock cutoff produces unpredictable results because the cutoff shifts based on slot time. Always use Time Range for wall-clock cutoffs.

### 3.7 Day of Week — the weekend booking gap

The Day of Week field is a multi-select of Monday through Sunday. **Default state when a rule is created is empty — no days selected.**

**Critical behavior:** an empty Day of Week field does **not** mean "no days" — it means **all days, including Saturday and Sunday**. This is the single most common source of unintentional weekend booking availability in AdCap-configured tenants.

Any rule without an explicit Day of Week:
- Applies to Saturday and Sunday if technicians have shifts on those days.
- Makes those slots visible to CSRs and (for availability purposes) to Scheduling Pro.
- Results in bookings appearing on days that were intended to be closed.

**Correct configuration:** every active strategic rule must have an explicit Day of Week selection. Mon–Fri operations: select Mon, Tue, Wed, Thu, Fri. Mon–Sat: add Saturday.

**Audit protocol:**
1. Export the strategic rules CSV.
2. Filter to Active.
3. Check the Day of Week column — any blank cell is a rule with no day restriction.
4. Cross-reference blank-day rules with the job types that received unintended weekend bookings.

**Important nuance:** even with correct Day of Week on all AdCap rules, **Scheduling Pro weekend booking is NOT controlled by AdCap.** See §5 for SP's independent weekend blocking.

### 3.8 Rule lifecycle and debt patterns

| Pattern | Description | Action |
|---|---|---|
| **Expired date range rules** | Activation Period end date passed; rules show Active in UI but do not fire | Deactivate; rename if needed |
| **DNU-prefixed rules** | Manually named "DNU" but never formally deactivated; still firing | Deactivate immediately |
| **Versioned duplicates** | v0309 rules built March 2026 + v031226 rules built later, both Active simultaneously; most-restrictive wins, possibly unintended | Deactivate the superseded version |
| **Unnamed / test rules** | Names like "40", "test"; created during platform testing | Delete if no historical data; deactivate otherwise |

**Cleanup protocol:**

1. Export all strategic rules to CSV.
2. Classify each rule into one of four buckets:
   - **ACTIVE — Keep:** correctly named, current-version active rules.
   - **ACTIVE — Bad Name:** active but non-standard name; needs rename.
   - **INACTIVE — Review:** inactive but intent unclear.
   - **INACTIVE — Delete:** DNU, unnamed, expired, superseded.
3. For each DELETE candidate: deactivate if any historical audit need; delete if zero usage.
4. For each ACTIVE — Bad Name: rename without changing other configuration.
5. Verify all remaining Active rules have explicit Day of Week, correct Shift Type, and valid Activation Period.

---

## 4. Scheduling Pro

### 4.1 Platform overview

Scheduling Pro (SP) is ServiceTitan's customer-facing online self-service booking widget. **It is administered through a separate web platform — WorkConnect (workconnect.servicetitan.com)** — not through go.servicetitan.com. The platforms share authentication but have separate session management.

Each deployment is called a **scheduler**. A single ServiceTitan instance can have multiple schedulers active simultaneously, each with independent configuration. Schedulers are identified by a unique key in the format `sched_[alphanumeric]` (e.g., `sched_zr5hroo2vzfzw5ug241ylpv7`).

The scheduler editor is a 3-step wizard: **(1) Review and Customize**, **(2) Reserve with Google**, **(3) Install**. All material configuration happens in Step 1.

### 4.2 Capacity mode selection

The Capacity Selection tab offers four modes:

| Mode | When to use |
|---|---|
| **Adaptive Capacity** | Recommended for operations with a configured AdCap ruleset. SP displays slots based on AdCap availability calculation. **SP honors AdCap availability results but does NOT honor AdCap Day of Week, Time Range, or Lead Time configurations** — these must be re-implemented in SP Advanced Settings (§4.3, §4.4). |
| **Adjustable Capacity Planning (ACP)** | Legacy mode. Mutually exclusive with AdCap as the SP capacity source. |
| **Business Hours** | No capacity restriction. Every slot within business hours shows as available regardless of actual booking load. Not appropriate for operations managing dispatch capacity. |
| **Custom Capacity** | Manual specification of available time blocks and tech counts. Used for fixed-slot operations or when AdCap is not configured. |

**Display Preference** sub-setting:
- *Arrival Windows:* displays availability as ranges (8 AM–12 PM). Matches AdCap arrival window configuration.
- *Business Hours:* displays specific appointment times. Less common for field service.

**First Available toggle:** ON shows the soonest available slot rather than requiring the customer to browse the calendar. Standard config = ON.

### 4.3 Buffers

Buffers prevent bookings that are too close to the current time (same-day prevention) or too far in advance.

| Parameter | Description |
|---|---|
| Buffer Name | Descriptive name |
| Service Area | Multi-select; all zones or specific |
| Job Type | Multi-select |
| Buffered By | Integer + Days/Hours. "1 Day" = no same-day booking |
| Start | DateTime |
| End | DateTime, optional. Omit for ongoing |

**Same Day Service Buffer pattern:** a buffer named "Same Day Service Buffer" set to 1 Day applied to all service areas and all job types prevents any same-day online booking, routing same-day demand through the phone/CSR channel.

The buffer is checked at the moment a customer reaches the Availability step. If the earliest slot within the buffer window is the current day, no slots for today are shown.

### 4.4 Blocked Dates

Blocked Dates are time-range rules that prevent all booking within a specified window. Unlike buffers (which push availability forward), Blocked Dates create hard blackout windows. **They are the primary and only mechanism for preventing Scheduling Pro weekend bookings.**

| Parameter | Description |
|---|---|
| Rule Name | Display name (e.g., "No Sunday", "8AM Block") |
| Service Area | Zone scope |
| Job Type | Job type scope |
| Start Date + Time | Block start |
| End Date + Time | Block end |
| Repeat | Daily or Weekly |

**Standard configuration for Mon–Fri operations:**

| Rule Name | Start Time | End Time | Repeat | Effect |
|---|---|---|---|---|
| 8AM Block | 07:00 | 08:00 | Daily | Prevents pre-business-hours bookings |
| No Saturday | 08:00 Sat | 21:00 Sat | Weekly | Blocks all of Saturday |
| No Sunday | 07:00 Sun | 21:00 Sun | Weekly | Blocks all of Sunday |

> **Critical per-scheduler isolation:** Blocked Date rules are configured **independently per scheduler**. A "No Sunday" rule in Scheduler A has zero effect on Scheduler B even if both serve the same zones and job types. **Every active scheduler must have weekend blocks explicitly configured.** This is the most common source of unintended weekend online bookings in multi-scheduler deployments.

### 4.5 Date Range and Time Zone

**Date Range:** how far in advance customers can book. Standard = 3 Months. Too short limits advance booking; too long creates uncertainty for ops without long-range staffing visibility.

**Time Zone:** must match the primary service area timezone. The sub-setting "Dynamically show customer time zone based on their location" adjusts for out-of-area visitors but can confuse single-timezone service areas. Typically OFF.

### 4.6 Booking Preferences and standard scheduler settings

| Setting | Recommended | Effect |
|---|---|---|
| Auto-advance when customer completes a section | ON | Auto-progress through booking steps |
| Send Scheduling Pro appointments as jobs to the Dispatch Board | ON | Completed bookings appear as scheduled jobs immediately |
| Include job type summary in appointment notes | ON | Job description appears in appointment notes |
| Block bookings if address is invalid | OFF | Allow booking with unverifiable addresses (rural addresses commonly fail validation) |

### 4.7 Abandoned sessions and follow-up SMS

When abandoned-session sync is enabled, sessions where a customer started but didn't complete booking are pushed to ServiceTitan after a configurable delay.

| Parameter | Standard | Description |
|---|---|---|
| Sync delay | 5 minutes | Wait before pushing |
| Send as | Lead | Pushes as a lead, not a job |
| Same-day follow-up | ON | CSR is notified to follow up |
| Tag | "SP Abandoned" or similar | Standard identification |

**Follow-up SMS** parameters:

| Parameter | Standard |
|---|---|
| Enable SMS follow up | ON |
| Phone number | Verified, non-toll-free ST tracking number |
| Send Message After | 15 minutes |
| Always follow up even if no availability | ON |
| Follow-up type | AI agent if configured, else Standard message |

**Toll-free restriction:** numbers with "Unverified" status cannot send SMS. The outbound number must be local or a verified toll-free.

### 4.8 Marketing integration

| Setting | Effect |
|---|---|
| Use Marketing Pro to assign campaigns to my jobs/bookings | ON = SP reads UTM/attribution from web session and assigns matching MP campaign |
| Default Marketing Campaign | Assigned when no attribution data matches — typically "Direct Web Traffic" |

When ON, this integrates online bookings into the same Marketing Pro attribution reporting as phone-originated jobs. See `06-phone-marketing-attribution.md` §5 for the full attribution model.

### 4.9 Session export — conversion diagnostics

Each scheduler supports a session export (XLSX) for a date range. Key analytical fields in the Sessions sheet:

| Field | Values | Use |
|---|---|---|
| Status | booked / abandoned / completed | Primary conversion metric |
| Last Step | Location / Service Selection / Availability / Contact Info / Booking Confirmation | Drop-off analysis |
| Timeslot Start | DateTime UTC | Day-of-week analysis (extract day name) |
| Category | Selected service category | Demand by category |
| Sourced from Marketing Pro | Yes/No | Attribution quality |
| Marketing Pro Campaign Name | Campaign name | Campaign-level conversion |
| Blocker Type | Reason string | Identifies blocked sessions vs. abandoned |

**Day-of-week analysis (weekend booking investigation):**

```python
import pandas as pd

df = pd.read_excel('scheduler_export.xlsx', sheet_name='Sessions')
booked = df[df['Status'] == 'booked'].copy()
booked['ts_dt'] = pd.to_datetime(booked['Timeslot Start'], utc=True)
booked['dow'] = booked['ts_dt'].dt.day_name()
weekend = booked[booked['dow'].isin(['Saturday', 'Sunday'])]
print(weekend[['Timeslot Start', 'dow', 'Category', 'Customer First Name']])
```

The `sched_[key]` in the export filename matches the WorkConnect URL, enabling traceback to the specific scheduler that allowed a weekend booking.

### 4.10 The Location step drop-off — most common conversion failure

The most common SP conversion failure is **mass abandonment at the Location step** — where customers enter zip code or address. Hundreds of sessions, near-zero completed bookings, majority abandoning at Location indicates a service area configuration problem rather than UX or demand.

**Root causes (in order of prevalence):**

1. **Zone coverage gap:** customer's zip falls outside all configured service zones in the scheduler's Service Area settings. Returns "no availability in your area"; customer exits.
2. **Zone/job-type mapping gap:** location is in a zone, but that zone has no active job types mapped for the selected category. Returns no availability.
3. **Zone lookup technical failure:** intermittent geocoding or zone-boundary errors. Some addresses in the same zip succeed while others fail.

**Diagnosis:**
- Check Blocker Type field in exported sessions.
- Cross-reference abandoned zip codes against the scheduler's service-area zone configuration.
- Test known addresses in edge zip codes using the scheduler's "Book a test job" link.

**Resolution:** expand zone coverage, ensure every zone has at least one active job type per offered category.

### 4.11 Multi-scheduler governance

ST tenants accumulate schedulers over time — campaign-specific, test, clones. Standard states:

| Status | Description | Action |
|---|---|---|
| Active | Live, embedded on website | Audit all settings quarterly |
| Draft | Created but not deployed | Either finalize or delete |
| Inactive | Disabled — not accepting bookings | Keep if historical data needed; delete otherwise |

Draft schedulers like "Copy of Copy of..." accumulate when schedulers are cloned for testing but never deployed. Delete unless actively in development.

**Standard settings template** — for consistent behavior across all active schedulers, define a standard and audit each scheduler against it. Key settings that must match:

- Capacity mode (Adaptive Capacity)
- Emergency phone number and button label
- Email required for customer info
- Abandoned session handling (5 min delay, Send as Lead, same-day follow-up, standard tag)
- Follow-up SMS enabled with consistent outbound number
- Marketing Pro attribution ON
- Same Day Service Buffer (1 Day, ongoing)
- No Saturday and No Sunday blocked dates
- 8AM Block (daily)
- Date Range (3 Months)
- Time Zone (consistent with service area)

Deviations must be intentional and documented (e.g., a scheduler for after-hours emergencies might omit weekend blocks).

---

## 5. AdCap vs. Scheduling Pro — The Scope Boundary

This is the most frequently misunderstood aspect of the platform. **Memorize this table.**

| Behavior | Controlled By | Notes |
|---|---|---|
| Slot availability (which slots show as bookable) | AdCap, when SP is in Adaptive Capacity mode | SP displays whatever AdCap calculates as available |
| **Day-of-week blocking for online bookings** | **Scheduling Pro Blocked Dates** | **AdCap Day of Week has NO effect on the SP widget** |
| Same-day booking prevention (online) | Scheduling Pro Buffer settings | AdCap has no same-day mechanism |
| Booking cutoff times for online customers | Scheduling Pro Buffer settings | AdCap Time Range does NOT affect SP cutoffs |
| After-hours online booking prevention | Scheduling Pro Blocked Dates | AdCap Time Range does NOT block SP after-hours |
| Weekend blocking for online bookings | Scheduling Pro No-Saturday/No-Sunday Blocked Dates | **ONLY** mechanism for SP weekend blocking |
| Job type visibility on SP widget | SP Job Types/Categories config | AdCap job-type scoping is invisible to SP display |

**The core principle:** AdCap governs capacity math. Scheduling Pro governs its own display, availability windows, and temporal restrictions independently.

**Common failure mode:** "Customer booked online on Sunday — but our AdCap rules don't allow Sunday." The cause is almost always: the AdCap rules correctly exclude Sunday, but Scheduling Pro has no Blocked Date rule for Sunday on that specific scheduler. Configure the SP Blocked Date.

---

## 6. Dispatch Pro — the AI dispatch optimizer

### 6.1 What Dispatch Pro is

Dispatch Pro (DP) is ServiceTitan's AI-driven dispatch optimization engine. It assigns techs to jobs based on revenue potential, tech skills, location, traffic, and current load — outperforming the dispatcher's "whoever is closest" default on the highest-volume call types.

### 6.2 Per-job-type configuration

DP is configured per job type:

- `Enable Dispatch Pro` — boolean.
- `Automatically enable Dispatch Pro for jobs in API calls` — sub-checkbox; only appears when "Enable Dispatch Pro" is checked.

### 6.3 Where DP underperforms

- Job types without declared required skills (DP defaults to "anyone").
- Tenants with stale tech-skill profiles.
- Job types with `Enable Dispatch Pro = OFF` (DP doesn't see them).
- Tenants where dispatchers manually override DP recommendations frequently (training data drift).

### 6.4 The repair-job-type override anti-pattern

A common point of friction is a request to **disable Dispatch Pro on certain repair job types** (drain clogs, electrical repair, etc.).

**The argument against the disable:** DP's value is **highest** precisely on reactive service / repair where dispatchers most often default to proximity. Disabling DP there erases the AI dispatching wedge on the highest-volume call types.

**Default recommendation:** keep DP enabled on all reactive service and repair job types unless there is a specific dispatcher-workflow reason that's been documented and signed off.

### 6.5 The DP / AdCap data-model handshake

For Dispatch Pro to optimize correctly, AdCap-side data must be clean:

1. **Skills** — every tech's skills profile must reflect actual competencies.
2. **Job-type required skills** — every job type must declare its required skills.
3. **BU bindings** — every tech and every job type must be bound to the correct BU.
4. **Zones** — if zone-based dispatch is used, every tech must have a primary zone.
5. **Tags** — Tag Selections in AdCap Advanced should reflect operational reality.

Skipping any of these starves DP of the inputs it needs.

### 6.6 The Dispatch Board AH Tech row

A common operational pattern: the **AH (After-Hours) Tech row** on the Dispatch Board is the daily reference for on-call coverage. **Before the on-call window starts**, the dispatcher updates this row with:

- Full name of the on-call tech.
- Direct phone number.
- Trade (HVAC, Plumbing, Electrical, Generator).

If the row is not updated before the on-call window starts, the answering service will not know who to contact for after-hours calls. Make this a daily checklist item with a hard cutoff.

### 6.7 Mode per Day (ST-77.3)

Dispatch Pro mode is now configured per calendar day rather than as a single global state. Each day can be set to **Assist** or **Auto**, and within Auto to **Full** or **Light**. Configuration lives under **Settings → Dispatch Pro → Mode per Day**.

Recommended pattern for operations transitioning from Assist to Auto: keep the **current day in Assist** (protect intraday dispatcher decisions from the board), allow **tomorrow and beyond to run Auto Light or Auto Full**. Review the week's mode schedule each morning before flipping. This decoupled configuration avoids the situation where a day's schedule is already well-formed and an accidental Auto-mode activation disrupts it.

### 6.8 After-hours auto-dispatch (ST-77.3)

ST-77.3 introduced an after-hours auto-dispatch path: an after-hours call (or booking event) triggers Dispatch Pro to find the on-call tech and auto-assign within seconds — without dispatcher intervention.

**Before enabling:** audit how after-hours calls currently flow to the on-call tech. If a third-party answering service, ScheduleEngine, or on-call platform already manages after-hours escalation, enabling this feature can create **duplicate dispatch events on the same on-call shift**. The system creates an assignment; the answering service also calls the tech. Evaluate the full chain before activating and disable redundant external routing if you turn this on.

### 6.9 Multi-appointment optimization (ST-77.3)

Return visits (e.g., equipment installs requiring a second appointment) and parts-runs are now candidates for Dispatch Pro optimization, not manual dispatcher placement. Prior to ST-77.3, only initial service calls were eligible.

Prerequisites:
- Job types for return visits and parts-runs must have `Enable Dispatch Pro = ON`.
- Skills must be configured on those job types — DP has no skill criteria to match against without them.
- Zones must be current, since drive-time optimization is the primary value add for multi-appointment sequences.

---

## 7. Capacity Hours vs. AdCap Adoption — A Critical Dashboard Distinction

**Critical learning that breaks naive dashboard builds.** ServiceTitan's **Scheduling Utilization** report tracks **AdCap tool adoption** (how often dispatchers used AdCap vs. manual dispatch), **NOT capacity utilization** (booked hours vs. available hours).

These are two separate data needs:

| Metric | Source | What it answers |
|---|---|---|
| AdCap Adoption | Scheduling Utilization report | "Are dispatchers using the AdCap tool?" |
| Capacity Hours | Dispatch / Capacity API | "How full are we?" |

A dashboard for "AdCap utilization by BU" cannot be built from the Adoption report — it needs the capacity-hours API. Confirm with your ServiceTitan CSM that the API returns Adaptive Capacity data (not just legacy ACP data) before committing to a dashboard build.

---

## 8. AdCap Audit Playbook

**Step 1 — Export.** Strategic Rules list. Capture Active vs. Inactive, version dates, capacity %.

**Step 2 — Classify.**
- Current version, in use → Keep.
- Expired prior version still Active → Inactivate.
- Duplicate from a prior tooling run → Manual remove.
- Cap % out of phase with seasonal philosophy → Update or version-fork.
- Naming doesn't match grammar → Rename via versioned suffix.

**Step 3 — Plan.** A target rule set with active/inactive flags. Verify Arrival Window Restrictions configuration as a separate worktrack.

**Step 4 — Execute.** UI-side work in `Settings → Adaptive Capacity → Strategic Rules`.

For Scheduling Pro: verify per-scheduler that No Saturday, No Sunday, Same Day Buffer, and 8AM Block are all configured (§4.4).

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
