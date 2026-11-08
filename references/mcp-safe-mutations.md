# Safe MCP mutations

## The cycle

1. **Read** the current definition (`get_form` / `get_workflow` / `get_menu` / `report_get` / `get_role`).
2. **Capture the guard**: `etag` (forms, workflows, menus) or `updated_at` → `expected_updated_at` (reports). Records patch and have no design etag.
3. **Analyze consumers**: `find_references` (both directions) for anything renamed, retyped, removed or whose contract changes. Respect its stated coverage; check external callers/exports yourself.
4. **Prefer targeted edits**: `edit_form`, `edit_workflow`, `edit_dashboard`, `edit_translations`, `update_role` deltas.
5. **Preserve** unrelated fields, column ids, layout, rules, permissions, trigger `enabled` state.
6. **Apply** once.
7. **Inspect the response**: errors, `warnings` (unresolved expression reads are the only signal of a misspelled variable), returned etag, returned `enabled`.
8. **Read back** and diff against intent: typed fields *and* layout; rules projected as expected; trigger enabled state unchanged unless asked.
9. **Validate/test**: `validate_workflow`, `validate_expression`, `save_version` issues, targeted tests.

## Full replacement rules

- `update_form`: send the **complete** field list. An omitted field is dropped together with its column and stored data. Keep every `column` id. To rename, keep the column and change the name. Settings you omit are carried forward by the server; fields are not.
- `update_workflow`: send the full definition based on `get_workflow`'s compiled graph. A trigger config is replaced wholesale — carry `enabled` forward from the top-level `enabled` read, or the trigger silently becomes disabled.
- `update_menu`: full replacement of that menu. For one dashboard tile use `edit_dashboard`.
- `reorder_menus`: the complete child list of one parent.
- `update_role`: `permissions` replaces the whole list; prefer `add_permissions` / `remove_permissions`. A removed key can still be granted by a wildcard — check.

## Concurrency

- Pass the etag from your read. A `stale_etag`/412 (or report 409) means someone else saved. **Re-read**, look at what changed, re-apply only your intended change to the fresh state, and re-verify that nothing of theirs is lost. Never replay the failed body.
- `update_form` without an etag still guards its own internal read-merge-write, but only your etag protects your *intent* against a human editing in the builder since your read.

## Errors

- Read `structuredContent.error` every time. `retryable: false` → do not retry; fix the cause or report.
- A timeout is an **ambiguous** outcome, not a failure. Re-read the resource (or list executions/records) before deciding whether another write is safe.
- A permission error is a `PERMISSION_BLOCKER`, not evidence the feature is missing. Never escalate, switch credentials, or route around it with another tool.
- Never invent tool names, arguments, fields or permission keys.

## Destructive / consequential actions — require explicit authorization

Delete form/field-with-data/record/workflow/menu/report/role · replace a critical definition wholesale · change production roles/permissions · activate a live trigger · `publish_application` · `report_publish` · irreversible data migration · revoke API keys.

Use the tool's own confirm-by-current-name argument with the *current* name from a fresh read. "Build me a skill" or "review my app" is not authorization for any of these.

## Activation

Live trigger modes (scheduled, date, before, after, after_async, webhook, on_error) save **disabled** unless the key holds `workflows:activate` (`get_application.api_key_can_activate_live_triggers`). Enable only what the user asked to activate, and say so. `run_workflow` executes a run; it never activates a trigger. A disabled scheduled/webhook trigger cannot be run headlessly.

## Checkpoints

Before a large or risky change set on an existing app: `save_version` with a description; read its `issues`. It is a restore point, not a publish. (Restore is not exposed as an MCP tool — see discovery notes; tell the user how rollback would happen before relying on it.)
