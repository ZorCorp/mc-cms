---
name: mc-cms
description: Use when the user asks about Master Concept's Cloud MS reseller-management data — Google Cloud / Workspace billing accounts (status, master reseller, per customer), customers (org + domain), virtual customers and their custom tiered SKU repricing, billing-account approval requests, internal teams and members, or the CMS audit log. Calls named tools on a private Cloud Run MCP endpoint (toolset `mc-cms`) that reads the read-only BigQuery mirror of Cloud MS (mc-cms.cms_to_bigquery). No VPN, no SQL to write.
---

# Cloud MS (via Cloud Run MCP)

> **Skill version:** the authoritative version is the `version` field in
> `.claude-plugin/plugin.json` — cite that when asked "which version".

Answer questions about **Master Concept's Cloud MS** — the internal system that manages the
firm's Google Cloud / Workspace **reseller** operation — by calling **named tools** on a
private **Cloud Run MCP endpoint** (Google's MCP Toolbox over the read-only BigQuery mirror
`mc-cms.cms_to_bigquery`, a CDC copy of the MySQL source). **No VPN, no MySQL, no SQL to
write for the common cases.**

- **Endpoint:** `https://bq-mc-cms-586459078049.asia-east2.run.app/mcp/mc-cms`
- That path exposes **only the `mc_cms_*` tools**. Never call the root `/mcp`, and never call
  a tool whose name does not start with `mc_cms_`.
- Read-only; the whole dataset is ~22 MB, so every query is effectively free.

## Setup (once per machine)

- Install `gcloud`; your **own** Google account needs Cloud Run **`run.invoker`** on the
  `bq-mc-cms` service (project `gen-lang-client-0674348445`, region `asia-east2`) — ask a GCP
  admin if you get `HTTP 403`. No SA impersonation / dataset grant needed.
- Run this yourself — do **not** ask the user to log in manually:
  ```bash
  gcloud auth print-identity-token >/dev/null 2>&1 || gcloud auth login
  ```

## How to query

Mint a fresh ID token and POST a JSON-RPC `tools/call`:

```bash
URL=https://bq-mc-cms-586459078049.asia-east2.run.app/mcp/mc-cms
TOKEN=$(gcloud auth print-identity-token)
curl -s -X POST "$URL" -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call",
       "params":{"name":"mc_cms_account_status","arguments":{"group_by":"status"}}}'
```

Rows are in `result.content[].text` (each a JSON object); print with
`| python3 -c 'import sys,json;[print(c["text"]) for c in json.load(sys.stdin)["result"]["content"]]'`.
`tools/list` returns the catalog. `HTTP 403` → no `run.invoker`; `401` → `gcloud auth login`.

## Tools (9) — the `mc-cms` toolset

Single entity, so tables are fixed — you pass no project/entity argument. Read-only; the
conventions (soft-delete filtering, current-state rows) are baked into each tool.

| Tool | Key args | Returns |
|---|---|---|
| **`mc_cms_billing_accounts`** | `name?`, `status?` | billing accounts (display_name, status, master, customer_id, created) |
| **`mc_cms_account_status`** | `group_by`: status·reseller | account counts by status or by master reseller |
| **`mc_cms_customer_lookup`** | `name` (org or domain) | a customer + their billing accounts |
| **`mc_cms_virtual_customer`** | `name?` | virtual customers → #accounts / #projects / #repriced SKUs / #tiers |
| **`mc_cms_repricing`** | `virtual_customer`, `sku?` | a virtual customer's tiered SKU pricing (tier 0 = highest) |
| **`mc_cms_approvals`** | `approve_type?`, `status?`, `year?` | approval requests by type & status |
| **`mc_cms_team_roster`** | `team?` | internal teams → active member count + managers |
| **`mc_cms_audit`** | `event?`, `operator?`, `since?` | audit-log activity by event type |
| **`mc_cms_run_sql`** | `sql` | **free-form fallback** — read-only SELECT for anything no tool covers |

**Prefer a named tool.** Only when none fits, use `mc_cms_run_sql`: write a plain `SELECT`,
fully-qualifying tables as `` `mc-cms`.cms_to_bigquery.cloud_ms_<table> ``, and apply the
conventions yourself (see `references/schema.md`) — most importantly `WHERE is_deleted = 0`
on the tables that have it. It is SELECT-only.

## What this data links to (and does not)

Cloud MS is the **registry / management** side: which customer owns which billing account,
its status, approvals, custom repricing. It is **not** the cost or the sales side:

- **Actual Google spend** per billing account lives in the reseller **billing exports**
  (the `gcp-bq` skill), keyed by the same account ids.
- **Orders / invoices / gross profit** live in the **BMS** mirror (the `bms2` skill).
- No single system holds every billing account id — Cloud MS covers the accounts opened
  through it, which is a large but not complete subset. Don't claim it is exhaustive.

## Local conventions worth knowing

- **Soft delete:** several tables carry `is_deleted` → filter `= 0` (the curated tools do).
- **Current-state CDC:** one row per entity; a `version` column is just the CDC revision.
- **Billing account id** is the `name` value with the `billingAccounts/` prefix stripped
  (format `XXXXXX-XXXXXX-XXXXXX`); `master_billing_account_name` is the reseller master.
- **Account status:** ACTIVE / CANCELLED / UNASSIGNED / SUSPENDED.

## Reports & charts

**Default output is a Markdown table** — return it directly. Build a visualization only when
asked ("chart / graph / dashboard"): render a self-contained **Artifact** (load the
`artifact-design` skill first), computed as **static HTML/CSS + inline `<svg>`** (the Artifact
CSP blocks inline `<script>`, so a JS-built page renders blank). Reach for horizontal bars for
rankings (accounts per customer, members per team) and KPI tiles for headline counts.

## Access control

Anyone with `run.invoker` on `bq-mc-cms` can call every `mc_cms_*` tool; project owners can
too. `run.invoker` is granted per **service**, so this is scoped to `bq-mc-cms` alone — but it
is not a per-tool or per-row boundary. Treat this as internal management data.

## Maintainer

The tools are **not** defined in this repo. They live in the `mc-cms` toolset of the
`bq-mc-cms` Cloud Run deployment, which the platform owner maintains in a private repo — ask
them to add or change a tool. This repo carries only the skill.

## Rules

1. **Read-only.** SELECT/WITH only.
2. Call **`mc_cms_*` tools** on `/mcp/mc-cms` only — never the root `/mcp`, never another prefix.
3. Trust the tools' baked-in conventions (`is_deleted=0`, current-state) — don't re-derive.
4. Report returned numbers; don't claim Cloud MS holds every billing account id (it doesn't).
5. Mint a fresh ID token per call; never ask the user to paste one.
