# MCP discovery

Goal: build an accurate, minimal picture of (a) what the platform can do right now and (b) what this app contains — before designing or mutating anything.

## Two different questions

| Question | Source | Never confuse with |
|---|---|---|
| What can the platform do? | `describe_platform` (prose), `describe_capabilities` (exact JSON schemas from the backend's `/meta/catalog`), `describe_node_type` | app inventory |
| What does this app contain? | `describe_platform section=inventory`, `list_forms`, `get_form`, `list_workflows`, `list_menus`, `list_roles`, `report_list`, `list_versions`, `list_saved_views` | platform capability |

A capability that exists but is unused in this app is **not missing**.

## Sequence

1. **Server & app.** `get_application`. Note `id`, `name` (needed verbatim for `publish_application` confirm), `published_version` (null = never published), `api_key_can_activate_live_triggers`.
2. **Profile.** Infer from the tool list:
   - full: authoring + records + admin + evaluation
   - design: forms, workflows, menus, dashboards, reports, translations, versions/publish
   - data: records, transitions, history, `simulate_viewer`, `preview_reference_options` (`data_describe_platform`, `data_list_forms`, `data_get_form`)
   - admin: roles, permissions, API keys, agents, knowledge bases (`admin_describe_platform`)
   - evaluation: datasets, runs, verdicts (`evaluation_describe_platform`)
   Do not attempt work outside the connected profile. Report the limitation and continue with authorized read-only work.
3. **Platform index.** `describe_platform` without arguments → section list. Open only what the task needs (`forms`, `forms.canvas`, `forms.detail_page`, `access_scope`, `menus`, `workflows`, `expressions`, `filters`, `reports`, `ui_workflows`, `dashboards`, `publishing`, `knowledge_bases`, `inventory`). Avoid `all`.
4. **Catalog index.** `describe_capabilities` without arguments → subtrees. Typical focused calls:
   - `forms.field_types`, `forms.definition_schema`, `access_scope` (alias; keeps its `$defs`)
   - `workflows.trigger_modes`, `workflows.trigger_event_types`, `workflows.compare_ops`, `workflows.filter_value_modes`, `workflows.variable_types`, `workflows.unsupported_capabilities`, `workflows.examples`, `workflows.webhook_providers`, `workflows.trigger_presets`
   - `menus.menu_types`, `menus.permission_modes`, `dashboards.widgets`, `ui_workflows.nodes`
   - `fingerprint`
   Dotted paths traverse object keys only, not array elements — for a node use `describe_node_type`.
5. **Nodes.** `describe_node_type <type>` for each node you plan to use. Read its **NOT for** line, deprecation flag and replacement. Package nodes (connectors) appear only if the deployment lists them.
6. **Inventory of the module in scope.** Summarize large inventories (counts, names, ids) rather than dumping them.
7. **Baseline.** Record ids, etags, enabled state of triggers, `list_versions` head, and anything published vs draft.

## Freshness

- The server re-registers tools when the backend catalog fingerprint changes. If `describe_capabilities section=fingerprint` differs from the one you used earlier, re-read affected schemas before authoring.
- `refresh_knowledge` reloads cached catalog/app context; use after external edits or when a tool reports stale context.
- Uploaded/source snapshots of the MCP server are context, not proof. The live schemas win; report contradictions (see "Known discrepancies").

## External API shapes

Before any step reads an `http_request` response: confirm the real shape from workspace scripts/CLIs, a prior `get_execution`, or user docs. Never print credentials. Declare it as the node's `response_schemas`. If unconfirmable, stop and ask for a sample, docs, or a builder "send request and auto-map" run.

## When the server is unavailable

- Say plainly that the App-Builder MCP server is not connected (or failed), and do not substitute other tools.
- Do design work only; label it `Designed`, and every implementation item `Not verified`.
- Mark capability assumptions as "documented in source snapshot, not live-verified".
- Provide the MCP implementation plan so it can be executed once connected.

## Known discrepancies (observed 2026-11-08, re-checked 2026-11-11; re-verify, they may be fixed)

| Observation | Classification |
|---|---|
| `assign_app_role` / `revoke_app_access` exist in source but are absent from the live tool list. Source shows they register only when the MCP process runs with `APP_BUILDER_MCP_ALLOW_MEMBERSHIP_WRITES=true` and the key holds `members:write` (granted by a Super Admin session). | `PERMISSION_BLOCKER` / configuration — **not** a missing feature |
| Source's capability summaries mention a `reports` catalog subtree; the live `describe_capabilities` index has none. Report schemas come from the report tools and `describe_platform section=reports`. | Documentation drift (`PLATFORM_DX`) |
| `describe_platform publishing` says versions "can be rolled back"; no MCP rollback/restore tool is exposed. | Candidate MCP tooling gap — runtime/UI capability presumed; verify before claiming |
| Record sharing exists at runtime; no MCP share tools (documented). | MCP tooling gap, not missing runtime |
| `simulate_viewer` resolves identity and enrollment profiles only; no tool shows which *records* a given role/user would see under `access_scope`. | `PLATFORM_DX` — test row security with real test users or reasoning, and say which |
| `hide_rules` audience `specific_people` needs platform user ids; no tool lists users. | Documented `PLATFORM_DX` |
| `find_references` does not cover external callers or exports. | Documented coverage limit — check those manually |

## Server behaviour to rely on (verified live 2026-11-11)

- **Tool profile decides what you see.** A server on the `design` profile exposes about 62 tools (no record, role, API-key or evaluation tools); `all` exposes about 103. If a tool family is missing, check the profile before calling it a platform gap.
- **"Not configured" is a 501, not an outage.** When a feature's backing config is missing (e.g. report images or rendering without file storage or craft CLIs), the tool returns HTTP 501 with the engine message naming what is missing, `retryable=false`. Do not retry; report it as a `CONFIG_BLOCKER`. Other 5xx bodies are blanked on purpose — treat them as server errors and report the status, not a guess.
- **Empty report tables keep their shape.** `report_view` table blocks always carry `rows` (`[]` when empty) and `total_rows`.
- **Formula privacy warnings are real.** `save_form` / `edit_form` warn when a formula reads a field that some roles cannot see (e.g. `hide_rules`). "Formula privacy check unavailable" means the check failed server-side — log it as a bug, do not ignore it.
- **Guide facts:** a boolean field `default` is a JSON boolean; new fields go through `edit_form` `add_field`; `get_workflow detail:full` returns `flow` when it fully represents the workflow and `save_workflow` accepts either `flow` or `definition`; one credential may reach several apps (`list_apps`, then `select_app`); WhatsApp exists only as `whatsapp_*` nodes from an installed package; form-scoped triggers scope by `source_form_id`.

## Compact response profile

Enable `APP_BUILDER_MCP_RESPONSE_PROFILE=compact-v1` in the environment of the
server instance the client connects to. It enables compact descriptions and
defaults resource ID schemas to string. Keep the variable unset or use
`legacy` for existing response shapes. The separate tool scope
(`all`, `design`, `data`, `admin`, `evaluation`) still controls discovery.

| Need | Read |
| --- | --- |
| Form structure and validation | `get_form` standard returns fields and `layout_outline`. Summary returns identity, counts and field references. |
| One canvas component or form field | `get_form` with `component_ids` (canvas IDs or field keys). A matching field comes back alongside its canvas element when present. |
| Workflow topology | `get_workflow` standard returns topology, trigger and condition outlines. Summary returns identity and counts. |
| One workflow node | `get_workflow` with `node_ids` for full matching configurations, or `node_query` for an exact ID, label or type. Multiple matches return candidate IDs. Older flow-only backends return matching steps and a flow outline for standard reads. |
| Diagnose a failed execution | `get_execution` standard returns complete errors, node statuses and timings. For a test run it includes the complete `test_report`. |
| Inspect one step's logged data | `get_execution` with `step_id` and `step_payload: "input"`, `"output"` or `"both"`. Follow `next_page` for more attempts. |
| Read full source | `detail: "full"` gives the existing form, workflow, execution, report or app representation. Add `include_node_outputs: true` only when you need the legacy bulk log view. |
| Read specific keys | `fields` selects top-level keys from the full object. Identity and concurrency fields stay. Choose either `fields` or an item selector. Item selectors return full matching items and take precedence over `detail`. |

List forms and workflows by `query` and continue with `next_cursor`. Use the
same filters with `cursor` in `list_executions` and `search_records`.
Search keeps the backend's filter and sort behavior and accepts `fields` for
record keys. `include_total: true` adds the matching count. Record cursors
continue by page, so concurrent writes can change the next page. Use
`detail: "full"` with `page` and `page_size` to get the legacy search shape.

For navigation, `list_menus` returns a 25-item outline with parent IDs,
sort order, access gates and form, report or dashboard targets. Use
`query` to find a menu and `get_menu` for its full config and etag.
`list_saved_views` pages the selected menu's views and keeps each complete
config, `can_manage` and `etag`. Use `view_id` for one view or `detail: "summary"`
for a smaller inventory. Continue with the same app or menu and query.
`detail: "full"` without filters or pagination returns the legacy array.

Use `edit_form` or `edit_workflow` for a targeted change and carry the exact
`etag`. Read `detail: "full"` before a replacement. For report draft updates,
carry `report_get.updated_at` as `expected_updated_at`. Form, workflow and
report create/update tools return receipts in compact mode. Request
`detail: "full"` on the write or fetch the saved resource to inspect its full
definition. Workflow edits that may generate step IDs retain the saved flow. Read errors, validation details and warnings before proceeding.

Bulk execution outputs can hit the existing log cap. Compact mode adds
`truncation` metadata with log counts and the targeted retrieval path.
A projection selects fields or items without shortening stored values.

`describe_platform` returns a short index. Use `section` for a focused guide,
or `tool` for the complete authoring guidance for a visible tool, for example
`{"tool":"report_save"}`. Use `describe_capabilities` for exact platform
schemas and `describe_node_type` for one node configuration. Claude Code Tool
Search can load supported MCP definitions on demand.
