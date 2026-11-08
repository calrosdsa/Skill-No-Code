# Platform gap analysis

Platform-gap detection is a deliverable, not a side note. It also feeds the App-Builder backlog.

## Categories

| Category | Meaning |
|---|---|
| `PLATFORM_MISSING` | the runtime/engine has no such capability (a feature the runtime has but MCP lacks is `PLATFORM_DX`, not this) |
| `PLATFORM_LIMITATION` | exists but a limit blocks the requirement (size, duration, semantics) |
| `PLATFORM_UNKNOWN` | cannot determine yet; say what evidence is missing |
| `PLATFORM_DX` | possible but error-prone/opaque/undocumented, or the MCP lacks a tool for an existing runtime feature |
| `PLATFORM_BUG` | behaviour contradicts documented contract — reproduced |
| `APP_ARCHITECTURE` | the app's design is the problem |
| `APP_CONFIGURATION` | a setting/permission/field in the app is wrong |
| `PERMISSION_BLOCKER` | the credential/profile/env gate prevents it |

## Before claiming a gap

1. Live catalog (`describe_capabilities`, `describe_platform`).
2. `workflows.unsupported_capabilities` — the platform's explicit "no", with supported alternatives.
3. Exact tool schemas (fields you need may exist under another name).
4. The connected profile and env-gated tools (e.g. membership tools need `APP_BUILDER_MCP_ALLOW_MEMBERSHIP_WRITES`).
5. Existing examples (`workflows.examples`, templates).
6. Source/tests if accessible (`workflow-engine/cmd/mcp-server`, `api/meta`) — snapshot, not authority.
7. Runtime vs MCP coverage: does the UI/runtime support it while MCP doesn't?
8. Permissions (`get_application`, role grants, key flags).
9. Safe reproduction if authorized.

Classify by evidence. Examples:

- No MCP import tool, runtime CSV import exists → `PLATFORM_DX` (MCP tooling gap), not missing import.
- Workflow run refused for lack of permission → `PERMISSION_BLOCKER`.
- No atomic idempotent operation for a required cross-boundary transaction → `PLATFORM_LIMITATION`.
- Versions and publish exist → existing capability.
- Workflow DB nodes bypass `access_scope` → documented behaviour → app must scope explicitly (`APP_ARCHITECTURE` obligation); optionally propose a platform enhancement ("run as initiator scope").

## Report template

```
ID:
Title:
Category:
Severity:                (critical | high | medium | low)
Affected capability:
Requirement:
Current behavior:
Expected behavior:
Evidence:                (tool calls, schema excerpts, source paths, execution ids)
Reproduction:
Architectural impact:
Security or reliability impact:
Recommended platform enhancement:   (reusable primitive, not customer-specific)
Suggested MCP interface:
Suggested backend/runtime change:
Components:              MCP server | engine | form builder | workflow builder | menu builder | runtime | permissions | testing | docs
Acceptance criteria:
Temporary workaround:
Workaround limitations:
Risk if unresolved:
Priority:
Blocker status:          blocks production | blocks feature | non-blocking
```

## Release gating

Block production release for unresolved critical security, correctness or data-loss risks. Do not block ordinary development for a missing optimization; record it in the backlog.

## Backlog

Keep a prioritized list (ID, title, category, severity, priority, status) in the task output, or in a file the user names. Prefer reusable primitives: e.g. "atomic outbox write + delivery worker", "row-scope-as-initiator for workflow reads", "MCP version restore", "role row-visibility simulation", "list app users", "concurrency limit per workflow".

## Candidate gaps observed in the current server (re-verify)

See [mcp-discovery.md](mcp-discovery.md#known-discrepancies-observed-2026-11-08-re-verify-they-may-be-fixed). Worked example: [examples/platform-gap-report.md](../examples/platform-gap-report.md).
