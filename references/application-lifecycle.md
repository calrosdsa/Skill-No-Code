# Application lifecycle

Open `describe_platform section=publishing` for the current model.

## Model

- **Design time**: every save to forms, menus, workflows, reports changes the draft app. End users do not see it.
- **Version checkpoint**: `save_version` records the design-time state as a restore point and returns `issues` (the same validation publish runs, non-blocking). Not live.
- **Publish**: `publish_application` validates and snapshots the **whole app** into the runtime. Refused if validation fails. Requires `confirm` = exact current app name. Publishes everything saved, including other people's unpublished changes.
- **History**: `list_versions` lists publishes, checkpoints and rollbacks.
- **Reports**: drafts are separate; `report_publish` releases one report draft; guard edits with `expected_updated_at`.

Versioning exists — never report ALM as missing because the app hasn't used it.

## Rollback

The platform documents rollback, but the connected MCP server exposes no rollback/restore tool (verify with the live tool list). Before relying on rollback, tell the user how it would be performed (builder UI, admin), and classify the missing MCP tool as a tooling gap if it matters.

## Release procedure (only on explicit authorization)

1. `get_application` — confirm app and current `published_version`.
2. Review what changed since the last publish (your changes and others' — `list_versions`, `get_workflow_history`, form reads). Surface anything you did not make.
3. `save_version` with a description; resolve or accept each `issue` with the user.
4. Confirm trigger activation state for every live workflow — publishing does not need, and should not imply, activation.
5. `publish_application` with confirm.
6. Verify the **runtime** result: `list_versions` shows the new published version; `simulate_menu_visibility` on published menus; smoke-test a record path.
7. Release notes: what went live, what remains disabled/draft, rollback point (version id).

## Safe migration

Additive change → backfill in bounded, idempotent batches → switch consumers → publish → remove old structure in a later release. Never combine destructive schema changes with a publish without a checkpoint and explicit approval.
