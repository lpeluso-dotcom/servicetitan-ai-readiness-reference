# 06 — Phone, Marketing, and Attribution

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `06-phone-marketing-attribution.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Marketing operations, integration engineers, and AI tooling that consumes call attribution data.

This file covers the dual-platform phone architecture (ServiceTitan tracking + external phone platform), tracking number taxonomy and naming, IVR configuration and after-hours routing, and the Marketing Pro attribution model that ties phone calls and online bookings to campaigns.

The single most important piece of leverage in this domain: **routing destination is determined by the tracking number the customer dialed, not by the customer record.** Tracking-number hygiene is the foundation of correct attribution.

---

## 1. The Dual-Platform Architecture

A typical ServiceTitan-based service operation runs phone routing across two platforms in combination:

**Platform 1 — ServiceTitan tracking numbers**

- Purpose: marketing attribution via Dynamic Number Insertion (DNI) and campaign assignment.
- Each ST tracking number is linked to a Marketing Pro campaign.
- Calls through ST tracking numbers are recorded, attributed to the campaign, and appear in MP reporting.

**Platform 2 — External communication platform** (e.g., Podium, RingCentral, etc.)

- Purpose: the actual phone system — IVR routing, voicemail, SMS inbox, staff communication.
- Numbers in this platform are **DID** (Direct Inward Dialing) numbers that physically ring phones.

**Standard call flow:**

```
Customer calls ST tracking number
    ↓ ST records call, attributes to linked campaign
Call forwarded to external platform DID number
    ↓ External platform routes to IVR, team, or individual
Call answered / voicemail taken
```

**The handoff:** ServiceTitan owns tracking and tagging; the phone platform owns routing. Most tracking numbers forward to a single live-agent answering service; some pass through an IVR layer first.

---

## 2. The DID Forwarding Constraint

### 2.1 The bypass problem

If an ST tracking number forwards **directly** to a staff member's personal mobile number (skipping the external platform), the attribution chain breaks at the ST tracking number layer:

- No call recording is captured (the call exits ST's system).
- No external-platform engagement occurs.
- After-hours routing in the external platform cannot intercept the call.
- SMS capability for that number chain is disrupted.

### 2.2 The correct chain

```
ST Tracking Number → External Platform DID → Staff Member's Phone
```

This chain maintains:
- ST call recording (call stays in ST's session through the tracking number layer).
- Campaign attribution (call attributed to the tracking number's linked campaign).
- External-platform routing (after-hours rules, ring groups, IVR logic apply at the DID layer).
- SMS capability (handled by the external platform DID).

**Consequence for staff DID lines:** each staff member who requires a direct line also requires a **dedicated** ST tracking number linked to an appropriate campaign. **One ST tracking number per DID** — numbers cannot be shared without losing per-person attribution.

---

## 3. Tracking Number Taxonomy

### 3.1 The seven categories

| Category | Description | Attribution Campaign Type |
|---|---|---|
| **Digital / PPC** | Google Ads, Meta, paid digital. Typically DNI (rotates per visitor). | PPC campaign per platform |
| **Organic / SEO** | Static number on website for organic-traffic attribution | Digital — Organic campaign |
| **Print** | Static numbers on direct mail, vehicle wraps, yard signs, business cards | Print campaign per piece |
| **Email** | Numbers embedded in email campaign content | Email campaign |
| **Staff Direct Lines** | DID chain numbers per staff member | Internal / Staff campaign |
| **AI / Overflow** | Numbers routing to AI answering or overflow center | AI or Overflow campaign |
| **Orphaned** | No campaign assigned — active inbound calls with no attribution | Requires assignment or deactivation |

### 3.2 The orphan number problem

A tracking number without a campaign assignment **accepts inbound calls but contributes zero data to source attribution**. Every job created from a call through an orphan appears in Marketing Pro as "unattributed" or with whatever default campaign is configured.

**Typical scale:** in tenants with dozens of orphan numbers, **20–40% of inbound revenue** can be unattributable to source. This is one of the highest-ROI cleanup targets in any marketing audit.

### 3.3 Common audit findings

A typical multi-trade tenant accumulates **100+ ST tracking numbers** over a few years. Audits commonly find:

- **5–10% of numbers have routing issues** (broken IVRs, wrong forward chains, dead-end menus).
- **15–25% of local numbers are orphaned** (no campaign tag).
- **A separate pool of toll-free numbers is intentionally untagged** because they rotate dynamically on the website (DNI pool — see §3.5).

The audit goal: **zero orphans in local-number space** and **complete attribution for every campaign-driven number**.

### 3.4 The dual-role outbound SMS conflict

A common configuration error: **a number configured as both the system-wide outbound SMS sender AND as an inbound tracking number with no campaign assigned.** Inbound calls or texts on this number have ambiguous attribution.

Two valid resolutions:
1. Assign a campaign to the number (clarify its inbound tracking purpose).
2. Remove it as an inbound tracking number (purely outbound).

### 3.5 DNI toll-free pools — the legitimate orphan exception

Toll-free numbers in **DNI (Dynamic Number Insertion) pools** rotate on the website to attribute web sessions to phone calls. They are **expected to have no campaign tag in ServiceTitan** — the attribution happens via the DNI vendor (CallRail, WhatConverts, Marchex, etc.) and is bridged to ST via lead-source mapping, not campaign tagging.

**Audit pattern:** DNI pools should be flagged as exempt in any orphan-tracking-number report to avoid false-positive cleanup tickets.

### 3.6 Internal staff DIDs — the emerging fifth category

A growing pattern: **DID lines for individual staff members** (CSRs, property-management coordinators, sales reps). Each requires:

1. A ServiceTitan tracking number.
2. A campaign tag in the form `Internal | Staff DID | [Name]`.
3. A forward chain that routes to the staff member's mobile or extension.
4. After-hours behavior consistent with the rest of the tenant (typically back to the voice agent).

Staff DIDs need explicit configuration; they are not interchangeable with general queue numbers.

---

## 4. The Canonical Tracking Number Naming Convention

Every tracking number must carry a campaign tag in this form:

```
[Channel] | [BU/Trade or ALL] | [Campaign Descriptor]
```

### 4.1 Channel values

- `Print` — door hangers, business cards, yard signs, brochures, car flyers, stickers
- `Email` — automated and one-time email sends
- `PPC` — Pay-Per-Click (Google Ads, Meta, etc.)
- `LSA` — Local Service Ads / Google Guaranteed
- `Partner` — referral partner / co-branded campaigns
- `Direct` — directly distributed numbers (e.g., business-customer direct lines)
- `Referral` — affiliate referrals
- `DNI` — Dynamic Number Insertion pool (intentionally untagged)
- `Internal` — staff DIDs and tracking numbers (not customer-facing marketing)
- `AI` — voice agent and overflow numbers

### 4.2 BU values

`HVAC`, `Plumbing`, `Electrical`, `Generators`, `ALL`.

### 4.3 Examples

```
Print | ALL | Door Hanger Spring 2026
Email | HVAC | Unsold Estimates Auto
Partner | HVAC | Trane
PPC | Plumbing | Google Ads
LSA | ALL | Google Guaranteed
Internal | Staff DID | [Name]
DNI | ALL | Toll-Free Pool
AI | ALL | After-Hours Voice Agent
```

### 4.4 Campaign-side dot notation alternative

Some tenants prefer a hierarchical dot-notation on Marketing Pro **campaigns** (rather than the pipe-delimited format on tracking-number names):

```
[Channel].[Trade/Scope].[Specificity].[Geo/Period]
```

```
PPC.HVAC.GoogleSearch.SC2026
PPC.ALL.GoogleSearchBrand.SC2026
PRINT.ALL.DirectMail.Spring2026
PRINT.HVAC.YardSign.Coastal2026
REFERRAL.HVAC.CustomerReferral
EMAIL.HVAC.MaintenanceReminder.2026
TRACKING.STAFF.DirectLine.HVAC.TeamLead
TRACKING.AI.AnsweringService.AfterHours
```

**API-friendliness:** dot notation enables programmatic filtering (`campaigns.where(name.startsWith('PPC.HVAC'))`) without custom field configuration. **Avoid spaces** in campaign names for API filtering compatibility.

Pick one convention per tenant; do not mix.

---

## 5. The Six Caller-Journey Paths

Codifying the canonical journeys lets CSR scripts and dashboards map any call back to its origin.

### Group 1 — PPC & LSA

1. Customer clicks paid ad → unique tracking number per campaign/BU.
2. Customer dials tracking number.
3. ServiceTitan tags the call (e.g., `PPC | HVAC | Google Ads`).
4. Call forwards to live-agent answering service.
5. Agent books the job or escalates to a customer-service queue.
6. After-hours route to the voice agent.

### Group 2 — Print (door hangers, business cards, yard signs)

1. Customer receives physical material.
2. Customer dials the number printed on the piece (each tech/campaign has a unique attribution number).
3. Tagged to the print campaign.
4–6. Same as Group 1.

### Group 3 — Email campaigns

1. Customer receives email (newsletter, estimate follow-up, unsold-estimate auto, birthday, welcome, loyalty, weather event).
2. Customer dials the tracked number.
3. Tagged to the email campaign and BU.
4–6. Same as Group 1.

### Group 4 — Voice agent (self-service)

1. Customer calls any standard tracking number → forwards to the answering service.
2. Voice agent answers.
3. Voice agent gathers issue type and follow-up info.
4. Voice agent books the appointment directly. Call ends.

### Group 5 — Voice agent escalation

1. Customer says "Agent" or voice agent cannot resolve.
2. Voice agent escalates to a live agent.

### Group 6 — Direct lines (existing business customers)

1. Existing customer dials a direct line that bypasses the new-customer queue.
2. Routes directly to a customer-service queue (or property-management queue).
3. Live agent handles the call. Often member-tagged and treated as priority inbound.

---

## 6. Standard Routing Destinations

Naming destinations consistently makes forward-chain documentation human-readable. A common pattern:

| Label | Purpose | Backed by |
|---|---|---|
| `CUST.SERV-1` | Customer Service queue | Multiple inbound numbers + queue |
| `DISP-HVAC-CQ` | HVAC Dispatch | Dedicated dispatch number |
| `DISP-ELEC/PLUM-CQ` | Electrical & Plumbing Dispatch | Dedicated dispatch number(s) |
| `DISP-ALL-1` | Membership queue (often via IVR key) | Membership coordinator |
| `SALES-CQ-1` | Sales queue | Sales rep group |

---

## 7. SMS Configuration

SMS capability is configured **independently per tracking number**. A tracking number without SMS enabled cannot receive text messages. For numbers used as primary customer contact points (website, signature, business card), SMS must be enabled.

### 7.1 Toll-free verification

Toll-free numbers (800, 855, 866, 877, 888 prefixes) require **carrier verification** before they can send outbound SMS. Numbers with "Unverified" status cannot send SMS through the ST platform. The verification process is initiated through ST support and **typically takes 2–4 weeks**.

### 7.2 The silent-failure trap

> Assigning a toll-free number as the Scheduling Pro follow-up SMS sender when that number is Unverified results in **zero abandoned-session follow-up SMS being sent — silently, with no error visible in the SP interface**.

**Always test with a real send** before configuring a number as the SP follow-up sender.

---

## 8. IVR Configuration and After-Hours Routing

### 8.1 IVR configuration requirements

An IVR (Interactive Voice Response) menu must have all key assignments populated. **The most common critical failure:** an IVR with **zero keys configured**. The menu plays, the caller presses any key, nothing happens, the call drops. This dead-ends every call passing through that IVR.

**Dead-end IVR symptoms:**
- Calls are answered but callers immediately redial or hang up.
- Voicemail is empty despite call volume appearing in reports.
- Customer complaints about "not being able to reach anyone" despite normal call counts.

**Required-fix template for any IVR:**

1. Log into the phone platform → Settings → Phones → [Number].
2. Add menu keys (e.g., Press 1 for Customer Service, Press 2 for Dispatch).
3. Add a call greeting recording (specific to the IVR's purpose).
4. Set after-hours routing (consistent with the rest of the tenant).
5. **Test inbound during business hours and after hours.**

### 8.2 After-hours routing

After-hours routing must be **explicitly configured per number** in the external platform. Behavior of unconfigured numbers varies by platform:

- Some platforms default to ringing indefinitely (no voicemail).
- Some default to immediate voicemail.
- Some default to disconnecting the call.

**Standard pattern for service operations with after-hours emergency availability:**

```
After 5 PM and weekends → Route to [overflow/AI answering service number]
```

Common pattern: **all standard tracking numbers route after-hours to a single voice-agent number** that handles overflow, captures basic info, and either books directly or escalates the next morning. Required:

- **One canonical after-hours number** known to both the answering service and the voice agent.
- **Consistent routing across tenant numbers** (exceptions documented).
- **Forwarding-permission settings verified** — some trunks require explicit permission to forward toll-free.

**Common exception:** AI direct-CSR lines (e.g., a dedicated AI sales line) may route after-hours to a different number than the standard voice agent. Document these exceptions explicitly; they are easy to forget in a phone audit.

For staff direct lines, after-hours routing options:
- Route to personal voicemail (staff preference).
- Route to central dispatch number.
- Route to after-hours answering service.

After-hours routing must be verified for **each DID number independently** — it cannot be configured globally across all numbers in most platforms.

---

## 9. Voice Agent Platforms

Two patterns observed in production:

| Pattern | Vendor | Status |
|---|---|---|
| ServiceTitan-native Voice Agent | ServiceTitan | Current incumbent in many tenants |
| Lace AI Voice Agent | Lace AI (formerly known for CSR scoring; expanding to voice) | Emerging; many tenants migrating from ST-native |

**Self-service flow (both vendors):**
1. Customer calls any tracking number → forwards to answering service.
2. Voice agent answers and gathers issue type and follow-up info.
3. Voice agent books the appointment directly. Call ends.

**Escalation flow:**
1. Customer says "Agent" or voice agent cannot resolve.
2. Voice agent escalates to a live agent.

---

## 10. Marketing Attribution

### 10.1 Two campaign types

**Tracking Campaigns** — linked to ServiceTitan tracking phone numbers. Attribution is automatic — when a call comes through a tracking number, the resulting job is assigned the linked campaign. Tracking campaigns drive the source attribution column in all ST revenue and lead reports.

**Pro Campaigns (Marketing Pro)** — active marketing initiatives with goals, spend tracking, automation rules, and customer enrollment. Pro campaigns can trigger automated customer communications (emails, SMS), segment customers for targeted outreach, and measure ROI. A Pro campaign can exist independently of tracking numbers.

> Both types appear in the same campaign list in Marketing Pro settings but serve different functions. **Mixing them without clear naming conventions leads to attribution confusion.**

### 10.2 Phone call attribution chain

```
Customer calls → ST tracking number → Job created → Campaign assigned automatically
```

Attribution is applied at job creation. **If a tracking number is not linked to a campaign at the time of the call, the job receives no source attribution.** Retroactive attribution assignment is not possible through the standard ST interface — it requires API access or manual field editing per job.

### 10.3 Online booking attribution chain

```
Customer visits website → UTM parameters captured →
  Scheduling Pro session → Job created →
  MP campaign assigned from UTM data
```

When Marketing Pro attribution is enabled in Scheduling Pro settings (see `03-dispatch-and-capacity.md` §4.8), the SP widget reads UTM parameters from the customer's session and maps them to the corresponding MP campaign. If no UTM data matches, the default campaign is assigned.

**Failure mode:** if the website does not pass UTM parameters consistently (broken UTM setup in Google Ads, campaign links not tagged), SP bookings accumulate in the default "Direct Web Traffic" campaign regardless of actual source. **This creates an inflated Direct attribution bucket that obscures true channel performance.**

### 10.4 Campaign goal types

ServiceTitan Marketing Pro supports a fixed set of campaign goal types. Common values:

- `Newsletter`
- `Cross-sell Relationship`
- `Inactive Customer`
- `Unsold Estimate Auto`
- `Opt In`
- `Marketing` (generic)

**Campaign mediums:** `Email`, `SMS`, `Direct Mail`, `Phone`, `Print`.

**Status values:** `Draft`, `Active`, `Finished`, `Stopped`.

**Delivery logic:** `One-Time`, `Automated`.

### 10.5 Campaign goals are required for ROI

Pro campaigns should have Goals configured to enable ROI measurement. Goals define the target metric (revenue, job count, conversion rate). Without goals:

- Campaign ROI cannot be calculated in MP.
- Benchmark reports have no target to compare against.
- Campaign comparison is qualitative rather than quantitative.

---

## 11. Phone System Audit Playbook

**Step 1 — Export.** Pull the ST tracking number list and the phone-platform number list.

**Step 2 — Classify each number.** Action buckets:

- Has a campaign tag → **Correct.**
- Local number, no campaign → **Orphan**, needs assignment or deactivation.
- Toll-free DNI pool → **Expected, exempt.**
- Routes to a broken IVR → **Critical fix.**
- Mismatched area code → Verify intentional.
- Dual-role outbound SMS sender → Resolve role conflict.
- Used by an individual staff member → Needs `Internal | Staff DID | [Name]` campaign and routing setup.

**Step 3 — Plan.** Produce:

- Master phone-number workbook with forward-chain maps.
- Critical-issues list (broken IVRs, dead-end routes, after-hours exceptions).
- Orphan list with assignment recommendations.
- Staff DID setup workbook if new lines are needed.

**Step 4 — Execute.** Some changes are in ST (campaign tags, deactivation); others are in the phone platform (IVR keys, forward chains). Track ownership per item.

---

## 12. The Architectural Lesson

> **Routing destination is determined by the tracking number the customer dialed, not by the customer record.**

This means tracking-number hygiene is the foundation of correct attribution and correct routing. A customer who calls the wrong number gets the wrong routing, and the call shows up in the wrong attribution bucket — both for marketing reporting and operational dispatch.

Every audit and every cleanup priority in this domain ladders up to that one principle.

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
