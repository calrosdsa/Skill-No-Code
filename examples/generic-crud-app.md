# Example — simple CRUD app (avoid overengineering)

**Request:** "Build an equipment-loan tracker: equipment, borrowers, loans with due dates; staff manage everything, borrowers see only their own loans; remind when overdue."

## Design (proportional)

One bounded context. No subflows beyond one reminder, no ledger, no sagas.

| Form | Key fields | Notes |
|---|---|---|
| Equipment | `asset_tag` (string, unique, searchable, title), `name`, `status` enum available/on_loan/retired | |
| Borrower | `email` (unique), `name` (title) | enrollment profile if borrowers log in (`create_user_on_submit`, email mapping) |
| Loan | `loan_no` autonumber `LN-{YYYY}-{####}`, `equipment` ref, `borrower` ref, `due_date` date **index**, `returned_at` datetime, `status` enum open/returned | `validation_rules`: `returned_at` required when status=returned |

Security: Staff role → view/create/edit on all three. Borrower role → view Loan only; `access_scope` on Loan: audience role=Borrower, filter `borrower eq current_user <borrower-form-id>.record_id`. Staff need no rule (unrestricted by being unmatched) — state that this is intended.

Workflow: `Loans | Loan | Overdue Reminder` — date trigger on `due_date` offset `+1d`, filter `status neq returned` → `email` to borrower. Saved disabled unless activation authorized.

Menus: module "Loans" → search Equipment, search Loans, add Loan; Borrower sees "My loans" search on Loan (gated by role).

## MCP sequence

1. `get_application`, `describe_platform forms`, `describe_capabilities forms.field_types`, `describe_node_type trigger`, `describe_node_type email`.
2. `create_form` ×3 (Equipment, Borrower, then Loan referencing their ids) → `get_form` each.
3. `validate_expression` for the record rule → `edit_form set_setting validation_rules`.
4. `list_roles` → create roles only if absent (`create_role`) — admin profile + approval.
5. `edit_form set_setting access_scope` on Loan → `get_form` readback.
6. `create_workflow` (date trigger) → `validate_workflow` → note `enabled:false`.
7. `create_menu` ×N → `simulate_menu_visibility` per role.
8. Test with `TEST-` records: rule rejection, `run_workflow trigger_record` with a test email.
9. `save_version` → report issues. Publish only if asked.

## Report

States per item (Created/Validated/Tested/Not verified), reminder trigger **disabled**, not published. No platform gaps. Roughly one screen of output — no 15-section document.
