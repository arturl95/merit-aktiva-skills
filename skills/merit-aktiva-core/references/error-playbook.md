# Error playbook

Merit's HTTP status taxonomy is loose: a 200 can contain an ASP.NET stack trace if the server validated successfully but the business logic rejected. Always inspect the body shape.

**Rule of thumb:** any body containing `at System.` or `Microsoft.AspNet` is an error regardless of status code.

## Status taxonomy

| Status | Typical cause | Body | Retry? |
|---|---|---|---|
| 200 (OK) | Happy path | `{ ... }` | n/a |
| 200 (logical error) | Validation slipped through | ASP.NET stack trace | Fix payload; not retryable unchanged |
| 400 | Malformed payload, wrong ApiId (incl. whitespace), signature mismatch | `{ "Message": "..." }` | No, fix locally |
| 401 + `api-wronglicense` | Company on Free/Standard license | `api-wronglicense` | No — user must upgrade |
| 404 | Wrong URL, missing query params, wrong v1/v2 version | empty | Fix URL |
| 429 | Rate limit | (headers include `Retry-After`) | Yes, after wait |
| 5xx | Server side | ASP.NET trace | Yes, with backoff |

## Recovery patterns by status

### On 400
The official docs identify two 400 scenarios: (a) malformed payload and (b) wrong ApiId (including accidental whitespace from copy-paste). Check both:
1. Confirm `MERIT_API_ID` is set correctly and contains no leading/trailing whitespace.
2. Confirm `MERIT_API_KEY` is set correctly.
3. Confirm timestamp is UTC and within ~5 minutes of server time.
4. Confirm the signature was **URL-encoded** before being appended to the query string.
5. Confirm you signed the **exact same bytes** you POSTed.
6. Confirm the base64 alphabet is **standard** (`+ / =`), not URL-safe (`- _`).
7. Validate the payload locally against the relevant skill's schema.

### On 401 + `api-wronglicense`
Surface to the user immediately:
> The connected Merit Aktiva company is on a Free or Standard plan; the API requires a Pro or Premium plan. Ask the account admin to upgrade, then re-issue API credentials.

Not retryable in code.

Common payload causes of 400:

- Missing required field (e.g. `NotTDCustomer` on a new customer, `CountryCode` on a new customer, `TaxId` on an invoice row).
- Unknown `Code` for a master-data reference (item code, customer reg no, account code).
- Wrong number type (string instead of decimal).
- Date in the wrong format (ISO `2026-05-12` instead of `20260512` in a body field).

### On 404
- Confirm v1 vs v2 path. `gettaxes` is v1, `sendinvoice` is v2.
- Confirm all three of `apiId`, `timestamp`, `signature` are in the URL.
- Confirm the endpoint name spelling.

### On 429
Honor `Retry-After`:

```python
if resp.status_code == 429:
    wait = int(resp.headers.get("Retry-After", "60"))
    time.sleep(wait)
    # retry; on second 429, exponential backoff: wait *= 2
```

### On 200 with ASP.NET trace
- Treat as a 422.
- Parse the first line of the trace — it usually names the missing or invalid field.
- Common offenders: missing `VatAmount` when `TaxId` is set on a GL row, missing `ItemCostAmount` on a credit-invoice stock row, sum of row totals not matching declared `TaxAmount`/`TotalAmount` after rounding.

## The 14 documented gotchas

Drawn from the official FAQ and per-endpoint pages, with symptom and fix.

### Validation & math

1. **Row totals must be rounded before summing.** Symptom: ASP.NET trace mentioning a sum mismatch. Fix: round each row to 2dp, then build `TaxAmount` and `TotalAmount` from the rounded values.
2. **TaxId required on every invoice row.** Even 0% lines. Fix: call `gettaxes`, find the zero-rate code, set `TaxId` on every row.
3. **Different VAT rates require different item codes.** Symptom: KMD INF reconciliation fails. Fix: use distinct `Item.Code` per VAT rate.
4. **`ItemCostAmount` required on credit-invoice rows for stock items.** Symptom: misleading "internal error". Fix: set `ItemCostAmount` when posting credit invoices for `Type=1` (stock) items.
5. **`VatAmount` mandatory on GL row when `TaxId` is set.** Symptom: ASP.NET trace on `sendglbatch`. Fix: pair every GL row with `TaxId` with an explicit `VatAmount`.

### Auth & transport

6. **URL-encode the signature.** Symptom: 400 even though local HMAC matches. Fix: URL-encode after base64.
7. **Timestamp must be UTC, current.** Symptom: 400 with correct signing logic. Fix: NTP-sync, format as `yyyyMMddHHmmss` from `now_utc()`.
8. **License tier matters.** Symptom: 401 + `api-wronglicense`. Fix: not a client bug — user must upgrade plan.

### Immutability & uniqueness

9. **Cannot update invoices.** Symptom: no `updateinvoice` endpoint exists. Fix: `deleteinvoice` + `sendinvoice`.
10. **No "next invoice number" endpoint.** Symptom: looking for `getmaxinvoiceno`. Fix: keep your own counter; collisions are not auto-resolved (no write locking).
11. **No write-locking.** Symptom: occasional duplicate `InvoiceNo` 400s under concurrency. Fix: serialize writes per-tenant in the client, or use a UUID-prefixed numbering scheme.
12. **Cannot delete payments freely.** Symptom: 400 on `deletepayment`. Fix: reverse via `sendpayment` of opposing type, or contact support for the rare cases.

### Listing & pagination

13. **Unfiltered customer/vendor list may stack-trace.** Symptom: 200 with ASP.NET trace on `getcustomers` with empty filter. Fix: always filter by `ChangedDate`, `RegNo`, or `VatRegNo`; paginate by `ChangedDate` high-water mark.

### Response handling

14. **200 with stack trace bodies.** Symptom: parsing succeeds but data is missing. Fix: before parsing JSON, check for `at System.` / `Microsoft.AspNet` substrings and treat the response as a 422.

## Operational defaults

- Timeout for any single request: 30 seconds.
- Retry budget for transient errors (429, 5xx, network): 3 attempts with exponential backoff (1s, 2s, 4s).
- Never retry on 400 or 401 (except a single retry on 400 after fixing clock skew or credentials).
- Log the response body shape (status + first 200 chars) for every non-2xx response.
