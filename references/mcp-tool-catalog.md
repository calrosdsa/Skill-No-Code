# MCP tool selection

Intent → tool. Exact arguments come from the live tool schema; this table only routes. If a tool named here is absent from the connected server, it is outside this profile, gated, or removed — do not invent a substitute.

## Discovery
| Intent | Tool |
|---|---|
| Platform prose / section | `describe_platform [section]` |
| Exact schema subtree | `describe_capabilities [section]` |
| One workflow node / UI step | `describe_node_type` |
| Reload cached knowledge | `refresh_knowledge` |
| App settings, activation grant | `get_application` |
| Other apps on this credential | `list_apps`, `select_app` |

## Forms & records
| Intent | Tool | Note |
|---|---|---|
| List / read form | `list_forms`, `get_form` | `get_form` gives etag |
| New form | `create_form` | default layout synthesized when no `layout` |
| Targeted change | `edit_form` ops: `add_field`, `set_field`, `remove_field` (confirm), `set_setting`, `set_detail_tabs` | preferred |
| Rename field, change type, restructure layout | `update_form` | full field list — an omitted field is **dropped with its data** |
| Delete form | `delete_form` | after `find_references` + explicit approval |
| Dependencies | `find_references id=<form>` or `<form-id>.<field>` | |
| Validate expression | `validate_expression` | record rules, formulas, conditions |
| Query | `search_records`, `aggregate_records`, `parse_filter`, `get_record` | |
| History | `get_record_history` | |
| Write | `create_record`, `update_record` (patch), `delete_record` (confirm) | test data only unless authorized |
| Document lifecycle | `transition_record` submit/cancel/amend | before-submit/cancel workflows run in its transaction |
| Picker / identity checks | `preview_reference_options`, `simulate_viewer` | |
| Translations | `list_form_translation_keys`, `get_translations`, `edit_translations`, `update_translations` | |

## Workflows
| Intent | Tool |
|---|---|
| List / read (compiled graph + flow + etag) | `list_workflows`, `get_workflow`, `get_workflow_history` |
| New | `create_workflow` (FLOW form) |
| Targeted | `edit_workflow` ops: `insert_step`, `update_step`, `remove_step`, `set_trigger`, `set_variables` |
| Full replacement / free-form graph | `update_workflow` |
| Validate | `validate_workflow` (read warnings) |
| Run (really executes) | `run_workflow` (`variables`, `trigger_record`) |
| Observe | `list_executions`, `get_execution` |
| Templates | `list_workflow_templates`, `get_workflow_template`, `instantiate_workflow_template`, `save_workflow_as_template` |
| Delete | `delete_workflow` after `find_references` |

## Navigation & UX
| Intent | Tool |
|---|---|
| Menus | `list_menus`, `get_menu`, `create_menu`, `update_menu` (full), `delete_menu`, `reorder_menus` (complete sibling list) |
| Dashboard tile change | `edit_dashboard` (never `update_menu` for one tile) |
| Visibility check | `simulate_menu_visibility` (role_id, user, super_admin, anonymous) |
| Mobile nav | `get_mobile_nav`, `update_mobile_nav` |
| Saved views | `list_saved_views`, `create_saved_view`, `update_saved_view`, `delete_saved_view` |

## Reports
`report_list`, `report_get` (→ `updated_at`), `report_create`, `report_update` (`expected_updated_at`), `report_validate`, `report_preview` (page images), `report_view` (live data), `report_render` (file), `report_publish`, `report_delete`, `report_craft_catalog`, `report_design_session`, `report_design_capture`, `report_image_*`.

## Security & admin
`list_roles`, `get_role`, `create_role`, `update_role` (prefer `add_permissions`/`remove_permissions`), `delete_role`, `list_permissions`; `assign_app_role`/`revoke_app_access` only when enabled (see discovery); `list_api_keys`, `set_api_key`, `revoke_api_key`; agents `list/get/create/update/delete_agent`; knowledge bases `list/create/update/delete_knowledge_base`, `add/list/delete_knowledge_document`.

## Lifecycle
`list_versions`, `save_version` (checkpoint + validation `issues`, not live), `publish_application` (live; confirm = exact app name; publishes **everything** saved, including others' changes).

## Evaluation (AI behaviours)
`create_evaluation_dataset`, `list_evaluation_datasets`, `add_evaluation_dataset_row`, `update_evaluation_dataset`, `run_evaluation`, `get_evaluation_run`, `list_evaluation_run_results`, `set_evaluation_verdict`, `score_evaluation_run`, `compare_evaluation_runs`. These evaluate platform agents/AI workflows — not this skill.
