# Form design

Open `describe_platform section=forms` and `describe_capabilities section=forms.field_types` (and `forms.definition_schema` when needed) before authoring. Field types, components and reserved names come from there, not from memory.

## Data model checklist (per form)

| Decide | Guidance |
|---|---|
| Identity | platform `id` is internal; add a **unique business key** (`unique: true`) for anything imported, upserted or referenced externally; `autonumber` for human document numbers |
| External ids | separate field per source system, unique (or composite deterministic key string) |
| Required | only true invariants; `required` is not valid on formula/autonumber |
| Indexed | `index: true` for fields used in filters/sorts/date triggers; unique/reference/enum are indexed automatically; json/file are not projected |
| Search | at least one text-like field `searchable` (API rejects otherwise when such fields exist) |
| Title | at least one `is_record_title` field |
| Relationships | `reference` (+ `display_field`, `reference_filter` enforced server-side twice); line items for owned children (`parent_link`, `line_item_count` aggregates) |
| Status | enum with explicit lifecycle; or `submittable` documents (draft → submitted → cancelled/amended) |
| Ownership | which context writes it; document on the form `description` |
| Audit | `created_by/updated_by` are null for API-key/system writes → add explicit actor/correlation fields if accountability matters |
| Formulas | stored, read-only, server-computed; validate with `validate_expression`; know their dependencies before renaming fields |
| Secrets | `secret` type only (encrypted, write-only, redacted) — never a string field or workflow config |
| Multi-value | `multi_enum` over json |
| JSON | only for genuinely schemaless payloads (raw external snapshots); never for fields you filter on |
| Retention | volume estimate, archival strategy for high-growth forms |

Prefer normalized typed fields and references. Snapshot copies only for as-of values, named explicitly (`unit_price_at_order`).

## Field rules (server-enforced)

| Rule | Semantics |
|---|---|
| `hide_rules` | hides value from matched viewers on every read (records, reports, workflow record steps) and rejects an update to it while masked. Additive (ANY firing hides). No Super Admin bypass. `when` evaluates the record's own fields |
| `show_rules` | newly authored ones are **normalized into negated `hide_rules`**; must share one audience; cannot be combined with typed hide_rules; only safely negatable static ops (eq/neq/is_null/not_null). Readback shows `hide_rules` — edit those afterwards |
| `read_only_rules` | blocks change on UPDATE when `when` holds against **stored** values; additive; no Super Admin bypass |
| `validation_rules` | record-level invariants, max 32 per form, each `{id (stable UUID), expr, message, when: save|submit}`; validate every `expr` |

Polarity matters: `when` describes when the field is *hidden/locked*. Read back after saving and check the projected rules.

## Layout ↔ typed rules synchronization

- A form created without `layout` gets a synthesized default builder layout (one "Details" section, one element per field).
- With no `layout` argument, typed rule changes are mirrored into the carried layout, renamed fields are retargeted, new fields get elements.
- With a `layout` whose element has an `advancedSettings` key (even empty), that layout becomes the **single source** for that field's hide/show/read-only rules — typed rules sent alongside are replaced.
- `access_scope` top-level is enforced; `layout.settings.accessScope` is only its mirror. A layout-only scope is refused, not promoted.
- `validation_rules` are mirrored to `layout.settings.recordRules`.

So: don't hand-edit layout rule blocks unless you mean to; after complex changes `get_form` and check **both** typed fields and the layout.

## Choosing the edit tool

| Change | Tool |
|---|---|
| add a field | `edit_form add_field` |
| change label, required, rules, enum values, index, formula… | `edit_form set_field` (per-key wholesale replace; rule arrays replace, `[]` clears) |
| form setting incl. `access_scope`, `validation_rules`, `submittable` | `edit_form set_setting` |
| detail tabs | `edit_form set_detail_tabs` |
| remove field (destroys data) | `find_references <form>.<field>` → approval → `edit_form remove_field confirm=<name>`; fix dependants yourself — they are not rewritten |
| rename, change type, restructure layout | `save_form` (`form_id` + `etag`) with the **complete** field list, keeping `column` ids |

## Schema migration (existing data)

1. `find_references` for the field/form; list workflows, menus, saved views, reports, expressions, other-app consumers.
2. Additive first: add the new field, backfill (workflow with `write_batch`/`update_records` in bounded batches, or `update_record` for small sets — authorized), switch consumers, then remove the old field in a later change.
3. Type change on populated fields: prefer add-new + migrate over in-place retype.
4. Checkpoint with `save_version` before, verify counts with `aggregate_records` after.
