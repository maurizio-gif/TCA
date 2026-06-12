# Tennis Club Ambrosiano — Cloud Accounting Software Analysis (carve-out, migration by December)

**Analysis date:** 12 June 2026
**Key requirements:**
1. **Import (POST)** of revenue data from the club's management system, broken down by **general-ledger account** and **payment type/method** (ideally also by revenue center).
2. **Extraction (GET)** of consolidated data (account balances, aggregated income/expenses, trial balance, payment schedule) to feed **dynamic external dashboards** (Power BI, Looker Studio, custom apps).
3. Italian tax compliance: SDI electronic invoicing, VAT, telematic receipts; the **398/91 regime** is relevant if the club operates as an ASD/SSD (e-invoicing mandatory since 1 Jan 2024; from 1 Jan 2026 the shift from VAT exclusion to VAT exemption brings new obligations).

**Scope note:** the requirement is **true general-ledger accounting** (chart of accounts, double-entry, balances). Products that only handle invoicing/receipts on a cash basis — most notably **Fatture in Cloud** — do not qualify as the accounting system itself; they are assessed here only as possible complementary tools.

---

## 1. Comparison table

| Software | True general-ledger accounting | Public documented API | POST revenue with account + payment method | GET consolidated data (balances/trial balance) | Italian compliance | Verdict |
|---|---|---|---|---|---|---|
| **TeamSystem Contabilità in Cloud** (ex Reviso) | ✅ | ✅ Documented REST API | ✅ POST vouchers/invoices onto a real chart of accounts | ✅ `accounts` resource exposes `balance`; entries by period | ✅ Italian TeamSystem product | **Finalist #1** |
| **Odoo** (Enterprise, l10n_it) | ✅ | ✅ XML-RPC/JSON-RPC over the whole ORM | ✅ `account.move` (full journal entries), `account.payment` with journal | ✅ `read_group` on `account.move.line` → trial balance, P&L | ✅ Native SDI (proxy code K95IV18), withholding, Ri.Ba., DDT | **Finalist #2** |
| **TeamSystem Enterprise** (TSE in Cloud) | ✅ | ✅ Developer portal incl. accounting API | ✅ likely ("accounting movements" services) | To be verified | ✅ | Possible but an over-sized ERP for a club |
| **Passepartout Passcom/Mexal** | ✅ | ⚠️ WebAPI documented (EduPass) but standard resources are document/master-data oriented | ⚠️ Via PassBuilder/Sprix customization built by a partner | ⚠️ Same | ✅ | Possible, but partner-dependent custom development |
| Fatture in Cloud (TeamSystem) | ❌ **Not accounting** — invoicing/receipts/cash, no chart of accounts, no double-entry | ✅ Excellent (OpenAPI, SDKs, webhooks) | ✅ (but onto categories/revenue centers, not GL accounts) | ❌ No balances/trial balance; lists only | ✅ SDI via API | Excluded as the accounting system; possible complementary invoicing/receipts layer |
| Zucchetti (Digital Hub, Tieni il Conto) | ✅/⚠️ | ❌ Docs only via sales/partner channel | ❌ Not publicly documented | ❌ | ✅ (DH = SDI only) | Excluded — API opacity |
| Aruba Electronic Invoicing | ❌ (SDI channel only) | ✅ Public apidoc | ❌ XML upload to SDI only | ❌ Invoice/notification search only | ✅ (SDI only) | SDI channel only, not an accounting backend |
| Wolters Kluwer Genya | ✅ | ❌ No public API found (Excel/XML import only) | ❌ | ❌ | ✅ | Excluded |
| Dylog / Datev Koinos / B.Point | ✅ | ❌ (file tracts or e-invoicing API only) | ❌ | ❌ | ✅ | Excluded; Koinos/B.Point are accountant-side tools |
| Sibill / Tot / Qonto / Agicap | ❌ Fintech/treasury, not accounting | ⚠️ Some good APIs (Qonto, Agicap) | ❌ | ❌ | ⚠️ | Wrong category; complementary at most |
| Xero | ✅ | ✅ Excellent (Reports API: TrialBalance, P&L) | ✅ | ✅ | ❌ **No SDI/Italy support** ("no sights for Italy") | Excluded |
| QuickBooks Online | ✅ | ✅ Excellent (Reports API) | ✅ | ✅ | ⚠️ Only via third-party add-on (FOR S.r.l.); Intuit withdrew from France in 2023 | Double-vendor risk: not recommended |
| Holded | ✅ (Spain) | ⚠️ API key, no consolidated report endpoints | ✅ partial | ❌ | ❌ Spain only (Veri*factu/AEAT) | Excluded |

---

## 2. The finalists in detail

### 2.1 TeamSystem Contabilità in Cloud (ex Reviso) — Finalist #1

A **true cloud general ledger** with a documented REST API ([api-docs.reviso.com](https://api-docs.reviso.com/), [Italian docs](https://www.reviso.com/it/assistenza/articoli/rest-api/)).

- **POST (requirement 1):** journal entries and invoices created via vouchers (official example: [`POST /vouchers/drafts/customer-invoices`](https://www.reviso.com/it/assistenza/articoli/restful-api-esempio-crea-fattura-manuale/)) onto a **real, customizable chart of accounts** — revenue can be posted to the right account with the relevant counterpart (cash, bank, POS), which covers the payment-type breakdown.
- **GET (requirement 2):** the `accounts` resource exposes `accountNumber`, `accountType` and **`balance`**, plus entries per fiscal year/period (`GET /accounts/:n/accounting-years/:y/periods/:p/vouchers`) — effectively a **trial balance via API**, ready for external dashboards.
- **Auth:** dual token (`X-AppSecretToken` + `X-AgreementGrantToken`); free demo environment (`?demo=true`, GET only).
- **Open issue to resolve before signing:** Reviso was rebranded "Contabilità in Cloud" and is sold on the TeamSystem Store ([rebranding notice](https://www.reviso.com/it/assistenza/articoli/reviso-diventa-contabilita-in-cloud/), [product page](https://www.teamsystem.com/store/contabilita-in-cloud/funzionalita/registrazioni-contabili/)) — **confirm the product roadmap with TeamSystem** given the December deadline.

### 2.2 Odoo Enterprise + Italian localization — Finalist #2

- **Compliance:** mature native Italian localization ([official docs](https://www.odoo.com/documentation/18.0/applications/finance/fiscal_localizations/italy.html)): `l10n_it_edi` sends/receives FatturaPA **through Odoo's accredited SDI proxy** (recipient code `K95IV18`); withholding taxes, Ri.Ba., DDT, VAT XML export; telematic receipts via Odoo PoS + certified RT printers.
- **API** ([External API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html)): XML-RPC/JSON-RPC with API keys over the entire ORM — `create` on `account.move` (any journal entry, lines per GL account) and `account.payment` (journal = payment method/account); `read_group`/`search_read` on `account.move.line` for balances, trial balance and P&L per period. No rate limits when on Odoo.sh/self-hosted.
- **Caveats:** on Odoo Online (SaaS) the External API is available **only on Custom plans** (alternatively Odoo.sh or self-hosted); implementation normally requires a partner and a real project — weigh against the December deadline.

### 2.3 Fallback options

- **TeamSystem Enterprise (TSE in Cloud):** public developer portal ([tse.docs.teamsystem.cloud](https://tse.docs.teamsystem.cloud/)) including an accounting API reference with "accounting movements" services — a real candidate on paper, but it is a full mid-market ERP: licensing and complexity are likely oversized for a tennis club. Requires being a TSE customer.
- **Passepartout Passcom/Mexal:** true accounting with a documented WebAPI ([EduPass](https://www.edupass.it/manuali/manualistica-passcom/manuale-prodotto?a=manuale-passweb-ecommerce%2Fconfigurazione%2Fpasscom--configurazione-gestionale%2Fweb-api-passcom)), but journal-entry POST and balance GET would almost certainly go through a PassBuilder/Sprix customization built by a Passepartout partner — feasible, with vendor lock-in and custom development to budget.

### 2.4 Where Fatture in Cloud still fits (complementary only)

Fatture in Cloud has the best-documented API on the Italian market ([developers.fattureincloud.it](https://developers.fattureincloud.it/), OpenAPI spec on [GitHub](https://github.com/fattureincloud/openapi-fattureincloud), 8 official SDKs, webhooks, API included in every paid plan) and natively supports `POST /receipts` and `POST /issued_documents` with `payment_account`, `rc_center` and line categories. But it has **no chart of accounts, no double-entry, no balance/trial-balance endpoints**: it is an invoicing/receipts/cash tool. If the chosen GL system lacks a good SDI front-end, FIC could serve as the invoicing/receipts layer feeding the GL — otherwise it drops out of the architecture entirely.

---

## 3. Recommendation

1. **First choice: TeamSystem Contabilità in Cloud (ex Reviso)** — the only Italian-native cloud product that satisfies both requirements through a publicly documented API (POST onto a real chart of accounts; GET account balances), at SME-friendly cost. Condition: written confirmation of the product roadmap from TeamSystem.
2. **Second choice: Odoo Enterprise with Italian localization** — the most powerful and most future-proof option (full GL API, native SDI, built-in dashboards), at the price of a partner-led implementation; choose it if the carve-out budget and timeline allow a proper project before December.
3. **Fallbacks:** TeamSystem Enterprise (if the group is standardizing on TeamSystem ERP anyway) or Passepartout via a partner POC.

**Suggested next steps (December deadline):**
1. Confirm with the accountant the post-carve-out accounting perimeter (full double-entry in-house vs. cash-basis management with the GL at the firm) — this validates the requirement assumption above.
2. Open test accounts: Contabilità in Cloud/Reviso demo + an Odoo trial with `l10n_it`.
3. Two-week proof of concept: POST one real day of revenue (courts, bar, membership fees) with accounts and payment methods + GET balances into a prototype dashboard (Power BI/Looker Studio).
4. Obtain TeamSystem's roadmap commitment for Contabilità in Cloud before contract signature.

---

## 4. Main sources

- Contabilità in Cloud / Reviso: [api-docs.reviso.com](https://api-docs.reviso.com/) · [REST API (IT)](https://www.reviso.com/it/assistenza/articoli/rest-api/) · [Invoice creation example](https://www.reviso.com/it/assistenza/articoli/restful-api-esempio-crea-fattura-manuale/) · [Authentication](https://www.reviso.com/authentication/) · [Rebranding](https://www.reviso.com/it/assistenza/articoli/reviso-diventa-contabilita-in-cloud/) · [TeamSystem Store](https://www.teamsystem.com/store/contabilita-in-cloud/funzionalita/registrazioni-contabili/)
- Odoo: [Italian localization 18.0](https://www.odoo.com/documentation/18.0/applications/finance/fiscal_localizations/italy.html) · [External API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html) · [OCA l10n-italy](https://github.com/OCA/l10n-italy)
- TeamSystem Enterprise: [tse.docs.teamsystem.cloud](https://tse.docs.teamsystem.cloud/) · TS Digital: [TS Digital Invoice API](https://www.teamsystem.com/store/ts-digital-invoice/funzionalita/api/)
- Fatture in Cloud: [developers.fattureincloud.it](https://developers.fattureincloud.it/) · [API Reference](https://developers.fattureincloud.it/api-reference/) · [OpenAPI spec (GitHub)](https://github.com/fattureincloud/openapi-fattureincloud) · [Authentication](https://developers.fattureincloud.it/docs/authentication/) · [Limits & quotas](https://developers.fattureincloud.it/docs/basics/limits-and-quotas/) · [Webhooks](https://developers.fattureincloud.it/docs/webhooks/) · [E-invoice](https://developers.fattureincloud.it/docs/guides/e-invoice-management/) · [API included in licenses](https://help.fattureincloud.it/help/articolo/666-integra-altri-software-fatture-cloud)
- Passepartout: [Passcom WebAPI (EduPass)](https://www.edupass.it/manuali/manualistica-passcom/manuale-prodotto?a=manuale-passweb-ecommerce%2Fconfigurazione%2Fpasscom--configurazione-gestionale%2Fweb-api-passcom) · [Accounting](https://www.passepartout.net/software/imprese/gestione-contabilita)
- Zucchetti: [Digital Hub third-party integration](https://help.zucchetti.it/cms/kb/soluzioni/fatturazione-elettronica/digital-hub/scopri-e-informati/faq/integrazioni/non-ho-gestionale-zucchetti-come-posso-integrarmi-con-digital-hub.html) · [Tieni il Conto PRO](https://www.zucchetti.it/website/cms/prodotto/8310-tieni-il-conto-pro.html)
- Aruba: [apidoc v1](https://fatturazioneelettronica.aruba.it/apidoc/docs.html) · [apidoc v2](https://fatturazioneelettronica.aruba.it/apidoc/v2/docs.html)
- Xero: [Reports API](https://developer.xero.com/documentation/api/accounting/reports) · [No Italy (community)](https://community.xero.com/business/discussion/115295423) — QuickBooks: [Reports API](https://developer.intuit.com/app/developer/qbo/docs/workflows/run-reports) · [FOR:QUICKBOOKS](https://www.quickbooksitalia.com/) · [France discontinuation](https://blogs.intuit.com/2023/06/12/discontinuation-of-quickbooks-in-france/) — Holded: [API](https://developers.holded.com/reference)
- ASD/SSD 398/91 regime: [TeamSystem Magazine](https://www.teamsystem.com/magazine/sport-e-wellness/regime-forfettario-asd-ssd/) · [2026 changes](https://fatturapro.click/associazioni-sportive-dilettantistiche-regime-398-91-e-novita-2026/) · [Mandatory e-invoicing for ASD since 2024](https://www.aruba.it/magazine/fatturazione-elettronica/associazioni-sportive-dilettantistiche-fattura-elettronica-obbligatoria-dal-1-gennaio-2024.aspx)
- Sports verticals: [TeamSystem Sportivi in Cloud](https://www.teamsystem.com/sport/sportivi-in-cloud/) · [Wansport](https://wansport.com/gestionale-per-tennis-club/) · [Golee](https://golee.it/gestionale-per-circoli/)
