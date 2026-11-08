# Pattern catalog mapped to App-Builder primitives

Select by requirement. Every pattern below has a cost; state it. "Primitive" names what the platform offers — re-verify with the live catalog before relying on a guarantee. If a guarantee is missing, the pattern is **partial**: say so.

Legend — **Cost**: L/M/H authoring + operational complexity.

## Structural

| Pattern | Use when | Primitives | Cost | Notes / limits |
|---|---|---|---|---|
| Layered / hexagonal | always, logically | UI (menus, UI workflows) → service workflows → domain (rules, before-triggers) → adapters (http_request) | L | Logical, not separate apps |
| Bounded context / modular monolith | >1 business capability | naming prefix, form ownership, subflow contracts | L | Default |
| Adapter | any external API | one subflow per external operation: `http_request` with `retry` + declared `response_schemas` → maps to internal variables | L | Only the adapter knows the vendor shape |
| Anti-corruption layer | messy/unstable external model | raw import form → normalize subflow → canonical form | M | Keep raw for replay/audit when needed |
| Facade | many callers need a multi-step op | one public subflow | L | |
| Dependency inversion | orchestrator vs implementations | orchestrator calls subflow by contract; strategy via config | L | |
| Separate apps | independent security/ownership/release | shared forms + cross-app links | H | Requires compatibility rules for shared forms |

## Domain

| Pattern | Primitives | Cost | Notes |
|---|---|---|---|
| Domain invariant | `validation_rules` (≤32/form, `when` save/submit), `unique`, `required`, `reference_filter` | L | Validate with `validate_expression` |
| Domain service | before-write workflow (data nodes) or subflow | L–M | Before-triggers must be fast |
| Application service / command handler | `on_demand_data_driven` workflow with declared trigger `inputs`; record action dispatch | L | Server enforces input ids/types/choices |
| State machine | enum `status` + `read_only_rules` keyed on stored status + before-write guard on allowed transitions; or `submittable` lifecycle with `transition_record` | M | Encode allowed transitions as a rule/config, not scattered `if`s |
| Strategy / policy | config form rows selecting behaviour; `if` on config value; separate subflows per strategy | M | |
| Specification | reusable filter groups / expressions | L | |
| Factory | creation subflow that sets defaults, numbering (`autonumber`) and child lines | L | |
| Versioned rules | config form: rule_key, version, effective_from/to, params; derived rows store version used | M | Needed for historical correction/replay |

## Data

| Pattern | Primitives | Cost | Notes |
|---|---|---|---|
| Repository boundary | context owns its forms; others go through its subflows | L | Not enforceable by platform — convention + review |
| Unit of work | `write_batch` (one transaction, ≤200 records, cross-form, retry returns first output); `save_records atomic:true` (≤200, one form); before submit/cancel workflow in document txn (data nodes only) | L | Never spans HTTP, email, separate executions, or multiple batches |
| Optimistic locking (design) | etags on forms/workflows/menus; `updated_at` on reports | L | Records: patch semantics, no etag guard → design for idempotency instead |
| Unique business key | `unique` field; deterministic key e.g. `<source>:<external_id>:<line>` | L | Foundation of idempotency and upsert |
| Audit log | record history (`get_record_history`) + explicit audit form for business events | L–M | `created_by/updated_by` null for API-key/system writes — add actor fields |
| Append-only ledger | ledger form, no edit/delete permissions for users, `read_only_rules` everyone; corrections as reversing rows | M | Workflows bypass `access_scope` — enforce append-only in workflow design |
| Materialized state | state form updated from ledger in same `write_batch` | M | Rebuildable from ledger |
| CQRS-lite | read models = dashboards/reports/aggregate_records over normalized forms; optional summary form | M | Only for real read-load or shape mismatch |
| Snapshot & replay | periodic snapshot rows + ledger since snapshot; replay subflow keyed by entity | H | Only when historical correction is a requirement |

## Integration & reliability

| Pattern | Primitives | Cost | Guarantee status |
|---|---|---|---|
| Idempotent consumer / inbox | inbox form with unique message/event id; `upsert_records` or unique-key write before processing | M | Real dedupe via unique constraint |
| Transactional outbox | outbox form row written in the same `write_batch` as the state change; separate after/scheduled workflow delivers and marks sent | M | Partial: delivery is at-least-once; external side must be idempotent. No exactly-once |
| Retry with backoff | `http_request.retry` policy | L | Retry only idempotent calls; check vendor idempotency keys |
| Circuit breaker | config/state row tracking consecutive failures; orchestrator skips when open | M | **Not a platform primitive** — hand-built; say so |
| Bulkhead | separate workflows/schedules per integration or tenant segment | L–M | No explicit concurrency-limit primitive verified |
| Rate limiter | bounded batch size per scheduled run; delay between calls (outside foreach) | M | Approximate; no token bucket primitive verified |
| Checkpoint / resume | sync_run form with cursor/watermark; each run resumes from last committed cursor | M | Commit cursor with data in one `write_batch` when local |
| Reconciliation | scheduled compare of external vs internal keys; mismatch → review queue form | M | Closes gaps left by at-least-once delivery |
| Dead-letter | failure form + `on_error` workflow; `human_task` for triage | M | |
| Saga | flow step `compensate` (undo runs in reverse when a later step fails); or process-manager state form | M–H | Compensation is best effort; not run for Before-mode triggers; design compensations idempotent |

## Workflow

| Pattern | Primitives | Cost | Notes |
|---|---|---|---|
| Orchestrator / worker | orchestrator → `subflow` calls | L | Subflow waits for result |
| Process manager | state form tracking a long process + date triggers / task_completed triggers advancing it | M–H | For anything > ~30 min (execution ceiling) |
| Reusable subflow | `executed_by_workflow` trigger + declared variables as contract | L | |
| Fan-out / fan-in | `foreach` (bounded `max_iters`, `continue_on_error`) ; nodes run sequentially | M | No parallel execution — don't promise it |
| Human-in-the-loop | `human_task` (non-blocking) + separate `task_completed` workflow | M | The creating run does not wait |
| Long wait | `date` trigger on indexed date field; not `delay` beyond ~30 min | L | |
| Deterministic replay | inputs + rule version stored; pure computation subflow | M–H | Only if correction/audit demands it |

## Do not introduce without a requirement

Event sourcing, full CQRS, sagas across many services, multiple apps, circuit breakers, snapshot/replay. Each must trace to a requirement (volume, audit, correction, external reliability, ownership). If the requirement is absent, the simpler design wins.

## Pattern write-up template (for each selected pattern)

```
Pattern:        Intent:            Requirement it serves:
Forms:          Workflows:         Menus:
Dependencies:   Transaction boundary:
Failure behaviour:                 Idempotency key:
Tests:          Complexity cost:   Unsupported guarantees / gaps:
```
