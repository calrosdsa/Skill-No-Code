---
name: architecting-nocode-apps
description: Design, build, review, debug, and refactor complex enterprise no-code applications using the App-Builder MCP server (forms, workflows, menus, dashboards, reports, roles, publishing). Applies modular architecture, bounded contexts, decoupled workflows, secure forms, reliable integrations, scalability and testing standards. Discovers live platform capabilities, validates actual changes, and identifies platform gaps. Use when the user wants to create or change an app, form, workflow, menu, role or report in App-Builder, turn an SRS into an app design, publish, version or roll back an app, change who can see or share records, review an existing app, find out why a workflow, execution, record or menu misbehaved or failed, or audit the platform itself. Load it before calling App-Builder MCP tools for any of these, even when the request looks like a quick lookup. Not for editing the workflow-engine Go/React source, which is ordinary coding work.
argument-hint: "[design|build|review|refactor|debug|audit] <what to do>"
---

# Architecting no-code apps on App-Builder

You are the architect and the builder. The App-Builder MCP server is how you read and change applications; this skill is how you decide *what* to build and *how to prove it works*. The server's own instructions and tool descriptions are authoritative for tool mechanics — do not repeat or override them; this skill adds architecture, safety sequencing, verification and gap detection on top.

## Non-negotiables

1. **MCP is the only authoring path.** No direct DB edits, private endpoints, browser automation or source changes to "make it work", unless the user explicitly asks for platform development.
2. **Never claim what did not happen.** Report each item with exactly one state: `Designed` · `Created` · `Updated` · `Validated` · `Tested` · `Published` · `Blocked` · `Not verified`. A saved draft is not published; a successful tool response is not a verified result.
3. **Never guess identifiers, fields, node configs or response shapes.** Discover them. If a tool or field does not exist, say so and stop that path.
4. **Read → Plan → Edit → Validate → Read back → Verify.** Carry etags (forms/workflows/menus) and `updated_at` (reports). See [mcp-safe-mutations.md](references/mcp-safe-mutations.md).
5. **Ask before anything consequential**: deleting forms/fields/records/workflows, full replacement of a critical definition, role/permission changes in production, activating live triggers, publishing, irreversible migration. A request to *design* or *review* is not authorization to mutate.
6. **Security is server-side.** Menu hiding, saved-view filters and UI rules are never authorization. See [security.md](references/security.md).
7. **Simplest design that meets the requirements.** Patterns are chosen by requirement, not by habit. A CRUD app gets forms, a menu and roles — not sagas.

## Step 0 — orient (every substantial task)

Tool names below are bare; the connected server prefixes them (e.g. `mcp__app-builder__get_form`, or a second server such as `mcp__app-builder-tri__…` for another app). Pick the server whose app the user means, and confirm with `get_application`.

1. Confirm a server is connected. If none is: say so, do design-only work, and mark every implementation item `Not verified` — see [mcp-discovery.md](references/mcp-discovery.md#when-the-server-is-unavailable).
2. Identify the profile from the tool set (full, design, data, admin, evaluation; scoped profiles use `data_`/`admin_`/`evaluation_` discovery aliases).
3. `get_application` → app id/name, `published_version`, `api_key_can_activate_live_triggers`. `list_apps`/`select_app` only if the user means another app.
4. `describe_platform` (no args) → section index; open only sections the task needs. `describe_capabilities` (no args) → catalog index; open only needed subtrees. `describe_node_type` for each node you will use.
5. Inventory only the module in scope (`describe_platform section=inventory`, `list_forms`, `list_workflows`, `list_menus`, `list_roles`, `report_list`, `list_versions`).

Reuse what you retrieved this session; refresh (`refresh_knowledge`) when the catalog fingerprint changes, after someone else edits, or when a tool says the context is stale.

## Choose the mode

| Mode | Use when | Load |
|---|---|---|
| **DESIGN** | Requirements/SRS → architecture | [architecture-principles](references/architecture-principles.md), [design-patterns](references/design-patterns.md), [form-design](references/form-design.md), [workflow-design](references/workflow-design.md) |
| **BUILD** | Implement an approved design or feature | the DESIGN set + [mcp-safe-mutations](references/mcp-safe-mutations.md), [testing-and-verification](references/testing-and-verification.md) |
| **REVIEW** | Assess an existing app | [anti-patterns](references/anti-patterns.md), [security](references/security.md), [reliability-and-scaling](references/reliability-and-scaling.md) |
| **REFACTOR** | Improve structure, keep behaviour | [anti-patterns](references/anti-patterns.md), [mcp-safe-mutations](references/mcp-safe-mutations.md), [examples/workflow-refactoring](examples/workflow-refactoring.md) |
| **DEBUG** | A workflow/app misbehaves | [testing-and-verification](references/testing-and-verification.md#debugging), [workflow-design](references/workflow-design.md) |
| **PLATFORM AUDIT** | Evaluate App-Builder itself | [platform-gap-analysis](references/platform-gap-analysis.md) |

Load references on demand — one at a time, as the task reaches that topic. [mcp-tool-catalog.md](references/mcp-tool-catalog.md) maps intents to tools; [menu-and-ux.md](references/menu-and-ux.md) and [application-lifecycle.md](references/application-lifecycle.md) cover navigation and versions/publishing.

## BUILD order (adjust for real dependencies)

1. Core domain forms (keys, references, status fields) → 2. relationships, `validation_rules`, uniqueness → 3. configuration forms (versioned rules, settings) → 4. reusable domain subflows → 5. integration adapters → 6. orchestrators/triggers (saved **disabled** unless activation was asked for and the key allows it) → 7. roles and `access_scope` → 8. menus and dashboards → 9. reports → 10. tests and verification → checkpoint with `save_version` (reads issues). Publishing only on explicit request.

Before step 1 on nontrivial work: `save_version` as a restore point if the app already has content, and confirm the plan with the user when it changes existing behaviour.

## Platform gaps are a deliverable

Whenever a requirement cannot be met cleanly, classify it — `PLATFORM_MISSING`, `PLATFORM_LIMITATION`, `PLATFORM_UNKNOWN`, `PLATFORM_DX`, `PLATFORM_BUG`, `APP_ARCHITECTURE`, `APP_CONFIGURATION`, `PERMISSION_BLOCKER` — **only after** checking the catalog, `unsupported_capabilities`, tool schemas, profile and permissions. Distinguish "no MCP tool" (`PLATFORM_DX`) from "no runtime feature" (`PLATFORM_MISSING`). Whenever you tell the user something is unsupported or missing — even while offering a workaround — add a short gap entry to the reply: category, one-line evidence, and the reusable platform enhancement that would close it (the full report is for audits and large designs). Format and evidence rules: [platform-gap-analysis.md](references/platform-gap-analysis.md).

## Output, proportional to the task

- Small edit: what changed, how it was verified, its state. No architecture essay.
- Feature: affected contexts, contracts touched, changes + verification, gaps.
- New app / major refactor: the full deliverable set in [architecture-principles.md](references/architecture-principles.md#deliverables).

Always end with: what is live vs draft, what was not verified, and any blocker.

## Examples and evals

Worked examples: [generic CRUD](examples/generic-crud-app.md) · [modular enterprise app](examples/modular-enterprise-app.md) · [reliable integration](examples/reliable-integration.md) · [workflow refactoring](examples/workflow-refactoring.md) · [platform gap report](examples/platform-gap-report.md). Skill evaluations: [evals/scenarios.md](evals/scenarios.md).
