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

## Known discrepancies (observed 2026-11-08; re-verify, they may be fixed)

| Observation | Classification |
|---|---|
| `assign_app_role` / `revoke_app_access` exist in source but are absent from the live tool list. Source shows they register only when the MCP process runs with `APP_BUILDER_MCP_ALLOW_MEMBERSHIP_WRITES=true` and the key holds `members:write` (granted by a Super Admin session). | `PERMISSION_BLOCKER` / configuration — **not** a missing feature |
| Source's capability summaries mention a `reports` catalog subtree; the live `describe_capabilities` index has none. Report schemas come from the report tools and `describe_platform section=reports`. | Documentation drift (`PLATFORM_DX`) |
| `describe_platform publishing` says versions "can be rolled back"; no MCP rollback/restore tool is exposed. | Candidate MCP tooling gap — runtime/UI capability presumed; verify before claiming |
| Record sharing exists at runtime; no MCP share tools (documented). | MCP tooling gap, not missing runtime |
| `simulate_viewer` resolves identity and enrollment profiles only; no tool shows which *records* a given role/user would see under `access_scope`. | `PLATFORM_DX` — test row security with real test users or reasoning, and say which |
| `hide_rules` audience `specific_people` needs platform user ids; no tool lists users. | Documented `PLATFORM_DX` |
| `find_references` does not cover external callers or exports. | Documented coverage limit — check those manually |
