# Authentication — full reference

Verified against [api.merit.ee/connecting-robots/reference-manual/authentication/](https://api.merit.ee/connecting-robots/reference-manual/authentication/) and cross-checked with four OSS client libraries.

## The recipe

```
dataToSign     := utf8_bytes( apiId + timestamp + httpBody )
hmacKey        := ascii_bytes( apiKey )
signatureBytes := hmac_sha256( hmacKey, dataToSign )
signature      := base64( signatureBytes )         // standard alphabet
url            := base + endpoint
                  + "?apiId="     + apiId
                  + "&timestamp=" + timestamp
                  + "&signature=" + url_encode(signature)
```

Rules:

- **HMAC-SHA256**, not plain SHA-256. The API Key is the HMAC secret.
- Concatenation order: `apiId + timestamp + body`. No separators. For empty payloads (e.g. `gettaxes`), the body is `{}` and is included literally in the signed string.
- `timestamp` format: `yyyyMMddHHmmss` (14 digits).
- **UTC** — not Tallinn local. Older docs and a few libraries say Tallinn; the current official spec is UTC. Requests too old or in the future are rejected (~5 min skew tolerance).
- Standard base64 (`+ / =`), then **URL-encode** before placing in the query string. `+` becomes a space otherwise.
- Place all three of `apiId`, `timestamp`, `signature` in the query string. Never in headers.
- Sign the exact bytes you will POST. If you `JSON.stringify` and sign, send those same bytes — pretty-printing or re-ordering keys after signing desyncs the signature.

## Worked example

```
ApiId      = 670fe52f-558a-4be8-ade0-526e01a106d0
ApiKey     = secret-key-here
timestamp  = 20240624205902     # UTC
body       = {}
dataToSign = "670fe52f-558a-4be8-ade0-526e01a106d020240624205902{}"
hmacKey    = "secret-key-here"  # ASCII bytes
signature  = base64(hmac_sha256(hmacKey, dataToSign))
```

Then URL-encode the signature before appending to the query string.

Sample URL from the official docs:

```
POST /api/v1/getcustdebtrep?apiId=670fe52f-558a-4be8-ade0-526e01a106d0&timestamp=20240624205902&signature=dt6dkfuj%2BOfX01YkvvAoN%2FfekAUGr6AvVlQhUUja9Qc%3D
```

Note `%2B` (`+`), `%2F` (`/`), `%3D` (`=`) — confirms standard base64 + URL-encoding.

Merit does not publish a canonical fixture (apiId, key, timestamp, body → signature). Validate against a live response on a trial company.

## curl one-liner

```bash
#!/usr/bin/env bash
set -euo pipefail

ENDPOINT="${1:-gettaxes}"
BASE="https://aktiva.merit.ee/api/v1"
BODY='{}'

TS=$(date -u +%Y%m%d%H%M%S)
DATA="${MERIT_API_ID}${TS}${BODY}"

# macOS: openssl dgst supports -mac hmac
SIG=$(printf '%s' "$DATA" \
  | openssl dgst -sha256 -mac HMAC -macopt "key:${MERIT_API_KEY}" -binary \
  | base64)

# URL-encode the signature
SIG_ENC=$(printf '%s' "$SIG" \
  | python3 -c "import sys, urllib.parse; print(urllib.parse.quote(sys.stdin.read(), safe=''))")

curl -sS -X POST \
  -H "Content-Type: application/json; charset=utf-8" \
  --data "$BODY" \
  "${BASE}/${ENDPOINT}?apiId=${MERIT_API_ID}&timestamp=${TS}&signature=${SIG_ENC}"
```

This works for any POST-shaped endpoint. For endpoints with a real body, replace `BODY='{}'` with the JSON to send — the same string must be both signed and POSTed.

## Cross-library notes

Four OSS clients implement this; here is what they got right and wrong.

| Library | Hash | Encoding | Notes |
|---|---|---|---|
| `jgangso/merit-aktiva-api-client` (PHP) | HMAC-SHA256 raw bytes | base64 (standard) + URL-encode | ✅ Canonical. |
| `KasparKipp/merit-aktiva` (TS/Node) | HMAC-SHA256 raw bytes | base64 (standard) + URL-encode | ✅ Canonical. |
| `akosmarton/merit-aktiva-api-go` (Go) | HMAC-SHA256 raw bytes | **`base64.URLEncoding`** | Works because URL-safe alphabet (`-`/`_`) is already URL-safe — never produces `+` or `/`. Non-canonical; could break if Merit tightens validation. |
| `infira/MeritAktivaAPI` (PHP) | HMAC-SHA256 returning **hex** (`raw=false`), then base64'd | base64 of hex string | ❌ Bug: signature is twice the length of canonical. May still be accepted by lenient server, do not copy. |

## Common pitfalls

1. **URL-encoding** — without it, `+` in the signature becomes a space when the server parses the query string, breaking validation.
2. **Hex vs raw bytes** — `hash_hmac()` in PHP without `$raw_output=true`, or `digest('hex')` in Node, gives hex; base64 of hex is wrong.
3. **URL-safe vs standard base64** — the spec mandates standard. URL-safe happens to work because it never produces problematic characters, but it is non-canonical.
4. **Timestamp clock skew** — sync to NTP. >5 minutes off is rejected.
5. **Body bytes mismatch** — sign the exact bytes you POST. If you `json_encode` once and sign that string, send that same string.
6. **Whitespace in credentials** — copy/paste of `ApiId` often includes a trailing space. Strip on read.
7. **Wrong base URL version** — `gettaxes` etc. are v1; `sendinvoice` etc. are v2. A 404 with a correctly-signed request usually means the wrong version.

## Source

- https://api.merit.ee/connecting-robots/reference-manual/authentication/
- https://github.com/akosmarton/merit-aktiva-api-go — `aktiva/http.go`
- https://github.com/jgangso/merit-aktiva-api-client — `src/Util/MeritApiClient.php`
- https://github.com/KasparKipp/merit-aktiva — `src/aktiva/authentication/getSignPayload.ts`
- https://github.com/infira/MeritAktivaAPI — `src/API.php` (suspected base64-of-hex bug)
