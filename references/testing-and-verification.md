# Testing and verification

## Data policy

Synthetic or user-approved test data only. Do not create, modify or delete real records to test without authorization. Prefix test records recognisably (e.g. `TEST-`), and clean them up only when asked or when the plan says so.

`run_workflow` really executes — emails, HTTP calls, writes. Before running: check which nodes have external effects; use test recipients/sandboxes; get authorization for anything off-platform.

## What to test

| Area | How |
|---|---|
| Record rules / required / unique | `create_record` / `update_record` with violating values → expect rejection with the rule message |
| References / reference_filter | `preview_reference_options`; write an out-of-filter value → expect rejection |
| Formulas | create record, `get_record`, compare computed value |
| Lifecycle | `transition_record` submit/cancel/amend; locked fields on submitted docs |
| State transitions | attempt illegal transition → expect rejection |
| Role restrictions | `simulate_menu_visibility`; real test users where available |
| Row-level / field masking | readback of `access_scope` / rules; runtime test via test users; otherwise state `Not verified` at runtime |
| Workflow side effects | `run_workflow` with `trigger_record`/`variables` → `get_execution` per-step outputs → `search_records` for durable effects |
| Duplicate execution | run the same input twice → no duplicate records/notifications |
| Concurrency | two updates on same entity; stale etag path for design edits |
| Error recovery | force a failing step (bad test endpoint) → on_error / dead-letter row / compensation ran |
| Historical correction | add correcting event → recompute → derived state matches expected |
| Reports/dashboards | `report_view` / `report_preview` / `aggregate_records` cross-check counts |
| Expressions | `validate_expression`, `validate_workflow` warnings = zero unexplained |

Verify **stored** results — a successful tool response is not proof.

## Evaluation profile

Use evaluation datasets/runs (`create_evaluation_dataset`, `run_evaluation`, `score_evaluation_run`, `compare_evaluation_runs`) for AI agents or AI-powered workflow behaviour inside the app. These are platform evaluations, distinct from this skill's own `evals/scenarios.md`.

## Debugging

1. Reproduce from evidence: `list_executions` → `get_execution` (status, per-step outputs, error classification).
2. Read the current definition (`get_workflow`) and its history (`get_workflow_history`) — did it change recently?
3. Check expressions (`validate_expression`; nil-on-misspelling), trigger filters (`was_updated`, fresh-read filters on date triggers), `enabled` state.
4. Check data: `get_record`, `get_record_history` of the triggering record.
5. Check permissions: role grants, `access_scope` (and that workflows bypass it), `simulate_viewer` for profile resolution.
6. Check external contracts: confirmed response shape vs `response_schemas`.
7. State a root cause with evidence. Propose the fix; apply only when authorized, via the safe-mutation cycle. No speculative or destructive changes while diagnosing.

## Reporting states

Per item: `Designed` / `Created` / `Updated` / `Validated` (schema/validator passed) / `Tested` (executed, durable result checked) / `Published` / `Blocked` (reason) / `Not verified` (and why).
