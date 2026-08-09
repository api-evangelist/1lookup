---
name: Enrich a company domain into firmographics and reachable contacts
description: >-
  Go from a company domain to a firmographic profile, then to named prospects, then reveal verified work
  contact details — spending the expensive 75-credit calls only on rows worth revealing.
api: https://app.1lookup.io/api/v1
operations:
  - POST /api/v1/company-firmographics
  - POST /api/v1/account-search
  - POST /api/v1/prospect-search
  - POST /api/v1/b2b-contact-append
  - GET /api/v1/account
mcp_tools: []
generated: '2026-08-09'
method: generated
source: https://app.1lookup.io/api
---

# Enrich a company and its contacts

Every operation below is printed in the 1Lookup API reference. **None of these are exposed through the
MCP connector** — the connector carries only `validate_phone`, `verify_email`, `ip_lookup`,
`bulk_verify` and `get_account`, so this flow is REST-only.

## Cost shape — read this first

This is the expensive corner of 1Lookup. Published credit rates:

| Operation | Credits |
|---|---|
| `POST /api/v1/prospect-search` | 2 per search |
| `POST /api/v1/company-firmographics` | 75 |
| `POST /api/v1/b2b-contact-append` | 75 per reveal |
| `POST /api/v1/account-search` | 75 (up to 100 companies) |

Firmographics, contact append and account search are documented as **not charged on a no-match**.
Prospect search returns preview rows with **masked surnames** and email-availability flags; the reveal
is a separate 75-credit `b2b-contact-append` call. Search wide, reveal narrow.

## Steps

1. **Check the balance.** `GET /api/v1/account`. At 75 credits a reveal, a careless loop is expensive.

2. **Either start from a known domain or find the accounts.**
   - Known domain → `POST /api/v1/company-firmographics`. Documented output: industry, SIC/NAICS,
     headcount, revenue, tech stack, growth signals.
   - Building a target list → `POST /api/v1/account-search`, which searches companies by headcount,
     revenue, location, technology and keywords and returns up to 100 companies per search with
     firmographics attached. One `account-search` at 75 credits beats 100 firmographics calls.
   - For professional-network context (HQ, founding year, specialties, funding history) use
     `POST /api/v1/company-profile-lookup` (25 credits, free on no match).

3. **Find the people.** `POST /api/v1/prospect-search` — search by title, seniority, location and
   employer. 2 credits per search. Use the returned email-availability flag to decide which rows are
   worth revealing; rows with no email available will not become reachable contacts.

4. **Reveal only the rows you will act on.** `POST /api/v1/b2b-contact-append` — accepts an email, a
   LinkedIn URL, or a name plus company, and returns verified work email, title, seniority and
   department. 75 credits per reveal, free on a no-match.
   - If you only need a mobile number from a professional profile URL or work/personal email, use
     `POST /api/v1/mobile-finder` (40 credits, success-billed) instead.
   - If you have a name and a company domain and only want the work email,
     `POST /api/v1/email-enrichment` is 5 credits.

5. **Validate before you use it.** A revealed email is still worth a 1-credit
   `POST /api/v1/email` check before it enters a sending system, and a revealed phone a 1-credit
   `POST /api/v1/phone` check for line type and DNC status. See
   `skills/1lookup-validate-contact-before-outreach.md`.

## Rules the agent must follow

- **Escalate cost deliberately.** Never call `b2b-contact-append` in a loop over an unfiltered search
  result. Filter on the preview flags first.
- **No idempotency key.** A retried reveal can bill twice. The 7-day cache is the only replay
  protection; do not set `bypass_cache` on a retry.
- **PII discipline.** These operations return personal data. 1Lookup states it deletes lookup data after
  30 days; your own retention is your responsibility, and GDPR/CCPA obligations follow the data out of
  the API.
- **402 means stop.** Insufficient credits mid-enrichment leaves a partially enriched list; record what
  succeeded before retrying.

## Cross-references

- `plans/1lookup-plans-pricing.yml` — the full per-product credit table and success-based billing list
- `mcp/1lookup-tool-crosswalk.yml` — why none of this is reachable from an MCP client
- `conventions/1lookup-conventions.yml` — caching, metering and error semantics
