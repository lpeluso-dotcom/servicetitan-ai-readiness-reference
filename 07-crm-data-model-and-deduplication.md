# 07 — CRM Data Model and Deduplication

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `07-crm-data-model-and-deduplication.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Data engineers, integration engineers, and AI tooling that reads, writes, warehouses, or deduplicates ServiceTitan CRM data.

This file is a data-structure reference for the ServiceTitan CRM domain — the schemas of **Customer**, **Location**, and **Contact** records as returned by the V2 API, the polymorphic Contact Hub, the `mergedToId` tombstone pointer model, and the dedup logic that follows from the schema constraints.

The CRM data model is the foundation for every customer-facing automation: voice agents, AI follow-up, marketing attribution, member-status checks, billing logic. Get the schema right and these all work; get it wrong and they all leak.

---

## 1. The Triad: Customer, Location, Contact

ServiceTitan CRM operates on a strict relational model with three core entities:

| Entity | What it represents | Cardinality |
|---|---|---|
| **Customer** | The financially responsible party — billing address, lifetime revenue, balance due | 1 record per legal/financial entity |
| **Location** | A physical service site — service address, equipment installed, jobs performed | M:1 to Customer |
| **Contact** | A specific human or communication endpoint — name, phone, email | M:N to Customer and Location (polymorphic) |

The split between Customer and Location matters most in commercial scenarios: a single Customer (the corporate entity that pays) commonly owns hundreds of Locations (the buildings serviced). In residential scenarios the addresses often coincide, but the schema is the same.

The Contact Hub is **polymorphic** — a single Contact record (e.g., a property manager) can be linked to multiple Customers and multiple Locations simultaneously. This is the structural difference from legacy CRMs that embedded contact fields as columns on Customer/Location rows.

---

## 2. Customer Schema

The Customer entity is the relational anchor of the CRM. It represents the legally or financially responsible party for all downstream activity. The Customer ID is the primary key and the parent identifier for all locations, jobs, invoices, and equipment associated with that account.

### 2.1 Endpoint

```
GET /crm/v2/tenant/{tenant}/customers
GET /crm/v2/tenant/{tenant}/customers/{id}
GET /crm/v2/tenant/{tenant}/export/customers   ← warehousing / dedup
```

The `/export/` endpoint returns a complete unfiltered snapshot **including merged tombstones** (records with `mergedToId > 0`). Use this for any dedup or warehousing work — the standard transactional GET filters merged records by default.

### 2.2 Field reference

| Field | Type | Notes |
|---|---|---|
| `id` | Integer (int64) | Primary key. Foreign key target for all locations, invoices, equipment. |
| `active` | Boolean | Operational status. `false` = deactivated; merged records are also `false`. |
| `name` | String | Legal or operational name of the financially responsible entity. |
| `address` | Object | Nested billing address (street, unit, city, state, zip, country, latitude, longitude). **Distinct from the service-location coordinates.** |
| `createdOn` | DateTime | Immutable record creation timestamp. |
| `createdById` | Integer | User who originated the record. |
| `modifiedOn` | DateTime | Last mutation timestamp. Used by delta-syncing pipelines. |
| `mergedToId` | Integer (int64) | **Tombstone pointer.** When `> 0`, this record was merged into the ID stored here. See §6. |
| `balanceDue` | Decimal | Aggregate of outstanding obligations linked to the parent. |
| `customFields` | Array of `{typeId, name, value}` | Tenant-defined extensions. See §5. |
| `externalData` | Array | Third-party identifier mappings (legacy systems, ERPs, etc.). |
| `doNotMail` / `doNotService` | Booleans | Operational restrictions. The Do-Not-Service flag in particular gates dispatching. |

### 2.3 Customer vs. Location address

A frequent confusion: the `address` on a Customer is the **billing address** — where invoices go. It is **not** necessarily the service address. In commercial accounts these are often very different (corporate HQ vs. the dozens of service sites). Always pull service addresses from the Location records.

---

## 3. Location Schema

Location records represent the physical, geographical sites where field technicians dispatch, estimates are conducted, and equipment is installed.

### 3.1 The M:1 cardinality rule

A Location is bound to **exactly one** Customer. Multiple Locations can hang off a single Customer, but no Location can have two Customer parents simultaneously.

This rule has a critical consequence for **real estate transactions**. If a residential property changes ownership, the historical Location record cannot simply be re-parented to the new homeowner's Customer ID — doing so would commingle the prior owner's invoice and equipment history with the new owner. The recommended workflow:

1. Create a **new** Location record for the new homeowner (under their Customer).
2. Migrate the deprecated Location to a placeholder Customer account (often named `New Homeowner` or similar) to preserve historical service data.
3. Future jobs at the address attach to the new Location.

**Consequence for dedup engines:** identical address strings frequently appear across two Location records that map to **different `customerId` values**. This is a valid historical property transfer, **not** a duplicate. The engine must interrogate `customerId` before merging on address match.

### 3.2 Endpoint

```
GET /crm/v2/tenant/{tenant}/locations
GET /crm/v2/tenant/{tenant}/locations/{id}
GET /crm/v2/tenant/{tenant}/export/locations
```

### 3.3 Field reference

| Field | Type | Notes |
|---|---|---|
| `id` | Integer (int64) | Primary key for the spatial location. |
| `customerId` | Integer (int64) | Required foreign key. The M:1 link to the parent Customer. |
| `active` | Boolean | Operational status. |
| `name` | String | Human-readable label (e.g., `"Main Warehouse"`, `"North Campus Facility"`). |
| `address.street` | String | Street routing info. |
| `address.unit` | String | Secondary designator. **Critical for high-density commercial / apartment dedup.** |
| `address.city` | String | Municipal boundary. |
| `address.state` | String | State / province / region. |
| `address.zip` | String | Postal routing code. |
| `address.country` | String | Two-letter ISO country code. |
| `address.latitude` | Float | Y-axis geo-coordinate. |
| `address.longitude` | Float | X-axis geo-coordinate. |
| `zoneId` | Integer | Operational dispatching/territory zone. |
| `taxExempt` | Boolean | Site-bound tax flag. |
| `mergedToId` | Integer (int64) | Tombstone pointer. Same semantics as Customer (§6). |
| `customFields` | Array | Tenant extensions. |

### 3.4 Why latitude/longitude matters

Standard text-based dedup against postal address strings inevitably encounters error rates from syntactic variations ("Street" vs "St.", "Highway" vs "Hwy"), localized abbreviations, and typos. **`address.latitude` / `address.longitude` provide a vastly superior dedup vector** — clustering on absolute coordinates bypasses spelling variation entirely. See §7.1 for the Haversine pattern.

---

## 4. Contact Schema and the Contact Hub

The Contact entity is structurally autonomous. Contact records exist centrally rather than as subservient child nodes locked to a single Customer or Location. The architecture is a **polymorphic many-to-many association matrix**: a single Contact can be associated with multiple Customers and multiple Locations simultaneously.

### 4.1 Why this matters

Legacy CRMs embedded contact fields directly on Customer/Location rows. ST's design means:

- Updating a contact's phone number is a single operation against one Contact ID; the change cascades to every associated Customer and Location.
- A property manager handling 30 buildings exists as **one** Contact with 30 Location associations and one Customer association — not 30 duplicate records.
- Removing a Contact from one Location does not affect any other association.

### 4.2 Field reference

| Field | Type | Notes |
|---|---|---|
| `id` | Integer | Primary key. |
| `name` | String | Human identifier. |
| `title` | String | Functional designation (e.g., `"Decision Maker"`, `"Facility Manager"`, `"Billing Contact"`). |
| `phone` | String | Primary phone. |
| `mobilePhone` | String | Mobile phone. |
| `email` | String | Email address. |
| `active` | Boolean | Operational status. |
| `optedOutOfSms` | Boolean | TCPA compliance flag for marketing SMS. |
| `optedOutOfEmail` | Boolean | Marketing email compliance flag. |
| `optedOutOfNotifications` | Boolean | Job-notification opt-out. |
| `mergedToId` | Integer | Tombstone pointer. |

### 4.3 Association objects

Because Contacts exist independently, the linkage to Customer and Location entities is held in **explicit Association objects** — bridge records connecting Contact ID to Customer ID or Location ID.

This is the structural layer that dedup engines must understand: **deleting a Contact ID is not the same as removing an Association.** Issuing a global DELETE against a Contact ID obliterates the contact from every association across the entire system, severing valid relationships with every Customer and Location that legitimately referenced it.

### 4.4 Three mutually exclusive severing operations

| Operation | Effect on the Contact record | Effect on associations |
|---|---|---|
| **Remove Association** | Record stays alive in the central hub | Deletes only one Customer↔Contact or Location↔Contact bridge |
| **Deactivate** | `active` flips to `false`; obscured from Call Booking UI; blocked from active marketing | All associations remain intact (for audit) |
| **Delete** | Record permanently destroyed | All associations dropped simultaneously |

**Operational cap:** the system enforces **a maximum of three Contact records merged simultaneously** in a single bulk operation. External scripts processing thousands of duplicate human records must batch in groups of ≤3.

---

## 5. Custom Fields — Schema Extensibility

ServiceTitan supports `customFields` arrays on Customer, Location, equipment, and operational records. The architecture returns custom fields as **a nested array of key-value pair objects**, not as flat columns.

### 5.1 Structure

```json
"customFields": [
  {
    "typeId": 4055,
    "name": "Gate Access Code",
    "value": "1234"
  },
  {
    "typeId": 4056,
    "name": "Marketing Source",
    "value": "Referral - Smith Family"
  }
]
```

| Key | Type | Meaning |
|---|---|---|
| `typeId` | Integer (int64) | Stable primary key for the field's global definition (e.g., `4055` = "Gate Access Code"). |
| `name` | String | Human-readable taxonomy of the field. |
| `value` | String | The actual data injected by the CSR or technician. |

### 5.2 ETL implications

Standard analytical algorithms (vector clustering, classifiers) require flat tabular structures. Pipelines must iterate the `customFields` array, unpack each object, and **pivot** based on `typeId` or `name` into standardized columns in the destination staging table.

### 5.3 The externalData array — strongest dedup vector

The `externalData` array houses third-party unique identifiers from legacy software or concurrent systems (Salesforce, ERPs, custom CRMs).

> If two ST Customer records both contain an **identical legacy account ID** in their `externalData` arrays, the probability that they represent a true duplicate approaches certainty — **overriding any string-distance signal**.

The dedup engine should always check `externalData` first; matches there are deterministic.

---

## 6. The `mergedToId` Tombstone Pattern

When ST users (or API calls) trigger a merge, the system does **not** hard-delete the duplicate row. Instead it executes a graph re-parenting operation that culminates in populating a tombstone pointer field: `mergedToId`.

### 6.1 The non-destructive merge sequence

1. The duplicate record's `active` flag transitions to `false` — record is hidden from Call Booking, search, dispatch boards.
2. The duplicate's `mergedToId` is stamped with the primary key of the surviving record.
3. Database backend re-parents foreign keys of every child entity (invoices, appointments, equipment, polymorphic Contact associations) to the surviving Customer or Location ID.

This preserves referential integrity and supports the "Undo a merge" UI utility. But it creates a logical trap for external dedup engines.

### 6.2 The dedup-engine consequence

Pulling an unfiltered `/export/` payload and immediately running TF-IDF or string-distance against the entire corpus will **disastrously re-flag records that ST has already merged**. Historical merges look like new duplicates because the tombstone records are still in the corpus.

### 6.3 The required pre-processing pipeline

Before any string-distance, spatial math, or vector clustering is computed, the engine must run two operations:

**Step 1 — Isolate and exclude tombstones.** Inspect `mergedToId` on every extracted record. **Any record with `mergedToId > 0` is a dead node** — prune from primary processing arrays. These records are functionally deactivated; including them creates recursive logical collisions (the engine "rediscovers" the same merge, generates a new merge proposal, the merge fails because the source is already merged, the engine retries, etc.).

```python
# Filter tombstones before any dedup work
def is_alive(record):
    return record.get('mergedToId', 0) == 0 and record.get('active', False)

alive_customers = [c for c in all_customers if is_alive(c)]
# only run dedup on alive_customers
```

**Step 2 — Relational graph traversal.** If the warehouse also powers BI dashboards, the reporting engine must traverse `mergedToId` chains for historical attribution. Legacy financial transactions linked to an ID that was later merged need to be attributed to the **final surviving primary ID** via chain-following.

```python
def resolve_to_living(records_by_id, record_id):
    """Walk mergedToId chains until we hit a living record."""
    current = records_by_id.get(record_id)
    seen = set()
    while current and current.get('mergedToId', 0) > 0:
        if current['id'] in seen:
            break  # cycle protection
        seen.add(current['id'])
        current = records_by_id.get(current['mergedToId'])
    return current
```

### 6.4 The merge cannot be performed via PATCH

The `mergedToId` field is **read-only via API**. PATCHing it directly does not work — population of `mergedToId` and the cascading re-parenting is a protected internal trigger that runs only when the merge is initiated through the platform's native merge interface or the merge endpoint.

**The correct workflow for an external dedup engine:**

1. Algorithm identifies a duplicate pair.
2. Algorithm outputs a structured **reconciliation report** (e.g., `Primary: Customer 1001, Duplicate: Customer 1002`).
3. Operators (or a UI-layer script) process the report through the platform's merge interface.
4. ST executes the protected merge sequence; `mergedToId` is populated correctly.

---

## 7. Dedup Logic — From Schema to Algorithm

The schema constrains which dedup approaches actually work. Each entity has a different optimal pattern.

### 7.1 Locations — spatial dedup with Haversine

Location records have absolute lat/long coordinates. The Haversine formula computes great-circle distance between two coordinate pairs:

$$d = 2r \arcsin\left(\sqrt{\sin^2\left(\tfrac{\phi_2 - \phi_1}{2}\right) + \cos(\phi_1)\cos(\phi_2)\sin^2\left(\tfrac{\lambda_2 - \lambda_1}{2}\right)}\right)$$

Where `φ` is latitude in radians, `λ` is longitude in radians, `r` is Earth's mean radius.

**Threshold:** if `d < 3–5 meters`, the two records occupy the same physical footprint.

**But identical coordinates do not mathematically authorize a merge.** The engine must then cross-reference `customerId`:

| Customer match? | Conclusion |
|---|---|
| Same `customerId` | **True duplicate** (data-entry error). Safe to merge. |
| Different `customerId` | **Likely property transfer** (§3.1). Do not merge — trigger a Levenshtein check on `address.unit` to determine if the coordinates represent two units within one structure. |

### 7.2 Contacts — exact match first, fuzzy fallback

The Contact Hub's autonomous structure means contact dedup must aggregate associations correctly before severing redundant records.

**Phase 1 — exact match via normalization.**

Phone normalization: strip parentheses, hyphens, periods, spaces — reduce to a canonical 10- or 11-digit numeric string.

```python
import re
def normalize_phone(p):
    if not p: return None
    digits = re.sub(r'\D', '', p)
    if len(digits) == 11 and digits.startswith('1'):
        digits = digits[1:]
    return digits if len(digits) == 10 else None
```

Email normalization: lowercase, strip whitespace.

```python
def normalize_email(e):
    return e.strip().lower() if e else None
```

Query for absolute 1:1 matches across distinct Contact IDs on the normalized values.

**Phase 2 — when matches are found, aggregate associations onto the surviving Contact ID.**

```
For matched duplicate Contact IDs:
  1. Aggregate the full association set (Customer + Location IDs) across all duplicates.
  2. POST the aggregated associations onto the surviving Contact.
  3. Issue Deactivate or Delete on the secondary Contact IDs.
```

This must happen **before** severing. Skipping the aggregation step destroys legitimate associations.

**Phase 3 — fuzzy fallback for incomplete records.**

For Contacts with missing phone/email, fall back on Jaro-Winkler distance against the `name` field. Jaro-Winkler weights prefix matches more heavily — well-suited to human names where typos cluster at the trailing end.

```
d_w = d_j + ℓ p (1 − d_j)
```

**Threshold:** if `d_w > 0.92` AND the two Contacts share at least one geographic Location association, an automated merge is justified.

### 7.3 Customers — TF-IDF + Cosine Similarity for commercial accounts

Commercial naming has heavy syntactic variance:
- "The Acme Corporation"
- "Acme Corp LLC"
- "Acme Corp - Main Billing"
- "ACME CORP"

TF-IDF (Term Frequency–Inverse Document Frequency) **down-weights** common syntactic noise (`The`, `LLC`, `Inc`, `Corporation`) and **up-weights** rare specific identifiers (`Acme`). Cosine similarity between the resulting vectors pierces through naming chaos.

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

names = [c['name'] for c in alive_customers]
vec = TfidfVectorizer(analyzer='char_wb', ngram_range=(2, 4))
matrix = vec.fit_transform(names)
similarity = cosine_similarity(matrix)
# similarity[i, j] > 0.85 → strong duplicate candidate
```

For residential customer names, fall back to Jaro-Winkler weighted by shared Location addresses.

---

## 8. Compliance and Edge-Case Integrity

### 8.1 The Do-Not-Service flag — restrictive inheritance

Customer records support a `doNotService` flag that gates dispatching. **During any merge, the surviving record must inherit the most restrictive value.**

If a high-value account is merged with an older duplicate carrying `doNotService = true`, and the surviving record inherits `false`, the business resumes service to a customer who should have been blocked — **operational gridlock** and potentially legal/safety consequences.

**Rule:** `surviving.doNotService = primary.doNotService OR duplicate.doNotService` (logical OR — if any duplicate is restrictive, the survivor inherits the restriction).

### 8.2 TCPA opt-outs — absolute restrictive inheritance

The Contact schema carries opt-out flags for SMS, email, and notifications. These are governed by the **Telephone Consumer Protection Act (TCPA)** and similar regulations.

When merging duplicate Contacts, the surviving record must enforce **absolute restrictive inheritance**:

> If **any** duplicate Contact in the matched cluster has an opt-out flag set (`optedOutOfSms = true`, `optedOutOfEmail = true`, `optedOutOfNotifications = true`), the surviving Contact must permanently inherit that restriction.

```python
def merge_compliance_flags(records):
    return {
        'optedOutOfSms': any(r.get('optedOutOfSms') for r in records),
        'optedOutOfEmail': any(r.get('optedOutOfEmail') for r in records),
        'optedOutOfNotifications': any(r.get('optedOutOfNotifications') for r in records),
    }
```

**Failure to honor opt-outs during dedup is direct legal exposure** under TCPA, CAN-SPAM, and analogous regulations.

### 8.3 Audit trail

ServiceTitan logs transactional mutations using **immutable Record Numbers** (not Record IDs), supporting extraction up to ~500 rows per query. Any external dedup pipeline must log its own activity so that ST audit trail entries (bulk deactivations, custom field transformations, association modifications) can be cross-referenced against the algorithmic decision (the Jaro-Winkler score, the Haversine distance, the TF-IDF cosine) that authorized the action.

---

## 9. Authentication and Multi-Tenancy

Standard ST OAuth 2.0 client credentials. See `01-tenant-architecture.md` §5.2 for details.

**Critical for warehousing multiple tenants:** ST primary keys are guaranteed unique only within a tenant. The warehouse schema must use a **composite primary key** of `(tenant_id, record_id)` to prevent collisions when ingesting from multiple tenants.

For Enterprise Hub deployments, **Network Groups** define subsets of tenant IDs that share a single Client ID and Secret pair.

**Sandbox:** the Integration Environment (`api-integration.servicetitan.io`) is a cloned-from-production sandbox. Develop and test dedup pipelines here before pointing them at production. Note that the sandbox is not provisioned by default for every tenant.

---

## 11. Installed Equipment API — ST-77 Schema Updates

The installed equipment endpoint (`/equipmentsystems/v2/tenant/{tenantId}/installed-equipment/{id}`) received confirmed writable fields and new read-only fields in ST-77.

### 11.1 Confirmed writable fields (ST-77)

| Field | Type | Notes |
|---|---|---|
| `manufacturerName` | String | Manufacturer name on the installed record. |
| `installedOn` | Date (ISO 8601) | Installation date. |
| `equipmentTypeId` | Integer | Equipment type classification. **Use this field, not `typeId`.** `typeId` silently fails on write — returns 200 OK with no change. |

### 11.2 New read-only fields (ST-77)

| Field | Type | Notes |
|---|---|---|
| `manufactured_on` | Date | Manufacture date of the unit. Enables warranty tracking without custom fields. |
| `predicted_replacement_*` | Date | Automated aging computation projecting when the unit will need replacement. Sourced from equipment type + install date + expected lifecycle. |
| `type` | String | Equipment type name. Prior ST versions returned `[object Object]` from the `{id, name}` shape; ST-77 returns the correct string. |

### 11.3 Integration implications

- **Warranty tracking and equipment aging dashboards** can now be driven from native fields rather than custom fields or external calculations.
- **Membership-tier service scheduling** can key off `predicted_replacement_*` to surface aging-equipment renewal offers.
- **Sync note:** if your integration syncs installed equipment, the new `manufactured_on` and `predicted_replacement_*` fields will start populating on records created or updated after the tenant upgrades to ST-77. Existing records may not have these fields until they are touched (updated or re-synced).
- **Do not PATCH `typeId`.** Use `equipmentTypeId` for equipment type writes. The `typeId` field returns 200 OK silently on write attempts.

---

## 10. Practical Pipeline Sequencing

For a working dedup pipeline:

1. **Pull** via `/export/` endpoints — Customers, Locations, Contacts (with associations), all custom fields.
2. **Filter tombstones** — exclude records where `mergedToId > 0` from primary processing.
3. **Normalize** — phone numbers, emails, addresses (for spatial matching, geocode if lat/long missing).
4. **Locations first** — Haversine distance with `customerId` cross-check.
5. **Contacts second** — exact match on normalized phone/email; aggregate associations before severing; fuzzy fallback only for incomplete records.
6. **Customers last** — TF-IDF on name; cross-check against billing address and `externalData`.
7. **Output reconciliation reports** — do not attempt to populate `mergedToId` directly via PATCH (§6.4).
8. **Merge via UI or merge endpoint** — let ST execute the protected merge sequence.
9. **Re-pull and validate** — verify each merge propagated correctly; check `mergedToId` chains.
10. **Log everything** — algorithmic score, merge decision, target IDs — for cross-reference against the ST audit trail (§8.3).

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
