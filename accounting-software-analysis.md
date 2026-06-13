# Tennis Club Ambrosiano — Cloud Accounting Software Analysis (carve-out, migration by December)

**Analysis date:** 13 June 2026 *(revised — ASD/SSD regime 398/91 suitability added)*
**Key requirements:**
1. **Import (POST)** of revenue data from the club's management system, broken down by **general-ledger account** and **payment type/method** (ideally also by revenue center).
2. **Extraction (GET)** of consolidated data (account balances, aggregated income/expenses, trial balance, payment schedule) to feed **dynamic external dashboards** (Power BI, Looker Studio, custom apps).
3. Italian tax compliance: SDI electronic invoicing, VAT, telematic receipts; the **398/91 regime** is the decisive filter if TCA operates as an ASD/SSD (e-invoicing mandatory since 1 Jan 2024; from 1 Jan 2026 shift from VAT exclusion to VAT exemption for sports activities).

**Scope note:** the requirement is **true general-ledger accounting** (chart of accounts, double-entry, balances). Products that only handle invoicing/receipts on a cash basis — most notably **Fatture in Cloud** — do not qualify as the accounting system itself.

---

## 0. Critical pre-condition: regime 398/91

Before any software decision, confirm with the accountant whether TCA operates under **Legge 398/1991**. Virtually every Italian amateur sports club (circolo tennis included) qualifies if commercial revenues are below €400,000/year. This regime changes the entire accounting architecture required:

- **Forfettario VAT:** pay only 1/3 of collected VAT on commercial income — standard accounting software calculates full VAT and does not support this mechanism natively.
- **Simplified bookkeeping:** a *registro IVA minori* is legally sufficient (full double-entry not mandatory, though advisable for control).
- **Rendiconto economico-finanziario:** the financial statement format for enti non commerciali differs from the standard Italian *bilancio d'esercizio*. Most general-purpose tools produce the latter.
- **Modello EAS:** annual declaration for non-commercial entities under D.Lgs. 36/2021 (Riforma dello Sport). No generic accounting software generates this.
- **Compensi sportivi dilettantistici:** athlete/coach payments below €10,000 are IRPEF-exempt — must be tracked per individual.

---

## 1. Comparison table

| Software | True GL | Public API | POST (account + payment) | GET (balances/trial balance) | Italian compliance | **Regime 398/91** | Verdict |
|---|---|---|---|---|---|---|---|
| **TeamSystem Sport** (Sportivi in Cloud) | ✅ GL + club mgmt | ⚠️ Partial — TeamSystem ecosystem | ✅ Native club workflows | ✅ ASD/SSD rendiconto | ✅ Full ASD/SSD | ✅ **Native support** | **Sports Specialist** |
| **TeamSystem Contabilità in Cloud** (ex Reviso) | ✅ | ✅ Documented REST API | ✅ POST vouchers/invoices onto real COA | ✅ `accounts.balance` + entries by period | ✅ Italian TeamSystem product | ⚠️ Requires configuration | **Finalist #1** |
| **Odoo** (Enterprise, l10n_it) | ✅ | ✅ XML-RPC/JSON-RPC over whole ORM | ✅ `account.move` (full journal entries), `account.payment` with journal | ✅ `read_group` on `account.move.line` → trial balance, P&L | ✅ Native SDI (proxy K95IV18), withholding, Ri.Ba., DDT | ❌ **Not native** — custom dev required | **Finalist #2*** |
| **TeamSystem Enterprise** (TSE in Cloud) | ✅ | ✅ Developer portal incl. accounting | ✅ likely ("accounting movements" services) | To be verified | ✅ | ⚠️ Requires configuration | Fallback — oversized ERP |
| **Passepartout Passcom/Mexal** | ✅ | ⚠️ WebAPI documented but resources are document/master-data oriented | ⚠️ Via PassBuilder/Sprix partner build | ⚠️ Same | ✅ | ⚠️ Via partner configuration | Fallback — partner-dependent |
| Fatture in Cloud (TeamSystem) | ❌ No chart of accounts, no double-entry | ✅ Excellent (OpenAPI, SDKs, webhooks) | ✅ (onto categories/revenue centers, not GL accounts) | ❌ Lists only, no balances | ✅ SDI via API | N/A | Complementary invoicing layer only |
| Zucchetti, Aruba, WK Genya, Dylog, etc. | Partial/No | ❌ API opacity or SDI-only | ❌ | ❌ | ✅ (SDI only for some) | N/A | Excluded |
| Xero, QuickBooks, Holded | ✅ (non-Italian) | ✅ Excellent APIs | ✅ | ✅ | ❌ No Italy/SDI support | N/A | Excluded |

*\* Odoo is only a realistic option if TCA is an SSD s.r.l. with commercial revenues above the €400,000 regime 398/91 threshold, or if a custom partner-built 398/91 module is budgeted.*

---

## 2. The finalists in detail

### 2.0 TeamSystem Sport (Sportivi in Cloud) — Sports Specialist (first choice for ASD/SSD)

Purpose-built for Italian sports associations ([TeamSystem Sport](https://www.teamsystem.com/sport/sportivi-in-cloud/)).

- **Regime 398/91 native:** handles IVA forfettaria (1/3 of collected VAT on commercial income), produces the *rendiconto economico-finanziario* required for non-commercial entities, manages quote associative as non-taxable, and tracks *compensi sportivi dilettantistici* within IRPEF exemption thresholds.
- **2024-2026 compliance:** covers mandatory e-invoicing for ASD/SSD (since Jan 2024) and the 2026 shift from VAT exclusion to VAT exemption for sports activities; D.Lgs. 36/2021 Riforma dello Sport.
- **Caveat:** standalone public REST API is less mature than Reviso/Odoo — verify dashboard integration depth. If Power BI / Looker Studio connectivity is essential, consider the two-layer architecture below.

### 2.1 TeamSystem Contabilità in Cloud (ex Reviso) — Finalist #1 (GL + API)

A **true cloud general ledger** with a documented REST API ([api-docs.reviso.com](https://api-docs.reviso.com/)).

- **POST:** journal entries and invoices via vouchers onto a real, customizable chart of accounts — covers the payment-type breakdown.
- **GET:** `accounts` resource exposes `balance` plus entries by fiscal year/period — effectively a trial balance via API, ready for dashboards.
- **Regime 398/91:** not native; requires configuration of the chart of accounts to separate institutional from commercial activity, and manual processes for forfettario VAT. Feasible but not automatic.
- **Open issue:** confirm product roadmap with TeamSystem (the product is the rebranded Reviso) before the December deadline.

### 2.2 Odoo Enterprise + Italian localization — Finalist #2 (conditional)

Only viable if TCA is an **SSD s.r.l. above the €400,000 threshold** (standard commercial accounting applies) or if a custom regime 398/91 module is in scope.

- **Compliance:** mature `l10n_it_edi` — FatturaPA through Odoo's accredited SDI proxy (recipient code `K95IV18`); withholding, Ri.Ba., DDT, VAT XML export; receipts via PoS + RT printers.
- **API:** XML-RPC/JSON-RPC with API keys over the entire ORM. No rate limits on Odoo.sh/self-hosted.
- **Regime 398/91 — not native:** the forfettario VAT mechanism, *rendiconto* format, Modello EAS, and exempt-compensation tracking are all absent; a partner-built module and careful COA setup are required. This adds significant cost to an already partner-heavy deployment.
- **Caveats:** External API available only on Custom plans, Odoo.sh, or self-hosted; partner-led implementation required.

### 2.3 Two-layer architecture (recommended for API + regime 398/91 coverage)

If TCA needs both native regime 398/91 support **and** a full REST API for external dashboards:

1. **Layer 1 — club management + regime 398/91:** TeamSystem Sport (or Wansport / Golee) handles membership, courts, forfettario VAT, rendiconto.
2. **Layer 2 — GL + API:** TeamSystem Contabilità in Cloud receives aggregated journal entries from layer 1 and exposes the full REST API for Power BI / Looker Studio.

This is more complex but is the architecture most Italian sports clubs with data ambitions end up with.

---

## 3. Recommendation

1. **First confirm TCA's legal form and regime** with the accountant before any software decision.
2. **If ASD/SSD in regime 398/91 (most likely):**
   - **Primary choice:** TeamSystem Sport — evaluate API depth vs. dashboard requirements.
   - **If API is insufficient:** two-layer architecture (TeamSystem Sport + TeamSystem CiC).
   - Odoo: not recommended without a custom 398/91 module.
3. **If SSD s.r.l. above €400,000 (commercial entity, standard accounting):**
   - **First choice:** TeamSystem Contabilità in Cloud (ex Reviso) — the only Italian-native cloud GL with a publicly documented REST API at SME-friendly cost.
   - **Second choice:** Odoo Enterprise with Italian localization — most powerful, at the price of a partner-led project.

---

## 4. Suggested next steps (December deadline)

1. Confirm with the accountant: ASD vs. SSD, and whether regime 398/91 applies (revenues vs. €400,000).
2. Demo TeamSystem Sport: verify regime 398/91 coverage and API/export capabilities.
3. Open a Reviso/CiC test account and run a two-week proof of concept: POST one real day of revenue + GET balances into a prototype dashboard.
4. If two-layer architecture: model the integration between the club system and CiC.
5. Obtain TeamSystem's written roadmap commitment for the chosen product before signing.

---

## 5. Main sources

- Contabilità in Cloud / Reviso: [api-docs.reviso.com](https://api-docs.reviso.com/) · [REST API (IT)](https://www.reviso.com/it/assistenza/articoli/rest-api/) · [Rebranding](https://www.reviso.com/it/assistenza/articoli/reviso-diventa-contabilita-in-cloud/)
- Odoo: [Italian localization 18.0](https://www.odoo.com/documentation/18.0/applications/finance/fiscal_localizations/italy.html) · [External API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html) · [OCA l10n-italy](https://github.com/OCA/l10n-italy)
- TeamSystem Enterprise: [tse.docs.teamsystem.cloud](https://tse.docs.teamsystem.cloud/)
- Fatture in Cloud: [developers.fattureincloud.it](https://developers.fattureincloud.it/) · [OpenAPI spec (GitHub)](https://github.com/fattureincloud/openapi-fattureincloud)
- Passepartout: [Passcom WebAPI (EduPass)](https://www.edupass.it/manuali/manualistica-passcom/manuale-prodotto?a=manuale-passweb-ecommerce%2Fconfigurazione%2Fpasscom--configurazione-gestionale%2Fweb-api-passcom)
- ASD/SSD 398/91: [regime forfettario (TeamSystem Magazine)](https://www.teamsystem.com/magazine/sport-e-wellness/regime-forfettario-asd-ssd/) · [2026 changes](https://fatturapro.click/associazioni-sportive-dilettantistiche-regime-398-91-e-novita-2026/) · [mandatory e-invoicing since 2024](https://www.aruba.it/magazine/fatturazione-elettronica/associazioni-sportive-dilettantistiche-fattura-elettronica-obbligatoria-dal-1-gennaio-2024.aspx)
- Sports verticals: [TeamSystem Sportivi in Cloud](https://www.teamsystem.com/sport/sportivi-in-cloud/) · [Wansport tennis](https://wansport.com/gestionale-per-tennis-club/) · [Golee circoli](https://golee.it/gestionale-per-circoli/)
