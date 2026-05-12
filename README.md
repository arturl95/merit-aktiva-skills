# Merit Aktiva for AI Agents

> Plug **Merit Aktiva** — Estonia's most-used accounting software — into Claude, ChatGPT, Cursor and any AI agent. Bookkeeping, invoicing, VAT (KMD), and the full **Merit Aktiva API** in six Claude Code skills. Bilingual: **English · Eesti keeles**.

[![Plugin](https://img.shields.io/badge/Claude_Code-plugin-blue)](https://code.claude.com/docs/en/plugins.md)
[![Version](https://img.shields.io/badge/version-0.1.0-green)](https://github.com/arturl95/merit-aktiva-skills/releases)
[![License](https://img.shields.io/badge/license-MIT-lightgrey)](LICENSE)
[![Rates verified](https://img.shields.io/badge/rates_verified-2026--05--12-success)](skills/estonian-bookkeeping/SKILL.md)
[![Made in Estonia](https://img.shields.io/badge/made_in-Estonia%20🇪🇪-blue)](#merit-aktiva-ai-agentidele--eestikeelne-juhend)

**[Eesti keeles allpool ↓](#merit-aktiva-ai-agentidele--eestikeelne-juhend)**

---

## What is this?

**Merit Aktiva for AI Agents** is an open-source [Claude Code](https://code.claude.com) skills plugin that lets AI agents — Claude, ChatGPT, Cursor, or anything that can read instructions — operate [Merit Aktiva](https://www.merit.ee/), Estonia's #1 accounting software (majandustarkvara). It wraps the official [Merit Aktiva API](https://api.merit.ee/) with HMAC-SHA256 authentication, the complete v2 endpoint surface, and — critically — the **2026 Estonian tax rules** (käibemaks 24%, distributed-profit CIT, payroll, KMD codes). Use it to automate sales invoicing, purchase invoices, e-arve import, bank reconciliation, GL adjustments, and month-end VAT (KMD) reporting from any LLM.

This plugin does **not** ship code. It ships *instructions* — six skills your AI reads to learn the API, the gotchas, and the Estonian bookkeeping rules. Your agent talks to `api.merit.ee` directly over HTTPS.

## Quick start

```bash
# Install the plugin
/plugin install github.com/arturl95/merit-aktiva-skills

# Set credentials (from Merit Aktiva → Settings → API Settings)
export MERIT_API_ID="your-api-id-guid"
export MERIT_API_KEY="your-api-key-secret"
```

Then ask Claude:

> "List Acme OÜ's unpaid invoices this month."
> "Post a €1,000 + 24% VAT consulting invoice to Acme OÜ for May."
> "Import last week's Swedbank statement and reconcile."
> "Verify the April KMD reconciles before I file."

Claude loads the right skill(s), resolves customer/item/tax codes, builds the API payload, shows it to you for confirmation, and posts it on your "yes".

## Features

- **Sales invoicing** — `sendinvoice` v2 with all the rounding and TaxId rules baked in. Credit invoices, e-arve dispatch via Omniva / Telema / bank, PDF retrieval, email delivery.
- **Purchase invoicing** — supplier bills, expense claims, approval queue, reverse-charge VAT for intra-EU and non-EU transactions.
- **Payments & bank reconciliation** — payments, prepayments, settlements, **camt.053** bank statement import with auto-matching via viitenumber.
- **General ledger & reports** — `sendglbatch` for adjustments, P&L, balance sheet (`getfinpos`), AR aging, sales/purchase reports, VAT reconciliation.
- **Estonian tax compliance** — 2026 rates (VAT 24% / 13% / 9%, CIT 22/78, social tax 33%, II pillar 2/4/6%), KMD line mapping, fringe-benefit math, deadlines, board-member-fee specifics. All citations to [emta.ee](https://www.emta.ee/en).
- **HMAC-SHA256 request signing** — verified against the official spec and four open-source clients (Go, TS, two PHP). Worked example included.
- **Confirmation guardrails on every write** — show the payload, wait for "yes". Optional batch-confirm mode for repetitive jobs.

## Authentication — HMAC-SHA256

Every Merit Aktiva API call carries three query parameters: `ApiId`, `timestamp` (UTC, `yyyyMMddHHmmss`), and `signature`.

```
timestamp  = UTC now formatted yyyyMMddHHmmss
dataToSign = utf8(apiId + timestamp + httpBody)
signature  = base64(hmac_sha256(apiKey, dataToSign))   # standard alphabet
url        = base + endpoint + "?ApiId=" + apiId
             + "&timestamp=" + timestamp
             + "&signature=" + urlencode(signature)
```

Common gotchas (full list in [`skills/merit-aktiva-core/references/authentication.md`](skills/merit-aktiva-core/references/authentication.md)):

- **URL-encode** the signature before appending — `+` becomes a space otherwise.
- **UTC**, not Tallinn local. Sync to NTP.
- **Sign the exact bytes** you POST. Re-serializing after signing breaks the signature.
- **HMAC-SHA256**, not plain SHA-256. Standard base64 alphabet, not URL-safe.

See [API seadistus on support.merit.ee](https://support.merit.ee/et/articles/368840-api-seadistus) for getting your API ID and API Key.

## The six skills

| Skill | What it does |
|---|---|
| **`merit-aktiva-core`** | Authentication, conventions, error handling. Foundation skill. |
| **`merit-aktiva-masters`** | Customers, vendors, items, tax codes, chart of accounts. Resolves names to GUIDs. |
| **`merit-aktiva-sales`** | Sales invoices, credit notes, e-invoicing (Omniva/Telema/bank), PDFs. |
| **`merit-aktiva-purchases-payments`** | Purchase invoices, expense claims, payments, camt.053 bank import. |
| **`merit-aktiva-ledger-reports`** | GL journal entries, P&L, balance, customer-debt, sales/purchase, VAT reports. |
| **`estonian-bookkeeping`** | 2026 Estonian tax rules — VAT, CIT, payroll, KMD codes, fringe benefits. Standalone (no Merit dependency). |

Plus a bundled subagent **`merit-bookkeeper`** for batch jobs (a stack of receipts, monthly retainers, bank reconciliation).

## How it compares

| Approach | Pros | Cons | Best for |
|---|---|---|---|
| **`merit-aktiva-skills`** (this) | Up-to-date 2026 tax rules baked in; AI-native; works in any LLM that reads instructions; bilingual | Requires Claude Code or compatible AI host | AI-driven bookkeeping, agent workflows |
| **[`KasparKipp/merit-aktiva`](https://github.com/KasparKipp/merit-aktiva)** (Node/TS) | Strongly typed; modern stack | Manual coding; no tax knowledge | Node integrations |
| **[`akosmarton/merit-aktiva-api-go`](https://github.com/akosmarton/merit-aktiva-api-go)** (Go) | Performant | Manual coding; uses URL-safe base64 (non-canonical) | Go backends |
| **Raw HTTP** | Maximum control | You write every signature, every payload, every error path | Custom one-offs |

## Use cases

- **AI bookkeeper for an Estonian OÜ** — process supplier invoices from email, decide accounts and VAT codes, post on confirmation.
- **ChatGPT KMD assistant** — pull period invoices, build the KMD line totals, cross-check against GL, surface discrepancies before filing.
- **e-arve sync** — read incoming e-invoices, route to approval queue, notify the responsible employee.
- **Monthly recurring invoices** — list of customers × monthly retainer → 30 `sendinvoice` calls with one batch confirm.
- **Bank reconciliation co-pilot** — daily camt.053 import + interactive matching of unmatched lines.

## FAQ

### What is the Merit Aktiva API?
The official REST API for [Merit Aktiva](https://www.merit.ee/), Estonia's most-used accounting software. Documented at [api.merit.ee](https://api.merit.ee/). Supports sales invoices, purchase invoices, customers, vendors, items, payments, bank statements, GL transactions, and reports.

### How do I get a Merit Aktiva API ID and API key?
Log into Merit Aktiva → **Settings → API Settings → Create Key**. You'll get an `ApiId` (GUID) and an `ApiKey` (HMAC secret). Step-by-step at [API seadistus](https://support.merit.ee/et/articles/368840-api-seadistus). Requires Pro or Premium license.

### Does Merit Aktiva have a sandbox?
No. The [official FAQ](https://api.merit.ee/faq/) confirms it: *"We don't have the sandbox. Just create a test company. Trial license allows you to create up to 100 documents."* Use a trial company for testing.

### How does HMAC signing work?
See the [Authentication](#authentication--hmac-sha256) section above and the full recipe in [`skills/merit-aktiva-core/references/authentication.md`](skills/merit-aktiva-core/references/authentication.md). It's HMAC-SHA256 over `apiId + timestamp + body`, base64-encoded and URL-encoded into the query string. UTC timestamp.

### Can I file KMD via API?
No. EMTA's KMD submission happens at [emta.ee](https://www.emta.ee/) (e-MTA portal). Merit Aktiva exposes the figures (via report endpoints) but does not push to EMTA. Manual filing remains.

### Is this official?
**No.** This is a community plugin, not affiliated with AS Merit Tarkvara. Cross-check rates and rules against [emta.ee](https://www.emta.ee/en) before filings.

### What if my LLM isn't Claude?
The skills are Markdown files. Any LLM that can read text and follow instructions can use them — copy the relevant SKILL.md into your system prompt or context. The plugin format is Claude Code's, but the content is portable.

### Why "AI agents" and not "MCP server"?
This plugin is a set of Claude Code *skills* (Markdown instructions), not an MCP server (a running process). The naming reflects what it actually is. The keywords `merit aktiva mcp` and `accounting mcp server` are addressed by the AI-agent framing because the use case overlaps.

## Security & disclaimer

- **Credentials**: only set `MERIT_API_ID` / `MERIT_API_KEY` as environment variables. Never paste them into chat. Never commit them to git.
- **Trial-first**: test against a Merit trial company before connecting to production. The skill prompts your AI to confirm before any first write in a session.
- **Tax accuracy**: tax rates in this plugin are last verified on the date in each skill's `last-verified` frontmatter. Estonia legislates frequently; cross-check current rates at [emta.ee](https://www.emta.ee/en) before filings.
- **Not an official Merit Tarkvara product.** Provided "as is" under [MIT License](LICENSE).

## Contributing

Issues and PRs welcome at [github.com/arturl95/merit-aktiva-skills](https://github.com/arturl95/merit-aktiva-skills). When updating tax rates, change the `last-verified` date in the affected SKILL.md and update the cited emta.ee URLs.

---

# Merit Aktiva AI agentidele — eestikeelne juhend

> Ühenda **Merit Aktiva** — Eesti enimkasutatav majandustarkvara — Claude'i, ChatGPT, Cursor'i ja teiste AI agentidega. **Merit Aktiva API**, raamatupidamise automatiseerimine, müügiarved, ostuarved, e-arved, käibedeklaratsioon (KMD) — kõik kuue Claude Code'i oskuse (skill) sees.

## Mis see on?

**Merit Aktiva AI agentidele** on avatud lähtekoodiga Claude Code'i plugin, mis õpetab AI agentidele (Claude, ChatGPT, Cursor jt) **Merit Aktiva integratsiooni**: ametliku [Merit Aktiva API](https://api.merit.ee/) HMAC-SHA256 autentimist, kõiki v2 endpointe ja — kõige tähtsam — **2026. aasta Eesti maksureegleid**: käibemaks 24%, jaotuspõhine tulumaks 22/78, palgamaksud, KMD koodid, erisoodustused.

Plugin ei ole MCP server ega koodikogu — see on kuus **Markdown-failis kirjeldatud oskust**, mida AI loeb ja õpib, et **Merit Aktiva liidesega** õigesti suhelda. Sinu agent suhtleb otse `api.merit.ee`-ga HTTPS-i kaudu.

**Kasutusjuhud**: AI raamatupidamine, e-arve import, automaatne ostuarve sisestus, KMD esitamine ChatGPT abil, pangaväljavõtte automaatne kooskõlastus, igakuiste arvete genereerimine.

## Kiire alustamine

```bash
# Paigalda plugin
/plugin install github.com/arturl95/merit-aktiva-skills

# Sea API andmed (Merit Aktiva → Seaded → API seadistus)
export MERIT_API_ID="sinu-api-id-guid"
export MERIT_API_KEY="sinu-api-võti-salasõna"
```

Seejärel küsi Claude'lt:

> "Näita Acme OÜ tasumata arveid sellel kuul."
> "Koosta Acme OÜ-le mai eest 1000 € + 24% käibemaks konsultatsioonarve."
> "Impordi eelmise nädala Swedbanki pangaväljavõte ja kooskõlasta."
> "Kontrolli aprilli KMD enne esitamist."

## Funktsioonid

- **Müügiarved** — `sendinvoice` v2; ümardamine, TaxId, kreeditarved, e-arve väljastamine Omniva / Telema / pangakanali kaudu, PDF, e-kiri.
- **Ostuarved** — tarnijaarved, kuluaruanded, kinnitusvoog, **pöördkäibemaks** EL-i ja kolmandate riikide tehingute jaoks.
- **Maksed ja pangaväljavõte** — maksed, ettemaksud, kooskõlastused, **camt.053** automaatne import viitenumbri järgi.
- **Pearaamat ja aruanded** — `sendglbatch` käsitsi raamatupidamiseks, kasum/kahjum (`getprofitloss`), bilanss (`getfinpos`), nõuded ostjate vastu, KMD kontroll.
- **Eesti maksureeglid (2026)** — käibemaks 24% / 13% / 9%, sotsiaalmaks 33%, tulumaks 22%, põhivabastus 700 €, II sammas 2/4/6%, KMD ridade kaardistus, juhatuse liikme tasu eripärad.
- **API allkirjastamine (HMAC-SHA256)** — ametliku spetsifikatsiooni ja 4 OSS kliendi (Go, TS, 2× PHP) vastu kontrollitud.
- **Kinnituse-värav igal kirjutamisel** — payload kuvatakse, oodatakse "jah". Vajadusel partii-režiim.

## Autentimine (API ID, API võti, HMAC allkiri)

Iga **Merit Aktiva API** kutse kannab kolme query-parameetrit: `ApiId`, `timestamp` (UTC, `yyyyMMddHHmmss`), `signature`.

```
timestamp  = praegune UTC kell formaadis yyyyMMddHHmmss
dataToSign = utf8(apiId + timestamp + httpBody)
signature  = base64(hmac_sha256(apiKey, dataToSign))
url        = base + endpoint + "?ApiId=" + apiId
             + "&timestamp=" + timestamp
             + "&signature=" + urlencode(signature)
```

API ID ja API võtme saad: Merit Aktiva → Seaded → API seadistus. Täielik samm-sammuline juhend [API seadistus](https://support.merit.ee/et/articles/368840-api-seadistus). Vajab Pro või Premium litsentsi.

## Kuus oskust (skills)

| Oskus | Mida teeb |
|---|---|
| **`merit-aktiva-core`** | Autentimine, konventsioonid, veakäsitlus. Aluse-oskus. |
| **`merit-aktiva-masters`** | Kliendid, hankijad, kaubad, KM-koodid, kontoplaan. Nimede → GUID-ide resolveerimine. |
| **`merit-aktiva-sales`** | Müügiarved, kreeditarved, e-arve väljastus, PDF-id. |
| **`merit-aktiva-purchases-payments`** | Ostuarved, kuluaruanded, maksed, pangaväljavõtte import. |
| **`merit-aktiva-ledger-reports`** | Pearaamatu kanded, aruanded, KMD kooskõlastus. |
| **`estonian-bookkeeping`** | 2026. aasta Eesti maksureeglid — KM, tulumaks, palgamaksud, KMD koodid. |

Lisaks **`merit-bookkeeper`** subagent partii-tööks (kviitungite hulk, kuumakse-arved, pangaväljavõte).

## Kasutusnäited

- **AI raamatupidamine OÜ-le** — AI agent loeb tarnijaarveid e-mailist, otsustab kontod ja KM-koodid, kannab pärast kinnitust raamatupidamisse.
- **ChatGPT KMD-assistent** — perioodi arvete summeerimine KMD ridade kaupa, pearaamatu vastu kontroll, lahknevuste väljatoomine enne EMTA-le esitamist.
- **E-arve sünk** — sissetulnud e-arvete suunamine kinnitusjärjekorda ja vastutava töötaja teavitamine.
- **Igakuised arved** — kliendinimekiri × kuumakse → 30 `sendinvoice` kutset ühe partii-kinnitusega.
- **Pangaväljavõtte kaaspilot** — igapäevane camt.053 import + interaktiivne sobitamine sobimatutele ridadele.

## KKK

### Mis on Merit Aktiva API?
Ametlik [Merit Aktiva](https://www.merit.ee/) REST API. Dokumentatsioon [api.merit.ee](https://api.merit.ee/). Toetab müügiarveid, ostuarveid, klientide ja hankijate haldust, kaubaartikleid, makseid, pangaväljavõtteid, pearaamatu kandeid ja aruandeid.

### Kuidas saada Merit Aktiva API võti?
Merit Aktiva → **Seaded → API seadistus → Loo võti**. Saad `ApiId` (GUID) ja `ApiKey` (HMAC salasõna). Vt [API seadistus](https://support.merit.ee/et/articles/368840-api-seadistus). Vajab Pro või Premium litsentsi.

### Kas Merit Aktival on testkeskkond?
Ei. Ametlik FAQ kinnitab: *"Meil ei ole sandboxi. Loo testfirma — prooviversioon lubab 100 dokumenti."* Testimine käib trial-firmas, sama hostiga.

### Kuidas ühendada ChatGPT Merit Aktivaga?
Selle plugina sisu töötab iga LLM-iga, kes oskab juhiseid lugeda. ChatGPT puhul kopeeri vajalikud `SKILL.md` failid süsteemiviipasse — agent suhtleb seejärel otse `api.merit.ee`-ga. Claude Code'i kasutajad saavad lihtsalt plugina paigaldada.

### Kuidas töötab HMAC-allkiri?
Vt [autentimise sektsiooni](#autentimine-api-id-api-võti-hmac-allkiri) ja täielikku retsepti [`skills/merit-aktiva-core/references/authentication.md`](skills/merit-aktiva-core/references/authentication.md). HMAC-SHA256 üle stringi `apiId + timestamp + body`, base64 + URL-encoded query-parameetrisse. UTC ajatempel.

### Kas saab KMD-d API kaudu esitada?
Ei. KMD esitamine käib [emta.ee](https://www.emta.ee/) e-MTA portaali kaudu. Merit Aktiva pakub ettevalmistatud andmed, kuid ei suhtle EMTA-ga otse.

### Kas see on ametlik AS Merit Tarkvara toode?
**Ei.** See on kogukonna plugin, ei ole AS Merit Tarkvaraga seotud. Maksumäärad ja reeglid kontrolli enne KMD/TSD esitamist [emta.ee](https://www.emta.ee/) vastu.

## Vastutus

- Maksumäärad on viimati kontrollitud kuupäeval, mis on iga oskuse `last-verified` väljal. Eesti seadusandlus muutub sageli — kontrolli enne deklaratsioonide esitamist [emta.ee](https://www.emta.ee/) andmeid.
- Esmase tootmiskeskkonna kasutamise eel testi trial-firmas. Oskus küsib sessiooni alguses kinnitust enne esimest kirjutamist.
- API ID ja API võti on tundlikud — ära kunagi pane neid versioonihaldusesse ega vestlusesse.

## Litsents

[MIT](LICENSE) · Autor: Artur · Kogukonna projekt, **mitte ametlik AS Merit Tarkvara toode**.

---

*Made for Estonian businesses. Built with [Claude Code](https://code.claude.com), the [Model Context Protocol](https://modelcontextprotocol.io) ecosystem, and the patience required to read every page of [api.merit.ee/connecting-robots/reference-manual/](https://api.merit.ee/connecting-robots/reference-manual/).*
