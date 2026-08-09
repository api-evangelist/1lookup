---
name: Validate a contact before you dial, text or email it
description: >-
  Check a phone number, email address and originating IP against 1Lookup before an agent acts on them,
  and stop on the signals that mean "do not contact".
api: https://app.1lookup.io/api/v1
operations:
  - POST /api/v1/phone
  - POST /api/v1/email
  - POST /api/v1/ip
  - GET /api/v1/account
mcp_tools: [validate_phone, verify_email, ip_lookup, get_account]
generated: '2026-08-09'
method: generated
source: https://app.1lookup.io/api
---

# Validate a contact before outreach

1Lookup has no OpenAPI document. Every path below is transcribed from the provider's API reference at
<https://app.1lookup.io/api>; request fields are only stated where the reference prints them.

## Auth

REST: `Authorization: Bearer YOUR_API_KEY` on every request, `Content-Type: application/json`.
Keys come from the dashboard at <https://app.1lookup.io>. Never put the key in client-side code.

MCP: connect `https://app.1lookup.io/api/mcp` and authorize with OAuth 2.1 (scope `lookup`). No key is
pasted; the client receives a scoped, revocable token.

## Steps

1. **Check the budget first.** `GET /api/v1/account` (MCP: `get_account`) returns organization details,
   subscription info, credit balances and API key usage. A 402 mid-run means the balance is exhausted,
   so read the balance before a batch of checks.

2. **Validate the phone.** `POST /api/v1/phone` (MCP: `validate_phone`) with body
   `{"phone_number": "+14155550132"}`. The documented result carries line type, carrier, country,
   reachability, DNC status and a fraud score. Costs 1 credit.
   - Do not dial or text when the number is not valid or not reachable.
   - Treat DNC status as a hard stop for marketing contact.
   - For live network reachability specifically, `POST /api/v1/hlr-lookup` (5 credits) is the deeper
     check; `POST /api/v1/phone-spam` scores spam/scam reputation and `POST /api/v1/phone-scrub`
     returns DNC status, line type and risk flags together.

3. **Verify the email.** `POST /api/v1/email` (MCP: `verify_email`) with body
   `{"email": "person@example.com"}`. Documented output covers syntax, domain/MX, deliverability, and
   disposable/role-based detection. Costs 1 credit. Do not send to an address that is undeliverable,
   disposable or role-based if sender reputation matters.

4. **Score the IP** when the contact arrived from a signup or session.
   `POST /api/v1/ip` (MCP: `ip_lookup`). Documented output covers geolocation plus VPN, proxy, Tor and
   datacenter detection. Costs 1 credit. A datacenter/hosting IP on a consumer signup is the standard
   fake-signup tell.

5. **Combine, don't chain blindly.** `POST /api/v1/fraud-detection` is not a documented endpoint — the
   fraud product is delivered as the `fraud_score` on the phone result plus the IP risk signals. Merge
   the three signals yourself.

## Rules the agent must follow

- **No idempotency contract.** 1Lookup documents no `Idempotency-Key`. A retried call can bill again.
  What protects you is the 7-day result cache: an identical lookup inside the window returns cached and
  is the cheap path. Never set `bypass_cache: true` on a retry — that forces a fresh, billable lookup.
- **Rate limit is 1,000 requests/minute per organization.** Read `X-RateLimit-Limit`,
  `X-RateLimit-Remaining` and `X-RateLimit-Reset` on every response and back off exponentially on 429.
- **Errors** come back as `{"success": false, "error": {"message", "code", "type"}}` — not RFC 9457.
  Handle `400` (fix the input), `401` (bad key), `402` (out of credits — stop the run, do not retry),
  `429` (back off), `500` (retry with backoff).
- **Every call is production.** There is no test mode and no test values; each call spends real credits.

## Cross-references

- `conventions/1lookup-conventions.yml` — caching, rate limiting, error envelope, metering
- `errors/1lookup-problem-types.yml` — status codes and the error envelope
- `authentication/1lookup-authentication.yml` — both auth models
- `mcp/1lookup-tool-crosswalk.yml` — which REST operation backs each MCP tool
