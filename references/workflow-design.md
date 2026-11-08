# Workflow design

Open `describe_platform section=workflows` (node list with NOT-for lines, examples, trigger modes) and `section=expressions` once per session; `describe_node_type` for each node before use.

## Authoring model (verify against live docs)

- Author in **FLOW** form: `trigger` + ordered `steps`; `if` (condition, then, else), `foreach` (`config.source_expr`, `body`), any other step is a node type. Branches rejoin where arrays end.
- The server compiles FLOW to the builder graph; `get_workflow` returns flow + compiled graph + etag.
- **No parallelism**: independent same-wave nodes run one at a time in authored order. Edges are never conditional.
- Optional per-step `compensate` (saga-style undo, reverse order, on later failure; not for Before-mode triggers).
- Step `description` explains *why* — fill it for non-obvious steps.

## Triggers

| Mode | Use for | Cautions |
|---|---|---|
| before (create/update/delete) | fast validation, mandatory invariants, same-txn derived writes | data nodes only where the docs say so; no slow HTTP; `TriggerRecord["id"]` is nil on create |
| before submit/cancel | posting derived/ledger rows in the document transaction | only data nodes; failure leaves document unchanged |
| after / after_async | non-critical background work, notifications, outbox delivery | cascades: writes may fire other triggers |
| scheduled | sweeps, reconciliation, time rules | bounded batches; idempotent |
| date | record-relative waits (renewal −30d, SLA +24h) | field must be date/datetime and indexed |
| webhook | authenticated external events | verify provider/auth in `workflows.webhook_providers`; inbox + dedupe |
| on_demand / on_demand_data_driven | commands from users/record actions | declare `inputs` on the trigger |
| executed_by_workflow | canonical reusable subflows | declared variables = contract |
| on_error | alerting, dead-letter, recovery | gets failed ids + message only |
| task_completed (event) | continuing after a `human_task` | separate workflow |

Live modes save disabled unless the key holds `workflows:activate`. Carry `enabled` through every trigger edit.

## Semantics that change designs

- `human_task` creates a durable task and **continues immediately**. The response arrives in a separate `task_completed` workflow. Never model "wait for approval" inside one run.
- `delay` / `wait_for_signal` are bounded by the ~30 min execution ceiling and are invalid inside a `foreach` body. Long waits → date trigger or state + scheduled sweep.
- `write_batch`: one transaction, ≤200 records, across forms, before-triggers run inside, after-triggers after commit, committed retry returns the first output. It does **not** span HTTP, email, other executions or a second batch.
- `save_records` atomic ≤200 on one form; non-atomic stops at row N with 1..N-1 committed. Re-run safe only with deterministic unique keys.
- `upsert_records` matches on the form's unique fields — a form without unique fields cannot be upserted into; create path needs all required fields.
- Workflow database nodes **bypass `access_scope`**. Scope explicitly in filters.
- A misspelled `Vars["x"]` evaluates to nil and the run "succeeds". Save/validate **warnings** are the only signal — read them.
- `email` dedupes per execution and node (a retried node does not resend), but a *new* execution for the same event sends again — cross-execution dedupe still needs a sent-log with a unique key.
- `show_message` reaches only the triggering UI session; `notification` is in-app only; `email` is the off-platform channel; SMS/chat only via listed connector nodes.
- `http_request`: declare `response_schemas` from a confirmed shape; set `retry` only for idempotent calls; credentials from `secret` fields.

## Expressions

Roots: `Vars["name"]`, `NodeOutputs["node_id"]["field"]`, `TriggerRecord["field"]` (nil when no record; guard `TriggerRecord != nil`), `Context[...]`, `AppSettings["name"]`, `Times["now"]`, `Trigger.node_id` / `Trigger.mode`, `Steps` (inside write_batch values). Never a bare identifier. Validate important expressions with `validate_expression`; run `validate_workflow` before testing.

## Naming

Configurable; default `<Context> | <Entity or Area> | <Operation>`, e.g. `CRM | Customer | Create`, `CRM | Integration | Sync Customers`, `CRM | Notification | Send Welcome Email`. Adopt the app's existing convention if it has one.

## Workflow contract template

```
Name:              Responsibility (one sentence):
Trigger:           Callers / dependents:
Inputs (declared variables / trigger inputs / TriggerRecord fields):
Outputs:           Preconditions:         Postconditions:
Forms read:        Forms written (owned?):
External effects:  Idempotency key:       Transaction boundary:
Error behaviour:   Retry policy:          Compensation:
Security context (whose permissions; access_scope bypassed?):
Observability (correlation id, run/status rows, debug points):
```

## Build sequence for complex workflows

1. Write the contract. 2. `describe_node_type` for each node. 3. `validate_expression` for key expressions. 4. Build domain subflows first. 5. Then orchestrators. 6. `validate_workflow`; fix every warning. 7. Run safe tests (`run_workflow` with `variables`/`trigger_record`, synthetic data, authorized). 8. `get_execution` per-step outputs. 9. Verify durable records. 10. Check trigger cascades: which other workflows fire on the forms this one writes (`find_references` on the form, `list_workflows`).

## Size heuristics

- One workflow ≈ one responsibility; > ~15 steps or several unrelated branches → split into subflows.
- But don't fragment: a subflow called once with no reuse or contract value is overhead.
- `foreach` always bounded (`max_iters`, page/batch size); fetch once and loop, never fetch inside a loop over a large list (N+1).
