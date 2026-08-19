---
name: Run a bulk validation job safely, with idempotent retries and a signed callback
description: >-
  Submit up to 100,000 rows to 1Lookup as one bulk job, make the submission safe to retry with an
  Idempotency-Key, then collect results either by polling or by verifying the signed webhook 1Lookup
  POSTs when the job finishes. Use this instead of looping single lookups above a few thousand rows.
api: https://app.1lookup.io/api/v1
operations:
  - 'POST /api/v1/bulk/jobs'
  - 'GET /api/v1/bulk/jobs/{job_id}'
  - 'GET /api/v1/bulk/jobs/{job_id}/results'
  - 'GET /api/v1/account'
mcp_tools: []
source: https://app.1lookup.io/api
generated: '2026-08-14'
method: generated
---

# Run a safe bulk job

1Lookup publishes no OpenAPI, so every path, header, field and status below is transcribed from the
provider's own reference at <https://app.1lookup.io/api>. Nothing here is invented; where the docs are
silent, this skill says so.

## Before you start

- Auth is a single header: `Authorization: Bearer sk_live_…`. Keys belong to the organization and carry
  its whole credit balance. A key on a free-plan organization returns **403 `UPGRADE_REQUIRED`**.
- Call `GET /api/v1/account` first. Read `data.tokens.total_available`. The whole job is priced before
  it starts, and if the balance cannot cover every row the job is refused with **402
  `INSUFFICIENT_CREDITS`** rather than stopping half way — so a pre-flight check is how you avoid a
  wasted round trip, not how you avoid a partial charge.
- Bulk is not available for every product. The supported `type` values are: `hlr_lookup`, `mnp_lookup`,
  `nt_lookup`, `email_validation`, `reverse_email_append`, `domain_authority`, `domain_age`,
  `backlink_overview`, `keyword_metrics`, `company_firmographics`, `b2b_contact_append`,
  `business_verify`, `business_lookup`, `company_profile_lookup`, `linkedin_profile_lookup`,
  `social_profile_check`, `property_lookup`, `social_post_lookup`, `video_transcript`,
  `audience_demographics`, `link_in_bio_lookup`. Search-style products are single-call by design and
  return **400 `INVALID_TYPE`** if you ask for them here.

## 1. Submit the job — always with an Idempotency-Key

```
POST /api/v1/bulk/jobs
Authorization: Bearer sk_live_…
Content-Type: application/json
Idempotency-Key: <a fresh UUID, one per job>

{ "type": "email_validation",
  "inputs": ["jane.doe@acme.com", "john.doe@example.com"],
  "webhook_url": "https://example.com/hooks/1lookup" }
```

- Maximum **100,000** rows per job; more returns **400 `INPUT_LIMIT_EXCEEDED`**.
- The key is remembered for **24 hours**. A retry with the same key returns the *original* job marked
  `Idempotent-Replay: true` instead of creating and charging for a second one. **Never mint a new key
  on retry** — that is how you pay twice.
- If the same key is still in flight you get **409 `IDEMPOTENCY_CONFLICT`**; wait and retry the *same*
  key.
- `webhook_url` is optional and must be HTTPS and publicly resolvable; private and loopback addresses
  are rejected.
- Response: `data.job_id`, `data.status` (`processing`), `data.accepted`.

## 2. Collect the results

**Polling** — free, and explicitly excluded from your lookup rate budget:

```
GET /api/v1/bulk/jobs/{job_id}
```

`status` moves `processing` → `completed`; `processed_count`, `successful_count` and `failed_count`
update as rows land. On completion `results_url` is a signed CSV link that **expires one hour after it
is issued** — fetch the job again for a fresh one rather than storing it.

Row-level results:

```
GET /api/v1/bulk/jobs/{job_id}/results?limit=1000&offset=0
```

`limit` defaults to 100, maximum 1000; page with `offset` until `data.pagination.has_more` is `false`.
Each row carries `id`, `input`, `status`, `tokens_consumed` and `response_data` (the same envelope the
single-lookup endpoint would have returned).

**Webhook** — one POST when the job finishes, signed as `X-1Lookup-Signature: sha256=<hex>`, an
HMAC-SHA256 of the **raw** body. Verify in constant time against the raw bytes *before* parsing.
Delivery is a single attempt with a 5-second timeout and no retries, and redirects are not followed, so
treat the callback exactly as the provider does: *a hint, not the record*. Return 2xx immediately, then
go and `GET` the job. A dropped delivery should cost you a few minutes of latency, never a batch.

## Error handling

Branch on `error.code`, never on `error.message` — codes are documented as stable, messages are not.
Retry **429, 500 and 503 only**, honouring `Retry-After` before falling back to exponential backoff;
every 4xx below 429 will fail again identically until the request changes. Log the `X-Request-Id`
header on every call — support asks for it.

See `errors/1lookup-problem-types.yml` for the full thirteen-code registry,
`conventions/1lookup-conventions.yml` for the idempotency and pagination contracts,
`asyncapi/1lookup-webhooks.yml` for the signature scheme, and
`rate-limits/1lookup-rate-limits.yml` for the four rate-limit tiers.

## When not to use this

Under a few thousand rows, single lookups with bounded concurrency (the docs' own example uses twenty
workers) are simpler. Above that, one bulk job is faster, cheaper to operate and immune to the rate
limits. From an MCP client, `bulk_verify` handles at most **50** phones/emails/IPs synchronously — it is
a different contract from this endpoint, and anything larger has to come back to REST.
