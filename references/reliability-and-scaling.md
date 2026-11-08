# Reliability and scaling

## Assessment (for high-volume or business-critical apps)

| Topic | Question | Typical answer |
|---|---|---|
| Volume | records/day, total, growth per form | drives index + archival plan |
| Query patterns | which filters/sorts are hot | `index: true` on those fields |
| Pagination | lists > a page? | page through `search_records`/`fetch_records` limits |
| Selectivity | does each scheduled fetch narrow by indexed field + watermark? | yes, always |
| Loops | `foreach` bounded? | `max_iters`, batch size, `continue_on_error` for isolation |
| Duration | can one run exceed the ~30 min ceiling? | split into batches / checkpointed runs |
| External limits | vendor rate limits / quotas | batch size per run, schedule spacing |
| Concurrency | can two runs touch the same entity? | unique keys + upsert; idempotent design |
| Retry amplification | retry at node + workflow + scheduler? | retry at one level only |
| Transaction scope | what must be atomic | ≤200 records per `write_batch` |
| Partial failure | one bad item kills the batch? | per-item isolation + failure rows |
| Recovery | how to resume / rerun | checkpoint cursor + idempotent replay |
| Reconciliation | how gaps are detected | scheduled compare + review queue |
| Retention | archive/delete policy | per form |
| Read load | dashboards/reports on large forms | indexed filters, aggregates, summary forms if needed |

## Data layering for integrations

Separate, and name each: **external source of truth** (not stored or stored raw) → **imported raw** (optional, for replay/audit) → **normalized internal** (canonical forms, unique external key) → **derived operational state** (materialized, rebuildable) → **business event history** (append-only ledger/audit).

## Idempotency

- Every retryable write has a deterministic unique key: `<source>:<external_id>[:<line>]` or `<doctype>:<record_id>:<event>`.
- Use `upsert_records` (match on the form's unique fields) or `save_records` with deterministic keys; `write_batch` retries return the first committed output.
- Side effects (email, HTTP POST): record an outbox/sent row with a unique key **before or atomically with** deciding to send; check it before sending; pass vendor idempotency keys where supported. Accept at-least-once; never claim exactly-once.
- Notifications: dedupe key per (recipient, event, entity).

## Incremental over full recompute

Process only changed entities (watermark on `updated_at` or external change cursor) plus time-driven events (date triggers). Keep full recompute as an explicit, bounded recovery tool, run per entity.

## Failure isolation and recovery

- Per-item `continue_on_error` + failure form row (entity, error, attempt, next_retry_at) = dead-letter.
- `on_error` workflow for alerting and human triage (`human_task`).
- Run-log form (`sync_run`/`engine_run`): started, finished, cursor, counts, status — the observability backbone.
- Correction = new event/reversal, then recompute affected entity.

## Verify guarantees before claiming them

Outbox, inbox, saga, circuit breaker and rate limiting are built from primitives. State which guarantee holds (atomic local write, unique dedupe) and which does not (exactly-once delivery, cross-boundary atomicity, concurrency limits). Missing primitives → platform gap report.
