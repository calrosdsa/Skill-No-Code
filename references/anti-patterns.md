# Anti-patterns and review

## Review procedure

1. Orient (SKILL.md step 0). Scope: whole app or one module.
2. Inventory: forms (fields, keys, indexes, rules, access_scope), workflows (triggers, enabled, size, subflow calls), menus (tree, gates), roles, reports, versions.
3. Build maps: subflow call graph, form ownership (who writes which form — from workflow definitions), trigger cascade graph (workflow writes form X → triggers on X).
4. Check each anti-pattern below with evidence.
5. Report findings, most severe first.

## Catalog

| Anti-pattern | Evidence to look for | Typical fix |
|---|---|---|
| God workflow | many unrelated branches/responsibilities, > ~15 steps, many forms written | split into subflows behind an orchestrator |
| Duplicated business rule | same expression/filter in several workflows/forms | one canonical subflow or record rule |
| Excessive fragmentation | subflows called once, no contract value | inline |
| Tight coupling to external API | raw body paths read across many steps; no `response_schemas` | adapter subflow |
| Circular dependency | subflow call cycle; trigger cascade loop (A writes X, X-trigger writes Y, Y-trigger writes X) | break with state guard / `was_updated` filter / redesign |
| Unbounded loop | `foreach` without `max_iters` over a growing query | bounded batches + checkpoint |
| N+1 queries | `fetch_records` inside a `foreach` | fetch once with `in` filter, map in memory |
| Full scans | filters on non-indexed fields of large forms; scheduled job without watermark | `index: true`; incremental cursor |
| Missing transaction boundary | multi-form related writes in separate nodes | `write_batch` |
| Blind retries | `retry` on non-idempotent POST; on_error re-running whole workflow | idempotency keys; retry one level |
| Non-idempotent operation | creates without unique key on retryable path | unique business key + upsert |
| Duplicate notifications | send on every update without dedupe | sent-log with unique key; trigger filter `was_updated` |
| Inconsistent ownership | several contexts write the same form directly | owner's public subflow |
| Incorrect state transitions | status freely editable; no guard | read_only_rules + transition guard |
| Unsafe field deletion | field removed without `find_references` | additive migration |
| Broken references | `find_references`/`save_version` issues | fix or remove dependants |
| Unnecessary custom HTML | html menu re-implementing search/forms | built-in menus |
| Menu-as-security | confidentiality relying on hidden menu | role form permissions + access_scope |
| Incomplete row security | access_scope without coverage for some roles (they become unrestricted) | `everyone` rule / full coverage |
| Unprotected privileged workflow | workflow DB nodes reading/writing across scopes on user command without explicit scoping | explicit filters; restrict who can dispatch |
| Hidden assumptions | expressions on undeclared vars; guessed response shapes; validate warnings ignored | fix warnings; confirm shapes |
| Hardcoded mutable config | thresholds/emails/rates in expressions | config form / AppSettings |
| Missing observability | no run log, no correlation id, no failure rows | run-log + failure forms |
| Missing regression tests | no repeatable test inputs | test plan with `trigger_record` fixtures |
| AI output trusted | `llm_call`/agent output written directly to records/decisions | validate shape + rules; human review for consequential |
| Over-engineering | sagas/CQRS/multi-app with no requirement | simplify |
| `human_task` as blocking wait | logic after task creation assumes the answer | `task_completed` workflow |
| Long `delay` | delay/wait > 30 min or inside foreach | date trigger / state + sweep |
| Disabled-by-accident trigger | trigger edited without carrying `enabled` | restore + verify |

## Finding format

```
ID / Title / Severity (critical|high|medium|low)
Evidence: tool + object id + exact observation
Impact:   correctness | security | data loss | performance | maintainability
Fix:      concrete change (tools, ops)
Verify:   how to prove the fix
```

## Scoring

Avoid scores. If the user asks for one, publish the method (e.g. counts of findings per severity per dimension, with weights) and the evidence behind every point.
