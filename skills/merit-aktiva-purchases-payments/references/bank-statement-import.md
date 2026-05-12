# Bank statement import (camt.053)

Merit Aktiva imports bank statements in **ISO 20022 camt.053** XML — the format every Estonian bank has used since 2018. The endpoint is `sendbankstatement`; Merit then auto-matches lines to open invoices.

## Where to get the file

| Bank | Path |
|---|---|
| Swedbank | Internetbank → Statements → Format: ISO XML (camt.053) |
| SEB | Online banking → Account statements → ISO XML (camt.053) |
| LHV | Online banking → Statements → ISO XML (camt.053) |
| Luminor | Online banking → Statements → ISO 20022 format |

For high-volume needs, all four banks offer **automated daily push** via FIDAVISTA or the EU PSD2 AISP API to a configured endpoint — outside the scope of this skill but worth wiring up.

## The endpoint

```json
POST /api/v2/sendbankstatement
{
  "BankAccountIBAN": "EE382200221020145685",
  "FileContent": "<base64-encoded-camt.053-xml>",
  "FileName": "stmt-2026-05-12.xml"
}
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `BankAccountIBAN` | string | yes | The IBAN whose statement is being imported. Must match an account in Merit (Settings → Banks). |
| `FileContent` | base64 string | yes | The entire XML file, base64-encoded. |
| `FileName` | string | recommended | For audit/diagnostic. |

Response: `{ "ImportId": "...", "LinesTotal": 27, "LinesMatched": 22, "LinesUnmatched": 5 }`.

## Auto-matching logic

Merit attempts to match each statement line to an open invoice in this order:

1. **`RefNo` (viitenumber)** — the Estonian reference-number standard. If the bank line carries a viitenumber matching an outstanding invoice's `RefNo`, it auto-matches and posts a `sendpayment` / `sendpurchasepayment` automatically. This is the highest-confidence path.
2. **Counterparty IBAN + amount** — if the line has the customer/vendor's known IBAN and the amount matches an outstanding invoice within ±€0.05, it matches.
3. **Amount + date proximity** — last resort; the line is suggested but flagged as needing manual confirmation.

Lines with no match end up as **"to reconcile"** entries in the Merit UI (Bank → Reconcile). The API does not currently expose these as a structured "unmatched" list; poll the bank-import history via `getpaymentimports`:

```json
POST /api/v2/getpaymentimports
{ "PeriodStart": "20260501", "PeriodEnd": "20260512" }
```

## Workflow for an AI agent reconciler

A typical agent flow:

```
1. Download camt.053 from the bank (or accept a file path from the user).
2. Validate it parses as XML and contains at least one BkToCstmrStmt.
3. Base64-encode the file content.
4. POST /api/v2/sendbankstatement.
5. Inspect ImportId, LinesMatched, LinesUnmatched.
6. If unmatched > 0:
     a. Fetch the import via getpaymentimports.
     b. For each unmatched line, propose a match based on:
        - Counterparty name fuzzy-matched against getcustomers / getvendors.
        - Amount + date matched against getinvoices (open) and getpurchaseinvoices (open).
     c. Show proposals to the user as a table.
     d. On user confirmation, post sendpayment / sendpurchasepayment for each accepted match.
7. Report final state.
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

## Encoding the file

```python
import base64
with open("stmt-2026-05-12.xml", "rb") as f:
    content_b64 = base64.b64encode(f.read()).decode("ascii")
```

Then include in the request body:

```python
{
    "BankAccountIBAN": "EE382200221020145685",
    "FileContent": content_b64,
    "FileName": "stmt-2026-05-12.xml"
}
```

For shell, use `base64 < stmt-2026-05-12.xml | tr -d '\n'`.

## Common pitfalls

1. **Wrong IBAN** — the `BankAccountIBAN` must exist in Merit's Banks config; otherwise the import is rejected.
2. **Mixed periods** — Merit expects the file's date range to align with its expected import cadence. Importing a 6-month-old statement may re-create payments that are already in the ledger; check for duplicates.
3. **Non-camt.053 formats** — Merit also accepts ISO 20022 camt.054 (debit/credit notification) and the legacy MT940 format, but support varies. Stick to camt.053 for reliability.
4. **File too large** — daily statements are tiny; monthly statements rarely exceed 200 KB after base64 encoding. If you're pushing 5 MB+, you're probably importing a wrong file.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/payments/ (bank-statement subsection)
- ISO 20022 camt.053 — https://www.iso20022.org/iso-20022-message-definitions
