---
name: Clean a whole contact list with a bulk job
description: >-
  Submit an asynchronous 1Lookup bulk job, poll it correctly without burning rate budget, and page the
  results — instead of looping single lookups.
api: https://app.1lookup.io/api/v1
operations:
  - POST /api/v1/bulk/jobs
  - GET /api/v1/bulk/jobs/{job_id}
  - GET /api/v1/bulk/jobs/{job_id}/results
  - GET /api/v1/account
mcp_tools: [bulk_verify, get_account]
generated: '2026-08-09'
method: generated
source: https://app.1lookup.io/api
---

# Bulk list hygiene

Grounded in the bulk endpoints printed in the 1Lookup API reference. No request-body field names are
published for job creation, so this skill describes the lifecycle, not an invented payload schema.

## When to use bulk

The reference states bulk jobs support **HLR, MNP, Number Type, Email Validation and Reverse Email
Append**. Anything outside that list has to go through the single-lookup POST endpoints. Looping single
lookups against a large list will hit the 1,000 req/min organization limit; a bulk job will not.

## Steps

1. **Budget the run.** `GET /api/v1/account` for the credit balance. Multiply rows by the product's
   credit rate (core validations 1 credit; HLR 5; MNP 3) before submitting. A run that outruns the
   balance fails with `402` partway through.

2. **Create the job.** `POST /api/v1/bulk/jobs`. It returns a job ID immediately — it does not block.
   (MCP clients call `bulk_verify`, which composes this lifecycle server-side and is billed 1 credit
   per record.)

3. **Poll status.** `GET /api/v1/bulk/jobs/{job_id}`. The reference is explicit that **status and result
   polling do not consume lookup-rate budget**, so polling is safe — but still poll on an interval, not
   in a tight loop. A completed job includes a **signed CSV download URL**.

4. **Take the results one of two ways.**
   - Download the signed CSV from the completed status response. **It expires after 1 hour** — fetch it
     promptly or re-read the job status to get a fresh link.
   - Or page the persisted rows: `GET /api/v1/bulk/jobs/{job_id}/results`. This endpoint is documented
     as paginated; the page parameter names are not published, so read them off the first response
     rather than assuming `page`/`limit`.

5. **Reconcile.** Rows the provider could not match are, for the success-billed products, not charged.
   Compare the returned row count against the submitted count before treating the list as clean.

## Rules the agent must follow

- **No webhooks.** 1Lookup publishes no callback or event surface — job completion is discovered by
  polling only. Do not wait on a webhook that does not exist.
- **No idempotency key.** Re-submitting a job after a timeout can create a second billable job. Record
  the returned job ID before retrying anything.
- **Cache interaction.** Identical lookups are served from the 7-day cache, so re-running a list you
  already cleaned inside the window is cheap. Do not set `bypass_cache` unless you specifically need
  fresh data.
- **Errors:** `{"success": false, "error": {"message","code","type"}}`. On `402` stop the run.

## Cross-references

- `conventions/1lookup-conventions.yml` — the job-and-poll pattern and rate-limit budget carve-out
- `plans/1lookup-plans-pricing.yml` — per-product credit rates for cost estimation
- `rate-limits/1lookup-rate-limits.yml` — 1,000 req/min organization limit and headers
