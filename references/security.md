# Server-enforced security

Open `describe_platform section=access_scope` and `describe_capabilities section=access_scope` when designing row security.

## Layers

| Layer | Mechanism | Enforced by server? |
|---|---|---|
| App/feature access | roles + permission keys (`list_permissions`) | yes |
| Form actions | `forms:<id>:view/create/edit/delete/share` on roles | yes |
| Rows | top-level `access_scope` | yes, for users — **not** for workflow DB nodes |
| Fields | `hide_rules`, `read_only_rules` (no Super Admin bypass) | yes |
| Reference choices | `reference_filter` | yes (picker + write) |
| Record sharing | runtime per-record share (needs `forms:<id>:share`) | yes; no MCP share tools |
| Navigation | menu gates | yes for menus — but menus are not data security |
| UI conditions, saved-view filters, menu default filters | — | **no** |

## access_scope behaviour (must shape designs)

1. Top-level `access_scope` is enforced; `layout.settings.accessScope` is only a mirror.
2. Rules whose audience matches a viewer are **ORed**.
3. A viewer matching **no** audience is **unrestricted**. If every viewer must be narrowed, include an `everyone` rule (and role rules for wider access).
4. Wildcard/Super Admin bypasses it.
5. Workflow database activities bypass it — privileged workflows must scope explicitly (filter by `TriggerRecord["created_by"]`, the department on the initiator's profile, etc.). Note: some nodes (e.g. `crm_convert_lead`) document enforcing the initiator's scope — check each node's description.
6. `current_user` works in access_scope/menus/dashboards, not in workflow filters (no viewer).
7. "Created by me" uses `created_by`; "assigned to me" needs a reference to an enrollment profile form compared with `<profile-form-id>.record_id`.

## Least privilege

- Design a role matrix: role × form × {view, create, edit, delete, share} + special permissions (`reports:read`, `workflows:*`…).
- Change roles with `update_role add_permissions/remove_permissions`; a removed key may still be granted by a wildcard (`forms:*:view`, `resource:*`).
- Membership tools are gated (env flag + `members:write`); absence means not enabled, not missing.
- Workflow activation needs explicit `workflows:activate`; `workflows:*` is insufficient.

## Accountability

`created_by`/`updated_by` are null for API-key writes (all MCP writes) and system-triggered workflows. If accountability matters, add explicit `actor_user` (reference to profile or user id string), `actor_type` (user/service/integration), `correlation_id`, and an audit-event form written in the same `write_batch` as the change.

## Other surfaces to check

Sensitive exports (reports, `report_render`, dashboards showing masked fields), cross-app shared forms, API key scopes (`list_api_keys`), webhook authentication, secrets (only `secret` fields; never echo), AI agents (`allowed roles`, tools, knowledge bases they can read), tenant isolation (never mix app ids).

## Verification

- `simulate_menu_visibility` per role for navigation.
- `simulate_viewer` for identity/enrollment resolution (`current_user` profile values). It does not show row visibility.
- Row/field enforcement: test with real test users of each role where available, or state that enforcement was verified by schema readback only (`Not verified` at runtime).
- Readback `get_form` to confirm `access_scope` and normalized field rules.

## Review checklist

- [ ] Every restricted form has an audience covering every role (or an `everyone` rule).
- [ ] No reliance on menu hiding or saved views for confidentiality.
- [ ] Privileged workflows filter explicitly and are not callable by roles that shouldn't trigger them.
- [ ] Field masks on salary/PII-style fields with correct `when` polarity.
- [ ] Secrets only in `secret` fields.
- [ ] Destructive permissions (delete, share) granted narrowly.
- [ ] Audit actor captured where `created_by` would be null.
