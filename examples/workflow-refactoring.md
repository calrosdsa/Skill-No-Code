# Example — refactoring a god workflow without changing behaviour

**Situation:** `Order Saved` (after trigger on Order, live, 34 steps) validates stock, recalculates totals, emails the customer, posts to an ERP over HTTP and updates a dashboard summary. Duplicate emails are reported.

## 1. Baseline (read-only)

- `get_workflow` → flow, compiled graph, **etag**, top-level `enabled` (true).
- `get_workflow_history`, `list_executions` + a few `get_execution` → real inputs/outputs to replay as fixtures.
- `find_references id=<workflow>` → callers, on_error links.
- Forms it writes; triggers on those forms (cascade check).
- Root cause of duplicates: the after trigger fires on every update, including its own summary write → re-entry. Evidence: execution chain ids.

## 2. Invariants to preserve

Totals formula result; stock rule rejection; one email per order confirmation; ERP posted once per order.

## 3. Target design

| New workflow | Trigger | Responsibility |
|---|---|---|
| `Sales | Order | Validate` | before update/create | stock + totals invariants (data nodes only, fast) — or move totals to a formula and stock to a record rule where expressible |
| `Sales | Order | Confirm` | after, filter `status was_updated` and `status eq confirmed` | write outbox rows (email + ERP) in one `write_batch` with unique keys |
| `Notify | Email | Deliver` | after_async on outbox | send, mark sent |
| `ERP | Order | Post` (adapter) | after_async on outbox | http_request with idempotency key, mark posted |
| Dashboard summary | — | replace with `aggregate` tiles; delete summary writes |

## 4. Change sequence (each step authorized)

1. `save_version "before order workflow refactor"`.
2. Create the new workflows — triggers saved **disabled**.
3. `validate_workflow` each; zero unexplained warnings.
4. Replay fixtures via `run_workflow trigger_record=…` on test records; compare outputs with baseline.
5. Cut-over (approval): disable the old trigger via `edit_workflow set_trigger` carrying all other config with `enabled:false` and the read etag; enable the new ones only if the key can activate and the user asked; otherwise hand the user the exact activation list.
6. Observe executions; confirm one email per confirmation.
7. Delete the old workflow only later, after `find_references` and approval.

On `stale_etag` at step 5: re-read, check what the other editor changed, re-apply only the trigger change.

## 5. Report

Behaviour-equivalence evidence (fixture comparisons), states per workflow, activation status, remaining risk (email at-least-once).
