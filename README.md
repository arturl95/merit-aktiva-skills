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

In any Claude Code session:

```bash
# 1. Add this repo as a plugin marketplace
/plugin marketplace add arturl95/merit-aktiva-skills

# 2. Install the plugin
/plugin install merit-aktiva-skills@merit-aktiva-skills
```

Then set credentials (from Merit Aktiva → Settings → API Settings) in your shell:

```bash
export MERIT_API_ID="your-api-id-guid"
export MERIT_API_KEY="your-api-key-secret"
```

See [Installation](#installation) below for alternative install methods (local clone, marketplace.json, etc.).

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

## Installation

Pick whichever method fits your setup.

### A — Add as a marketplace (recommended)

In any Claude Code session:

```bash
/plugin marketplace add arturl95/merit-aktiva-skills
/plugin install merit-aktiva-skills@merit-aktiva-skills
```

The first command registers this repo as a [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces). The second installs the `merit-aktiva-skills` plugin from it. Updates land via `/plugin update`.

### B — Local clone (for development or air-gapped use)

```bash
git clone https://github.com/arturl95/merit-aktiva-skills.git
claude --plugin-dir ./merit-aktiva-skills
```

`--plugin-dir` loads the plugin for the current session without installing it system-wide. You can stack multiple plugin directories.

### C — Persist as a known marketplace in `~/.claude/settings.json`

```json
{
  "extraKnownMarketplaces": {
    "merit-aktiva-skills": {
      "source": { "source": "github", "repo": "arturl95/merit-aktiva-skills" }
    }
  }
}
```

Then `/plugin install merit-aktiva-skills@merit-aktiva-skills`. Useful if you want the marketplace available without re-adding it each time.

### Verifying installation

```bash
/plugin list
```

You should see `merit-aktiva-skills` enabled. Try `merit-aktiva-core` in your next prompt:

> "Show me what merit-aktiva-core knows about authentication."

Claude should load the skill and summarise the HMAC signing recipe.

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

> Ühenda **Merit Aktiva** — Eesti enimkasutatav majandustarkvara — Claude'i, ChatGPT, Cursori ja teiste AI agentidega. **Merit Aktiva API**, raamatupidamise automatiseerimine, müügiarved, ostuarved, e-arved, käibedeklaratsioon (KMD) — kõik see on pakendatud kuude Claude Code'i oskusesse (skill).

## Mis see on?

**Merit Aktiva AI agentidele** on avatud lähtekoodiga Claude Code'i plugin, mis õpetab AI agentidele (Claude, ChatGPT, Cursor jt) selgeks **Merit Aktiva integratsiooni**: ametliku [Merit Aktiva API](https://api.merit.ee/) HMAC-SHA256 autentimise, kõik v2 endpoint'id ja — mis kõige tähtsam — **2026. aasta Eesti maksureeglid**: käibemaks 24%, jaotatud kasumi tulumaks 22/78, palgamaksud, KMD koodid ja erisoodustused.

See plugin ei ole MCP server ega kooditeek — see koosneb kuuest **Markdown-failis kirjeldatud oskusest**, mida AI loeb ja omandab, et **Merit Aktiva liidesega** korrektselt suhelda. Sinu agent suhtleb `api.merit.ee`-ga otse üle HTTPS-i.

**Kasutusjuhud**: AI raamatupidamine ja AI-toega raamatupidamine, e-arvete import, ostuarvete automaatne sisestamine, KMD koostamine ChatGPT abil, pangaväljavõtete automaatne kooskõlastamine ja igakuiste korduvate arvete genereerimine.

## Kiire alustamine

Käivita Claude Code'is:

```bash
# 1. Lisa see repo plugina-turuplatsina
/plugin marketplace add arturl95/merit-aktiva-skills

# 2. Paigalda plugin
/plugin install merit-aktiva-skills@merit-aktiva-skills
```

Seejärel sea API andmed terminalis (Merit Aktiva → Seaded → API seadistus):

```bash
export MERIT_API_ID="sinu-api-id-guid"
export MERIT_API_KEY="sinu-api-võti-salasõna"
```

Alternatiivsete paigaldusviiside (kohalik kloon, `--plugin-dir`, `settings.json` kaudu) jaoks vaata [Installation](#installation) sektsiooni inglisekeelses osas.

Seejärel küsi Claude'ilt:

> "Näita Acme OÜ selle kuu tasumata arveid."
> "Koosta Acme OÜ-le mai eest konsultatsiooniteenuse arve summas 1000 € + 24% käibemaks."
> "Impordi eelmise nädala Swedbanki pangaväljavõte ja kooskõlasta laekumised."
> "Kontrolli aprilli KMD-d enne esitamist."

## Funktsioonid

- **Müügiarved** — `sendinvoice` v2; ümardamised, TaxId, kreeditarved, e-arvete väljastamine Omniva, Telema või pangakanalite kaudu, PDF-ide ja e-kirjade genereerimine.
- **Ostuarved** — hankija arved, kuluaruanded, kinnitusringid, **pöördkäibemaks** EL-i ja kolmandate riikide tehingutel.
- **Maksed ja pangaväljavõtted** — laekumised, tasumised, ettemaksud, kooskõlastamised, **camt.053** automaatne import ja sidumine viitenumbri alusel.
- **Pearaamat ja aruanded** — `sendglbatch` käsitsi kannete tegemiseks, kasumiaruananne (`getprofitloss`), bilanss (`getfinpos`), ostjate tasumata arved, KMD kontroll.
- **Eesti maksureeglid (2026)** — käibemaks 24% / 13% / 9%, sotsiaalmaks 33%, tulumaks 22%, maksuvaba tulu 700 €, II sammas 2/4/6%, KMD ridade kaardistamine, juhatuse liikme tasu eripärad.
- **API allkirjastamine (HMAC-SHA256)** — testitud ja valideeritud ametliku spetsifikatsiooni ning nelja avatud lähtekoodiga kliendi (Go, TS, 2× PHP) põhjal.
- **Kinnitusnõue andmete muutmisel** — enne API päringu teele panemist kuvatakse *payload* ja oodatakse kasutaja kinnitust ("jah"). Toetab ka partii-režiimi (*batch mode*).

## Autentimine (API ID, API võti, HMAC allkiri)

Iga **Merit Aktiva API** päring sisaldab kolme URL-i parameetrit (*query parameter*): `ApiId`, `timestamp` (UTC, `yyyyMMddHHmmss`) ja `signature`.

```
timestamp  = praegune UTC kell formaadis yyyyMMddHHmmss
dataToSign = utf8(apiId + timestamp + httpBody)
signature  = base64(hmac_sha256(apiKey, dataToSign))
url        = base + endpoint + "?ApiId=" + apiId
             + "&timestamp=" + timestamp
             + "&signature=" + urlencode(signature)
```

API ID ja API võtme genereerimiseks ava: Merit Aktiva → Seaded → API seadistus. Vaata ka samm-sammulist juhendit: [API seadistus](https://support.merit.ee/et/articles/368840-api-seadistus). API kasutamine eeldab Pro või Premium paketi olemasolu.

## Kuus oskust (skills)

| Oskus | Kirjeldus |
|---|---|
| **`merit-aktiva-core`** | Autentimine, API konventsioonid ja veakäsitlus. Baasoskus, millele toetuvad teised. |
| **`merit-aktiva-masters`** | Kliendid, hankijad, artiklid, KM-koodid ja kontoplaan. Nimede vastendamine GUID-ideks. |
| **`merit-aktiva-sales`** | Müügiarved, kreeditarved, e-arvete väljastamine ja PDF-ide genereerimine. |
| **`merit-aktiva-purchases-payments`** | Ostuarved, kuluaruanded, maksed ja pangaväljavõtete import. |
| **`merit-aktiva-ledger-reports`** | Pearaamatu kanded, finantsaruanded ja KMD andmete kontroll. |
| **`estonian-bookkeeping`** | 2026. aasta Eesti maksureeglid — käibemaks, tulumaks, palgamaksud ja KMD koodid. |

Lisaks on saadaval **`merit-bookkeeper`** alamagent (*subagent*) massandmete töötlemiseks (suurem hulk tšekke, igakuised korduvad arved, mahukad pangaväljavõtted).

## Kasutusnäited

- **AI-raamatupidaja väikeettevõttele** — AI agent loeb e-kirjadest hankijate arveid, määrab õiged kulukontod ja KM-koodid ning sisestab need pärast inimese kinnitust raamatupidamisse.
- **ChatGPT KMD-assistent** — perioodi arvete summeerimine KMD ridade lõikes, andmete võrdlemine pearaamatuga ja lahknevuste tuvastamine enne EMTA-le esitamist.
- **E-arvete sünkroniseerimine** — sissetulevate e-arvete suunamine kinnitusringi ja vastutavate töötajate teavitamine.
- **Igakuiste arvete masskoostamine** — kliendinimekiri × kuutasu → 30 `sendinvoice` päringut, mis saadetakse teele üheainsa koondkinnitusega.
- **Pangaväljavõtete assistent** — igapäevane **camt.053** failide import ja interaktiivne abi tundmatute laekumiste sidumisel.

## KKK

### Mis on Merit Aktiva API?
See on ametlik [Merit Aktiva](https://www.merit.ee/) REST API. Dokumentatsioon asub aadressil [api.merit.ee](https://api.merit.ee/). API toetab müügi- ja ostuarveid, klientide ning hankijate haldust, artikleid, makseid, pangaväljavõtteid, pearaamatu kandeid ja finantsaruandeid.

### Kuidas saada Merit Aktiva API võti?
Ava Merit Aktiva ja liigu: **Seaded → API seadistus → Loo võti**. Sealt saad `ApiId` (GUID) ja `ApiKey` (HMAC salasõna). Loe lähemalt juhendist: [API seadistus](https://support.merit.ee/et/articles/368840-api-seadistus). API kasutamiseks on vajalik Pro või Premium pakett.

### Kas Merit Aktival on testkeskkond?
Ei ole. Ametlik KKK ütleb: *"Meil ei ole sandboxi. Loo testfirma — prooviversioon lubab 100 dokumenti."* Testimine toimub tavalises prooviversiooni (*trial*) ettevõttes, kasutades sama API aadressi.

### Kuidas ühendada ChatGPT Merit Aktivaga?
Selle plugina sisu töötab iga keelemudeliga (LLM), mis suudab juhiseid järgida. ChatGPT puhul kopeeri vajalike `SKILL.md` failide sisu süsteemiviipa (*system prompt*) — seejärel oskab agent otse `api.merit.ee`-ga suhelda. Claude Code'i kasutajad saavad plugina lihtsalt käsurealt paigaldada.

### Kuidas töötab HMAC-allkiri?
Vaata [autentimise sektsiooni](#autentimine-api-id-api-võti-hmac-allkiri) ja täielikku tehnilist kirjeldust failist [`skills/merit-aktiva-core/references/authentication.md`](skills/merit-aktiva-core/references/authentication.md). Lühidalt: tehakse HMAC-SHA256 räsiallkiri stringist `apiId + timestamp + body`, mis kodeeritakse Base64 formaati ja lisatakse URL-kodeerituna päringu parameetritesse. Ajatempel peab olema UTC ajavööndis.

### Kas saab KMD-d API kaudu esitada?
Ei saa. KMD esitamine toimub jätkuvalt [emta.ee](https://www.emta.ee/) e-MTA portaali kaudu. Merit Aktiva API kaudu saab kätte eeltäidetud andmed, kuid tarkvara ei saada neid automaatselt EMTA-sse.

### Kas see on ametlik AS Merit Tarkvara toode?
**Ei.** Tegemist on kogukonna loodud pluginaga, mis ei ole AS-iga Merit Tarkvara seotud. Enne KMD/TSD esitamist kontrolli alati maksumäärad ja reeglid üle [emta.ee](https://www.emta.ee/) lehelt.

## Vastutus

- Maksumäärad on viimati kontrollitud kuupäeval, mis on märgitud iga oskuse `last-verified` väljale. Kuna Eesti maksuseadusandlus võib muutuda, kontrolli andmeid enne deklaratsioonide esitamist alati [emta.ee](https://www.emta.ee/) lehelt.
- Enne toodangukeskkonnas (*production*) kasutamist testi lahendust kindlasti prooviversiooni (*trial*) ettevõttes. Agent küsib sessiooni alguses alati kinnitust, enne kui teeb API kaudu esimese andmeid muutva päringu.
- API ID ja API võti on konfidentsiaalsed andmed — ära kunagi lisa neid versioonihaldusesse (nt Git) ega jaga avalikes vestlustes.

## Litsents

[MIT](LICENSE) · Autor: Artur · Kogukonna projekt, **tegemist ei ole ametliku AS Merit Tarkvara tootega**.

---

*Made for Estonian businesses. Built with [Claude Code](https://code.claude.com), the [Model Context Protocol](https://modelcontextprotocol.io) ecosystem, and the patience required to read every page of [api.merit.ee/connecting-robots/reference-manual/](https://api.merit.ee/connecting-robots/reference-manual/).*