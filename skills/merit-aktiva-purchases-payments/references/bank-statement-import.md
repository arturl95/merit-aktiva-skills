# Bank statement import (camt.053)

Merit Aktiva imports bank statements in **ISO 20022 camt.053** XML — the format every Estonian bank has used since 2018. The endpoint is `sendcamt53`; Merit then auto-matches lines to open invoices.

## Where to get the file

| Bank | Path |
|---|---|
| Swedbank | Internetbank → Statements → Format: ISO XML (camt.053) |
| SEB | Online banking → Account statements → ISO XML (camt.053) |
| LHV | Online banking → Statements → ISO XML (camt.053) |
| Luminor | Online banking → Statements → ISO 20022 format |

For high-volume needs, all four banks offer **automated daily push** via FIDAVISTA or the EU PSD2 AISP API to a configured endpoint — outside the scope of this skill but worth wiring up.

## The endpoint

```
POST /api/v2/sendcamt53
Content-Type: application/xml (or text/xml)

<?xml version="1.0" encoding="UTF-8"?>
<Document xmlns="urn:iso:std:iso:20022:tech:xsd:camt.053.001.02">
  ...camt.053 XML content...
</Document>
```

The request body is the **raw XML file** — not a JSON wrapper. Supported schema versions: `camt.053.001.02` and `camt.053.001.10`.

> **Important:** The endpoint name is `sendcamt53`, not `sendbankstatement`. The body is raw XML, not a JSON object with a `FileContent` base64 field. The Merit auth signature is still appended to the URL as query parameters (same as all other endpoints).

Response: a plain-text Estonian message such as `"Imporditi 22 makserida (ridu kokku 27)"` (imported 22 payment rows, 27 total). There is no structured JSON response with `ImportId`, `LinesTotal`, etc.

## Auto-matching logic

Merit attempts to match each statement line to an open invoice in this order:

1. **`RefNo` (viitenumber)** — the Estonian reference-number standard. If the bank line carries a viitenumber matching an outstanding invoice's `RefNo`, it auto-matches and posts a `sendpayment` / `sendPaymentV` automatically. This is the highest-confidence path.
2. **Counterparty IBAN + amount** — if the line has the customer/vendor's known IBAN and the amount matches an outstanding invoice within ±€0.05, it matches.
3. **Amount + date proximity** — last resort; the line is suggested but flagged as needing manual confirmation.

Lines with no match end up as **"to reconcile"** entries in the Merit UI (Bank → Reconcile). The API does not expose a structured unmatched-lines list. Use `getpayments` with `DateType: 1` (ChangedDate) to poll for newly-created payment records after an import.

## Workflow for an AI agent reconciler

A typical agent flow:

```
1. Download camt.053 from the bank (or accept a file path from the user).
2. Validate it parses as XML and contains at least one BkToCstmrStmt.
3. POST /api/v2/sendcamt53 with the raw XML as the request body.
4. Parse the plain-text response to get imported count vs total count.
5. If unmatched lines exist (total > imported):
     a. In the Merit UI (Bank → Reconcile), unmatched lines appear for manual review.
        The API does not expose a structured unmatched-lines list.
     b. For each unmatched line, propose a match based on:
        - Counterparty name fuzzy-matched against getcustomers / getvendors.
        - Amount + date matched against getinvoices (open) and getpurchorders (open).
     c. Show proposals to the user as a table.
     d. On user confirmation, post sendpayment / sendPaymentV for each accepted match.
6. Report final state.
```

## XML structure crash course

camt.053 is a structured bank statement. The relevant nodes for matching:

```
<BkToCstmrStmt>
  <Stmt>
    <Acct>
      <Id><IBAN>EE38...</IBAN></Id>
    </Acct>
    <Bal>...</Bal>
    <Ntry>                    <!-- one per transaction -->
      <Amt Ccy="EUR">123.45</Amt>
      <CdtDbtInd>CRDT</CdtDbtInd>     <!-- CRDT incoming, DBIT outgoing -->
      <BookgDt><Dt>2026-05-12</Dt></BookgDt>
      <NtryDtls>
        <TxDtls>
          <RltdPties>
            <Dbtr><Nm>Acme OÜ</Nm></Dbtr>
            <DbtrAcct><Id><IBAN>EE49...</IBAN></Id></DbtrAcct>
          </RltdPties>
          <RmtInf>
            <Strd>
              <CdtrRefInf><Ref>20260042</Ref></CdtrRefInf>  <!-- the viitenumber -->
            </Strd>
          </RmtInf>
        </TxDtls>
      </NtryDtls>
    </Ntry>
  </Stmt>
</BkToCstmrStmt>
```

The `<CdtrRefInf>/<Ref>` element is the viitenumber that drives Merit's primary auto-match. If your customer-facing invoices include the viitenumber on the PDF (Merit does this automatically), >90% of inbound payments match without manual intervention.

## Sending the file

The XML is posted as a raw body — no base64 encoding or JSON wrapper needed:

```python
import requests, hashlib, hmac, time, json

# Build the Merit auth params (same pattern as all Merit API calls)
# ... see merit-aktiva-core for the full signature construction ...

with open("stmt-2026-05-12.xml", "rb") as f:
    xml_bytes = f.read()

response = requests.post(
    f"https://aktiva.merit.ee/api/v2/sendcamt53?ApiId=...&timestamp=...&signature=...",
    data=xml_bytes,
    headers={"Content-Type": "application/xml"}
)
print(response.text)  # "Imporditi 22 makserida (ridu kokku 27)"
```

For shell:

```bash
curl -X POST \
  "https://aktiva.merit.ee/api/v2/sendcamt53?ApiId=...&timestamp=...&signature=..." \
  --data-binary @stmt-2026-05-12.xml \
  -H "Content-Type: application/xml"
```

## Common pitfalls

1. **Wrong endpoint name** — use `sendcamt53`, not `sendbankstatement`. The latter does not exist.
2. **JSON body instead of raw XML** — the body must be raw XML, not a JSON object with a `FileContent` base64 field. Wrong content type or body format results in rejection.
3. **Mixed periods** — Merit expects the file's date range to align with its expected import cadence. Importing a 6-month-old statement may re-create payments that are already in the ledger; check for duplicates.
4. **Non-camt.053 formats** — Merit also accepts ISO 20022 camt.053.001.10 (newer schema). Stick to camt.053.001.02 or camt.053.001.10 for reliability; other formats are not guaranteed.
5. **File too large** — daily statements are tiny; monthly statements are small. If you're pushing megabytes, you're probably sending the wrong file.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/bank-statement/
- ISO 20022 camt.053 — https://www.iso20022.org/iso-20022-message-definitions
