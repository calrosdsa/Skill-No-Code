# Example — modular enterprise app (CRM + Sales + Notifications)

**Request:** "Customers, opportunities and quotes; when a quote is accepted create an order and notify the account owner; sales reps see only their accounts."

## Bounded contexts

| Context | Owns | Public operations | Invariants |
|---|---|---|---|
| Customer | Account, Contact | `CRM | Customer | Upsert` (by `account_code` unique) | one account per code |
| Sales | Opportunity, Quote (+ Quote Line items), Order | `Sales | Quote | Accept` | accepted quote immutable; order total = quote total |
| Notification | Notification Log | `Notify | Event | Send` (dedupe key) | one notification per (event, entity, recipient) |

Sales references Customer via `account` reference; it never writes Account directly.

## Key decisions

- **Accept quote** = `on_demand_data_driven` record action on Quote → `Sales | Quote | Accept` subflow:
  1. guard: `TriggerRecord["status"] == "sent"` else `show_message` and stop.
  2. `write_batch`: update Quote status=accepted; create Order with `quote` ref and `order_key = "quote:" + id` (unique); create Notification Log row (outbox) `dedupe_key = "quote_accepted:" + id + ":" + owner`.
  3. Quote `read_only_rules` audience everyone, when status eq accepted.
- **Notification delivery** = after_async trigger on Notification Log (status=pending) → `email` → update status=sent. Retry-safe: re-run checks status first; email may still be sent twice on crash between send and update — documented at-least-once.
- **Row security**: `access_scope` on Account/Opportunity/Quote/Order: role SalesRep → `owner eq current_user <rep-profile>.record_id`. SalesManager matches no rule and is therefore unrestricted — intended here, and stated explicitly in the security matrix. Any future role must be added deliberately (a role not covered sees everything). Workflows bypass scope: the Accept subflow only acts on the triggering record, which the user could already see.
- **Totals**: `line_item_count` aggregate sum on Quote lines; formula for tax.

## Why not more

No separate apps (single team, single release cadence). No saga: all local writes fit one `write_batch`; the only external effect (email) is behind an outbox.

## Contracts (abbreviated)

```
Sales | Quote | Accept
Trigger: on_demand_data_driven on Quote     Inputs: none (TriggerRecord)
Writes: Quote (owned), Order (owned), Notification Log (via outbox row — owned by Notification context, write allowed by contract)
Idempotency: Order.order_key unique; batch retry returns first output
Transaction: one write_batch          Error: guard failure → show_message, no writes
Security: dispatchers need forms:<quote>:edit; record visible to caller
```

Build order: Customer forms → Sales forms + rules → Notification Log → subflows → triggers (disabled) → roles/scope → menus per role → dashboard (pipeline by stage via `aggregate`) → tests (double-accept produces one order and one log row) → `save_version`.
