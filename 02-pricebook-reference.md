# 02 — Pricebook Reference

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `02-pricebook-reference.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Pricebook administrators, integration engineers, and AI tooling that reads/writes pricebook data.

This file is the consolidated technical reference for the ServiceTitan Pricebook subsystem: API endpoints, object schemas observed on live tenants, the dynamic pricing model, configurable equipment workflow, the four pricing-rule sibling tables, the export-format trap, and the catalogue of silent-failure gotchas that ST does not document.

Several worked examples reference a real multi-trade tenant snapshot (HVAC / Plumbing / Electrical / Generators, ~2.7K services / 7.7K materials / 225 equipment items at the time of writing). These are illustrative — your numbers will differ; the patterns generalize.

---

## 1. Why the Pricebook Is the Foundation

Every job touches the Pricebook. The tech selects services, materials, and equipment in the field; the resulting invoice rolls into Accounting; the historical line items roll into dashboards and Titan Intelligence (TI). Bad codes, ambiguous categories, or duplicate items pollute every downstream system simultaneously.

**The Pricebook is the highest-leverage cleanliness target in the entire tenant.** A single wrong category assignment shows up in dashboards forever. A single reused SKU corrupts historical line items irreversibly.

---

## 2. The Four-Entity Pricebook Model

| Entity | Definition | Has serial # | Has variants | Pricing model |
|---|---|---|---|---|
| **Service** | Sellable line item — labor + bundled materials/equipment | No | Sometimes | Dynamic (default) or static |
| **Material** | Small consumable (drip pans, valves, fittings) | No | No | Static |
| **Equipment** | Major equipment (water heaters, furnaces, condensers, panels, generators) | Yes | No (or use Configurable Equipment) | Static |
| **Configurable Equipment** | One logical equipment item with picklists for variants (size, fuel, efficiency tier) | Yes | Yes (variants) | Static (per variant) |

**Estimate Templates** (also called Service Templates) bundle one or more of the above into a sellable package. Good / Better / Best / Ultra is the standard bundling pattern (see §11).

---

## 3. API Endpoint Reference

Base URL: `https://api.servicetitan.io/pricebook/v2/tenant/{tenantId}/`

All requests require:
```
Authorization: Bearer {oauth2_access_token}
ST-App-Key: {app_key}
```

### 3.1 Primary endpoints

| Entity | Path | Methods | Pagination |
|---|---|---|---|
| Categories | `/categories` | GET, POST, PATCH, DELETE | Yes |
| Services | `/services` | GET, POST | Yes |
| Services (single) | `/services/{id}` | GET, PATCH, DELETE | — |
| Materials | `/materials` | GET, POST | Yes |
| Materials (single) | `/materials/{id}` | GET, PATCH, DELETE | — |
| Equipment | `/equipment` | GET, POST | Yes |
| Equipment (single) | `/equipment/{id}` | GET, PATCH, DELETE | — |
| Equipment Types | `/equipment-types` | GET, POST, PATCH, DELETE (as of ST-76.1) | Yes |
| Discounts and Fees | `/discounts-and-fees` | GET | Yes |
| Membership Types | `/memberships/v2/.../membership-types` | GET | Yes |
| Membership Discounts | `/memberships/v2/.../membership-types/{id}/discounts` | GET | Per-type fan-out |
| Recurring Service Types | `/memberships/v2/.../recurring-service-types` | GET | Yes |
| Materials Markup | `/materialsmarkup` | GET, POST, PUT | — |

### 3.2 Pagination

All list endpoints return paginated results. Response includes `totalCount`, `page`, `pageSize`. Maximum `pageSize` is **250** in most cases.

```python
def get_all_pages(base_url, headers, params):
    results = []
    page = 1
    while True:
        params['page'] = page
        params['pageSize'] = 250
        resp = requests.get(base_url, headers=headers, params=params)
        data = resp.json()
        results.extend(data['data'])
        if len(results) >= data['totalCount']:
            break
        page += 1
    return results
```

### 3.3 The `?calculatePrices=true` query mode

When `/services` is called with `?calculatePrices=true`, the response includes a `calculatedPrice` object on each service describing what the dynamic pricing engine would produce *at the moment of the GET*. This is informational, not authoritative — the actual invoice price is recomputed at invoice-creation time using the markup rules then in effect.

Use this mode to feed price-search experiences (e.g., a voice agent looking up "what does a 50 gal NG water heater install cost"). Do not use it to derive pricing rules — those live in the markup engine.

### 3.4 Rate limits

The pricebook family enforces a rate limit cap of approximately **30 requests per 60 seconds**. A 145-row bulk material create executed sequentially through this governor takes ~10 minutes. Bulk endpoints (`POST /pricebook` and `PATCH /pricebook` for bulk create/update) significantly reduce this — adopt them when available.

---

## 4. Object Schemas (Observed on Live Tenants)

The shapes returned by the live API are richer than what most schema docs carry. Every field listed below has been observed on a real ST tenant call.

### 4.1 Service object

```json
{
  "id": 62295156,
  "code": "WHEH-140",
  "displayName": "50 gal Hybrid Water Heater Install",
  "description": "...",
  "warranty": { ... },
  "categories": [{ "id": 123, "name": "Plumbing > Residential > Water Heaters" }],
  "price": 0,
  "memberPrice": 0,
  "addOnPrice": 0,
  "addOnMemberPrice": 0,
  "taxable": false,
  "account": "Revenue",
  "hours": 0,
  "isLabor": false,
  "recommendations": [],
  "upgrades": [],
  "assets": [
    { "alias": null, "fileName": "uuid.", "isDefault": true,
      "type": "Image", "url": "Images/Equipment/ferguson/uuid." }
  ],
  "serviceMaterials": [{ "skuId": 12345, "quantity": 1 }],
  "serviceEquipment": [
    { "skuId": 75499590, "quantity": 1 },
    { "skuId": 77672794, "quantity": 1 }
  ],
  "active": true,
  "crossSaleGroup": null,
  "paysCommission": false,
  "bonus": 0,
  "commissionBonus": 0,
  "modifiedOn": "2026-03-16T10:54:58Z",
  "createdOn": "2024-01-15T00:00:00Z",
  "source": "...",
  "externalId": "...",
  "externalData": [],
  "businessUnitId": null,
  "cost": 0,
  "soldByCommission": null,
  "defaultAssetUrl": "...",
  "budgetCostCode": null,
  "budgetCostType": null,
  "calculatedPrice": { /* present when ?calculatePrices=true */ },
  "useStaticPrices": false
}
```

**Field notes:**

- `displayName` — the human-readable name. **Not** `name`. Mappers that read `name` will silently get empty strings.
- `categories` — array of `{id, name}` objects. The `name` is the full breadcrumb: `"Parent > Child > Grandchild"`.
- `assets[]` — list of attached images/files. **Not** `imageUrl`.
- `isLabor` — boolean. **Not** `serviceType === "Labor"`.
- `useStaticPrices` (plural, with the trailing `s`) — the actual ST field name. The singular form `useStaticPrice` is silently ignored. See §7.1.
- `addOnPrice` / `addOnMemberPrice` — accessory pricing for items added on top of a base service.
- `calculatedPrice` — only populated when GET includes `?calculatePrices=true`. Object shape, contains the dynamic-pricing result.
- `serviceMaterials` / `serviceEquipment` — arrays of `{skuId, quantity}`. `skuId` matches the equipment or material `id`. **There is no DELETE endpoint for individual links** — see §7.7.

### 4.2 Material object

```json
{
  "id": 87654321,
  "code": "RDP-6",
  "displayName": "6\" Round Duct Pipe",
  "description": "6-inch round duct pipe",
  "price": 15.50,
  "cost": 8.25,
  "memberPrice": 12.00,
  "active": true,
  "taxable": true,
  "unitOfMeasure": "EA",
  "account": "Revenue",
  "costOfSaleAccount": "Cost of Goods Sold:Materials",
  "assetAccount": null,
  "categories": [{ "id": 456, "name": "Round Duct Pipe" }],
  "vendorPricingLinks": [
    { "id": 111, "vendorId": 222, "vendorName": "Ferguson",
      "cost": 7.50, "vendorCode": "RDP6", "isPrimary": true }
  ],
  "primaryVendor": { /* see §4.4 */ },
  "isInventory": false,
  "modifiedOn": "2026-03-16T10:55:17Z",
  "createdOn": "2024-06-01T00:00:00Z"
}
```

**Unit-of-measure enum** (commonly observed): `EA` (each), `BX` (box), `RL` (roll), `SH` (sheet), `BG` (bag), `FT` (foot), `GL` (gallon), `CS` (case), `PR` (pair), `PK` (pack).

### 4.3 Equipment object

```json
{
  "id": 76293459,
  "code": "RE50M2RH95646893",
  "displayName": "50 gal. Medium 4.5kW 2-Element Residential Electric Water Heater",
  "description": "HTML description with specs",
  "active": true,
  "price": 0,
  "memberPrice": 0,
  "addOnPrice": 0,
  "addOnMemberPrice": 0,
  "manufacturer": "RHEEM WATER HEATERS ONLY",
  "model": "646893",
  "manufacturerWarranty": { "duration": 72, "description": "6-Year Tank & Parts" },
  "serviceProviderWarranty": { "duration": 12, "description": "1-Year Labor Warranty" },
  "categories": [62289918, 75114937],
  "assets": [...],
  "primaryVendor": { /* see §4.4 */ },
  "otherVendors": [...],
  "account": "Revenue",
  "costOfSaleAccount": "Cost of Goods Sold:Equipment",
  "cost": 494.7,
  "unitOfMeasure": null,
  "isInventory": false,
  "modifiedOn": "2026-04-03T16:53:11.215Z",
  "source": "ferguson",
  "externalId": "RE50M2RH95646893",
  "externalData": [],
  "isConfigurableEquipment": true,
  "variationsOrConfigurableEquipment": [76332404],
  "typeId": 111,
  "displayInAmount": false,
  "generalLedgerAccountId": 3516288,
  "createdOn": "2026-01-25T23:31:07.776Z",
  "defaultAssetUrl": "...",
  "dimensions": { "height": 46.5, "width": 24.5, "depth": 24.5 }
}
```

**Equipment-specific field notes:**

- `manufacturer` and `model` are separate fields. Brand can also be inferred from `source` (e.g., `"ferguson"`).
- `categories` here is a **flat array of integer IDs** (not the `{id, name}` shape services use). Resolve names via `/categories`.
- `manufacturerWarranty` / `serviceProviderWarranty` — `duration` is in months; `description` is free text.
- `isConfigurableEquipment` — boolean. The variant-management toggle (§5).
- `variationsOrConfigurableEquipment` — array of equipment IDs that are variants of this item when it's a configurable root. **Read-only via API** — UI only (§5, §7.3).
- `typeId` — integer Equipment Type ID. **Read-only via API** — UI only (§7.2).
- `dimensions` — object with `height`, `width`, `depth` (units typically inches).

### 4.4 The `primaryVendor` object

```json
"primaryVendor": {
  "id": 76293462,
  "vendorName": "Ferguson",
  "vendorId": 95,
  "memo": null,
  "vendorPart": "RE50M2RH95646893",
  "cost": 494.7,
  "active": true,
  "primarySubAccount": { "id": 76472611, "cost": 494.7, "accountName": "499958" },
  "otherSubAccounts": [...]
}
```

Sub-accounts are GL ledger sub-codes that vendor invoices post against. `accountName` is the numeric ledger string (e.g., `"499958"`), not the GL display name. `isPrimary` on a `vendorPricingLinks` row marks the canonical vendor for cost purposes — multiple rows may have `isPrimary: true` if multiple vendors are kept active for the same item.

---

## 5. Configurable Equipment

### 5.1 What it is

Configurable equipment is ServiceTitan's mechanism for letting techs **swap a generic placeholder for a specific model** at estimate time. When an equipment item has the **"Enable Configurable Equipment"** toggle ON (`isConfigurableEquipment: true`), the tech sees a swap dropdown on the estimate. They pick the exact unit instead of selling the generic root. This is how Good/Better/Best presentation works for water heaters, condensers, panels, etc.

**Concrete example.** A service `WHNG-130` ("50 gal Gas Water Heater Install") links to a *root* equipment item — the generic. That root item has `isConfigurableEquipment: true` and a `variationsOrConfigurableEquipment` array listing the specific Rheem / State / Rinnai models the tech can choose from. The variant items (children) appear in the swap dropdown when the tech is in the estimate configurator on the mobile app.

### 5.2 The three requirements for an item to appear as a variant

Every variant search bug traces to one of these three preconditions failing.

**1. Equipment Type must be ASSIGNED (not blank) on the variant.**

- Where: Equipment → [item] → Details tab → Type dropdown.
- If `Type = "Select Equipment Type..."` (i.e., blank), the item will **not appear** in the variant search dropdown anywhere.
- **Cannot be set via API** — `typeId`, `equipmentTypeId`, `type`, `equipmentType` are all silently ignored. UI only. (See §7.2.)
- When you set a Type that has Capacity Levels configured (e.g., "Tank Water Heater" uses Gallons), ST shows a yellow banner: *"Capacity is missing for this item, please select one based on the chosen equipment type"* — you must also set Capacity Level before saving.

**2. `isConfigurableEquipment` must be FALSE on the VARIANT item (not the root).**

- Where: Equipment → [item] → Configurable Equipment tab → "Enable Configurable Equipment" toggle.
- ST excludes items with this toggle ON from appearing in *other* items' variant search dropdowns. An item that is itself a configurable root cannot be listed as a variant of another root.
- The **root** item must have the toggle ON. The **variant** items must have it OFF.
- **This IS PATCHable via API:** `PATCH /equipment/{id}` with `{ "isConfigurableEquipment": false }`.

**3. Item must be Active.** Deactivated items do not appear in any search.

### 5.3 The root-vs-variant rule

An equipment item is **either** a root (configurable, toggle ON, linked to services) **or** a variant (not configurable, toggle OFF, appears as an option under roots). **It cannot be both.**

If you blanket-set `isConfigurableEquipment: true` on every equipment item via API, every item becomes invisible to every other item's variant search. This is one of the most common pricebook outages — easy to cause via a well-intentioned bulk PATCH, and the only fix is to batch-flip the variants back to false.

### 5.4 ST UI procedure for adding variants — step-by-step

1. **Pricebook → Equipment**
2. Toggle **Edit Mode** ON (top right).
3. Search for the **root / parent** equipment item by code.
4. Right-click → **"View/Edit Equipment"**.
5. Click the **"Configurable Equipment"** tab.
6. Click **"+ Add Equipment"** at the bottom.
7. In the search dropdown, type the **NAME** of the variant (not the code).
8. Wait 5–10 seconds for results to load.
9. Select the item — it appears in the Equipment Variations table.
10. Click **Save** (top right).

### 5.5 Search behavior gotchas (the variant-search dropdown is brittle)

- **Searches by NAME only.** Entering a code returns nothing.
- **Returns max ~6 results.** If your item isn't in the top 6, try a more specific search term.
- **5–10 second delay** before results appear. Patience.
- **Only returns items with Equipment Type assigned.** Blank-type items are invisible.
- **Name truncation** in the dropdown makes similar items hard to distinguish.
- **Best search terms:** the distinguishing 2–3 words. For "State 50 gal. 4.5kW Tall Electric Water Heater", search "State 50 gal" or "4.5kW Tall".

### 5.6 What works vs. doesn't work via API for configurable equipment

**Works:**

| Action | Method | Endpoint |
|---|---|---|
| Toggle `isConfigurableEquipment` | PATCH | `/equipment/{id}` body `{ "isConfigurableEquipment": true \| false }` |
| Read equipment details | GET | `/equipment/{id}` |
| Read equipment list | GET | `/equipment?page=1&pageSize=200&active=True` |
| List a root's variants | GET | `/equipment?parentEquipmentId={id}` |

**Does NOT work (silently ignored):**

| Action | What happens |
|---|---|
| Set `typeId` | Returns 200, value stays null |
| Set `equipmentTypeId` | Returns 200, value stays null |
| Set `type` | Returns 200, value stays null |
| Set `equipmentType` | Returns 200, value stays null |
| Modify `variationsOrConfigurableEquipment` array | Returns 200, array unchanged |

Equipment Type assignment and the variant array are **UI-only**.

### 5.7 Variant-not-appearing — troubleshooting decision tree

```
VARIANT item not in search dropdown?
├── Is "Enable Configurable Equipment" OFF on the VARIANT?
│   └── ON → Turn it OFF (UI or API PATCH isConfigurableEquipment: false)
│       ⚠️ THIS IS THE #1 BLOCKER
├── Is Equipment Type assigned (not "Select Equipment Type...")?
│   └── NO → Set in UI (Details tab → Type) + set Capacity Level if required
├── Did you search by NAME (not code)?
│   └── Used code → Search by name
├── Is the item Active (not Deactivated)?
│   └── Deactivated → Reactivate first
├── Did you wait 5–10 seconds?
│   └── Retry with patience
└── Still not appearing?
    ├── Try different search terms — most unique 2-3 words from the name
    ├── Check if search index needs time to refresh after Type was just set
    └── Possible ST bug — some items may not index properly
```

---

## 6. The Categories Export Format Trap

### 6.1 The encoding problem

The ServiceTitan pricebook Excel export (Settings > Pricebook > Export) represents category hierarchy using columns `Category1` through `Category9`. Each column corresponds to a depth level in the tree.

**The encoding rule:** Each row in the Categories sheet populates **only the column corresponding to its own depth level**. All shallower columns (parent levels) are left blank. The parent-child relationship is encoded by **row order and position**, not by explicit foreign-key references.

**Example — what the export looks like:**

| Row | Id | Category1 | Category2 | Category3 | Category4 | Category5 |
|---|---|---|---|---|---|---|
| 1 | 100 | Electric Department | | | | |
| 2 | 101 | | Electrical Services | | | |
| 3 | 102 | | | Residential | | |
| 4 | 103 | | | | Lighting | |
| 5 | 104 | | | | | Fixtures |
| 6 | 105 | | | | | Outdoor Fixtures |

Row 5 ("Fixtures", id 104) has only `Category5` populated. A naive parser reading row 5 in isolation would classify Fixtures as a root-level node with no parent context — **incorrectly**. The full path is `Electric Department > Electrical Services > Residential > Lighting > Fixtures`, derived by reading the preceding rows.

### 6.2 Positional reconstruction algorithm

```python
def reconstruct_category_paths(categories_df):
    """
    categories_df: pandas DataFrame from the 'Categories' sheet of an ST pricebook export.
    Returns: dict mapping category ID → full path string.
    """
    current = {f'Category{i}': '' for i in range(1, 10)}
    id_to_path = {}

    for _, row in categories_df.iterrows():
        # Find the deepest populated level in this row
        deepest_col = None
        deepest_level = 0

        for level in range(1, 10):
            col = f'Category{level}'
            val = row.get(col, '')
            if pd.notna(val) and str(val).strip():
                deepest_col = col
                deepest_level = level

        if deepest_col is None:
            continue

        # Update running state at the deepest level
        current[deepest_col] = str(row[deepest_col]).strip()

        # Clear all deeper levels
        for deeper in range(deepest_level + 1, 10):
            current[f'Category{deeper}'] = ''

        # Build full path from current state
        path_parts = [
            current[f'Category{i}']
            for i in range(1, deepest_level + 1)
            if current[f'Category{i}']
        ]
        full_path = ' > '.join(path_parts)
        id_to_path[str(row.get('Id', '')).strip()] = full_path

    return id_to_path
```

### 6.3 Why this matters

Failing to apply the positional reconstruction algorithm produces what appears to be a large number of "orphan" categories — categories classified at a single depth level with no visible parent. This is a systematic error that invalidates any category hierarchy analysis derived from the export.

**Worked example:** An initial export audit classified ~188 Electrical services as "root-level orphans" when they were correctly nested 4–5 levels deep in the Electric Department hierarchy. The false orphan classification triggered a remediation effort targeting non-existent problems while genuine miscategorizations were lost in the noise.

**Verification rule:** Before any pricebook category analysis, verify the reconstruction is working by tracing 3–5 known categories through the export and confirming the reconstructed path matches what is displayed in the ST Pricebook UI.

### 6.4 Export vs. API representation

| Aspect | XLSX Export | API Response |
|---|---|---|
| Parent reference | Implicit (row position) | Explicit `parentId` / `parentCategoryId` field |
| Hierarchy reconstruction | Required algorithm | Walk the `parentId` chain |
| Multi-level path | Must be computed | Must be computed |
| Category names | In `Category1`–`Category9` columns | In `name` field |
| Use case | Bulk import template | Programmatic read/write |

For programmatic hierarchy reconstruction, use the API. The export is primarily useful for preparing bulk import files and is the only way to make many bulk changes.

### 6.5 Multi-category items — the export limitation

The XLSX export does not support representing multi-category assignments. When a service is assigned to multiple categories, the export shows only one (typically the primary). **Bulk import files cannot create multi-category assignments — this requires the API.**

API payload for multi-category:

```json
PATCH /pricebook/v2/tenant/{tenantId}/services/{serviceId}
{
  "categories": [
    { "id": 21002017 },
    { "id": 62291292 }
  ]
}
```

The `categories` array is a **complete replacement, not append**. To add a category to an existing multi-category assignment, include all existing category IDs plus the new one. Omitting an existing category ID removes that assignment.

Legitimate reasons for multi-category assignment:
1. **Dispatch fees** that legitimately apply across trades.
2. **Cross-trade accessories** — e.g., a generator install service that needs to appear in both Electrical and Generators trees.
3. **Cross-residential/commercial fees** — items billed identically to both segments.

Document the dual home in the description so a future audit knows it was intentional.

---

## 7. The "Don't Do" Catalogue (Silent-Failure Gotchas)

ServiceTitan returns **HTTP 200 OK on every PATCH**, including PATCHes with misspelled fields, read-only fields, or deprecated fields. There is no error, no warning, no diagnostic header. You only learn the field was ignored if you GET the item afterward and diff.

> **The cardinal rule:** every write needs `GET → mutate → PATCH → GET → diff`. **Always GET after PATCH.**

Below is the catalogue of silent failures observed on production tenants. Every one returns 200 OK.

### 7.1 `useStaticPrice` (singular) is silently ignored — use `useStaticPrices` (plural)

The actual field name is `useStaticPrices` with a trailing `s`. The singular form is silently ignored. Documentation has historically carried both spellings; use the plural in any new code.

### 7.2 Equipment `typeId` PATCH is silently ignored — UI only

Every variant — `typeId`, `equipmentTypeId`, `type`, `equipmentType` — returns 200 OK and does nothing. The Equipment Type can only be set in the ST UI: **Pricebook → Equipment → [item] → Details tab → Type dropdown**.

### 7.3 `variationsOrConfigurableEquipment` array PATCH is silently ignored — UI only

The array of variant equipment IDs on a configurable root item is read-only. Variant lists must be managed in the UI: **Pricebook → Equipment → [root item] → Configurable Equipment tab → "+ Add Equipment"**.

The corollary: the *flag* `isConfigurableEquipment` IS PATCHable. Only the array of variants is not. So you can toggle an item to be a configurable root via API, but you have to walk into the UI to attach its children.

### 7.4 `isConfigurable` is the wrong field name — use `isConfigurableEquipment`

`isConfigurable` does not exist on the equipment schema. PATCHing it returns 200 with no change. Use the longer name.

### 7.5 `useStaticPrices: null` IS the dynamic-pricing default — not a bug

On services, `useStaticPrices` tri-states:

| Value | Meaning |
|---|---|
| `null` | Dynamic-priced (default for most tenants). `cost` and the markup engine produce the invoice price. |
| `true` | Static-priced. `price` and `memberPrice` are canonical; markup engine bypassed. |
| `false` | Explicit dynamic. Same as `null` for most purposes. |

Do not treat `useStaticPrices: null` as a missing field, missing data, or a sync bug. The vast majority of services on a dynamic-pricing tenant are `null`. Only services with `useStaticPrices = true` carry a settable API price.

### 7.6 `cost` PATCH on dynamic-pricing services is silently dropped

PATCHing `cost` on a service whose `useStaticPrices` is `null` or `false` returns 200 OK with no change. Cost on dynamic-pricing services must be set in the ST UI only.

This is **not the same as cost being read-only** — equipment `cost` IS PATCHable, and so is service `cost` for static-priced services. The behavior is gated on `useStaticPrices`.

**Operational consequence:** `cost = 0` on a dynamic-pricing service is **not** a data error. ST does not return a cost value for these via API. Reports that flag zero-cost services should split queries by `use_static_prices`:

```sql
-- Real data quality issue: static-priced service with no price
SELECT code, name, price FROM pb_services
WHERE active = 1 AND use_static_prices = 1 AND price = 0;

-- NOT a bug: dynamic-priced services with cost = 0 (expected)
SELECT code, name FROM pb_services
WHERE active = 1 AND (use_static_prices IS NULL OR use_static_prices = 0)
  AND cost = 0;
```

Static-priced services with `price = 0` AND `active = 1` are the actual billing risk — those genuinely will invoice at $0.

### 7.7 No DELETE on service-equipment / service-material links

The endpoint `DELETE /services/{serviceId}/equipment/{equipmentId}` returns 404 — it does not exist. The only way to remove a single linked equipment from a service is to GET → filter the array client-side → PATCH the service back with the filtered array.

```javascript
const svc = await getService(token, serviceId);
const filtered = svc.serviceEquipment.filter(eq => eq.skuId !== removeId);
await patchService(token, serviceId, { serviceEquipment: filtered });
```

Same pattern for `serviceMaterials`. The match is on `skuId` (which equals the equipment/material `id`).

### 7.8 Unknown / misspelled fields return 200 with no change

Generalization of §7.1 through §7.4. ST's PATCH endpoints accept any JSON body and return 200 OK regardless of whether any of the fields are recognized. There is no per-field acknowledgement. **Diff after PATCH or you do not actually know whether your change took effect.**

### 7.9 `service-titan:makeAnApiCall` (Make.com module) drops request bodies

The Make.com native ServiceTitan integration has a module called `service-titan:makeAnApiCall`. **It silently drops request bodies.** Any PATCH or POST routed through it lands at ST as a body-less call, which the API treats as a no-op.

Use HTTP modules with direct OAuth, or front your writes through a serverless proxy with a clean `fetch()` call.

### 7.10 `/task-management/` (with hyphen) returns 404 — use `/taskmanagement/`

Not pricebook-specific but in the same family of silent-failure path bugs. ST's task management API lives at `/taskmanagement/v2/`, no hyphen. Tasks are also **READ + CREATE only** — PATCH and PUT on tasks return 200 OK and do nothing.

### 7.11 `active=Any` returns inactives FIRST in pagination

Tenants with large numbers of inactive items (40,000+ inactive services is common after years of operation) hit a sort-order trap. When `/services` is called with `active=Any` (or no `active` filter), ST returns **inactives first** in pagination. With a default 50-page cap (10,000 rows), the active services may be buried past the cap and never returned.

**Fix:** default sync calls to `active=True`. Active-only payloads are a fraction of the size and every downstream consumer almost certainly filters `active=1` anyway.

### 7.12 Equipment uses static pricing — `$0 price` IS a real gap (unlike services)

Unlike services, equipment items use **static pricing** in ST. There is no calculated-price equivalent. So `price = 0 AND active = 1` on equipment is a legitimate data error — the item will sell at $0 if a tech adds it to an estimate.

A real audit on a multi-trade tenant found 87 equipment items with $0 price but >$0 cost — about **$146K in cost exposure with no revenue offset**. Run this query monthly.

### 7.13 Configurable-equipment variation conflict on save

When you try to save an equipment item in the ST UI whose `variationsOrConfigurableEquipment` array contains another item that is itself flagged `isConfigurableEquipment: true`, ST throws:

> *Action cancelled — Equipment {id} is Configurable and cannot be assigned as a Variation.*

This typically hits after a bulk-set of `isConfigurableEquipment: true` on too many items. Workaround: turn OFF `isConfigurableEquipment` on the variant first → save → turn ON the toggle on the *root* → re-attach the variant → save.

**Long-term fix:** an item is either a root or a variant, never both. Run a periodic sweep on items intended as pure variants and PATCH them back to `isConfigurableEquipment: false`.

### 7.14 Deactivated codes — do not reuse

Reusing a deactivated pricebook code for a different item creates permanent semantic ambiguity in historical line items. Never reuse a deactivated code; always mint a new one.

---

## 8. Dynamic Pricing Model

### 8.1 The mental model

Most multi-trade tenants run dynamically-priced services. The model:

- **Service price** = (`cost` × markup multiplier from `materialsmarkup` rules) + adjustments. Computed at invoice time, not stored.
- **Member price** = service price minus the member discount that applies to the service's category, sourced from membership-discount rules (§9).
- **Equipment price** = static `price` field on the equipment item. No markup engine.
- **Material price** = static `price` field. No markup engine.

### 8.2 The markup multiplier

A common markup pattern: **per-item-tiered multiplier sliding from `5.50x` (lowest direct cost) down to `1.50x` (highest direct cost)**. Rationale:

- A $5 fitting marked up at 1.5x produces only $7.50 — not enough to cover handling, returns, and write-offs.
- A $4,000 hybrid water heater marked up at 5.5x ($22,000) is unsellable. The market sets a ceiling.

A sliding multiplier reconciles both ends and produces realistic margins across the full cost spectrum. Configure these rules under `/pricebook/.../materialsmarkup`.

### 8.3 The `calculatedPrice` field

When a GET call uses `?calculatePrices=true`, ST attaches a `calculatedPrice` object to each service describing what the dynamic engine would produce *at the moment of the GET*. This is informational, not authoritative — the actual invoice price is recomputed at invoice creation time using the markup rules then in effect.

### 8.4 Member pricing

`memberPrice` is a real settable field on services, materials, and equipment. But on tenants using rule-based membership pricing, it is **for display only** — at invoice time the engine applies the membership-discount rule for the customer's active membership (§9). The displayed `memberPrice` may not match the price the customer actually sees on the invoice.

---

## 9. The Pricing-Rule Subsystem (Four Sibling Tables)

Together with the per-customer `memberships` table, these four tables capture ST's dynamic-pricing rule definitions. They are the inputs that explain *why* a particular invoice line came out at the price it did.

### 9.1 `discounts-and-fees`

Standalone discounts and fees. Endpoint: `GET /pricebook/v2/.../discounts-and-fees`.

```sql
CREATE TABLE pb_discounts_and_fees (
  code TEXT PRIMARY KEY,
  id INTEGER,
  display_name TEXT NOT NULL,
  description TEXT DEFAULT '',
  type TEXT,                    -- "Discount" | "Fee" | etc.
  amount_type TEXT,             -- "Percent" | "Dollar"
  amount REAL DEFAULT 0,
  amount_limit REAL DEFAULT 0,
  taxable INTEGER DEFAULT 0,
  account TEXT DEFAULT '',
  active INTEGER DEFAULT 1,
  categories_json TEXT DEFAULT '[]',
  pays_commission INTEGER DEFAULT 0,
  exclude_from_payroll INTEGER DEFAULT 0,
  hours REAL DEFAULT 0,
  bonus REAL DEFAULT 0,
  commission_bonus REAL DEFAULT 0,
  cross_sale_group TEXT,
  updated_at TEXT
);
```

The `pays_commission` flag controls whether the discount/fee feeds into payroll commission calculations.

### 9.2 `membership-types`

Membership type definitions. Endpoint: `GET /memberships/v2/.../membership-types`.

```sql
CREATE TABLE pb_membership_types (
  id INTEGER PRIMARY KEY,
  name TEXT NOT NULL,
  display_name TEXT DEFAULT '',
  active INTEGER DEFAULT 1,
  discount_mode TEXT,                -- "Basic" | "Units"
  discount REAL DEFAULT 0,
  use_membership_pricing_table INTEGER DEFAULT 0,
  show_membership_savings INTEGER DEFAULT 1,
  auto_calculate_invoice_templates INTEGER DEFAULT 0,
  billing_template_id INTEGER,
  revenue_recognition_mode TEXT,
  location_target TEXT,
  duration_billing TEXT,
  tag_type_ids_json TEXT DEFAULT '[]',
  recurring_services_json TEXT DEFAULT '[]'
);
```

`discount_mode`:
- `"Basic"` — flat percent discount on everything.
- `"Units"` — category-by-category or SKU-by-SKU rules (see §9.3).

`recurring_services_json` lists the recurring service templates available under this membership type (see §9.4).

### 9.3 `membership-discounts`

Per-membership-type discount rules. **No list endpoint exists** — must fan-out per membership type:

```
GET /memberships/v2/.../membership-types/{id}/discounts
```

```sql
CREATE TABLE pb_membership_discounts (
  id INTEGER PRIMARY KEY,
  membership_type_id INTEGER NOT NULL,
  target_id INTEGER NOT NULL,    -- category ID or SKU ID, depending on parent's discount_mode
  discount REAL DEFAULT 0
);
```

Sync pattern: `DELETE FROM pb_membership_discounts WHERE membership_type_id IN (...)` then upsert. Per-type 404/500 errors should log and skip — they should not abort the whole sync.

### 9.4 `recurring-service-types`

Templates for auto-scheduling recurring memberships. Endpoint: `GET /memberships/v2/.../recurring-service-types`.

```sql
CREATE TABLE pb_recurring_service_types (
  id INTEGER PRIMARY KEY,
  active INTEGER DEFAULT 1,
  recurrence_type TEXT,                -- "Yearly" | "Monthly" | etc.
  recurrence_interval INTEGER,
  recurrence_months_json TEXT DEFAULT '[]',
  recurrence_days_string TEXT,
  recurrence_week TEXT,
  duration_type TEXT,
  duration_length INTEGER,
  invoice_template_id INTEGER,
  business_unit_id INTEGER,
  job_type_id INTEGER,
  priority TEXT,
  campaign_id INTEGER,
  job_summary TEXT,
  estimated_revenue REAL,
  duration_estimate INTEGER
);
```

A membership type's `recurring_services_json` array names which of these templates apply. When the renewal cycle triggers, ST schedules a job per recurring service against the customer's location.

---

## 10. Categories — Tree Structure and Multi-Category Items

### 10.1 The standard category tree shape

A clean category tree has a consistent depth pattern:

- **Level 1:** Department (HVAC, Plumbing, Electrical, Generators)
- **Level 2:** Residential vs. Commercial
- **Level 3:** Service mode (Service / Sales / Maintenance / Install / Replace)
- **Level 4:** Equipment family (Water Heaters, Furnaces, Panels)
- **Level 5+:** Variant (Electric / Natural Gas / Propane / Tankless)

Every item lives at a leaf. Multi-category items are the explicit, intentional exception (§6.5).

### 10.2 Category creation via API

```json
POST /pricebook/v2/tenant/{tenantId}/categories
{
  "name": "Commercial Lighting",
  "active": true,
  "parentCategoryId": 62587123,
  "categoryType": "Services"
}
```

`parentCategoryId` sets the new category's position in the hierarchy. Category creation via API uses explicit parent IDs (unlike the export's positional encoding).

### 10.3 Common category-related audit findings

Across multi-trade tenant audits, the same classes of issues recur:

- **Commercial services parented under residential or generic branches** of the category tree.
- **Dispatch fees in only one category** when they should be multi-categorized.
- **Generator and cross-trade items orphaned** outside their proper department tree.
- **Lighting / specialty subcategories missing entirely**; items live at the wrong level.
- **DNU items still active** and appearing in tech pickers.
- **Duplicate items with slightly different names** (`Drain Auger` vs. `Drain Auger Service`) that should be merged.
- **Items with no markup tier set** falling through to a default that doesn't match current pricing.
- **Estimate Templates referencing deactivated SKUs.**

---

## 11. Estimate Templates — Good / Better / Best / Ultra

For any installable equipment family (water heaters, furnaces, condensers, electrical panels, generators), the four-tier template structure follows a consistent logic:

| Tier | What's included | Typical GP% | Customer profile |
|---|---|---|---|
| **Good** | Equipment + minimum code-required accessories | Highest (often 47–55%) | Price-sensitive |
| **Better** | Good + extended warranty kit + branded smart accessory | Mid (40–54%) | Most common middle pick |
| **Best** | Better + premium accessory tier (water-treatment, leak detection) | Lower (36–48%) | Quality-driven |
| **Ultra** | Best + tier-up equipment swap (hybrid, condensing) | Lowest (35–38%) | Aspirational; long-term cost-of-ownership focus |

Higher tiers carry lower GP percentages but **higher absolute gross-profit dollars** and better long-term retention via warranty and smart-home stickiness.

**Implementation rule:** Each tier should reference the **same equipment SKU** when possible, with different bundled accessories — not four separate equipment SKUs. This keeps the Pricebook lean and enables clean tier-mix reporting downstream.

### 11.1 Labor tiers

A widely-used three-tier labor pricing scaffold for installation work:

| Tier | Labor structure | Rationale |
|---|---|---|
| **Tier 1: Main Install** | Flat labor rate (e.g., $325 flat at $110/hr burdened) | Predictable customer-facing pricing |
| **Tier 2: Add-On Services** | Flat labor reduced ~20% from Tier 1 (e.g., $260 flat) | Tech is already on site — discount for efficiency |
| **Tier 3: Accessories** | Hours × direct rate (e.g., $50/hr direct); spiff included in job cost | Variable, low-friction add-ons |

Tech spiff (incentive payment) is included in job cost. In Estimate Template line economics, spiff appears as a negative number reducing gross profit.

---

## 12. The Ten Golden Rules of Pricebook Item Creation

1. **Codes are SKUs and SKUs are stable.** Once on a customer invoice, never reuse with different meaning.
2. **Trade prefixes are reserved.** Use the existing prefix dictionary; create new prefixes only when no existing one applies.
3. **Names are search targets.** Most-searched word leads. Industry abbreviations preferred.
4. **Categories form a tree; items live at leaves.** Multi-category is the intentional exception.
5. **Multi-category items must be intentional and documented.**
6. **Estimate Templates are bundles, not items.** Don't create N SKUs for N tiers of the same equipment.
7. **Markup is per-item-tiered.** Not flat.
8. **Labor tiers exist for a reason.** Tier 1 main install, Tier 2 add-on (already on site), Tier 3 accessories.
9. **Deactivate, don't delete.** Reporting depends on it.
10. **Never reuse a deactivated code for a different item.**

---

## 13. Bulk Import — XLSX Format

The pricebook XLSX import (Settings → Pricebook → Import) accepts a file with specific sheet structure:

**Services sheet required columns:**
- `Name` — service display name
- `Code` — unique service code (SKU)
- `Category.Id` — numeric ST category ID of the target category
- `Active` — TRUE/FALSE
- `Price` — service price (or 0 for dynamic-priced)
- `Hours` — estimated labor hours

**Import behavior:**
- Matched by `Code`: existing services with matching codes are updated.
- New codes: new services are created.
- Category assignment: **single category only** (multi-category requires API).
- Deactivation: setting `Active=FALSE` deactivates the service.

---

## 14. Common SQL Recipes (for a Pricebook Mirror)

A pattern for ingesting and querying pricebook data in a local store (e.g., D1, Postgres, BigQuery). Schemas above (§9) plus matching tables for `pb_services`, `pb_materials`, `pb_equipment`. The queries below assume that shape.

### 14.1 The "real billing risks" query

```sql
SELECT code, display_name, price, cost
FROM pb_services
WHERE active = 1 AND use_static_prices = 1 AND price = 0;

SELECT code, display_name, price, cost
FROM pb_equipment
WHERE active = 1 AND price = 0 AND cost > 0;
```

### 14.2 The configurable-root-with-no-variants query

```sql
SELECT e.id, e.code, e.display_name
FROM pb_equipment e
WHERE e.is_configurable_equipment = 1
  AND e.active = 1
  AND NOT EXISTS (
    SELECT 1 FROM pb_equipment v
    WHERE v.parent_equipment_id = e.id AND v.active = 1
  );
```

### 14.3 The orphan-equipment-type query

```sql
SELECT code, display_name
FROM pb_equipment
WHERE active = 1 AND (type_id IS NULL OR type_id = 0);
```

These items will not appear in any variant search dropdown (§5.2 requirement #1). Set Equipment Type via UI.

---

## 15. Pricebook Audit Playbook

Every pricebook cleanup follows the same four-phase shape (covered generally in `00-overview-and-naming.md` §5; this section is the pricebook-specific application).

**Step 1 — Export.** Pull the full pricebook export. Categories sheet must be parsed positionally (§6).

**Step 2 — Classify.** For each item, decide:
- Correct? (right category, right code, right markup tier)
- Wrong category? (commercial under residential, generator under wrong trade)
- Needs multi-category? (dispatch fees, cross-trade items)
- Orphan? (no category at all, or stranded under a deprecated branch)
- Deactivate? (DNU, replaced, no recent line items)

**Step 3 — Plan.** Produce:
- Excel of changes with `current state` and `target state` columns.
- Execution Markdown describing API payloads.
- Multi-category item list with explicit dual homes documented.

**Step 4 — Execute.** Via the API, an edge proxy, or browser automation for UI-only changes (Equipment Type, configurable-equipment variant lists). Validate each batch — if a bulk PATCH errors, bisect to isolate the offending record.

---

## 16. Forward-Roadmap Items to Track

Capabilities ST has been adding to the pricebook API surface. Track these in your own roadmap:

- **Bulk POST/PATCH `/pricebook` endpoints** — `PricebookBulk_Create` / `PricebookBulk_Update`. A 145-row create takes ~10 minutes today through the per-item rate limit; bulk would do it in one call. Same authentication, batched payload.
- **Reporting API as generic export layer** — `POST /reporting/v2/.../report-category/{cat}/reports/{reportId}/data` returns rows from saved reports as JSON. Replaces hand-rolled SQL aggregations and stays correct when ST changes the schema. See `10-api-release-notes-st75-st76.md` §3.
- **Materials Markup endpoints** — `GET/POST/PUT /pricebook/v2/.../materialsmarkup` exposes the markup-engine rules that drive every dynamic-priced service's invoice price. Today this is opaque to most integrators.
- **Pricebook category CRUD via API** — `POST/PATCH/DELETE /categories/{id}` is published but adoption is uneven. Useful as you grow the pricebook with new equipment types.

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants.*
