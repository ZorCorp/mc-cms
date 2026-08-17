# Cloud MS schema & join map (BigQuery mirror)

All tables are `` `mc-cms`.cms_to_bigquery.cloud_ms_<name> ``. Columns match the MySQL source.
**44 tables**, ~22 MB total. A Datastream CDC mirror: **current-state** rows (one per PK),
soft-deleted rows carry `is_deleted = 1` (filter `= 0`), and a `version` column is the CDC
revision number (not business data). Dates are `DATETIME`.

Cloud MS is MC's Google Cloud / Workspace **reseller management** system. The spine is the
**billing account**; customers, virtual customers, approvals and audit all hang off it.

## Billing accounts & customers

### cloud_ms_billing_account  (the reseller sub-accounts)
`name` (PK — `billingAccounts/XXXXXX-XXXXXX-XXXXXX`; strip the prefix for the account id),
`display_name`, `status` (ACTIVE / CANCELLED / UNASSIGNED / SUSPENDED), `open` (0/1),
**`master_billing_account_name`** (the reseller master account — same ids the `gcp-bq`
billing exports use), **`customer_id`** (→ cloud_ms_customer), `entitlement_id`, `creator`,
`create_time`, `update_time`. Also `cloud_ms_billing_account_role`, `_history`, `_tag`,
`_auto_send_email_{to,cc,bcc}`.

### cloud_ms_customer  (the end-customer org)
**`customer_id`** (PK), `account_id`, `org_display_name`, `domain`, `creator`, **`is_deleted`**.
Join `cloud_ms_billing_account.customer_id = cloud_ms_customer.customer_id`.

## Virtual customers & repricing

### cloud_ms_virtual_customer
`id` (PK), `name`, `channel_service_account_id`, looker-studio fields, `create_time`.
### cloud_ms_virtual_customer_billing_account  — `virtual_customer_id`, `billing_account_name`
### cloud_ms_virtual_customer_project          — `virtual_customer_id`, `project_id`
### cloud_ms_virtual_customer_tiered_repricing  — `virtual_customer_id`, `sku_id`, `tier`, `price`
Custom per-SKU pricing by tier (**tier 0 = highest price, higher tier = cheaper**). Join
`sku_id` → `cloud_ms_google_cloud_sku` for descriptions. Also `_proportion_repricing`,
`_email`, `_role`.

### cloud_ms_google_cloud_sku
`sku_id` (PK), `sku_description`, `service_id`, `service_description`, `product_taxonomy`,
`per_unit_quantity`. Plus `cloud_ms_default_google_cloud_sku_tiered_pricing` (the defaults).

## Approvals

### cloud_ms_approve_request
`id` (PK), `requester`, `request_reason`, `content` (JSON), `approve_type`
(e.g. BILLING_ACCOUNT_CREATE, BILLING_ACCOUNT_GCP_ROLES_MODIFICATION,
BILLING_ACCOUNT_PROJECT_UNLINK, BILLING_ACCOUNT_TERMINATION), `status`
(APPROVED / REJECTED / BYPASSED / …), `resolved_time`, `create_time`.
### cloud_ms_approval_flow / cloud_ms_approver_result / cloud_ms_approval_group(_member) / cloud_ms_approval_cc / cloud_ms_approval_revoker
The per-request approval steps, assignees, groups and results.

## People / org

### cloud_ms_user_profile  — `email` (PK), `state` (ACTIVE / RESIGNED), `resignation_date`, `team_id`
### cloud_ms_team           — `id` (PK), `name`, `description`, `state`, `approval_auto_reject_days`
### cloud_ms_team_manager   — `team_id`, `email`
Also `cloud_ms_user_role`, `cloud_ms_user_profile_line_manager`, `_regional_manager`.
Count members with `up.state = 'ACTIVE'` to exclude resigned staff.

## Audit & misc

- **cloud_ms_audit_log** — `id`, `event_name` (BILLING_ACCOUNT_MODIFIED / _CREATED,
  CUSTOMER_CREATED, APPROVE_REQUEST_APPROVED, …), `key`, `content` (JSON), `time`, `operator`.
- **cloud_ms_reminder** (+ `_assignee`), **cloud_ms_async_task**, **cloud_ms_system_config**,
  **cloud_ms_billing_export_drive_file**, **cloud_ms_looker_studio_template**.

## Cross-system links (for context — not in this dataset)

- Billing account id → **actual Google cost**: the reseller billing exports (`gcp-bq` skill).
- Billing account id → **customer company / orders / GP**: the BMS mirror (`bms2` skill).
- **No single system has every billing account id.** Cloud MS covers the accounts opened /
  managed through it — a large subset, not the complete set. Don't present it as exhaustive.

## Gotchas

- **Soft delete** — filter `is_deleted = 0` where the column exists (customer, and others).
- **Billing account id** = `REPLACE(name, 'billingAccounts/', '')`.
- **account_id** on `cloud_ms_billing_account` is the reseller **channel** id (only a few
  distinct) — NOT the per-account id. Use `name` for the account, `customer_id` for the customer.
- **Resigned staff** stay in `cloud_ms_user_profile` with `state = 'RESIGNED'`.
