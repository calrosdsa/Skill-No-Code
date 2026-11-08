# Example — platform gap report

Two worked entries, based on behaviour documented by the current MCP server. Re-verify before filing.

```
ID:                     GAP-001
Title:                  Workflow database nodes cannot run under the initiator's row scope
Category:               PLATFORM_LIMITATION
Severity:               high
Affected capability:    workflows / access_scope
Requirement:            A user-dispatched bulk action must only touch records that user can see
Current behavior:       describe_platform access_scope: "Workflow database activities bypass access_scope"
Expected behavior:      An opt-in per node or per workflow: enforce the initiating user's access_scope
Evidence:               describe_platform section=access_scope (live); crm_convert_lead node documents
                        initiator scope enforcement, showing the primitive exists for one node
Reproduction:           access_scope on form F for role R; on_demand workflow update_records on F without
                        filter; dispatch as R user → records outside scope updated
Architectural impact:   every privileged workflow must re-implement scope filters (duplication, drift)
Security impact:        data exposure / unauthorized writes if a filter is forgotten
Recommended enhancement: generic "run data nodes as initiator" execution option
Suggested MCP interface: trigger/node config flag `enforce_initiator_scope: true`, shown in describe_node_type
Suggested backend change: pass initiator identity to data activities; apply access_scope compiler
Components:             engine, workflow builder, MCP schema, docs
Acceptance criteria:    with flag on, the reproduction updates only in-scope records; tests per data node
Temporary workaround:   explicit filters using TriggerRecord / initiator profile in every node
Workaround limitations: manual, unverifiable by platform, easy to miss in edits
Risk if unresolved:     silent cross-scope writes after future edits
Priority:               P1
Blocker status:         non-blocking with workaround + review checklist
```

```
ID:                     GAP-002
Title:                  No MCP tool to restore a saved version
Category:               PLATFORM_DX (MCP tooling gap)
Severity:               medium
Affected capability:    application lifecycle
Requirement:            Safe automated refactors need a scripted rollback path
Current behavior:       list_versions/save_version/publish_application exist; publishing docs mention
                        rollback; no restore tool in the live tool list
Expected behavior:      restore_version(version_id, confirm) with the same validation as publish
Evidence:               live tool list; describe_platform section=publishing
Reproduction:           n/a (absence)
Architectural impact:   rollback requires a human in the builder
Reliability impact:     slower recovery from a bad automated change
Recommended enhancement: MCP restore tool reusing the backend rollback endpoint (if it exists — verify)
Suggested MCP interface: restore_version {version_id, confirm: <app name>} → issues, new draft etag
Components:             MCP server, docs
Acceptance criteria:    restore + readback equals checkpoint; list_versions records the rollback
Temporary workaround:   save_version before changes; user restores via UI
Workaround limitations: manual
Risk if unresolved:     low–medium
Priority:               P2
Blocker status:         non-blocking
```

Note what is **not** a gap: membership tools absent because `APP_BUILDER_MCP_ALLOW_MEMBERSHIP_WRITES` is unset → `PERMISSION_BLOCKER`/configuration.
