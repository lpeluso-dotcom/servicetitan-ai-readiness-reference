# 08 — CSR Scoring Framework and AI Tools

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `08-csr-scoring-and-ai-tools.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Operations leaders, contact-center managers, and AI tooling teams configuring CSR scoring, dispatch optimization, or capacity prediction.

This file covers the AI-tool layer that consumes the data the rest of this series teaches you to clean: ServiceTitan's native AI suite (Titan Intelligence, Dispatch Pro, Adaptive Capacity), AI CSR scoring platforms (Lace AI and similar), and the canonical 10-step inbound-call rubric that those scoring platforms grade against.

The single insight that ties everything in this file together: **AI tools are not a substitute for clean data.** Every model in this domain is a function of the upstream taxonomy. A tenant that hasn't completed the cleanup work in `00`–`07` will see AI outputs that look reasonable but cannot be trusted at the operational level.

---

## 1. The Native ServiceTitan AI Suite

ServiceTitan ships four native AI tools. Each consumes a specific slice of the configuration domain and produces a specific operational output.

| Tool | What it produces | Primary data dependencies |
|---|---|---|
| **Titan Intelligence (TI)** | Revenue forecasting, CLV scoring, churn flags, replacement opportunities, lead-source ROI | Job types, BUs, customer history, marketing campaigns, equipment records |
| **Dispatch Pro (DP)** | Optimized tech assignment per job | Skills (tech + job type), BUs, zones, job-type duration, tag selections |
| **Adaptive Capacity (AdCap)** | Slot availability calculations | Strategic rules, tech shifts, BUs, skills, tags, job-type duration |
| **AI CSR Scoring** (typically third-party) | Per-call scoring against a rubric, objection categorization | Call recordings, ST job/customer/membership data, the rubric definition |

### 1.1 Titan Intelligence (TI)

ST's growing AI suite. Capabilities at the time of writing:

- **Revenue forecasting** by BU and job class.
- **Customer lifetime value (CLV)** scoring.
- **Churn risk** flags.
- **Equipment-replacement opportunity** detection (age + call history).
- **Lead-source ROI** modeling.

TI's outputs are only as good as the upstream taxonomies. A tenant with messy job types and orphan tracking numbers will receive forecasts that look reasonable but cannot be trusted at the BU level — the source attribution is wrong, so the lead-source ROI is wrong, so the spend recommendations are wrong.

### 1.2 Dispatch Pro — what it actually optimizes

DP's optimization takes these inputs:

- Job revenue potential (job type × historical sold rate).
- Tech skills (declared on tech profile, required by job type).
- Tech location and traffic.
- Tech current load and remaining capacity.
- Customer history (member, repeat, complaint flag).

It outputs an assignment that maximizes expected revenue per tech-hour while honoring skill and capacity constraints.

**Where DP underperforms:**

- Job types without declared required skills (DP defaults to "anyone").
- Tenants with stale tech-skill profiles.
- Job types with `Enable Dispatch Pro = OFF` (DP doesn't see them).
- Tenants where dispatchers manually override DP frequently (training data drift).

For per-job-type configuration and the repair-job-type override anti-pattern, see `03-dispatch-and-capacity.md` §6.

### 1.3 Adaptive Capacity (AdCap)

Rules-engine driven. Evaluates every booking attempt against the slot's date/time, the job type's BU and skill requirements, tech shifts overlapping the slot, the global Availability Threshold (drive-time buffer, typically 15 min), and all matching Strategic Rules under most-restrictive-wins.

Output: a capacity percentage. `0%` = unavailable. `< 100%` = partially booked. `> 100%` = intentionally overbooked (sales rules often run at 110% to absorb cancellation/reschedule rates).

Full Strategic Rule architecture in `03-dispatch-and-capacity.md` §3.

### 1.4 AI CSR Scoring

External platforms (Lace AI is a common example) integrate with ST call data to score every inbound call automatically against a rubric. The rubric drives coaching and identifies systemic objection patterns.

**Common rubric architecture:**
- 10 steps × 10 points each = 100 total.
- Some steps are conditional (`N/A` for new customers, existing members, etc.) and don't penalize the score.
- Seasonal cross-sell add-ons can be wired into specific steps.
- Calls below a threshold get flagged for supervisor review.

The full 10-step pattern is in §3 below.

---

## 2. The Data-Readiness Checklist for ST AI Tools

Before turning on **any** AI tool — DP, TI, AdCap, or external AI CSR scoring — run this checklist. A tenant that passes is genuinely AI-ready. Until it passes, every AI tool is fighting noise.

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

Each of these maps to a section in this reference series; see file `00`–`07` for the cleanup work.

---

## 3. The Canonical 10-Step CSR Rubric

Widely used inbound-call scoring rubric for trades CSRs. AI scoring platforms grade each call against these criteria after it ends.

| # | Step | Points | Conditional? |
|---|---|---|---|
| 1 | Greeting | 10 | — |
| 2 | Appreciate Membership | 10 | N/A for new customers |
| 3 | Personalization | 10 | — |
| 4 | Contact Information | 10 | — |
| 5 | Problem Discovery | 10 | N/A if customer's issue is already known |
| 6 | Value & Company Identity | 10 | — |
| 7 | Service Details (+ optional cross-sell) | 10 | Cross-sell often seasonal |
| 8 | Offer Membership | 10 | N/A for existing members |
| 9 | Job Confirmation (+ optional cross-sell) | 10 | Cross-sell often seasonal |
| 10 | Closure | 10 | — |

Total: **100 points.** Conditional steps marked N/A do not penalize the score.

### 3.1 Step-by-step scoring criteria

#### Step 1 — Greeting (10 pts)

- Agent mentions the company name at the start of the call.
- Agent identifies themselves to the customer immediately.
- Sample: *"Thank you for calling [Company], this is [Name] — how can I help you today?"*

#### Step 2 — Appreciate Membership (10 pts, conditional)

- Agent thanks returning members.
- N/A for new customers — does not penalize.
- Sample: *"I can see you're one of our valued members — thank you so much for your continued loyalty."*

#### Step 3 — Personalization (10 pts)

- Agent asks for and uses the customer's name.
- Agent engages in appropriate small talk and expresses empathy.
- Sample: *"I completely understand, [Name] — that sounds really frustrating. Don't worry, we'll take great care of you today."*

#### Step 4 — Contact Information (10 pts)

- Agent collects or confirms address, ZIP, phone, email.
- New customers: collect all fields.
- Returning customers: confirm existing info is current.

#### Step 5 — Problem Discovery (10 pts, conditional)

- Agent asks clarifying questions and quickly understands the issue.
- N/A if the customer's need is already known from a prior inquiry.

#### Step 6 — Value & Company Identity (10 pts)

- Agent shares something about who the company is or what makes them different.
- Brand pillars (e.g., a three-pillar slogan around community ownership, work quality, and customer-as-family — a common pattern in trades) should come through naturally, not as verbatim recitation.

#### Step 7 — Service Details + Seasonal Cross-Sell (10 pts)

- Agent explains what services will be performed.
- Agent describes any prerequisites.
- Agent explains fees, costs, and discounts.
- Agent reminds the customer payment is due day-of-service.
- 🌸 **Seasonal cross-sell** (if active) — example: spring HVAC tune-up special pitched on plumbing calls.
- All criteria required for full points.

#### Step 8 — Offer Membership (10 pts, conditional)

- Agent offers the membership program.
- Agent explains the benefits.
- Both required for full points — offering without explaining benefits does not score full.
- N/A if customer is already a member.

#### Step 9 — Job Confirmation + Seasonal Cross-Sell (10 pts, conditional)

- Agent confirms appointment date and time.
- Agent repeats appointment details back (address, phone).
- Agent offers an arrival window.
- 🌸 **Seasonal cross-sell** (if active) — example: in-home plumbing inspection pitched on HVAC bookings; aim for same-day scheduling.
- N/A if no appointment was scheduled.

#### Step 10 — Closure (10 pts)

- Agent thanks the customer.
- Agent mentions appointments can be booked online.
- Agent tells the customer they can contact the company anytime.
- Agent wishes the customer a good day.

---

## 4. The "Recoverable" Frame — Common Objections

AI CSR scoring data consistently identifies a small number of objection categories that account for most lost bookings. Understanding which are recoverable drives coaching priorities.

| Objection | Typical share | Recoverable? |
|---|---|---|
| Immediate Service Unavailability | ~30% | Partially — capacity step-down rules and CSR-side language for managing expectations |
| Service Fee / Dispatch Fee | ~24% | **Highly** — the #1 recoverable objection; needs a standard playbook line |
| Wanted price over the phone | 10–15% | **Highly** — Steps 5/6 must precede any pricing discussion |
| Will call back | 10–15% | Partially — same-day callback campaigns convert these warm leads |
| Did not book (no clear reason) | 15–20% | Variable — coaching opportunity |

---

## 5. The Two Highest-Leverage Playbook Lines

Two specific objection types account for ~35% of lost bookings combined and both are highly recoverable with a memorized response.

### 5.1 The dispatch-fee playbook line

Every CSR should have a memorized, standard answer to dispatch-fee objections. A common framing:

> *"I totally understand — the dispatch fee is what gets a licensed pro to your door, fully equipped and ready to diagnose the problem. It also goes toward the repair if you book the work that day. Most of our customers find it pays for itself in the first ten minutes."*

The point is consistency — across all CSRs, on every call. Without a standard answer, agents improvise, and improvisation produces variance.

### 5.2 The price-over-the-phone playbook line

**Don't give a number before Steps 5 and 6** (Problem Discovery and Value). Customers who get a price too early shop around and don't call back. A common framing:

> *"I understand wanting a number. Here's the honest answer: the price depends on what we find when we get there — and our techs will give you the full quote on-site, no obligation. The dispatch fee is $[X] and that's the only fixed cost up front."*

---

## 6. Seasonal Cross-Sell Wiring

Cross-sells are scored items wired into Steps 7 and 9. Common spring pattern:

| Inbound call | Cross-sell at Step | Cross-sell offer |
|---|---|---|
| Plumbing call | Step 7 | Spring HVAC Cleaning Special (great rate for new HVAC customers) |
| HVAC call | Step 9 | Promotional In-Home Plumbing Inspection (same-day with HVAC visit) |

**Pre-fill the playbook with `$______` placeholders** before the season goes live so the launch is a one-touch activation. The most common cross-sell launch failure is: pricing not finalized → CSR has no number → cross-sell is mumbled or skipped.

---

## 7. The Coaching Loop

The data flow that turns scoring into operational improvement:

1. AI scoring platform tags every call with a score and flagged objections.
2. Calls below threshold are queued for supervisor review.
3. Supervisors pull recordings, identify patterns (per-agent and systemic).
4. Patterns become coaching topics or playbook updates.
5. New playbook lines get tested in calls; AI scoring confirms or disconfirms improvement.

**The loop closes weekly.** Without it, the scoring data is interesting but not operational. Tenants that buy AI CSR scoring without staffing the supervisor-review and coaching steps get dashboards but not improvement.

---

## 8. Voice Agent Integration with the Scoring Framework

For tenants running an AI voice agent (ST-native or third-party like Lace), the rubric still applies — voice agents are scored against the same criteria. Specific implications:

- **Step 1 (Greeting):** the voice agent's greeting is fixed in configuration. Set it once correctly. ("Thank you for calling [Company]. I'm [Agent Name] — how can I help you today?")
- **Steps 4–5 (Contact Info, Problem Discovery):** the agent's intake script needs structured prompts. This is where most voice-agent quality variance lives.
- **Step 6 (Value):** voice agents struggle here. They tend to skip company-identity content unless explicitly prompted.
- **Step 8 (Offer Membership):** typically not handled by voice agents in self-service flows; gets skipped or escalated.
- **Step 10 (Closure):** voice agents can do this consistently if scripted correctly.

Tracking the voice agent's rubric scores alongside human CSR scores is one of the highest-value uses of AI CSR scoring — it's the only way to know objectively whether voice agents are conversion-positive or conversion-negative.

---

## 9. The Compounding Effect of Clean Data

Every section in this reference series feeds into the AI tool layer:

- **Pricebook cleanup** (`02`) → Dispatch Pro revenue scoring is correct.
- **AdCap rule hygiene** (`03`) → Capacity forecasts match operational reality.
- **Job type cleanup** (`04`) → DP routing matches skill requirements; TI segmentation is meaningful.
- **Form-driven structured data** (`05`) → AI can extract findings from completed forms.
- **Tracking number hygiene** (`06`) → Lead-source ROI is accurate; campaign attribution is trustworthy.
- **CRM dedup** (`07`) → Customer history (CLV, churn risk) reflects reality; voice agent doesn't talk to "John Smith" twice as if they're different people.

A tenant that completes the cleanup work in `00`–`07` unlocks AI tools that produce trustworthy outputs. A tenant that skips the cleanup work activates AI tools that confidently produce nonsense.

> **The hard truth:** you cannot AI your way out of a configuration problem. Every model in this domain is a function of the upstream taxonomy. Clean the taxonomy first; activate the AI second.

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
