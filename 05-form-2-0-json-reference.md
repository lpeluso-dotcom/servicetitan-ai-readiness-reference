# 05 — Form 2.0 JSON Reference

> **Part of:** ServiceTitan AI-Readiness Reference  
> **File:** `05-form-2-0-json-reference.md`  
> **Author:** Luke Peluso, Business Systems Manager, Quality Service Company  
> **Status:** Public reference — community-sourced findings  
> **Audience:** Developers programmatically authoring, importing, or migrating ServiceTitan Form 2.0 documents.

This file is the complete schema and behavior reference for ServiceTitan Form 2.0 — the JSON-driven dynamic form engine embedded in the ServiceTitan mobile and web platforms. Every detail here is derived from live tenant exports, import-validation responses, and runtime testing on production tenants. None of it is from official ServiceTitan documentation, which does not publicly document Form 2.0 at the level required for programmatic authoring.

For Forms as a configuration entity (types, triggers, naming, audit), see `04-job-types-forms-tags.md` §3.

---

## 1. Architecture and Activation

ServiceTitan Form 2.0 is a **client-side rendered** dynamic form engine. Forms are authored as JSON documents and imported via **Settings → Operations → Forms → Import**. Once imported, forms attach to job types, call types, equipment records, customer records, or location records via the `ownerTypes` property.

Form 2.0 differs from the legacy Form 1.0 system in critical ways:
- Form 2.0 supports **conditional logic** through a rule engine that evaluates field values at runtime and toggles field visibility.
- Form 1.0 forms do not support conditional logic and use a different schema.
- The presence of `"IsForm20": true` at the root of the JSON document is the flag that activates the Form 2.0 rendering engine. **Without this flag, conditional logic is silently ignored.**

The rendering engine is entirely client-side. Forms are evaluated in the browser or mobile app, not on the server. This has critical implications:
- The engine has strict performance limits — most importantly the **two-action-per-rule constraint** (§9).
- All rule evaluation happens in real time as technicians interact with the form.
- Every rule condition is re-evaluated on every field change.

Forms are stored as **versioned definitions**. Each import creates a new definition version. The `definition.version` integer increments with each published change. The platform maintains version history but rollback through the API or import mechanism is not directly supported in the standard UI.

---

## 2. Top-Level JSON Envelope

The root of every Form 2.0 JSON document contains exactly four top-level keys:

```json
{
  "FormData": { ... },
  "FormTriggers": [],
  "IsForm20": true,
  "IsLocked": false
}
```

| Key | Type | Required | Notes |
|---|---|---|---|
| `FormData` | Object | Yes | All form metadata and the full field/rule definition |
| `FormTriggers` | Array | Yes | Trigger objects for automatic form dispatch; typically `[]` for manually-used forms |
| `IsForm20` | Boolean | Yes | **Must be `true`** to activate the Form 2.0 rendering engine |
| `IsLocked` | Boolean | Yes | When `true`, prevents editing in the UI; set to `false` for editable imports |

Key order at the top level is not enforced by the parser, but conventional order from live exports is `FormData` first, then `FormTriggers`, `IsForm20`, `IsLocked`.

---

## 3. The FormData Object

The largest object in the document. Contains all form-level metadata and the nested definition.

```json
{
  "FormData": {
    "id": 9999,
    "name": "Form Display Name",
    "active": true,
    "locked": false,
    "hide": false,
    "ownerTypes": ["Job"],
    "applyTagsTargets": ["Job"],
    "showOnMobile": true,
    "allowEmptyToPresent": false,
    "canDelete": false,
    "sendOnEvents": [],
    "dwyerBigBoardReport": false,
    "definition": { ... },
    "createdOn": "2026-01-01T00:00:00.000Z",
    "modifiedOn": "2026-01-01T00:00:00.000Z",
    "BusinessUnits": null,
    "ApplyTagTypes": null
  }
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | Integer | Yes — **must be non-null** | Any integer; ServiceTitan overwrites with its own ID on import. `null` causes 400. |
| `name` | String | Yes | Display name in Forms list and form header |
| `active` | Boolean | Yes | `true` = available for use |
| `locked` | Boolean | Yes | `false` = editable in UI |
| `hide` | Boolean | Yes | `false` = visible in forms list |
| `ownerTypes` | Array of strings | Yes | Record types the form attaches to. See §16. |
| `applyTagsTargets` | Array of strings | Yes | Record types that receive tags from `ApplyTags` rule actions |
| `showOnMobile` | Boolean | Yes | `true` = appears in mobile app |
| `allowEmptyToPresent` | Boolean | Yes | `true` = can be presented even with no fields filled |
| `canDelete` | Boolean | Yes | `false` = cannot be deleted from UI |
| `sendOnEvents` | Array | Yes | Event triggers for automatic dispatch; typically `[]` |
| `dwyerBigBoardReport` | Boolean | Yes | Integration flag for Dwyer Big Board reporting; typically `false` |
| `definition` | Object | Yes | Versioned form definition. See §4. |
| `createdOn` | ISO 8601 string | Yes | Format: `"2026-01-01T00:00:00.000Z"` |
| `modifiedOn` | ISO 8601 string | Yes | UTC |
| `BusinessUnits` | null or Array | Yes | `null` = applies to all BUs |
| `ApplyTagTypes` | null or Array | Yes | `null` = all tag types eligible for ApplyTags action |

---

## 4. The Definition Object — Two-Level Nesting

The `definition` key inside `FormData` is a versioned wrapper around the actual form definition. **This two-level nesting is consistent across all live exports.**

```json
"definition": {
  "id": 1,
  "version": 1,
  "published": true,
  "definition": {
    "rootSection": { ... },
    "rules": [ ... ],
    "hasConditionalLogic": true
  },
  "printOptions": []
}
```

### 4.1 Outer definition

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | Integer | Yes — **must be non-null** | Any integer; overwritten on import. `null` causes 400 at path `'definition'`. |
| `version` | Integer | Yes | Increments on each published change. Use `1` for new forms. |
| `published` | Boolean | Yes | `true` = active published version |
| `definition` | Object | Yes | Inner definition object |
| `printOptions` | Array | Yes | Print layout config; typically `[]`. **Must be at this level**, not inside the inner `definition`. |

### 4.2 Inner definition

```json
"definition": {
  "rootSection": { ... },
  "rules": [ ... ],
  "hasConditionalLogic": true
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `rootSection` | Object | Yes | Root container for all form fields and sections |
| `rules` | Array | Yes | Array of rule objects. Empty `[]` if no conditional logic. |
| `hasConditionalLogic` | Boolean | Yes | **Must be `true` when any rules exist.** If `false` or absent while rules are present, conditional logic does not execute. |

> **Trap:** `printOptions` placement. In hand-crafted forms it is often incorrectly placed inside the inner `definition`. Live exports confirm it belongs at the outer `definition` level alongside `id`, `version`, `published`. Incorrect placement does not error on import but may cause print layouts to be ignored.

---

## 5. Root Section and Unit Hierarchy

The `rootSection` is the top-level container for all form content. Every field, sub-section, and static element is a "unit" nested within a section's `units` array.

```json
"rootSection": {
  "unitType": "Section",
  "canBeDuplicated": false,
  "units": [ ... all top-level units ... ],
  "name": "Root",
  "id": "00000000-0000-4000-8000-000000000000"
}
```

### 5.1 Three confirmed unit types

| `unitType` | Description |
|---|---|
| `Field` | An interactive form field. Must have a `fieldType`. |
| `Section` | A container that groups fields. Can be nested. Can be duplicatable. |
| `StaticElement` | A non-interactive HTML display element. Has no `fieldType`. |

### 5.2 The Section object

```json
{
  "unitType": "Section",
  "canBeDuplicated": false,
  "units": [ ... nested units ... ],
  "name": "Section Display Name",
  "id": "valid-uuid-v4"
}
```

When `canBeDuplicated` is `true`, the technician sees an "Add Another" button — useful for multi-unit jobs where the same inspection repeats per unit.

Sections can be nested to any depth; practical forms rarely exceed two levels. **Conditional logic can target a `Section` unit directly** — toggling a section hides or shows all of its contents simultaneously. This is the most efficient way to manage large groups of related fields. (See §13 Pattern 2.)

---

## 6. Field Types — Complete Catalog

All field types listed here have been confirmed from live tenant exports. Any `fieldType` value not in this list causes a `400 ValidationException` on import: `"Error converting value 'X' to type 'ServiceTitan.Forms.Client.Contracts.FormFieldType'"`.

### 6.1 Text

```json
{
  "fieldType": "Text",
  "unitType": "Field",
  "isRequired": false,
  "name": "Field Label",
  "description": "Helper text shown below the field",
  "id": "valid-uuid",
  "smartFieldTag": ""
}
```

`smartFieldTag` is optional — when populated with a valid ServiceTitan smart field identifier (e.g., `"JobNumber"`, `"CustomerName"`), the field auto-populates from the associated record. See §18.

### 6.2 Dropdown

```json
{
  "fieldType": "Dropdown",
  "values": ["Option A", "Option B", "Option C"],
  "unitType": "Field",
  "isRequired": false,
  "name": "Field Label",
  "description": "",
  "id": "valid-uuid"
}
```

The `values` array must come **before** `unitType` in the key ordering to match the confirmed schema from live exports. JSON parsers are technically order-agnostic, but the ingestion engine has been observed to behave differently with certain key orderings in edge cases.

When used as a condition trigger, the `Compare` condition's `right` value must **exactly match** one of the strings in `values`, including capitalization and spacing.

### 6.3 Checkbox

The most important field type for conditional logic — the primary trigger for show/hide rules.

```json
{
  "fieldType": "Checkbox",
  "values": ["Yes"],
  "horizontalLayout": true,
  "unitType": "Field",
  "isRequired": false,
  "name": "Field Label",
  "description": "",
  "id": "valid-uuid"
}
```

> **Critical:** The `values` array for a Checkbox must always be `["Yes"]`. This is not a display value — it is the internal value that the rules engine evaluates. When writing a `Compare` condition targeting a checkbox, the `right` value must always be the string `"Yes"`, never `"true"`, `"1"`, `"checked"`, or the field's display name.

`horizontalLayout` controls whether the label appears to the right of the checkbox (`true`) or below it (`false`). Live exports consistently use `true`.

### 6.4 Picture

Camera/photo capture button in the mobile app.

```json
{
  "fieldType": "Picture",
  "unitType": "Field",
  "isRequired": false,
  "name": "Photo Label",
  "id": "valid-uuid"
}
```

> **Critical:** The correct value is `"Picture"`, **not** `"Photo"`. `"Photo"` causes `400 ValidationException`. The `Picture` field type does not support `description` or `values` — including them does not cause an error but they are ignored.

### 6.5 Stoplight

Three-state toggle: **Green (Pass), Yellow (Warning/Monitor), Red (Fail/Replace)**. The most visually powerful field type for inspection and assessment forms.

```json
{
  "fieldType": "Stoplight",
  "unitType": "Field",
  "isRequired": false,
  "name": "Inspection Item",
  "description": "Green = Pass, Yellow = Monitor, Red = Fail",
  "id": "valid-uuid"
}
```

Ideal for: equipment condition assessments, safety checks, inspection pass/fail items, urgency indicators, technician recommendations. The `description` is strongly recommended — tells the technician what each color means in context.

Using Stoplight as a rule trigger has **not been confirmed** from live exports. Use it as an output/assessment field rather than a logic trigger.

### 6.6 Number

```json
{
  "fieldType": "Number",
  "unitType": "Field",
  "isRequired": false,
  "name": "Numeric Field Label",
  "description": "",
  "id": "valid-uuid"
}
```

No min/max validation properties have been confirmed from live exports. Renders a numeric keyboard input on mobile.

### 6.7 Date

```json
{
  "fieldType": "Date",
  "smartFieldTag": "",
  "unitType": "Field",
  "isRequired": false,
  "name": "Date Field Label",
  "description": "",
  "id": "valid-uuid"
}
```

`smartFieldTag` is present in all Date field exports, even when empty. Include for schema consistency.

### 6.8 StaticElement (not a fieldType — a unitType)

Non-interactive HTML for headers, instructions, warnings, informational text.

```json
{
  "unitType": "StaticElement",
  "htmlContent": "<h3>Section Title</h3><p>Instructions for this section.</p>",
  "header": false,
  "isPrintViewOnly": false,
  "id": "valid-uuid"
}
```

| Property | Notes |
|---|---|
| `htmlContent` | Full HTML string. Supports `<h1>`–`<h6>`, `<p>`, `<strong>`, `<em>`, `<ul>`, `<li>`, `<br>`. Complex CSS is not supported. |
| `header` | When `true`, renders with header styling. Typically `false`. |
| `isPrintViewOnly` | When `true`, only appears in print/PDF, not in the interactive form. |

`StaticElement` units do **not** have `fieldType`, `isRequired`, `name`, `description`, or `values`.

### 6.9 Field-type confirmed/unconfirmed summary

| Field Type | Status | Source |
|---|---|---|
| `Text` | ✓ Confirmed | Multiple live exports |
| `Dropdown` | ✓ Confirmed | Multiple live exports |
| `Checkbox` | ✓ Confirmed | Multiple live exports |
| `Picture` | ✓ Confirmed | Live water-heater check-up form export |
| `Stoplight` | ✓ Confirmed | Live plumbing maintenance checklist export |
| `Number` | ✓ Confirmed | Live water-heater check-up form export |
| `Date` | ✓ Confirmed | Multiple live exports |
| `Photo` | ✗ Invalid | Causes 400 ValidationException — use `Picture` |
| `Image` | ? Unconfirmed | Not tested |
| `Signature` | ? Unconfirmed | Not tested |
| `RadioGroup` | ? Unconfirmed | Not tested |
| `MultiSelect` | ? Unconfirmed | Not tested |
| `Toggle` (as fieldType) | ✗ Invalid use | `Toggle` is an action type, not a field type |

---

## 7. UUID Requirements

Every unit in a Form 2.0 document — including the `rootSection`, every `Field`, every `Section`, and every `StaticElement` — requires a unique `id` containing a valid UUID.

### 7.1 Valid UUID format

```
[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}
```

All characters must be **lowercase hexadecimal** (`0–9` and `a–f`). Standard 8-4-4-4-12 hyphenated format.

### 7.2 Common invalid UUID patterns

| Invalid pattern | Error | Fix |
|---|---|---|
| `"g0000000-0000-4000-8000-000000000001"` | `"invalid or empty Id: '0'"` — `g` parses as `0` | Replace `g` with `a`–`f` |
| `"h0000000-..."` | Same | Replace `h` |
| `"p0000000-..."` | Same | Replace `p` |
| `"n0000000-..."` | Same | Replace `n` |
| `null` | `"Required property 'id' expects a non-null value"` | Provide any valid UUID string |
| `""` (empty) | Schema validation failure | Provide any valid UUID string |

### 7.3 Safe hand-crafted UUID strategy

When building forms programmatically without a UUID generator, use a structured prefix scheme that stays within valid hex characters:

```
[section-prefix][sequence]-0000-4000-8000-[form-prefix][sequence]
```

Examples:
```
a0000001-0000-4000-8000-000000000001   ← first field
a0000001-0000-4000-8000-000000000002   ← second field
b0000001-0000-4000-8000-000000000001   ← first field of second section
```

In Python, `str(uuid.uuid4())` generates valid UUIDs.

### 7.4 Uniqueness

All UUIDs within a single form document must be unique. Duplicates have not been confirmed to cause an import error, but they cause unpredictable behavior in the rules engine — rule actions reference fields by UUID.

---

## 8. The Rules Engine

The Form 2.0 rules engine evaluates a list of rule objects at runtime. Each rule has a set of conditions and a set of actions. When all conditions are satisfied (given `"whenMatch": "All"`), all actions in that rule execute.

Rules are stored in the `rules` array inside the inner `definition`:

```json
"definition": {
  "rootSection": { ... },
  "rules": [
    {
      "whenMatch": "All",
      "actions": [ ... ],
      "conditions": [ ... ]
    }
  ],
  "hasConditionalLogic": true
}
```

### 8.1 Rule object structure

```json
{
  "whenMatch": "All",
  "actions": [
    { "type": "Toggle", "unit": "field-uuid", "mode": "On", "isMet": false }
  ],
  "conditions": [
    { "type": "Compare", "operator": "Equals", "left": "checkbox-uuid", "right": "Yes" }
  ]
}
```

| Property | Type | Required | Notes |
|---|---|---|---|
| `whenMatch` | String | Yes | `"All"` = all conditions must be true (AND logic). `"Any"` (OR) has not been confirmed from live exports. |
| `actions` | Array | Yes | **Maximum 2 actions per rule.** See §9. |
| `conditions` | Array | Yes | One or more condition objects. Multiple with `"All"` = AND logic. |

> **The cardinal Form 2.0 bug:** `condition` (singular) object instead of `conditions` (plural) array, and missing `whenMatch`. The wrong shape causes silent failure with "Something went wrong" on the Forms page.
>
> ```json
> // ❌ WRONG — silent failure
> { "condition": { "type": "InitialLoad" }, "actions": [...] }
>
> // ✅ CORRECT — confirmed working
> { "whenMatch": "All", "conditions": [{ "type": "InitialLoad" }], "actions": [...] }
> ```

---

## 9. The Micro-Rule Constraint (Two Actions Max)

> **The single most important constraint in Form 2.0.** The rendering engine enforces **a maximum of two (2) actions per rule.**

This limit exists to preserve client-side rendering performance. The engine evaluates rules synchronously on every field change; rules with large action sets create blocking operations that cause the preview renderer to fail silently.

### 9.1 The trap

A form with a single `InitialLoad` rule containing 25 `Toggle Off` actions — the natural way to write "hide all fields on load" — fails completely on preview. **The form imports successfully** (the schema validator does not check this limit), but the rendering engine rejects it at preview time with a generic "Something went wrong" error.

### 9.2 Solution: micro-rule splitting

Every rule with more than 2 actions must be split into multiple rules with the same conditions:

```python
def make_rules(conditions, unit_ids, mode):
    """Split unit_ids into micro-rules of max 2 actions each."""
    rules = []
    for i in range(0, len(unit_ids), 2):
        chunk = unit_ids[i:i+2]
        rules.append({
            "whenMatch": "All",
            "actions": [{"type": "Toggle", "unit": uid, "mode": mode, "isMet": False}
                        for uid in chunk],
            "conditions": conditions
        })
    return rules
```

### 9.3 Before/after example

**Invalid (5 actions in one rule):**
```json
{
  "whenMatch": "All",
  "actions": [
    {"type": "Toggle", "unit": "id-1", "mode": "Off", "isMet": false},
    {"type": "Toggle", "unit": "id-2", "mode": "Off", "isMet": false},
    {"type": "Toggle", "unit": "id-3", "mode": "Off", "isMet": false},
    {"type": "Toggle", "unit": "id-4", "mode": "Off", "isMet": false},
    {"type": "Toggle", "unit": "id-5", "mode": "Off", "isMet": false}
  ],
  "conditions": [{"type": "InitialLoad"}]
}
```

**Valid (split into 3 micro-rules):**
```json
[
  {"whenMatch": "All",
   "actions": [
     {"type": "Toggle", "unit": "id-1", "mode": "Off", "isMet": false},
     {"type": "Toggle", "unit": "id-2", "mode": "Off", "isMet": false}],
   "conditions": [{"type": "InitialLoad"}]},
  {"whenMatch": "All",
   "actions": [
     {"type": "Toggle", "unit": "id-3", "mode": "Off", "isMet": false},
     {"type": "Toggle", "unit": "id-4", "mode": "Off", "isMet": false}],
   "conditions": [{"type": "InitialLoad"}]},
  {"whenMatch": "All",
   "actions": [
     {"type": "Toggle", "unit": "id-5", "mode": "Off", "isMet": false}],
   "conditions": [{"type": "InitialLoad"}]}
]
```

Behavior is identical — all five fields hidden on load — but the form renders correctly. **A 25-field hide-all-on-load rule becomes 13 micro-rules.** Yes, really.

---

## 10. Condition Types

### 10.1 InitialLoad

Fires exactly once when the form is first opened. Used to set the initial visibility state. Every field that should be hidden by default needs an `InitialLoad` rule that toggles it `Off`.

```json
{"type": "InitialLoad"}
```

No additional properties.

### 10.2 Compare

Evaluates a field's current value against a static comparison value. The primary condition type for show/hide logic triggered by user input.

```json
{
  "type": "Compare",
  "operator": "Equals",
  "left": "field-uuid-to-evaluate",
  "right": "value-to-compare-against"
}
```

| Property | Notes |
|---|---|
| `operator` | `"Equals"`, `"NotEquals"`, `"LessThan"`, `"GreaterThan"`, `"LessThanOrEquals"`, `"GreaterThanOrEquals"`, `"Contains"` |
| `left` | UUID of the field being evaluated |
| `right` | String value to compare against. Exact match, case-sensitive. |

`LessThan` / `GreaterThan` family is intended for `Number` and `Date` fields. For `Dropdown` and `Checkbox`, use `Equals` / `NotEquals`.

**Checkbox-specific:** when targeting a `Checkbox`, `right` must always be `"Yes"`. There is no confirmed way to trigger a rule on an unchecked checkbox using `Compare` — the `InitialLoad` pattern handles the default-hidden state.

### 10.3 IsFilled

Evaluates whether a field has any value, regardless of what it is.

```json
{
  "type": "IsFilled",
  "field": "field-uuid-to-evaluate"
}
```

> **Property name difference:** `Compare` uses `left` for the field UUID; `IsFilled` uses `field`. Confirmed from live exports. Using `left` in an `IsFilled` condition causes the condition to never evaluate correctly.

Useful for chaining: show a follow-up field after a primary field is filled, without caring about the specific value.

---

## 11. Action Types

### 11.1 Toggle

Shows or hides a field, section, or static element.

```json
{
  "type": "Toggle",
  "unit": "target-field-or-section-uuid",
  "mode": "On",
  "isMet": false
}
```

| Property | Notes |
|---|---|
| `type` | Always `"Toggle"` |
| `unit` | UUID of the field, section, or static element to show/hide |
| `mode` | `"On"` = show, `"Off"` = hide |
| `isMet` | Always `false` in confirmed live exports. Purpose not fully documented; setting to other values not tested. |

`Toggle` can target any unit type. **Targeting a `Section` hides/shows the entire section and all nested contents simultaneously** — far more efficient than individual field toggles.

### 11.2 ApplyTags

Applies one or more ServiceTitan tags to a record when conditions are met.

```json
{
  "type": "ApplyTags",
  "applyTagsTargets": ["Job"],
  "applyTags": [12345, 67890],
  "isMet": false
}
```

| Property | Notes |
|---|---|
| `applyTagsTargets` | Array of record types to tag. Typically `["Job"]`. Must match the form's `applyTagsTargets`. |
| `applyTags` | Array of integer tag IDs from the tenant. Tag IDs are tenant-specific. |
| `isMet` | Always `false` in confirmed exports |

Powerful for workflow automation: when a tech checks "Customer interested in replacement," the form auto-tags the job with a "Sales Lead" tag, which can trigger CSR follow-up workflows or reporting filters.

---

## 12. Cross-Section AND Logic

Multiple conditions in a single rule's `conditions` array with `"whenMatch": "All"` implement AND logic — all must be simultaneously true for actions to execute.

This enables cross-section dependencies: a field is visible only when two or more checkboxes from different sections are both checked.

```json
{
  "whenMatch": "All",
  "actions": [
    {"type": "Toggle", "unit": "line-set-field-uuid", "mode": "On", "isMet": false}
  ],
  "conditions": [
    {"type": "Compare", "operator": "Equals", "left": "equipment-checkbox-uuid", "right": "Yes"},
    {"type": "Compare", "operator": "Equals", "left": "refrigerant-checkbox-uuid", "right": "Yes"}
  ]
}
```

In this example, "Line Set Included" only appears when both the Equipment AND Refrigerant checkboxes are checked.

**Important:** cross-section AND rules must still respect the 2-action limit. **The number of conditions is not limited — only the number of actions.**

Real-world use cases:
- Line Set / TXV Kit → Equipment ✓ AND Refrigerant ✓
- Surge Protector / Low Voltage Wire → Equipment ✓ AND Electrical ✓
- Any field that only makes sense when two scopes of work are both present

---

## 13. Standard Patterns and Recipes

### Pattern 1: Hide all conditional fields on load, show on checkbox

The most common pattern. All conditional fields hidden when the form opens, revealed when the user checks a checkbox.

```python
# Step 1: Hide all conditional fields on load
rules = make_rules([{"type": "InitialLoad"}], all_conditional_field_ids, "Off")

# Step 2: Show fields when checkbox is checked
rules += make_rules(
    [{"type": "Compare", "operator": "Equals", "left": CHECKBOX_ID, "right": "Yes"}],
    all_conditional_field_ids, "On"
)
```

### Pattern 2: Section gate (the efficient pattern)

Hide an entire section on load, show it when a checkbox is checked. **More efficient than hiding individual fields** because one rule targets the section container.

```python
rules = make_rules([{"type": "InitialLoad"}], [SECTION_ID], "Off")
rules += make_rules(
    [{"type": "Compare", "operator": "Equals", "left": CHECKBOX_ID, "right": "Yes"}],
    [SECTION_ID], "On"
)
```

A 10-field group becomes 2 rules instead of 20.

### Pattern 3: Dropdown-driven visibility

Show different field groups based on a dropdown selection.

```python
# Hide all option groups on load
rules = make_rules([{"type": "InitialLoad"}],
                    GROUP_A_IDS + GROUP_B_IDS + GROUP_C_IDS, "Off")

# Show Group A when dropdown = "Option A"
rules += make_rules(
    [{"type": "Compare", "operator": "Equals", "left": DROPDOWN_ID, "right": "Option A"}],
    GROUP_A_IDS, "On"
)
# Show Group B when dropdown = "Option B"
rules += make_rules(
    [{"type": "Compare", "operator": "Equals", "left": DROPDOWN_ID, "right": "Option B"}],
    GROUP_B_IDS, "On"
)
```

### Pattern 4: Cross-section AND gate

Show a field only when two conditions from different sections are both true.

```python
rules += make_rules(
    [
        {"type": "Compare", "operator": "Equals", "left": CB_SECTION_A, "right": "Yes"},
        {"type": "Compare", "operator": "Equals", "left": CB_SECTION_B, "right": "Yes"},
    ],
    [CROSS_SECTION_FIELD_ID], "On"
)
```

### Pattern 5: Chained IsFilled

Show a follow-up field only after a primary field has been filled.

```python
rules += make_rules(
    [{"type": "IsFilled", "field": PRIMARY_FIELD_ID}],
    [FOLLOWUP_FIELD_ID], "On"
)
```

---

## 14. Import and Validation Behavior

### 14.1 Import process

Forms imported via **Settings → Operations → Forms → Import**. Single JSON file per import. **No batch import in the standard UI.**

### 14.2 The two-stage validation

ServiceTitan validates imports in two stages — failures at each stage produce different error patterns.

**Stage 1 — Schema validation (server-side, on import).** Validates JSON structure, required fields, data types, UUID formats. Errors produce HTTP 400 with `ValidationException` and a specific `title`. **The form is rejected and not saved.**

**Stage 2 — Rendering validation (client-side, on preview).** The rendering engine validates logic and structure. Errors produce the generic "Something went wrong" message. **The form has been saved to the database** but cannot be rendered. The micro-rule constraint (§9) is enforced at this stage, not Stage 1.

> **This two-stage validation is why a form can import successfully but fail on preview.** The server accepts the JSON, but the client rendering engine rejects it.

### 14.3 ID overwriting

When imported, ServiceTitan overwrites `FormData.id` and `definition.id` with internally generated IDs. The values you provide are ignored. **However, both fields must be present and non-null integers** — the schema validator enforces presence even though the values are discarded.

---

## 15. Error Catalog

| HTTP Status | Error Message | Path | Stage | Root Cause | Fix |
|---|---|---|---|---|---|
| 400 | `Required property 'id' expects a non-null value` | `'definition'` | Stage 1 | `definition.id` is `null` | Set to any integer |
| 400 | `Required property 'id' expects a non-null value` | `''` (root) | Stage 1 | `FormData.id` is `null` | Set to any integer |
| 400 | `Unit '...' has an invalid or empty Id: '0'` | — | Stage 1 | UUID contains non-hex character (g, h, p, n) | Replace with valid hex UUID |
| 400 | `Error converting value 'Photo' to type 'FormFieldType'` | `'fieldType'` | Stage 1 | `fieldType: "Photo"` is invalid | Change to `"Picture"` |
| 400 | `Error converting value 'X' to type 'FormFieldType'` | `'fieldType'` | Stage 1 | Unknown `fieldType` value | Use only confirmed field types (§6) |
| UI | `Something went wrong` (Forms page) | — | Stage 2 | `condition` (singular object) used instead of `conditions` (array) + missing `whenMatch` | Use `conditions` array, add `whenMatch: "All"` |
| UI | `Something went wrong` (preview) | — | Stage 2 | Rule has >2 actions | Split into micro-rules (§9) |
| UI | `Something went wrong` (preview) | — | Stage 2 | Invalid UUID in unit | Validate all UUIDs |
| UI | `Something went wrong` (preview) | — | Stage 2 | `IsForm20` not `true` | Add `"IsForm20": true` at root |
| Silent | Logic does not trigger | Runtime | — | `hasConditionalLogic` is `false` or absent | Set to `true` |
| Silent | Logic does not trigger | Runtime | — | Checkbox `right` not exactly `"Yes"` | Change to `"Yes"` |
| Silent | Field always visible | Runtime | — | Missing `InitialLoad` hide rule | Add hide rule paired with show rule |

---

## 16. Form Owner Types

The `ownerTypes` array in `FormData` controls which ServiceTitan record types the form attaches to. Multiple types can be specified.

| Value | Description |
|---|---|
| `"Job"` | Form appears on job records |
| `"Call"` | Form appears on call/booking records |
| `"Equipment"` | Form appears on equipment records |
| `"Customer"` | Form appears on customer records |
| `"Location"` | Form appears on location/property records |

`applyTagsTargets` uses the same values and controls which record types receive tags from `ApplyTags` rule actions. Typically `applyTagsTargets` matches `ownerTypes`.

---

## 17. Print Options

The `printOptions` array controls how the form renders when printed or exported to PDF. In all confirmed live exports, this array is empty (`[]`). Custom print layouts are configured through the UI after import, not through JSON.

**Placement** — must be at the outer `definition` level (alongside `id`, `version`, `published`), not inside the inner `definition`. Misplacement does not error on import but causes print options to be ignored.

---

## 18. SmartFieldTag System

The `smartFieldTag` property on `Text` and `Date` fields enables auto-population from ServiceTitan record data. When a form is opened in the context of a specific job, customer, or location, fields with valid smart field tags are automatically populated.

Confirmed `smartFieldTag` values from live exports include identifiers for **job number**, **customer name**, **service address**, **technician name**. The complete list is not publicly documented and varies by tenant configuration.

The property should be set to an empty string `""` when not used — it is present in all confirmed `Date` field exports.

---

## 19. Form Triggers

The `FormTriggers` array at the root level supports automatic form dispatch based on ServiceTitan events. When populated, trigger objects specify the event type and conditions for automatic dispatch.

In all forms developed and tested empirically, `FormTriggers` is `[]`. **Trigger configuration is typically managed through the ServiceTitan UI after import.** Including triggers in import JSON has not been tested.

For Forms-as-configuration (when triggers attach forms automatically by job type, etc.), see `04-job-types-forms-tags.md` §3.2.

---

## 20. Python Builder Framework

A complete reusable framework for building Form 2.0 JSON programmatically. All helper functions produce output validated against the confirmed schema.

```python
#!/usr/bin/env python3
"""
ServiceTitan Form 2.0 Builder Framework
All field types and rule structures confirmed from live tenant exports.
"""
import json, re, sys
from datetime import datetime, timezone

UUID_PAT = re.compile(r'^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$')
VALID_FIELD_TYPES = {'Text', 'Dropdown', 'Checkbox', 'Picture', 'Stoplight', 'Number', 'Date'}

# ── Field constructors ───────────────────────────────────────────────────

def text(fid, name, description="", required=False, smart_tag=""):
    u = {"fieldType": "Text", "unitType": "Field", "isRequired": required,
         "name": name, "description": description, "id": fid}
    if smart_tag:
        u["smartFieldTag"] = smart_tag
    return u

def dropdown(fid, name, values, description="", required=False):
    return {"fieldType": "Dropdown", "values": values, "unitType": "Field",
            "isRequired": required, "name": name, "description": description, "id": fid}

def checkbox(fid, name, description=""):
    return {"fieldType": "Checkbox", "values": ["Yes"], "horizontalLayout": True,
            "unitType": "Field", "isRequired": False, "name": name,
            "description": description, "id": fid}

def picture(fid, name, required=False):
    """Photo capture. fieldType MUST be 'Picture', NOT 'Photo'."""
    return {"fieldType": "Picture", "unitType": "Field",
            "isRequired": required, "name": name, "id": fid}

def stoplight(fid, name, description="", required=False):
    return {"fieldType": "Stoplight", "unitType": "Field",
            "isRequired": required, "name": name, "description": description, "id": fid}

def number(fid, name, description="", required=False):
    return {"fieldType": "Number", "unitType": "Field",
            "isRequired": required, "name": name, "description": description, "id": fid}

def date_field(fid, name, description="", required=False, smart_tag=""):
    return {"fieldType": "Date", "smartFieldTag": smart_tag, "unitType": "Field",
            "isRequired": required, "name": name, "description": description, "id": fid}

def static(fid, html):
    return {"unitType": "StaticElement", "htmlContent": html,
            "header": False, "isPrintViewOnly": False, "id": fid}

def section(fid, name, units, duplicatable=False):
    return {"unitType": "Section", "canBeDuplicated": duplicatable,
            "units": units, "name": name, "id": fid}

# ── Rule constructors ────────────────────────────────────────────────────

def toggle_action(uid, mode):
    return {"type": "Toggle", "unit": uid, "mode": mode, "isMet": False}

def make_rules(conditions, unit_ids, mode):
    """Auto-split into micro-rules of max 2 actions (required by ST engine)."""
    rules = []
    for i in range(0, len(unit_ids), 2):
        chunk = unit_ids[i:i+2]
        rules.append({
            "whenMatch": "All",
            "actions": [toggle_action(uid, mode) for uid in chunk],
            "conditions": conditions
        })
    return rules

def hide_on_load(ids):
    return make_rules([{"type": "InitialLoad"}], ids, "Off")

def show_when_equals(field_id, value, ids):
    return make_rules(
        [{"type": "Compare", "operator": "Equals", "left": field_id, "right": value}],
        ids, "On"
    )

def show_when_checked(checkbox_id, ids):
    return show_when_equals(checkbox_id, "Yes", ids)

def show_when_both_checked(cb1, cb2, ids):
    return make_rules(
        [
            {"type": "Compare", "operator": "Equals", "left": cb1, "right": "Yes"},
            {"type": "Compare", "operator": "Equals", "left": cb2, "right": "Yes"},
        ],
        ids, "On"
    )

def show_when_filled(field_id, ids):
    return make_rules([{"type": "IsFilled", "field": field_id}], ids, "On")

def apply_tags(tag_ids, targets, conditions):
    return [{
        "whenMatch": "All",
        "actions": [{"type": "ApplyTags", "applyTagsTargets": targets,
                     "applyTags": tag_ids, "isMet": False}],
        "conditions": conditions
    }]

# ── Form assembler ───────────────────────────────────────────────────────

def build_form(name, units, rules, owner_types=None):
    now = datetime.now(timezone.utc).strftime('%Y-%m-%dT%H:%M:%S.') + '000Z'
    return {
        "FormData": {
            "id": 9999,
            "name": name,
            "active": True, "locked": False, "hide": False,
            "ownerTypes": owner_types or ["Job"],
            "applyTagsTargets": ["Job"],
            "showOnMobile": True,
            "allowEmptyToPresent": False,
            "canDelete": False,
            "sendOnEvents": [],
            "dwyerBigBoardReport": False,
            "definition": {
                "id": 1, "version": 1, "published": True,
                "definition": {
                    "rootSection": {
                        "unitType": "Section",
                        "canBeDuplicated": False,
                        "units": units,
                        "name": "Root",
                        "id": "00000000-0000-4000-8000-000000000000"
                    },
                    "rules": rules,
                    "hasConditionalLogic": len(rules) > 0
                },
                "printOptions": []
            },
            "createdOn": now, "modifiedOn": now,
            "BusinessUnits": None, "ApplyTagTypes": None
        },
        "FormTriggers": [], "IsForm20": True, "IsLocked": False
    }
```

---

## 21. Pre-Import Validation Script

Standalone script that validates a Form 2.0 JSON file before import. Catches all known failure conditions in a single pass.

```python
#!/usr/bin/env python3
"""
ServiceTitan Form 2.0 — Pre-Import Validator
Run: python3 validate_form.py path/to/form.json
"""
import json, re, sys

UUID_PAT = re.compile(r'^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$')
VALID_FIELD_TYPES = {'Text', 'Dropdown', 'Checkbox', 'Picture', 'Stoplight', 'Number', 'Date'}

def get_all_units(section):
    units = []
    for u in section.get('units', []):
        units.append(u)
        if u.get('unitType') == 'Section':
            units.extend(get_all_units(u))
    return units

def validate(path):
    with open(path) as f:
        form = json.load(f)

    errors, warnings = [], []
    fd = form.get('FormData', {})
    defn_outer = fd.get('definition', {})
    defn_inner = defn_outer.get('definition', {})
    all_units = get_all_units(defn_inner.get('rootSection', {}))
    rules = defn_inner.get('rules', [])

    # Top-level flags
    if not form.get('IsForm20'):
        errors.append("IsForm20 is not true — Form 2.0 engine will not activate")

    # Required non-null IDs
    if fd.get('id') is None:
        errors.append("FormData.id is null — causes 400 error on import")
    if defn_outer.get('id') is None:
        errors.append("definition.id is null — causes 400 error at path 'definition'")

    # hasConditionalLogic
    if rules and not defn_inner.get('hasConditionalLogic'):
        warnings.append("Rules exist but hasConditionalLogic is not true — logic may not execute")

    # UUID and fieldType validation
    seen_ids = set()
    for u in all_units:
        uid = u.get('id', '')
        name = u.get('name', u.get('htmlContent', '[unnamed]'))[:50]

        if not UUID_PAT.match(uid):
            errors.append(f"Invalid UUID on '{name}': {uid!r}")
        elif uid in seen_ids:
            warnings.append(f"Duplicate UUID on '{name}': {uid!r}")
        seen_ids.add(uid)

        ft = u.get('fieldType')
        ut = u.get('unitType')
        if ft and ft not in VALID_FIELD_TYPES and ut not in ('StaticElement', 'Section'):
            errors.append(f"Unknown fieldType '{ft}' on '{name}' — will cause 400 on import")

    # Micro-rule constraint
    for i, r in enumerate(rules):
        action_count = len(r.get('actions', []))
        if action_count > 2:
            errors.append(
                f"Rule {i} has {action_count} actions (max 2) — "
                f"will cause 'Something went wrong' on preview"
            )

    # Rule reference integrity
    unit_ids = {u.get('id') for u in all_units}
    for i, r in enumerate(rules):
        for a in r.get('actions', []):
            if a.get('type') == 'Toggle' and a.get('unit') not in unit_ids:
                errors.append(f"Rule {i} action references unknown unit UUID: {a.get('unit')!r}")
        for c in r.get('conditions', []):
            if c.get('type') == 'Compare' and c.get('left') not in unit_ids:
                errors.append(f"Rule {i} condition references unknown field UUID: {c.get('left')!r}")

    # Report
    print(f"\nValidation Report: {path}")
    print(f"  Units : {len(all_units)}  |  Rules : {len(rules)}")
    if errors:
        print(f"\n  ERRORS ({len(errors)}):")
        for e in errors:
            print(f"    ✗ {e}")
    if warnings:
        print(f"\n  WARNINGS ({len(warnings)}):")
        for w in warnings:
            print(f"    ⚠ {w}")
    if not errors and not warnings:
        print("  ✓ All checks passed — safe to import")
    elif not errors:
        print("  ✓ No blocking errors — warnings are advisory only")
    return len(errors) == 0

if __name__ == "__main__":
    path = sys.argv[1] if len(sys.argv) > 1 else input("Form JSON path: ").strip()
    sys.exit(0 if validate(path) else 1)
```

---

## 22. Design Patterns and Best Practices

### 22.1 Use Section containers for large conditional groups

When a checkbox controls visibility of 5+ related fields, wrap them in a `Section` container and target the section with a single `Toggle` action. **Two rules total instead of ten.**

```
Checkbox: "Equipment Included?"
  └── Section: "Equipment Details" (hidden by default)
        ├── Dropdown: System Type
        ├── Dropdown: Tonnage
        ├── Text: Make
        ├── Text: Model
        └── Picture: Photo
```

One `InitialLoad` rule hides the section. One `Compare` rule shows it. Two rules.

### 22.2 Always pair InitialLoad hide with Compare show

Every field participating in conditional logic needs **exactly two rules**: one to hide on load, one to show when the condition is met. Omitting the `InitialLoad` hide means the field is visible by default — the show rule becomes redundant.

### 22.3 Use Stoplight for assessment fields

Any field that asks a tech to assess a condition — equipment state, safety concern, urgency, pass/fail — should be `Stoplight` rather than `Dropdown` or `Text`. Visual color coding communicates urgency at a glance and makes data more actionable in reporting.

### 22.4 Always include descriptions on Stoplight fields

The `description` renders as helper text below the label. For Stoplight, include `"Green = Good condition, Yellow = Monitor, Red = Immediate replacement needed"`.

### 22.5 UUID naming convention for hand-crafted forms

Encode section and sequence in the UUID for readable debugging:

```
[section-code][sequence]-0000-4000-8000-[form-code][sequence]
```

Example for an HVAC form:
```
a1000001-0000-4000-8000-000000000001  ← HVAC section, field 1
a1000001-0000-4000-8000-000000000002  ← HVAC section, field 2
a2000001-0000-4000-8000-000000000001  ← Plumbing section, field 1
```

### 22.6 Validate before every import

Run the validator (§21) before every import. The two-stage validation means some errors are only caught at preview, after the form has been saved. Pre-import validation catches all known errors before they reach the platform.

---

## 23. Annotated Example — Checkbox-Driven Conditional Section

A form where a "Replacing Existing Equipment?" checkbox reveals a group of fields about the existing unit.

```json
{
  "FormData": {
    "id": 9999,
    "name": "Equipment Replacement Form",
    "active": true, "locked": false, "hide": false,
    "ownerTypes": ["Job"],
    "applyTagsTargets": ["Job"],
    "showOnMobile": true,
    "allowEmptyToPresent": false,
    "canDelete": false,
    "sendOnEvents": [],
    "dwyerBigBoardReport": false,
    "definition": {
      "id": 1, "version": 1, "published": true,
      "definition": {
        "rootSection": {
          "unitType": "Section",
          "canBeDuplicated": false,
          "units": [
            {
              "fieldType": "Checkbox",
              "values": ["Yes"],
              "horizontalLayout": true,
              "unitType": "Field",
              "isRequired": false,
              "name": "Replacing Existing Equipment?",
              "description": "Check to reveal existing equipment details",
              "id": "bb000001-0000-4000-8000-000000000001"
            },
            {
              "unitType": "Section",
              "canBeDuplicated": false,
              "units": [
                {"fieldType": "Text", "unitType": "Field", "isRequired": false,
                 "name": "Existing Make / Brand", "description": "",
                 "id": "bb000001-0000-4000-8000-000000000002"},
                {"fieldType": "Text", "unitType": "Field", "isRequired": false,
                 "name": "Existing Model #", "description": "",
                 "id": "bb000001-0000-4000-8000-000000000003"},
                {"fieldType": "Stoplight", "unitType": "Field", "isRequired": false,
                 "name": "Existing Equipment Condition",
                 "description": "Green = Good, Yellow = Failing, Red = Failed",
                 "id": "bb000001-0000-4000-8000-000000000004"},
                {"fieldType": "Picture", "unitType": "Field", "isRequired": false,
                 "name": "Photo — Existing Equipment",
                 "id": "bb000001-0000-4000-8000-000000000005"}
              ],
              "name": "Existing Equipment Details",
              "id": "bb000001-0000-4000-8000-000000000010"
            }
          ],
          "name": "Root",
          "id": "00000000-0000-4000-8000-000000000000"
        },
        "rules": [
          {
            "whenMatch": "All",
            "actions": [
              {"type": "Toggle", "unit": "bb000001-0000-4000-8000-000000000010",
               "mode": "Off", "isMet": false}
            ],
            "conditions": [{"type": "InitialLoad"}]
          },
          {
            "whenMatch": "All",
            "actions": [
              {"type": "Toggle", "unit": "bb000001-0000-4000-8000-000000000010",
               "mode": "On", "isMet": false}
            ],
            "conditions": [
              {"type": "Compare", "operator": "Equals",
               "left": "bb000001-0000-4000-8000-000000000001", "right": "Yes"}
            ]
          }
        ],
        "hasConditionalLogic": true
      },
      "printOptions": []
    },
    "createdOn": "2026-01-01T00:00:00.000Z",
    "modifiedOn": "2026-01-01T00:00:00.000Z",
    "BusinessUnits": null,
    "ApplyTagTypes": null
  },
  "FormTriggers": [],
  "IsForm20": true,
  "IsLocked": false
}
```

Notice: only **2 rules total** for a 4-field conditional section, because the inner Section is the toggle target (Pattern 2, §13).

---

## 24. Schema Reference Summary

### FormData schema tree

```
FormData (object, required)
├── id (integer, required, non-null)
├── name (string, required)
├── active (boolean, required)
├── locked (boolean, required)
├── hide (boolean, required)
├── ownerTypes (array of strings, required)
├── applyTagsTargets (array of strings, required)
├── showOnMobile (boolean, required)
├── allowEmptyToPresent (boolean, required)
├── canDelete (boolean, required)
├── sendOnEvents (array, required)
├── dwyerBigBoardReport (boolean, required)
├── definition (object, required)
│   ├── id (integer, required, non-null)
│   ├── version (integer, required)
│   ├── published (boolean, required)
│   ├── definition (object, required)
│   │   ├── rootSection (Section object, required)
│   │   ├── rules (array of Rule objects, required)
│   │   └── hasConditionalLogic (boolean, required)
│   └── printOptions (array, required)
├── createdOn (ISO 8601 string, required)
├── modifiedOn (ISO 8601 string, required)
├── BusinessUnits (null or array)
└── ApplyTagTypes (null or array)
```

### Unit schemas by type

```
Field unit (unitType = "Field")
├── fieldType (string, required) — one of confirmed types (§6)
├── unitType (string, required) = "Field"
├── isRequired (boolean, required)
├── name (string, required)
├── description (string, optional for Picture)
├── id (string UUID, required)
├── values (array of strings) — Dropdown and Checkbox only
├── horizontalLayout (boolean) — Checkbox only
└── smartFieldTag (string) — Text and Date only

Section unit (unitType = "Section")
├── unitType (string, required) = "Section"
├── canBeDuplicated (boolean, required)
├── units (array of units, required)
├── name (string, required)
└── id (string UUID, required)

StaticElement unit (unitType = "StaticElement")
├── unitType (string, required) = "StaticElement"
├── htmlContent (string, required)
├── header (boolean, required)
├── isPrintViewOnly (boolean, required)
└── id (string UUID, required)
```

### Rule schema

```
Rule object
├── whenMatch (string, required) = "All"
├── actions (array, required, max 2 items)
│   ├── Toggle action
│   │   ├── type = "Toggle"
│   │   ├── unit (string UUID, required)
│   │   ├── mode (string) = "On" or "Off"
│   │   └── isMet (boolean) = false
│   └── ApplyTags action
│       ├── type = "ApplyTags"
│       ├── applyTagsTargets (array of strings)
│       ├── applyTags (array of integers)
│       └── isMet (boolean) = false
└── conditions (array, required)
    ├── InitialLoad condition
    │   └── type = "InitialLoad"
    ├── Compare condition
    │   ├── type = "Compare"
    │   ├── operator (string)
    │   ├── left (string UUID)
    │   └── right (string)
    └── IsFilled condition
        ├── type = "IsFilled"
        └── field (string UUID)
```

---

*Maintained by Luke Peluso, Business Systems Manager, Quality Service Company. Contact: open an issue or pull request in the repository. Not an official ServiceTitan document — community-sourced findings validated on live tenants. The Form 2.0 JSON schema is not publicly documented at this level of detail by ServiceTitan; corrections and additions welcome.*
