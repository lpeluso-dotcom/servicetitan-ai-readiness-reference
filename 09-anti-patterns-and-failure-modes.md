# 09 — Anti-Patterns and Known Failure Modes

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `09-anti-patterns-and-failure-modes.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** ServiceTitan administrators, integration engineers, dispatch leads, and AI tooling teams who need to recognize the failure shapes before they happen — and the recoverable shapes when they do.

This file is the curated catalogue of **what not to do** in a ServiceTitan tenant pursuing AI readiness, plus the most-encountered runtime failure modes and their workarounds. The patterns here are extracted from real tenant audits and integration work; every entry has been observed in production at least once.

A note on tone: these are not theoretical. Each anti-pattern below has burned someone — usually multiple operators on multiple tenants. The corrections are battle-tested.

---

## 1. Configuration Anti-Patterns

The big one-liners. Each row is a "do not do this" with the corrective pattern. These are what an LLM should warn the user about when it sees these shapes coming up in conversation.

| # | Anti-pattern | Why it's bad | Correct approach |
|---|---|---|---|
| 1 | Reading individual rows from the Pricebook Categories sheet without iterating with running state | Produces false orphan classifications — items appear root-level when they're correctly nested. | Iterate top-to-bottom with a running `current` dict; populate the deepest non-empty `CategoryN` and clear deeper levels. See `02-pricebook-reference.md` §6. |
| 2 | Renaming a Business Unit | Breaks all historical reporting tied to that BU; revenue rollups, dashboards, and scheduled reports lose continuity silently. | Deactivate the prior BU; create a new one; migrate explicitly. Keep the old record. |
| 3 | Deleting (rather than deactivating) tags, items, job types, forms, tracking numbers, campaigns, or AdCap rules | Destroys historical reports. Deleted records leave dangling foreign keys in jobs and invoices. | **Always deactivate.** Universal rule across the platform. |
| 4 | Running multiple browser-automation sessions in parallel against the same tenant | Chrome-extension thrash; ServiceTitan session state becomes inconsistent; silent partial writes. | One session at a time. Sequence work into named workstreams. |
| 5 | Disabling Dispatch Pro on reactive service / repair job types | Erases the AI dispatching wedge on the highest-volume call types — exactly where DP outperforms human dispatch most. | Default DP `ON` for reactive service and repair. See `03-dispatch-and-capacity.md` §6. |
| 6 | Assigning tags to job types before standardizing the tag-type dictionary | Creates redundant rework when the dictionary later changes. Every dictionary edit invalidates the per-type assignments. | Tag dictionary cleanup **first**, then map to job types in a single pass. See `04-job-types-forms-tags.md` §7 (cleanup sequencing). |
| 7 | Treating AdCap rules and Scheduling Pro day-of-week settings as one system | AdCap can say "100% available" while Scheduling Pro is closed. Day-of-week behavior breaks because they are independent. | Configure **both** schedulers' day-of-week settings explicitly. See `03-dispatch-and-capacity.md` §5. |
| 8 | Using the Scheduling Utilization report as a capacity-hours metric in a dashboard | The report tracks **AdCap tool adoption** (did the dispatcher use AdCap?), NOT capacity utilization (how full are we?). They are not the same data. | Use the Dispatch / Capacity API for capacity hours. See `03-dispatch-and-capacity.md` §7. |
| 9 | Reusing a deactivated pricebook code for a different item | Mixed-semantics history under one identifier; reports lie about what was sold. | Create a new code; never reuse. Once a code touches a customer invoice, its meaning is permanent. |
| 10 | Adding tracking numbers without a campaign tag | Live calls flow through with zero attribution. Marketing Pro reports show "unattributed" or default-bucketed revenue. | Tag at creation. Orphan tracking numbers are direct cleanup debt — see `06-phone-marketing-attribution.md` §3.2. |
| 11 | Letting CSRs improvise on dispatch-fee objections | Booking variance across agents; the #1 recoverable objection isn't recovered consistently. | Standardize the playbook line. See `08-csr-scoring-and-ai-tools.md` §5. |
| 12 | Giving prices over the phone before Steps 5/6 of the inbound rubric | Customers shop and don't return. The price-over-phone objection compounds the problem when the CSR caves. | Discovery + Value first, then pricing. See `08-csr-scoring-and-ai-tools.md` §5. |
| 13 | Using free-text custom fields where a pick list could work | Becomes regex hell at analysis time; AI extraction is unreliable; reporting categories drift. | Pick lists wherever possible. See `04-job-types-forms-tags.md` §6. |
| 14 | Skipping the **Arrived** button on Field Pro | Permanently bad timestamp data; on-site time is unrecoverable; capacity math is wrong. | Train techs to back-stamp from the timestamp menu rather than skip. |
| 15 | Tapping **Done** before the customer signs | Invoice locks before signature; awkward UX; sometimes signature flow gets skipped entirely. | Always wait for signature. Done triggers signature → form → save. |
| 16 | Adding custom items in Field Pro with made-up prices | Breaks Pricebook discipline; pollutes downstream margin and revenue data; no historical comparability. | Stop and escalate. Add the missing item to the Pricebook correctly so the next tech finds it. |
| 17 | Routing all after-hours calls to a single number without testing the IVR for that number | Caller experience can dead-end if any layer is misconfigured — and you find out from a customer complaint, not from monitoring. | Test inbound during business hours **and** after-hours, on every tracking number, after any forward-chain change. |
| 18 | Configuring an IVR with zero menu keys assigned | Calls dead-end immediately; the system answers, the menu plays, no key works, the caller hangs up. | Always configure at least one menu key + greeting + after-hours route. See `06-phone-marketing-attribution.md` §8.1. |
| 19 | Forwarding an ST tracking number directly to a personal mobile (bypassing the phone platform's DID layer) | Breaks the attribution chain at the tracking number layer; no call recording from the platform; no IVR/after-hours rules apply; SMS capability disrupted. | Maintain the chain `ST tracking → phone-platform DID → staff phone`. See `06-phone-marketing-attribution.md` §2. |
| 20 | Setting a toll-free tracking number as the Scheduling Pro follow-up SMS sender without verifying the toll-free number is SMS-verified | Zero abandoned-session follow-up SMS get sent — silently, with no error visible in the SP interface. | Verify toll-free SMS capability with a test send before assigning. The verification process takes 2–4 weeks via ST support. |
| 21 | Authoring AI-tool-generated rules (Atlas-generated AdCap, etc.) without versioning | Hard to roll back; hard to audit what changed when; the AI can regenerate and double-stack rules. | Use the `vMMDDYY` versioned suffix pattern. See `00-overview-and-naming.md` §3.4. |
| 22 | Authoring multiple AdCap rules expecting them to chain or prioritize | They don't chain. They evaluate independently. The most-restrictive (lowest %) wins. | Embrace most-restrictive-wins as a design constraint. Author specific tight rules over general loose ones. See `03-dispatch-and-capacity.md` §3.2. |
| 23 | Building dashboards before the underlying taxonomy is clean | Locks in messy taxonomy as "the way it is" — once a dashboard exists, the taxonomy becomes politically harder to change. | Clean tags, job types, BUs, tracking numbers **first**. Then build dashboards. See `00-overview-and-naming.md` §1. |
| 24 | Treating cleanup work as a one-time project | Configuration drifts continuously as new staff add things. Cleanup is a quarterly cycle, not a finish line. | Schedule recurring audits. Pricebook quarterly, phone numbers quarterly, AdCap rules monthly. |
| 25 | Skipping the planning document in favor of "just do it" | Failed executions cannot be re-run without a plan; tribal knowledge evaporates when the operator leaves. | Every cleanup workstream has a plan + an execution Markdown + screenshots. See `09` §4. |
| 26 | Letting one person own all configuration changes without written documentation | Bus-factor of one. Staff turnover means lost institutional memory and silent regressions. | Document the patterns. Cross-train. Use a knowledge base like the one this file lives in. |
| 27 | Running the AI tools (DP, AdCap, TI) and the manual workflow simultaneously without precedence rules | Dispatchers override DP; training data drifts; AI accuracy degrades; everyone blames the AI. | Set explicit precedence: when does a human override DP? How is the override logged? Track override rates as a coaching signal. |
| 28 | Ignoring the TitanAdvisor health score as a "vanity metric" | The score correlates with real operational visibility. Dismissing it dismisses real configuration gaps. | Use TitanAdvisor as a north-star metric for cleanup work. Identify the 5–10 missing scheduled reports it surfaces. See `09` §4.8. |
| 29 | Storing ST OAuth credentials in automation tool configs (Make, Zapier) directly | Credential rotation requires touching every tool; rotation gets postponed; secrets leak in logs. | Front the API with a Cloudflare Worker (or equivalent). Hold credentials as Worker secrets. Tools call the Worker. See `01-tenant-architecture.md` §5. |
| 30 | Hard-coding sequential UUIDs in Form 2.0 JSON using non-hex characters (`g`, `h`, `n`, `p`) | Schema validation fails with `Unit '...' has an invalid or empty Id: '0'` — the parser substitutes `0` for invalid hex chars. | Use only `0-9` and `a-f` in UUIDs. Generate properly with `uuid.uuid4()` or use a structured prefix scheme. See `05-form-2-0-json-reference.md` §7. |
| 31 | Putting more than 2 actions in a single Form 2.0 conditional logic rule | The rendering engine silently rejects it; preview throws "Something went wrong"; the form imports but never displays. | Split into micro-rules with max 2 actions each. See `05-form-2-0-json-reference.md` §9. |
| 32 | Using `condition` (singular object) instead of `conditions` (plural array) in Form 2.0 rules | Rules silently don't fire. The form imports but conditional logic does nothing. | Always use `conditions` array, even for single-condition rules. Always include `whenMatch: "All"`. |
| 33 | Issuing a hard `DELETE` against a Contact ID that has multiple polymorphic associations | Severs all the contact's valid links across other Customers and Locations — not just the duplicate one. | Use **Remove Association** to sever a single link; use **Deactivate** for soft removal; reserve **Delete** for genuinely orphan contacts. See `07-crm-data-model-and-deduplication.md` §4.4. |
| 34 | PATCHing `mergedToId` directly via the API to merge customer records | The field is a system-protected internal trigger. PATCHing it does not perform the cascading re-parenting; it just sets a field on a still-active record. | Output a reconciliation report and run merges through the native UI flow. The platform's merge operation handles the graph re-parenting correctly. See `07-crm-data-model-and-deduplication.md` §6.4. |
| 35 | Inheriting opt-out / TCPA-restriction flags by majority vote during a contact merge | Loses opt-out status if the surviving record didn't carry it; TCPA exposure. | **Restrictive inheritance:** if any merged contact carries a `false` outreach preference, the survivor inherits the restriction. See `07-crm-data-model-and-deduplication.md` §8.2. |
| 36 | PATCHing estimate or proposal template `items[]` expecting delta-append behavior | The `items[]` array on template PATCH is **full-replacement** — any item absent from the payload is deleted from the template. Also, `modifiedOn` does not advance after `items[]` changes, only after a soft-delete (DELETE endpoint). `modifiedOnOrAfter`-gated syncs will silently miss item-level changes. | Always send the complete `items[]` array on PATCH. After editing template items, force a full downstream backfill on any sync that gates on `modifiedOn`. See `02-pricebook-reference.md` §7.15. |

---

## 2. Common Failure Modes — Symptom / Cause / Workaround

These are the runtime errors and "weird behaviors" that come up most often when working with ServiceTitan tenants, APIs, automation tools, and integrations.

### 2.1 Browser automation: blank page after deep-link navigation

- **Symptom:** A direct deep-link URL (e.g., `https://go.servicetitan.com/#/Settings/job-types/Edit/{id}`) loads a blank page or skeleton with no data.
- **Cause:** ServiceTitan's SPA routing requires the framework to be loaded from a parent URL first. Direct deep links don't always hydrate the modal data.
- **Workaround:** Navigate to `https://go.servicetitan.com/#/Settings` (or the relevant root), then click through the menu to the target page. After any session disconnect or extension reload, re-enter via the root.

### 2.2 Browser automation: extended sessions degrade

- **Symptom:** After 30–60 minutes of automation, clicks stop registering, modals don't close, or random elements throw "stale reference" errors.
- **Cause:** ST's session state and the browser's DOM state drift over long automation runs. Memory pressure and JS context fragmentation contribute.
- **Workaround:** Periodically restart the browser. Persist partial results to `outputs/` (or a similar location) frequently — `window` state is lost on reload but file state isn't. Build automation to be resumable from the last completed index.

### 2.3 `BundleValidationError` from Make.com or similar iPaaS write proxy

- **Symptom:** Make.com (or Zapier, n8n) HTTP/MCP module returns `BundleValidationError` on a Pricebook write.
- **Cause:** Either the Make scenario's input bundle validation rejects the payload format before it reaches the ST API, or the ST-specific connector silently drops the request body. The ST `service-titan:makeAnApiCall` module is known to drop bodies on PATCH/POST.
- **Workaround:**
  1. Try a single-item payload first to isolate whether it's batch logic or per-record.
  2. If it's per-record, bisect the failed batch to find the offending record.
  3. If using the ST-specific Make module, switch to a generic HTTP module with direct OAuth — the native module is unreliable for writes.

### 2.4 CSR can't see a job type they expect in the booking flow

- **Symptom:** CSR opens the booking flow; an expected job type doesn't appear in the picker.
- **Cause:** BU mismatch. The customer's location is bound to one BU; the job type is bound to a different BU. The picker filters job types by the location's BU.
- **Workaround:** Confirm the location's BU matches the job type's BU. Either rebind the location to the correct BU, or use a job type bound to the location's existing BU.

### 2.5 AdCap shows availability but Scheduling Pro is closed (or vice versa)

- **Symptom:** Customer can't book online; CSR sees green availability in the GAA tool. Or the inverse.
- **Cause:** AdCap and Scheduling Pro day-of-week settings are independent. AdCap might be configured "Mon–Sat 8a–6p" while Scheduling Pro is "Mon–Fri only."
- **Workaround:** Verify both schedulers' day-of-week settings explicitly. They must be configured consistently per-channel. See `03-dispatch-and-capacity.md` §5.

### 2.6 Capacity dashboard shows percentages that don't match operational reality

- **Symptom:** A Plecto / Power BI dashboard sourced from the Scheduling Utilization report shows 30% capacity when the team feels 90% booked.
- **Cause:** The Scheduling Utilization report tracks **AdCap tool adoption** (how often dispatchers used AdCap), not capacity utilization (booked hours vs. available hours). Different metric.
- **Workaround:** Switch the data source to the Dispatch / Capacity API. Confirm with the ST CSM that the API returns Adaptive Capacity data, not legacy ACP data. See `03-dispatch-and-capacity.md` §7.

### 2.7 After-hours calls dead-end despite working business-hours routing

- **Symptom:** Inbound calls during business hours route fine; after-hours calls drop or hit unintended voicemail.
- **Cause:** After-hours forward chain misconfigured on a specific tracking number, or the forwarding-permission flag is missing on toll-free trunks.
- **Workaround:** Re-test every tracking number's after-hours behavior individually. The standard pattern is "route to the voice agent number"; document any exceptions explicitly.

### 2.8 Voice agent escalation stops working

- **Symptom:** Customer says "Agent" but the voice agent doesn't escalate to a human queue.
- **Cause:** Trigger phrase configuration drift, or the live-agent queue is offline / out of hours / underprovisioned.
- **Workaround:** Test with the canonical phrases ("agent," "live person," "human," "representative"). Verify the destination queue's status. Check the voice agent's escalation configuration for changes.

### 2.9 Pricebook export says an item is orphaned but the UI shows it correctly categorized

- **Symptom:** An automated audit script flags a pricebook item as having no category; the ST UI shows the item correctly nested in its tree.
- **Cause:** The Categories sheet was read without positional iteration. Each row in the export only populates the deepest `CategoryN` it reaches; everything to the left is implied by the prior row's state. See `02-pricebook-reference.md` §6.
- **Workaround:** Re-parse with the running-state algorithm. Verify by tracing 3–5 known items through the algorithm and checking they match the UI.

### 2.10 Scheduled report runs but the recipient doesn't read it

- **Symptom:** Daily / weekly reports running for months with no engagement; the recipient's inbox has hundreds of unread copies.
- **Cause:** Common organic clutter — the report owner left the company, or the recipient's role changed and the report is no longer relevant.
- **Workaround:** Audit the scheduled-report list quarterly. Cross-reference recipients with active employees. Survey recipients: "Are you using this?" Deactivate reports with no active reader.

### 2.11 Form import fails with `"Required property 'id' expects a non-null value. Path 'definition'"`

- **Symptom:** A 400 error on Form 2.0 JSON import.
- **Cause:** `FormData.id` or `definition.id` is set to `null`. The schema validator rejects null IDs even though ST will overwrite them on import.
- **Workaround:** Set `id` to any integer (`9999`, `1`, the literal `"id": 1`). ST will replace with a real ID upon import. See `05-form-2-0-json-reference.md` §3.

### 2.12 Form 2.0 conditional logic doesn't fire at runtime

- **Symptom:** Form imports without errors, but show/hide rules don't trigger when the technician interacts with the form.
- **Cause(s) — check in this order:**
  1. `hasConditionalLogic` is `false` or missing on `definition.definition`. Set to `true`.
  2. `IsForm20` is missing or `false` at the document root. Must be `true`.
  3. Used `condition` (singular object) instead of `conditions` (plural array). Fix to plural array.
  4. Missing `whenMatch: "All"` on the rule object.
  5. For checkbox conditions, `right` is not exactly the string `"Yes"` (uppercase Y).
  6. The targeted field was never hidden by an `InitialLoad` rule, so it's already visible — the Toggle action has nothing to toggle.

### 2.13 Form preview throws "Something went wrong"

- **Symptom:** Form imports successfully but the preview page shows a generic "Something went wrong" error.
- **Cause:** Most likely a rule with more than 2 actions (the micro-rule constraint), or an invalid (non-hex) UUID somewhere in the document.
- **Workaround:** Validate the rules array — every rule must have ≤ 2 actions. Validate every UUID — only `0-9` and `a-f`. See `05-form-2-0-json-reference.md` §7 and §9.

### 2.14 OAuth `invalid_client` or `401 Unauthorized` on API calls

- **Symptom:** Worker / Make / direct curl calls return 401, or token requests return `invalid_client`.
- **Cause(s):**
  1. Wrong environment — the integration sandbox uses `auth-integration.servicetitan.io` and `api-integration.servicetitan.io`; production uses `auth.servicetitan.io` and `api.servicetitan.io`. Mixing them gives `invalid_client`.
  2. Credentials rotated and the consumer wasn't updated.
  3. `ST-App-Key` header missing on API calls (auth succeeds; API call fails 401).
  4. Token expired in cache without refresh logic.
- **Workaround:** Verify the auth domain matches the API domain. Check credentials are current. Confirm `ST-App-Key` is set on every API call. Add proper token-cache refresh-before-expiry logic.

### 2.15 Tracking number reported in ST but not present in phone platform (or vice versa)

- **Symptom:** ST shows a tracking number forwarded to "555-XXX-XXXX"; that number doesn't exist in the phone platform's number list.
- **Cause:** Forward chain was set in ST when a number existed; later the phone platform deleted or repurposed the number; the ST forward target is now stale.
- **Workaround:** Quarterly cross-reconciliation: pull the ST tracking-number list and the phone-platform DID list; cross-reference; any "ST forwards to a number that doesn't exist in the phone platform" is a critical fix.

### 2.16 Pricebook write returns 200 but the change doesn't appear

- **Symptom:** PATCH or POST returns 200 OK; subsequent GET shows the old data.
- **Cause(s):**
  1. The ST-specific Make connector silently drops the request body (returns 200 from Make, no actual API call body).
  2. Field name typo — e.g., `useStaticPrice` (singular) instead of `useStaticPrices` (plural). ST silently ignores unknown fields.
  3. `typeId` was sent on configurable equipment — that field is UI-display-only on writes; the actual write must go through equipment-type linking.
- **Workaround:** Always run `GET → mutate → PATCH → GET → diff` for any pricebook write. Don't trust 200 alone. Field-name typos are silent failures; verify the diff. See `02-pricebook-reference.md` §7.

### 2.17 Pricebook sync only returns inactive items (active items missing)

- **Symptom:** A sync that previously returned ~3K services now returns ~10K records, all inactive; active services are missing.
- **Cause:** ST's `/services` endpoint returns inactive services first when `active=Any` (or no filter). With a 50-page cap (10,000 rows) the actives can be buried past the cap and never sync.
- **Workaround:** Default `active=True` on all pricebook syncs (services, materials, equipment). Most downstream consumers filter `active=1` anyway, so this is purely net-positive. See `02-pricebook-reference.md` §7.

### 2.18 Customer merge in UI succeeds but historical equipment / invoices still reference the old ID

- **Symptom:** Customer A merged into Customer B; Customer B's profile shows the consolidated history, but external warehouse data still has equipment / invoices linked to Customer A's ID.
- **Cause:** The external warehouse pulled before the merge; the local snapshot has stale foreign keys. ST's internal re-parenting cascades on the live database but doesn't push to external warehouses.
- **Workaround:** When following the `mergedToId` pointer in external data, follow the chain to the surviving ID. Schedule a re-sync of customer / location / job tables after any bulk merge operation. See `07-crm-data-model-and-deduplication.md` §6.

### 2.19 Credential-fill browser automation breaks on Adaptive Login (ST-77.3)

- **Symptom:** Browser automation script that pre-fills username and password fails to complete login. The login page now shows an intermediate identity or auth-method selection step the script doesn't handle.
- **Cause:** ST-77.3 introduced **Adaptive Login** — a two-step flow where the tenant identifies itself first, then the auth method is selected. Single-pass credential-fill scripts get stuck at the intermediate step.
- **Workaround:** Migrate to **session-reuse** — persist the browser session state (cookies, local storage) between runs. Re-authenticate only when the session expires. Do not re-login fresh on every run. See `glossary.md` — **Adaptive Login**.

---

## 3. Strategic Anti-Patterns (Managerial)

These are not technical mistakes — they're organizational shapes that cause the technical anti-patterns above to recur.

| Anti-pattern | Why it's bad |
|---|---|
| Treating ServiceTitan cleanup as a one-time project | Configuration drifts continuously as new staff add things. Cleanup is a quarterly cycle, not a finish line. |
| Letting one person own all configuration changes without documentation | Bus-factor of one. Staff turnover means lost institutional memory. |
| Skipping the planning document in favor of "just do it" | Failed executions can't be re-run without a plan; tribal knowledge evaporates. |
| Authoring AI-tool-generated rules (Atlas-generated AdCap, etc.) without versioning | Hard to roll back; hard to audit what changed when. |
| Building dashboards before the underlying taxonomy is clean | Locks in messy taxonomy as "the way it is." |
| Skipping the audit phase in favor of plan + execute | You don't know what you're cleaning up; cleanup steps double-handle data. |
| Running AI tools and manual workflow simultaneously without precedence rules | Dispatchers override DP, training data drifts, AI accuracy degrades. |
| Ignoring the TitanAdvisor score as "vanity" | The score correlates with real operational visibility. |
| Treating the Pricebook as a one-person job | Pricebook touches every transaction; one-person ownership is fragile. Cross-train at minimum 2 people. |
| Activating an AI tool without first running the data-readiness checklist | Produces outputs that look reasonable but cannot be trusted; degrades trust in AI generally. |

---

## 4. Audit Playbooks — The Cleanup Workstreams

The standard shape of any ST cleanup workstream. These are the four-phase patterns that keep the work auditable.

### 4.1 General audit methodology

Every cleanup audit follows the same four-phase shape:

1. **Inventory** — export the current state (Excel, JSON, or screenshots). Capture the timestamp of the export.
2. **Classify** — group records into action buckets (correct / rename / re-categorize / deactivate / delete-after-confirmation).
3. **Plan** — produce an import-ready file or API payload set that effects the changes. Output is reproducible from inventory.
4. **Execute** — apply via API, browser automation, or manual UI. Capture before-and-after evidence.

Capture artifacts at each phase. The plan output should be a single import-ready Excel **and** an execution Markdown — both regenerable from the inventory if needed.

### 4.2 Pricebook audit playbook

**Step 1 — Export.** Pull the full pricebook export. Categories sheet must be parsed positionally (`02-pricebook-reference.md` §6).

**Step 2 — Classify.** For each item, decide:
- Correct? (right category, right code, right markup tier)
- Wrong category? (commercial under residential, generator under wrong trade)
- Needs multi-category? (dispatch fees, cross-trade items)
- Orphan? (no category at all, or stranded under a deprecated branch)
- Deactivate? (DNU items, replaced items, no recent line items)

**Step 3 — Plan.** Produce:
- Excel of changes with current state and target state columns.
- Execution Markdown describing API payloads.
- Multi-category item list with explicit dual homes documented.

**Step 4 — Execute.** Via Worker proxy + automation. Validate each batch. If `BundleValidationError`, bisect to isolate the offending record.

**Common findings:**
- Commercial services parented under residential or generic branches.
- Dispatch fees in only one category that should be in multiple.
- Cross-trade items (generator install) outside their secondary tree.
- Specialty subcategory missing entirely (e.g., Commercial Lighting).
- DNU items still active and appearing in tech pickers.

### 4.3 Phone-system audit playbook

**Step 1 — Export.** ST tracking-number list + phone-platform DID list.

**Step 2 — Classify each number.**
- Has a campaign tag → correct.
- Local number, no campaign → orphan; needs assignment or deactivation.
- Toll-free DNI pool number → expected, exempt from individual tagging.
- Routes to a broken IVR → critical fix.
- Mismatched area code → verify intentional.
- Dual-role outbound SMS sender → resolve role conflict.
- Used by an individual staff member → needs a Staff DID campaign.

**Step 3 — Plan.**
- Master phone-number workbook with forward-chain maps.
- Critical-issues list (broken IVRs, dead-end routes, after-hours exceptions).
- Orphan list with assignment recommendations.
- Staff DID setup workbook if new lines are needed.

**Step 4 — Execute.** Some changes are in ST (campaign tags, deactivation); others are in the phone platform (IVR keys, forward chains). Track ownership per item.

### 4.4 Forms audit playbook

**Step 1 — Export.** Forms list + Forms_History (one-year submission counts).

**Step 2 — Classify each form.**
- Active and used → rename to convention if needed.
- Active but zero submissions → investigate; deactivate if confirmed dead.
- Duplicate of another form → resolve to canonical, deactivate the duplicate.
- Year-prefixed legacy → roll forward or deactivate.
- PDF that should be ST Form (or vice versa) → migrate.
- Triggers wrong → flag for downstream Job Types cleanup.

**Step 3 — Plan.** Rename / deactivate spreadsheet. Trigger work blocked on Job Types cleanup completion.

**Step 4 — Execute.** Mostly UI work in ST Forms admin.

### 4.5 AdCap audit playbook

**Step 1 — Export.** Strategic Rules list. Capture Active vs. Inactive flag, version dates, capacity %, time ranges.

**Step 2 — Classify.**
- Current version, in use → keep.
- Expired prior version still Active → inactivate.
- Duplicate from a prior tooling run (e.g., Atlas-AI generation) → manual remove.
- Cap % out of phase with seasonal philosophy → update or version-fork.
- Naming doesn't match grammar → rename via versioned suffix.

**Step 3 — Plan.** Target rule set with active/inactive flags. Verify Arrival Window Restrictions configuration as a separate worktrack.

**Step 4 — Execute.** UI-side work in `Settings → Adaptive Capacity → Strategic Rules`.

### 4.6 Permissions audit playbook (people, roles, BUs)

**Step 1 — Export.** Employees, technicians, roles, role-permission matrix, BU assignments.

**Step 2 — Classify.**
- Active employee, correct role → keep.
- Active employee, wrong role / over-permissioned → adjust.
- Inactive employee still active in ST → deactivate.
- Tech with stale skill profile → update.
- Tech with wrong BU binding → rebind.
- Custom role created for one person → consolidate to standard role if possible.

**Step 3 — Plan.** Target role-permission matrix; tech-to-BU-group mapping.

**Step 4 — Execute.** UI work plus an audit of scheduled reports (often surfaces obsolete reports running on stale recipients).

### 4.7 Scheduled-reports audit (paired with permissions)

A common finding alongside permissions audits: **dozens of scheduled reports running daily/weekly that nobody reads.**

**Pattern:**
- Export scheduled-report list with recipients.
- Cross-reference recipients with active employees.
- Survey active recipients: "Are you using this?"
- Deactivate reports with no active reader.
- Identify priority reports that should be created (TitanAdvisor health-score gap analysis often surfaces 5–10 missing scheduled reports).

### 4.8 TitanAdvisor health-score audit

ServiceTitan provides a tenant health-scoring system (TitanAdvisor) that grades configuration cleanliness. The score is informational but useful as a north-star metric for cleanup work.

A typical gap analysis identifies **8–12 priority scheduled reports** that, if added, materially improve the health score and (more importantly) deliver real operational visibility. Common categories:

- Membership pipeline reports.
- Dispatching for Dollars (DP-on vs. DP-off comparisons).
- Capacity utilization (true hours, not adoption %).
- Sold-rate by job type.
- Tracking-number performance.
- CSR call-scoring summary.

Performing a live audit requires UI access — browser automation works if the connection is reliable.

---

## 5. The "Don't AI Your Way Out of a Configuration Problem" Principle

Every anti-pattern in this file ultimately traces back to one mistake: trying to fix a configuration problem by adding more AI on top of it.

- A tenant with messy job types adds Dispatch Pro and gets bad routing recommendations. The fix isn't "tune the AI"; it's clean the job types.
- A tenant with orphan tracking numbers activates Marketing Pro attribution and gets wrong lead-source ROI. The fix isn't "add more campaign rules"; it's tag the orphan numbers.
- A tenant with sloppy form configurations turns on AI form summaries and gets hallucinated findings. The fix isn't "fine-tune the summarizer"; it's clean the forms.
- A tenant with duplicate customer records activates AI follow-up and the voice agent calls the same person twice. The fix isn't "add a dedup step in the agent"; it's run a CRM dedup pass.

**The compounding logic:**
- Clean configuration → trustworthy AI → high-leverage automation.
- Messy configuration → unreliable AI → automation that creates more cleanup work than it eliminates.

The discipline is to do the boring cleanup work first. Every section in this reference series exists because someone tried to skip it and learned the hard way.

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
