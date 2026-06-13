# Tennis Club Ambrosiano — Cloud Accounting Software Analysis (carve-out, migration by December)

**Analysis date:** 13 June 2026
**Key requirements:**
1. **Import (POST)** of revenue data from the club's management system, broken down by **general-ledger account** and **payment type/method** (ideally also by revenue center).
2. **Extraction (GET)** of consolidated data (account balances, trial balance, aggregated income/expenses) to feed **dynamic external dashboards** (Power BI, Looker Studio, custom apps).
3. Italian accounting compliance: SDI electronic invoicing, IVA, withholding taxes, Ri.Ba.

**Scope:** true general-ledger accounting (chart of accounts, double-entry, balances). Products that only handle invoicing/receipts — notably Fatture in Cloud — do not qualify as the accounting backend.

---

## 1. Comparison table

| Software | True GL | Public API | POST (account + payment) | GET (balances/trial balance) | Italian compliance | Verdict |
|---|---|---|---|---|---|---|
| **TeamSystem Contabilità in Cloud** (ex Reviso) | ✅ | ✅ Documented REST API | ✅ Vouchers/invoices onto real COA | ✅ `accounts.balance` + entries by period | ✅ | **Finalist #1** |
| **Odoo** (Enterprise, l10n_it) | ✅ | ✅ XML-RPC/JSON-RPC over whole ORM | ✅ `account.move` (full journal entries), `account.payment` with journal | ✅ `read_group` on `account.move.line` → trial balance, P&L | ✅ Native SDI (proxy K95IV18), withholding, Ri.Ba., DDT | **Finalist #2** |
| **TeamSystem Enterprise** (TSE in Cloud) | ✅ | ✅ Developer portal incl. accounting | ✅ likely ("accounting movements" services) | To be verified | ✅ | Fallback — oversized ERP |
| **Passepartout Passcom/Mexal** | ✅ | ⚠️ WebAPI documented but resources are document-oriented | ⚠️ Via PassBuilder/Sprix partner build | ⚠️ Same | ✅ | Fallback — partner-dependent |
| Fatture in Cloud (TeamSystem) | ❌ No chart of accounts, no double-entry | ✅ Excellent (OpenAPI, SDKs, webhooks) | ✅ (onto categories, not GL accounts) | ❌ Lists only, no balances | ✅ SDI via API | Complementary invoicing layer only |
| Zucchetti, Aruba, WK Genya, Dylog, etc. | Partial/No | ❌ API opacity or SDI-only | ❌ | ❌ | ✅ (SDI only for some) | Excluded |
| Xero, QuickBooks, Holded | ✅ (non-Italian) | ✅ Excellent APIs | ✅ | ✅ | ❌ No Italy/SDI support | Excluded |

---

## 2. CiC vs Odoo — Direct Comparison

Both finalists satisfy Requirement 01 (POST into a real GL by account and payment type) and Requirement 02 (GET balances and trial balance via API). The comparison below covers costs, technical capabilities, and implementation effort.

*Prices indicative (June 2026) — verify exact EUR with each vendor before signing.*

### Costs

| Dimension | TeamSystem CiC | Odoo Custom SaaS |
|---|---|---|
| **Pricing model** | Per transaction volume — users unlimited | Per user / month |
| **Entry cost** | ~€25–29/mese (piano base, ~6,000 entries/year) | ~€34–42/utente/mese (Custom plan) |
| **Typical TCA (5 users)** | ~€45–59/mese → **~€540–700/anno** | ~€190/mese → **~€2,300/anno** (licenze sole) |
| **Implementation (one-time)** | €0–2,000 — self-setup, no partner | €10,000–25,000 — partner required |
| **Year 1 total** | **~€540–2,700** | **~€12,300–27,500** |
| **Year 2+ annual** | ~€540–700 | ~€2,300 (5 users) |

### Technical Features

| Dimension | TeamSystem CiC | Odoo Custom SaaS |
|---|---|---|
| **True GL / double-entry** | Yes | Yes |
| **POST by GL account + payment (Req. 01)** | Yes — vouchers onto customizable COA; counterpart journal = payment method | Yes — `account.move` per GL line; `account.payment` with journal |
| **GET balances / trial balance (Req. 02)** | Yes — `accounts.balance` per account + period; REST endpoint ready for dashboards | Yes — `read_group` on `account.move.line`; full trial balance + P&L per period |
| **API type** | REST — documented at api-docs.reviso.com | XML-RPC / JSON-RPC — full ORM; most powerful option on the market |
| **API included in plan** | Yes — all plans | Custom plan only (Standard excludes API) |
| **SDI / FatturaPA** | Yes — via TeamSystem ecosystem | Yes — native `l10n_it_edi`, SDI proxy K95IV18 |
| **Withholding taxes / Ri.Ba. / DDT** | Yes | Yes |
| **Multi-company** | No | Yes — Custom plan |
| **Multi-country** | No — Italy only | Yes — 100+ localizations |
| **ERP expansion (hypothetical)** | Not possible — accounting ceiling | Full suite available without platform change |

### Implementation & Ease of Use

| Dimension | TeamSystem CiC | Odoo Custom SaaS |
|---|---|---|
| **Setup complexity** | Low — Italian product, guided onboarding | Medium-High — Italian partner required |
| **Time to go live** | Days to a few weeks | Weeks to months (partner project) |
| **Learning curve** | Low — intuitive Italian UI | Medium — broad interface; training + Odoo admin required |
| **Ongoing maintenance** | Low — fully managed SaaS | Medium — Odoo admin role, module compatibility on upgrades |
| **Free trial** | 14 giorni | Yes — odoo.com trial |
| **Support** | TeamSystem (Italian) — chat/email Base; phone Advanced | Italian partner + Odoo community |

---

## 3. Recommendation

1. **First choice: TeamSystem CiC** — satisfies both API requirements, Italian-native, predictable flat cost (~€540–700/anno), no partner needed, 14-day trial. Best fit for a single accounting entity.
2. **Second choice: Odoo Custom SaaS** — if TCA eventually needs multi-company, multi-country, or ERP expansion on the same platform. Year 1 cost is significantly higher (~€12,300–27,500) but Year 2+ licence cost (~€2,300/anno) is competitive once implementation is amortized.

**Suggested next steps:**
1. Run both trials in parallel — POST one real day of revenue, GET balances into a prototype dashboard.
2. Confirm CiC's REST API is sufficient for dashboard granularity needed; if yes, it wins on cost and simplicity.
3. Obtain TeamSystem's roadmap commitment before signing (December deadline).
4. If Odoo: engage an Italian partner immediately — December is tight for a partner-led project.

---

## 4. Main sources

- Contabilità in Cloud / Reviso: [api-docs.reviso.com](https://api-docs.reviso.com/) · [REST API (IT)](https://www.reviso.com/it/assistenza/articoli/rest-api/) · [pricing](https://www.teamsystem.com/store/fatturazione-e-contabilita/contabilita-in-cloud/prezzi/)
- Odoo: [Italian localization 18.0](https://www.odoo.com/documentation/18.0/applications/finance/fiscal_localizations/italy.html) · [External API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html) · [pricing configurator](https://www.odoo.com/pricing-configurator) · [OCA l10n-italy](https://github.com/OCA/l10n-italy)
- TeamSystem Enterprise: [tse.docs.teamsystem.cloud](https://tse.docs.teamsystem.cloud/)
- Fatture in Cloud: [developers.fattureincloud.it](https://developers.fattureincloud.it/) · [OpenAPI spec](https://github.com/fattureincloud/openapi-fattureincloud)
- Passepartout: [Passcom WebAPI (EduPass)](https://www.edupass.it/manuali/manualistica-passcom/manuale-prodotto?a=manuale-passweb-ecommerce%2Fconfigurazione%2Fpasscom--configurazione-gestionale%2Fweb-api-passcom)
