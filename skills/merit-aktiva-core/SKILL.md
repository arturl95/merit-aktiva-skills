---
name: merit-aktiva-core
description: Authenticate and call the Merit Aktiva REST API — HMAC-SHA256 query-string signing, request conventions, rate limits, error handling. Use before any other Merit skill in this plugin.
last-verified: 2026-05-12
---

# merit-aktiva-core

The foundation skill for every Merit Aktiva interaction. Load this whenever you are about to talk to `aktiva.merit.ee/api/*`.

## When to use

- Before any other `merit-aktiva-*` skill in this plugin runs its first API call.
- Any time you see a 401, 429, or 200-with-body-that-looks-like-an-error.
- When the user asks about Merit auth, signing, or rate limits.

## Credentials

Read from environment:

- `MERIT_API_ID` — GUID from Merit UI → Settings → API Settings.
- `MERIT_API_KEY` — the HMAC secret.

If either is unset, **stop and ask the user**. Never hard-code, never write to a file, never echo into shell history.

**Before any first write call in a session**, confirm the connected company is a trial company unless the user has already said "production". Merit has no separate sandbox; the trial company shares the same hostname.

## Base URLs

- EE v1: `https://aktiva.merit.ee/api/v1/`
- EE v2: `https://aktiva.merit.ee/api/v2/`

Use v2 by default. The following endpoints live only on v1: `gettaxes`, `getaccounts`, `getcustomers`, `getvendors`, `getitems`. Mixing v1 and v2 in one integration is fine.

## Signing recipe (4 lines)

```
timestamp  = UTC now formatted yyyyMMddHHmmss
dataToSign = utf8(apiId + timestamp + httpBody)
signature  = base64(hmac_sha256(apiKey, dataToSign))
url        = base + endpoint + "?ApiId=" + apiId + "&timestamp=" + ts + "&signature=" + urlencode(signature)
```

Critical:

- **HMAC-SHA256**, not plain SHA-256.
- Concatenation: `apiId + timestamp + body`. No separators. Empty body = empty string.
- Timestamp is **UTC**, not Tallinn local.
- Standard base64 alphabet (`+ / =`), then **URL-encode** before appending — `+` becomes a space otherwise.
- All three params (`ApiId`, `timestamp`, `signature`) live in the **query string**, never in headers.
- Sign the exact bytes you POST. Re-serializing later (whitespace, key order, unicode escapes) breaks the signature.

See [authentication.md](references/authentication.md) for a worked example, cross-library notes, and a curl one-liner.

## Request envelope

- HTTP `POST` (even for read-shaped endpoints with a filter body).
- `Content-Type: application/json; charset=utf-8`.
- Body: JSON. For endpoints that take no payload, send `{}` (not empty string).

## The 6 errors you will hit

| Status | Cause | What to do |
|---|---|---|
| 200 + body containing `at System.` or `Microsoft.AspNet` | Logical/validation error slipped past 400 check | Treat as 422. Parse the first line of the trace — it usually names the missing field. |
| 400 | Malformed payload; invalid ApiId; whitespace in credentials | Validate payload locally against the schema in the relevant skill; check for accidental whitespace in env vars. |
| 401 (no body) | Wrong ApiId | Re-check env vars; re-check signing (timestamp UTC, base64 + URL-encode). |
| 401 + body `api-wronglicense` | Company is on Free/Standard | Not retryable. Tell the user the company must upgrade to Pro/Premium. |
| 404 | Wrong path or missing required query params | Confirm v1 vs v2 path; confirm all three of ApiId/timestamp/signature are in the URL. |
| 429 | Rate limited | Sleep `Retry-After` seconds, then exponential backoff. See `conventions.md`. |

See [error-playbook.md](references/error-playbook.md) for the full taxonomy and the 14 known gotchas with recovery patterns.

## Rate limits

100 requests per minute per `ApiId`. On 429, response headers include `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (unix), `Retry-After` (seconds). Honor `Retry-After`.

## When to read each reference

- [authentication.md](references/authentication.md) — before the first call in a fresh integration, or on any persistent 401.
- [conventions.md](references/conventions.md) — when serializing decimals/dates, implementing pagination, or sending attachments.
- [error-playbook.md](references/error-playbook.md) — on any unexpected response.

## Guardrails

- All write operations (any endpoint starting with `send*` or `delete*`) require **explicit user confirmation**: show the full payload, summarise the effect (e.g., "Will create invoice 2026-0123 for €1,220.40 including 24% VAT, charged to customer Acme OÜ"), and wait for an explicit "yes" / "go" before posting.
- For batch operations, show a summary table of every planned write and require a single explicit confirmation, or per-row review if the user asks for it.
- Never invent values — TaxId GUIDs come from `gettaxes`, account codes from `getaccounts`, customer/vendor/item identifiers from the relevant `get*` endpoints.

## Cross-references

- For master data (customers, vendors, items, taxes, accounts): `merit-aktiva-masters`.
- For sales invoicing and e-invoicing: `merit-aktiva-sales`.
- For purchase invoices, payments, bank statement import: `merit-aktiva-purchases-payments`.
- For GL journals and reports: `merit-aktiva-ledger-reports`.
- For Estonian VAT codes, account mapping, tax rules: `estonian-bookkeeping`.
