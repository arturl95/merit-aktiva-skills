# Wire-format conventions

## HTTP

- All endpoints accept `POST` with a JSON body (and require it, even for endpoints that take no payload — send `{}`).
- `Content-Type: application/json; charset=utf-8`.
- All authentication is in the query string (see [authentication.md](authentication.md)).

## Dates and timestamps

| Field | Format | Timezone |
|---|---|---|
| Body date fields (DocDate, DueDate, BatchDate, …) | `YYYYMMDD` (compact 8-digit) | Business date, local |
| Body date-time fields (PaymDate occasionally) | `YYYYMMDDHHmm` | Local |
| **Signing** `timestamp` query param | `yyyyMMddHHmmss` (14 digits) | **UTC** |
| Response date fields (since 2024-02-08) | ISO-like `YYYY-MM-DDTHH:mm:ss` | Local |

The signing timestamp is the only one that must be UTC. Body dates are local business dates.

## Decimals

- Use `.` as the separator, locale-neutral.
- Send as JSON numbers, not strings.
- Documented precisions per schemas:
  - Amounts: 18.2
  - Quantities: 18.3
  - Prices: 18.7
  - Currency / VAT rates: 18.7
- Percentages are whole numbers (`5` = 5%).
- **Round each invoice row to 2 decimals before summing** `TaxAmount` and `TotalAmount`. The server validates row totals against the declared sums.

## Booleans, nulls, strings

- Booleans: lowercase `true` / `false`.
- Null: lowercase `null`.
- Strings: UTF-8. No length checks beyond the per-field documented maxima.
- Languages used in language-aware fields (`SalesInvLang`, `NameEN`/`NameRU`/`NameFI`/`NameET`, etc.): `ET`, `EN`, `FI`, `PL`, `RU`.

## Currency

- Default is `EUR` (EE/FI) or `PLN` (PL).
- For non-default currency on v2 endpoints, `CurrencyRate` defaults to the ECB rate of the document date if you omit it. v1 requires explicit `CurrencyRate`.

## Rate limits

- 100 requests per minute per `ApiId`.
- On 429 the response carries:
  - `X-RateLimit-Limit`
  - `X-RateLimit-Remaining`
  - `X-RateLimit-Reset` (unix timestamp)
  - `Retry-After` (seconds)
- Pattern: honor `Retry-After`, then back off exponentially if 429 returns again.

## Payload size

- **Maximum 500 rows per submission** for orders/invoices/offers/GL transactions. Split larger batches.
- Attachments (PDF on sales invoice, files on GL batch and purchase invoice) are sent as inline base64 strings. No documented hard size cap; keep payloads under a few MB.

## Pagination / incremental sync

There is no cursor pagination. Two patterns:

1. **Date-window filter** — pass `PeriodStart`/`PeriodEnd` (or equivalent date filter) and split big ranges into smaller windows.
2. **High-water mark** — store the maximum `UpdatedDate` (invoices, payments) or `ChangedDate` (customers, vendors, items) seen, and pass it back as the filter on the next poll. Narrow the window by 1 day to handle clock skew.

Large unfiltered list calls (e.g. `getcustomers` with no filter) can return ASP.NET stack traces — always filter.

## Concurrency

- The API does not lock for writes. The FAQ states this explicitly: "We never lock for writing to allow our partners to write faster."
- Consequence: the client must guarantee uniqueness of `InvoiceNo` and avoid races. There is **no endpoint to fetch the next invoice number** — keep a server-side counter or use a UUID-derived numbering scheme.

## Immutability

- Invoices have **no update endpoint** — delete and recreate.
- GL rows have **no delete endpoint** — reverse with a new batch.
- Payments have constrained deletion (complex GL relations).

## Response shape

- Older endpoints returned a JSON-encoded string (`"\"{...}\""`). Since 2024-02-08, most return plain JSON objects.
- Errors come as either `{ "Message": "..." }` or, occasionally, a raw ASP.NET stack trace as the body even with status 200. Always inspect the body shape, not just the status code.
