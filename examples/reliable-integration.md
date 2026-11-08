# Example — reliable high-volume external sync

**Request:** "Sync ~50k products and daily stock changes from a supplier REST API every night; never duplicate; report failures."

## Discovery first

- Confirm the API's response shape from a workspace script, docs or a previous execution. If none: stop and ask for a sample/docs. Declare `response_schemas`.
- Credentials in a Connection form with `secret` fields.
- `describe_node_type http_request` (retry, pagination options), `write_batch`, `upsert_records`, `trigger` (scheduled).

## Data layering

| Form | Role | Key |
|---|---|---|
| Supplier Connection | config (base_url, secret token) | single record |
| Sync Run | run log + checkpoint | `run_key` unique; fields: started_at, cursor, pages_done, upserted, failed, status |
| Product | normalized canonical | `supplier_sku` unique, indexed `updated_at_source` |
| Sync Failure | dead-letter | `failure_key = run_key:sku` unique; error, payload excerpt (no secrets), attempts |

Raw payload storage only if replay/audit requires it.

## Workflows

1. `Catalog | Integration | Fetch Page` (adapter subflow): `http_request` with `retry` (GET is idempotent) → maps to internal list via declared schema → returns `{items, next_cursor}`.
2. `Catalog | Integration | Apply Page`: `transform` → `save_records atomic:true` (≤200) keyed on `supplier_sku`, or `write_batch` that also advances `Sync Run.cursor` in the same transaction.
3. `Catalog | Integration | Nightly Sync` (scheduled orchestrator): load/create Sync Run; `foreach` bounded pages per run (`max_iters` chosen so a run finishes well under the ~30 min ceiling); per page call 1 then 2; `continue_on_error` per item path writes Sync Failure; stop when no cursor. If pages remain, the next scheduled tick resumes from the stored cursor.
4. `Catalog | Integration | Reconcile` (weekly scheduled): compare supplier counts/checksums with `aggregate_records`; mismatches → review rows.
5. `on_error` workflow → notification/`human_task` to integration owner.

## Guarantees stated honestly

- Idempotent: unique `supplier_sku` + upsert semantics; reprocessing a page is safe.
- Atomic: page apply + cursor advance in one transaction (≤200 rows); not across pages.
- Not guaranteed: no concurrency limit primitive verified — ensure only one schedule; rate limiting approximated by page size per run. Record as `PLATFORM_LIMITATION`/`PLATFORM_UNKNOWN` if the vendor limit demands more.

## Tests

Run against a sandbox endpoint or with a small `max_iters`; run twice → identical counts; inject a bad item → one Sync Failure, others applied; kill mid-run (or simulate by limiting iterations) → next run resumes from cursor.
