# Tennis Club Ambrosiano — Cloud Accounting Software Analysis (carve-out, migration by December)

**Analysis date:** 13 June 2026 *(rev. 2 — pricing, features, and scalability comparison added)*
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

**Pricing (per transaction volume — users unlimited):**
| Plan | Price | Annual entries |
|---|---|---|
| Base | ~€25–29/mese | ~6,000 voci/anno |
| Medium | ~€45–59/mese | ~15,000 voci/anno |
| Advanced | on request | unlimited |

*Figures indicative; confirm at teamsystem.com/store/contabilita-in-cloud/prezzi.*

- **Users:** unlimited included — no per-user cost. The whole TCA team at the same flat rate.
- **Typical TCA cost:** ~€45–59/mese → ~€540–700/anno. Implementation: guided self-setup, no partner required (€0–2,000).
- **POST:** journal entries and invoices via vouchers onto a real, customizable chart of accounts — revenue posted per GL account with correct counterpart (cash, bank, POS).
- **GET:** `accounts` resource exposes `balance` plus entries by fiscal year/period — trial balance via REST API, ready for Power BI / Looker Studio.
- **Italian compliance:** native IVA, SDI/FatturaPA via TeamSystem ecosystem, export to studio commercialista.
- **Regime 398/91:** not native; requires COA configuration and manual processes for forfettario VAT.
- **Ceiling:** accounting only — no HR, CRM, member management, multi-company, or ERP expansion path.
- **Open issue:** confirm product roadmap with TeamSystem before the December deadline.

### 2.2 Odoo Enterprise + Italian localization — Finalist #2 (conditional — but uniquely scalable)

The only option that starts as accounting and can grow into a **full ERP — multi-module, multi-company, multi-country — without changing platform**.

**Pricing (Custom plan, SaaS hosted by Odoo):**
| Users | Monthly (Custom SaaS) | Annual |
|---|---|---|
| 5 users | ~€170–210/mese | ~€2,000–2,500 |
| 10 users | ~€340–420/mese | ~€4,000–5,000 |

*~€34–42/utente/mese indicativo per l'Italia (exact EUR: odoo.com/pricing-configurator). The Standard plan (~€22–28/utente/mese) excludes the external API and multi-company — Custom is required.*

- **Implementation (one-time):** partner-led project — Italian localization, data migration, training, and any custom modules: **€10,000–25,000**. Total Year 1: ~€12,000–27,500.
- **The scalability path (key differentiator):**
  - Phase 1 — Custom SaaS: accounting + SDI, Odoo hosts everything. All 50+ modules available to activate.
  - Phase 2+ (hypothetical, outside current scope): HR, multi-company, multi-country, custom modules — all on the same platform without a system migration.
- **Multi-country:** 100+ official localizations — if the parent group has foreign entities, one Odoo instance handles all.
- **API:** XML-RPC/JSON-RPC over the full ORM — most powerful data access available. No extra cost on Custom plan.
- **Italian accounting compliance:** `l10n_it_edi` — FatturaPA via SDI proxy K95IV18, withholding, Ri.Ba., DDT, VAT export. Fiscal receipts (corrispettivi) via direct AdE connection — no RT printer required.
- **Regime 398/91 — not native:** forfettario VAT, rendiconto, Modello EAS all require a partner-built custom module.

### 2.3 Head-to-head: CiC vs Odoo

| Dimension | TeamSystem CiC | Odoo Custom SaaS |
|---|---|---|
| **Pricing model** | Per transaction volume, users unlimited | Per user/month |
| **Year 1 total cost** | ~€540–2,700 | ~€12,300–27,500 |
| **Year 2+ annual** | ~€540–700 | ~€2,300 (5 users) |
| **API** | REST, documented, all plans | XML-RPC/JSON-RPC, Custom plan only |
| **Multi-company** | No | Yes (Custom plan) |
| **Multi-country** | Italy only | 100+ localizations |
| **ERP modules** | None | Full suite (HR, CRM, POS, Inventory…) |
| **Setup complexity** | Low — self-service | Medium-High — partner required |
| **Regime 398/91** | Config required | Custom module required |
| **Growth path** | Accounting ceiling | Full ERP, multi-entity |

### 2.3 Two-layer architecture (recommended for API + regime 398/91 coverage)

If TCA needs both native regime 398/91 support **and** a full REST API for external dashboards:

1. **Layer 1 — club management + regime 398/91:** TeamSystem Sport (or Wansport / Golee) handles membership, courts, forfettario VAT, rendiconto.
2. **Layer 2 — GL + API:** TeamSystem Contabilità in Cloud receives aggregated journal entries from layer 1 and exposes the full REST API for Power BI / Looker Studio.

This is more complex but is the architecture most Italian sports clubs with data ambitions end up with.

---

## 3. Recommendation

**Step 0 (before anything):** confirm with the accountant whether TCA operates under regime 398/91. This is the single most important decision gate.

**If ASD/SSD in regime 398/91 (most likely):**
1. **TeamSystem Sport** as the primary system (native 398/91 + club management).
2. If Power BI / Looker Studio integration is critical and TeamSystem Sport's API is insufficient: two-layer architecture — TeamSystem Sport + **TeamSystem CiC** as the GL/API layer.
3. Odoo: not recommended without a custom 398/91 module. Could be scoped for a Phase 2 if TCA plans multi-entity expansion.

**If SSD s.r.l. above €400,000 (full commercial accounting, no 398/91):**
1. **TeamSystem CiC** — first choice for cost, simplicity, and documented REST API (~€540–700/anno, users unlimited, no partner needed).
2. **Odoo Custom SaaS** — second choice if TCA needs ERP scalability, multi-company, or multi-country capabilities. Justified when Phase 2-3 expansion is planned; Year 1 cost ~€12,000–27,500.

**When Odoo's higher cost is justified (hypothetical — outside current scope):** if TCA eventually needs to manage additional entities on the same platform, integrate across multiple countries, or expand beyond pure accounting — Odoo avoids a future system migration. Year 2+ cost (~€2,300/anno for 5 users) is competitive once the Year 1 implementation is amortized. If TCA stays a single accounting entity, CiC is decisively more cost-effective at ~€540–700/anno.

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
