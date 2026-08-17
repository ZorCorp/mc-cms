# mc-cms — Cloud MS reseller management, read-only, via Cloud Run MCP

A Claude Code plugin (skill only — no MCP server, proxy or binary) that answers questions
about **Master Concept's Cloud MS**: the internal system that manages the firm's Google
Cloud / Workspace **reseller** operation — billing accounts, customers, virtual customers and
their custom tiered repricing, billing-account approvals, internal teams, and an audit log.

The agent signs a short-lived Google ID token with `gcloud` and `curl`s a private Cloud Run
endpoint that serves Google's MCP Toolbox over a read-only BigQuery mirror of the MySQL
source. No VPN, no database client, nothing to install beyond `gcloud`.

> 🔒 **Internal.** The endpoint is `--no-allow-unauthenticated`; you need Cloud Run
> `run.invoker` on `bq-mc-cms` to reach it. This repo carries no credentials and no data —
> only the skill.

## Install

From the marketplace, then log in once per machine:

```bash
gcloud auth login
```

If a call returns `HTTP 403`, ask a GCP admin for `run.invoker` on the `bq-mc-cms` service
(project `gen-lang-client-0674348445`, region `asia-east2`).

## What it talks to

| | |
|---|---|
| Endpoint | `https://bq-mc-cms-586459078049.asia-east2.run.app/mcp/mc-cms` |
| Toolset | `mc-cms` — 9 tools, all named `mc_cms_*` |
| Data | `mc-cms.cms_to_bigquery` (BigQuery mirror, read-only, ~22 MB) |
| Guards | SELECT only (`writeMode: blocked`), service account has `dataViewer` only |

## Contents

| File | Purpose |
|---|---|
| `SKILL.md` | The skill — endpoint, the 9 tools, conventions, report rules |
| `references/schema.md` | Table/join map, soft-delete & billing-id conventions, cross-system links |

## Where the tools are defined

Not here. They live in the `mc-cms` toolset of the `bq-mc-cms` Cloud Run deployment, which the
platform owner maintains in a private repo. Ask them to add or change a tool — this repo
carries only the skill.

## Related skills

Cloud MS is the **registry / management** side. Actual Google **cost** per billing account is
in the reseller billing exports (the `gcp-bq` skill); **orders / invoices / gross profit** are
in the BMS mirror (the `bms2` skill). The three link by billing-account id, but no single
system holds every id.
