# Architecture principles for no-code apps

## Logical layers (not separate apps)

```
User experience     menus · dashboards · search views · forms · reports · UI workflows
        │  collects input, starts commands; holds no consequential rules
        ▼
Application services  orchestrator workflows · canonical subflows (one per business operation)
        ▼
Business domain     validation_rules · formulas · before-write workflows · state fields · versioned config
        ▼                         ▼
Persistence          Integrations
forms/records ·      http_request · email · AI nodes · webhooks
ledgers · audit      behind adapters that translate to internal contracts
```

Dependencies point downward/inward: UI depends on services, services on domain, adapters translate the outside world into domain contracts. The domain never depends on a menu, a page or an external payload shape.

## Bounded contexts

Identify contexts by business capability and data ownership, not by screen. For each context record:

| Item | Question |
|---|---|
| Responsibilities | What decisions does it own? |
| Owned forms | Which forms may only it write? |
| Invariants | What must always be true? (→ `validation_rules`, unique keys, before-write workflows) |
| Public operations | Which subflows/record actions do others call? |
| Contracts | Inputs/outputs of those operations (declared variables, result shape) |
| Events / states | Status fields, transitions, ledger rows emitted |
| External dependencies | Which systems, via which adapter |
| Security boundary | Roles, `access_scope`, field rules |
| Consistency | What must be atomic (one `write_batch` / one before-trigger) vs eventually consistent |
| Failure expectations | What happens if it is down or a step fails |

Default to a **modular monolith**: one app, contexts separated by naming, form ownership and subflow contracts. Split into multiple apps (shared forms, cross-app links) only for independent security, release cadence, ownership or reuse needs — and then document the shared-form compatibility contract.

## Decoupling rules

- UI workflows and record actions collect input and dispatch; consequential rules run server-side.
- Menus are never a dependency of business logic.
- External payloads are mapped to internal forms/variables at the adapter boundary (declared `response_schemas`), never read raw downstream.
- One business operation → one canonical implementation (usually a subflow). Callers reuse it.
- No circular subflow calls. Draw the call graph.
- A context writes another context's owned records only through that context's public operation.
- Do not combine unrelated responsibilities just because they share a trigger — use separate workflows on the same trigger, or an orchestrator that calls separate subflows.
- Prefer references to copies. Copy (snapshot) data only when you need its value *as of* an event (price at order time), and name it so.
- Mutable business configuration (thresholds, windows, rates, lists) lives in configuration forms with effective dates or `AppSettings`, not hardcoded in expressions.

## SOLID / GRASP translated

| Principle | In App-Builder |
|---|---|
| Single responsibility | one workflow = one reason to change; one form = one entity |
| Open/closed | add behaviour via new subflows/config rows, not edits to a god workflow |
| Liskov / contracts | a subflow's declared inputs/outputs stay stable; extend, don't repurpose |
| Interface segregation | small subflows with narrow inputs over one with 20 optional variables |
| Dependency inversion | orchestrators depend on subflow contracts; adapters isolate vendors |
| Information expert | the context owning the data computes rules about it |
| Low coupling / high cohesion | measure with `find_references` and the subflow call graph |
| Controller | record actions / triggers delegate to an application-service workflow |

## Transferable principles from rule-heavy systems

Extracted from a real rule-heavy SRS case study (only the principles carried over, no domain content):

- **Source-of-truth separation.** An external system owns its facts; the app owns derived state. No silent write-back.
- **Deterministic and replayable.** Same inputs + same rule version → same outputs. Any derived state can be recomputed from an append-only event ledger.
- **Rules are configuration** with effective dates and versions, not code.
- **Explain every derived change**: store the rule version and inputs that produced it.
- **Bounded work.** Periodic jobs touch only changed entities plus time-driven events; a full recompute is a recovery tool, not the critical path.
- **Immutable audit** of state changes, configuration changes, exports and adjustments.
- **Failure isolation.** One entity failing does not block the batch; the next run catches up (checkpoint + reconciliation).
- **Corrections are new events**, never edits to history.

Apply these only where the requirements need them.

## Deliverables

For a new app or major refactor (skip sections that do not apply; keep small tasks small):

1. Requirement → implementation matrix (requirement, context, forms/workflows/menus, state)
2. Confirmed assumptions and open questions
3. Bounded-context map
4. Form/data model (keys, references, indexes, ownership, retention)
5. Workflow contracts ([template](workflow-design.md#workflow-contract-template))
6. Menu and UX architecture
7. Integration architecture
8. Security matrix (role × form × action, `access_scope`, field rules, privileged workflows)
9. Failure and recovery strategy
10. Performance strategy
11. Test plan
12. MCP implementation plan (ordered tool calls, approvals needed)
13. Actual MCP changes with state per item
14. Platform gap report
15. Release/operational notes (what is live, what is disabled, how to roll back)
