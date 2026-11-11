# Menus, dashboards and UX

Open `describe_platform section=menus` (includes the visibility rule) and `describe_capabilities section=menus.menu_types` for exact configs.

## Choose built-ins first

| Need | Use |
|---|---|
| list/search records | `search` menu (+ saved views) |
| create a record | `add` menu |
| grouping / sections | `parent`, `module` |
| KPIs, tiles, canvas | `dashboard` (edit tiles with `edit_dashboard`) |
| published report | `report` menu |
| truly custom page | `html` (sandboxed) — only when no built-in fits; justify it |
| record-level commands | detail-page custom actions → `on_demand_data_driven` workflow |

`custom` is deprecated. Avoid HTML pages that re-implement search, forms or dashboards.

Navigation follows responsibilities: one module per role family or bounded context; a form without a menu is invisible at runtime (intended for internal/ledger forms).

## Visibility (server-enforced on the runtime snapshot)

A menu is visible only if its own gates **and every ancestor's** pass: `required_permission`; `permission_mode: role` + `required_role_ids`; for search/add, `forms:<id>:view` / `:create` on its form. Hidden parent → hidden subtree. `hidden_from_nav` is layout only. Super Admin passes gates but not a missing `form_id`.

**Hiding a menu never protects data.** Restrict data with role form permissions and `access_scope`.

## Rules

- Gate a section once, on its parent.
- `reorder_menus` needs the complete child list of one parent.
- `save_menu` with `menu_id` + `etag` replaces the whole menu; for a dashboard tile use `edit_dashboard set_widget`.
- Dashboard "my X" tiles use `value_mode: current_user` with qualified profile values; set `ignoreMissingProfile` when viewers may lack a profile.
- Saved views shared by others can be changed only by their owner or `menus:write` holders.

## Verify before publish

1. `simulate_menu_visibility` for each role (and anonymous, super_admin) → check `visible`, `shown_in_nav`, `failed_gate`.
2. `save_version` → read `issues` (stale role refs, missing report, broken slugs, deleted data sources).
3. Every role reaches every form it needs; no role reaches what it shouldn't via direct link.
