# Tennis Club Ambrosiano — Analisi software di contabilità cloud (carve-out, migrazione entro dicembre)

**Data analisi:** 12 giugno 2026
**Requisiti chiave:**
1. **Import (POST)** dei dati di ricavo dal gestionale del club, con dettaglio per **conto/categoria contabile** e **tipologia/metodo di pagamento** (e idealmente centro di ricavo).
2. **Estrazione (GET)** di dati consolidati (saldi, entrate/uscite aggregate, bilancino, scadenzario) per alimentare **dashboard dinamiche esterne** (Power BI, Looker Studio, app custom).
3. Conformità fiscale italiana: fatturazione elettronica SDI, IVA, corrispettivi; rilevante il regime **398/91** se il club opera come ASD/SSD (e-fattura obbligatoria dal 1/1/2024; dal 1/1/2026 passaggio da esclusione a esenzione IVA con nuovi adempimenti).

---

## 1. Tabella comparativa

| Software | API pubblica documentata | POST ricavi con conto + metodo pagamento | GET dati consolidati (saldi/bilancino) | Conformità Italia | Verdetto |
|---|---|---|---|---|---|
| **Fatture in Cloud** (TeamSystem) | ✅ Eccellente (OpenAPI, SDK 8 linguaggi, webhook) | ✅ Fatture, corrispettivi, prima nota cassa, con `payment_account`, `category`, `rc_center` | ⚠️ Solo liste filtrabili + 1 endpoint aggregato; **no piano dei conti, no bilancino** | ✅ SDI via API; corrispettivi solo registrazione (no invio AdE) | **Finalista** (se basta gestione "per cassa") |
| **Contabilità in Cloud** (ex Reviso, TeamSystem) | ✅ API REST contabile documentata | ✅ POST vouchers/fatture su piano dei conti vero | ✅ `accounts` con campo `balance`, movimenti per periodo | ✅ Prodotto italiano TeamSystem | **Finalista** (contabilità vera + API) |
| **Odoo** (Enterprise, l10n_it) | ✅ XML-RPC/JSON-RPC su tutto l'ORM | ✅ `account.move` (prima nota completa), `account.payment` con journal | ✅ `read_group` su `account.move.line` → trial balance, P&L | ✅ SDI nativo (codice K95IV18), ritenute, Ri.Ba., DDT | **Finalista** (più potente, più oneroso) |
| TeamSystem Enterprise (TSE in Cloud) | ✅ Portale developer con API contabilità | ✅ probabile (servizi "movimenti contabili") | Da verificare | ✅ | ERP sovradimensionato per un circolo |
| Passepartout Passcom/Mexal | ⚠️ WebAPI documentata (EduPass) ma risorse standard documentali | ⚠️ Via personalizzazione PassBuilder/Sprix di un partner | ⚠️ Idem | ✅ | Possibile, ma dipendenza da partner e sviluppo custom |
| Zucchetti (Digital Hub, Tieni il Conto) | ❌ Doc solo via commerciale/partner | ❌ non documentato pubblicamente | ❌ | ✅ (DH = solo SDI) | Escluso per opacità API |
| Aruba Fatturazione Elettronica | ✅ apidoc pubblico | ❌ Solo upload XML a SDI | ❌ Solo ricerca fatture/notifiche | ✅ (solo SDI) | Solo canale SDI, non backend contabile |
| Wolters Kluwer Genya | ❌ Nessuna API pubblica trovata | ❌ (import Excel/XML) | ❌ | ✅ | Escluso |
| Dylog / Datev Koinos / B.Point | ❌ (solo tracciati file o API fatturazione) | ❌ | ❌ | ✅ | Esclusi; Koinos/B.Point sono strumenti del commercialista |
| Sibill / Tot / Qonto / Agicap | ⚠️ API anche buone (Qonto, Agicap) | ❌ Non sono contabilità generale (fintech/tesoreria) | ❌ | ⚠️ | Categoria sbagliata; al più complementari |
| Xero | ✅ API e Reports API eccellenti (TrialBalance, P&L) | ✅ | ✅ | ❌ **Nessun supporto SDI/Italia** ("no sights for Italy") | Escluso |
| QuickBooks Online | ✅ API e Reports API eccellenti | ✅ | ✅ | ⚠️ Solo via add-on terzo (FOR S.r.l.); Intuit ritirata dalla Francia nel 2023 | Rischio fornitore doppio: sconsigliato |
| Holded | ⚠️ API key, no report consolidati | ✅ parziale | ❌ | ❌ Solo Spagna (Veri*factu/AEAT) | Escluso |

---

## 2. Approfondimento sui tre finalisti

### 2.1 Fatture in Cloud (TeamSystem) — API v2

Il developer portal migliore del mercato italiano: [developers.fattureincloud.it](https://developers.fattureincloud.it/), spec OpenAPI pubblica ([github.com/fattureincloud/openapi-fattureincloud](https://github.com/fattureincloud/openapi-fattureincloud)), SDK ufficiali in PHP, JS/TS, Python, Java, C#, Ruby, Go.

**Autenticazione e costi**
- OAuth 2.0 (Authorization Code / Device Code) **oppure token manuale che non scade** — ideale per uno script interno del club ([doc autenticazione](https://developers.fattureincloud.it/docs/authentication/)).
- **API incluse in tutte le licenze a pagamento** (nessun piano API separato). Piani indicativi 2026: Standard ~12 €/mese → Complete ~29-49 €/mese. **Attenzione: la funzione Corrispettivi richiede il piano Premium o superiore.**

**Import ricavi (POST)** — pienamente supportato per il caso d'uso del club:
- `POST /c/{company_id}/issued_documents` — fatture attive, note di credito, ricevute; ogni rata in `payments_list` ha `due_date`, `amount`, `status`, `paid_date` e **`payment_account`** (es. "Cassa bar", "Banca Intesa", "POS"): registrare la rata come pagata = registrare l'incasso sul conto.
- `POST /c/{company_id}/receipts` — **corrispettivi** (`till_receipt`/`sales_receipt`) con `payment_account` a livello documento, **`rc_center`** (centro di ricavo, es. "Campi", "Bar", "Scuola tennis") e **`category`** su ogni riga.
- `POST /c/{company_id}/cashbook` — prima nota cassa per entrate generiche (`amount_in` + `payment_account_in`).
- Anagrafiche gestibili via API: `GET/POST /settings/payment_accounts`, `/settings/payment_methods`, `/settings/vat_types` (incluse nature di esenzione per il 398/91); liste rapide in `/info/*`.

**Estrazione per dashboard (GET)** — il punto debole:
- Liste con filtri SQL-like (`q=date >= '2026-01-01' and amount_gross > 100`), fieldset, paginazione su `/issued_documents`, `/receipts`, `/cashbook`, `/received_documents`, `/taxes`.
- **Unico endpoint aggregato**: `GET /receipts/monthly_totals`. **Non esistono** endpoint di saldi per conto, bilancino o cashflow: l'aggregazione per le dashboard va fatta lato vostro (staging DB / Power Query / ETL), con sync incrementale via polling su `updated_at` o **webhook** (78 tipi di evento, formato CloudEvents).
- Nessun connettore Power BI/Looker nativo; esistono connettori ufficiali **Zapier** e **Make** (quest'ultimo con moduli per corrispettivi e cashbook).

**Limiti**: ~1.000 richieste/ora e ~20.000/mese **condivise tra tutte le app della stessa azienda** (il caricamento massivo iniziale va rateizzato). Invio SDI via API solo per documenti creati in FIC; trasmissione telematica corrispettivi all'AdE fuori perimetro (resta al Registratore Telematico).

**Verdetto**: perfetto per il requisito 1, parziale sul requisito 2 (aggregazione client-side). Non è una contabilità in partita doppia: niente piano dei conti né bilancio.

### 2.2 TeamSystem Contabilità in Cloud (ex Reviso)

Vera **contabilità generale cloud** con API REST documentata ([api-docs.reviso.com](https://api-docs.reviso.com/), [doc italiana](https://www.reviso.com/it/assistenza/articoli/rest-api/)).

- **POST**: creazione registrazioni/fatture via vouchers (es. [`POST /vouchers/drafts/customer-invoices`](https://www.reviso.com/it/assistenza/articoli/restful-api-esempio-crea-fattura-manuale/)) su un **piano dei conti vero**.
- **GET**: la risorsa `accounts` espone `accountNumber`, `accountType`, **`balance`** e i movimenti per esercizio/periodo (`GET /accounts/:n/accounting-years/:y/periods/:p/vouchers`) — è esattamente il "bilancino via API" che serve per le dashboard.
- Autenticazione a doppio token (`X-AppSecretToken` + `X-AgreementGrantToken`); ambiente demo gratuito (`?demo=true`, solo GET).
- **Riserva da sciogliere subito**: Reviso è stato rebrandizzato "Contabilità in Cloud" e venduto sul TeamSystem Store — **verificare con TeamSystem la roadmap del prodotto** prima di impegnarsi sulla migrazione di dicembre.

**Verdetto**: l'unico software italiano nativo che copre entrambi i requisiti con API pubblica documentata.

### 2.3 Odoo (Enterprise + localizzazione italiana)

- **Conformità**: localizzazione italiana nativa e matura ([doc ufficiale](https://www.odoo.com/documentation/18.0/applications/finance/fiscal_localizations/italy.html)): `l10n_it_edi` con **invio/ricezione SDI tramite il proxy accreditato Odoo** (codice destinatario `K95IV18`), ritenute, Ri.Ba., DDT, export XML liquidazione, PoS con stampanti RT per i corrispettivi.
- **API** ([External API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html)): XML-RPC/JSON-RPC con API key su tutto l'ORM — `create` su `account.move` (qualsiasi registrazione di prima nota, con righe per conto del piano dei conti) e `account.payment` (journal = metodo/conto d'incasso); `read_group`/`search_read` su `account.move.line` per saldi, trial balance e P&L per periodo. Nessun rate limit se su Odoo.sh/self-hosted.
- **Avvertenza**: su Odoo Online (SaaS) l'External API è disponibile **solo sui piani Custom**; in alternativa Odoo.sh o self-hosted.

**Verdetto**: il più potente e flessibile (copre tutto, incluse dashboard native), ma richiede un progetto di implementazione con partner — da valutare rispetto alla scadenza di dicembre.

---

## 3. Raccomandazione

La scelta dipende da una domanda che il carve-out impone di chiarire con il commercialista: **il club deve tenere internamente una contabilità in partita doppia (piano dei conti, bilancio), o gli basta gestire fatturazione/corrispettivi/incassi lasciando la contabilità generale allo studio?**

1. **Se basta la gestione "per cassa"** (fatture, corrispettivi, incassi per conto e metodo di pagamento): **Fatture in Cloud** è la scelta migliore — API più matura d'Italia, costi bassi, time-to-market rapidissimo. Dashboard costruite aggregando i dati grezzi in uno staging (script + Power BI/Looker).
2. **Se serve contabilità generale con saldi via API**: **TeamSystem Contabilità in Cloud (ex Reviso)**, previa conferma della roadmap prodotto; in subordine **Odoo Enterprise** con localizzazione italiana, se c'è budget/tempo per un'implementazione con partner.
3. **Architettura ibrida consigliata** se il club è ASD/SSD in 398/91: gestionale club → POST corrispettivi/fatture con `rc_center` per separare attività istituzionale/commerciale → estrazione liste + webhook → staging DB → dashboard. La classificazione 398/91 resta a carico del flusso di import.

**Esclusi e perché**: Zucchetti (documentazione API solo via partner), Aruba (solo SDI), Genya/Dylog/Koinos/B.Point (nessuna API pubblica), Xero/Holded (nessuna conformità Italia), QuickBooks (conformità solo via partner terzo + rischio ritiro dal mercato UE), Sibill/Tot/Qonto/Agicap (fintech/tesoreria, non contabilità).

**Prossimi passi suggeriti** (vincolo dicembre):
1. Chiarire con il commercialista il perimetro contabile post carve-out (per cassa vs partita doppia).
2. Aprire account di test: developer account gratuito Fatture in Cloud + demo Reviso/Contabilità in Cloud.
3. Proof-of-concept di 2 settimane: POST di un giorno di incassi reali (campi, bar, quote) con conti e metodi di pagamento + estrazione e dashboard di prova.
4. Interpellare TeamSystem sulla roadmap di Contabilità in Cloud prima della firma.

---

## 4. Fonti principali

- Fatture in Cloud: [developers.fattureincloud.it](https://developers.fattureincloud.it/) · [API Reference](https://developers.fattureincloud.it/api-reference/) · [OpenAPI spec (GitHub)](https://github.com/fattureincloud/openapi-fattureincloud) · [Autenticazione](https://developers.fattureincloud.it/docs/authentication/) · [Limiti e quote](https://developers.fattureincloud.it/docs/basics/limits-and-quotas/) · [Webhooks](https://developers.fattureincloud.it/docs/webhooks/) · [E-invoice](https://developers.fattureincloud.it/docs/guides/e-invoice-management/) · [API incluse nelle licenze](https://help.fattureincloud.it/help/articolo/666-integra-altri-software-fatture-cloud)
- Contabilità in Cloud / Reviso: [api-docs.reviso.com](https://api-docs.reviso.com/) · [REST API (IT)](https://www.reviso.com/it/assistenza/articoli/rest-api/) · [Esempio crea fattura](https://www.reviso.com/it/assistenza/articoli/restful-api-esempio-crea-fattura-manuale/) · [Rebranding](https://www.reviso.com/it/assistenza/articoli/reviso-diventa-contabilita-in-cloud/) · [TeamSystem Store](https://www.teamsystem.com/store/contabilita-in-cloud/funzionalita/registrazioni-contabili/)
- Odoo: [Localizzazione Italia 18.0](https://www.odoo.com/documentation/18.0/applications/finance/fiscal_localizations/italy.html) · [External API](https://www.odoo.com/documentation/18.0/developer/reference/external_api.html) · [OCA l10n-italy](https://github.com/OCA/l10n-italy)
- TeamSystem Enterprise: [tse.docs.teamsystem.cloud](https://tse.docs.teamsystem.cloud/) · TS Digital: [API TS Digital Invoice](https://www.teamsystem.com/store/ts-digital-invoice/funzionalita/api/)
- Zucchetti: [Integrazione Digital Hub per terzi](https://help.zucchetti.it/cms/kb/soluzioni/fatturazione-elettronica/digital-hub/scopri-e-informati/faq/integrazioni/non-ho-gestionale-zucchetti-come-posso-integrarmi-con-digital-hub.html) · [Tieni il Conto PRO](https://www.zucchetti.it/website/cms/prodotto/8310-tieni-il-conto-pro.html)
- Aruba: [apidoc v1](https://fatturazioneelettronica.aruba.it/apidoc/docs.html) · [apidoc v2](https://fatturazioneelettronica.aruba.it/apidoc/v2/docs.html)
- Passepartout: [WebAPI Passcom (EduPass)](https://www.edupass.it/manuali/manualistica-passcom/manuale-prodotto?a=manuale-passweb-ecommerce%2Fconfigurazione%2Fpasscom--configurazione-gestionale%2Fweb-api-passcom) · [Contabilità](https://www.passepartout.net/software/imprese/gestione-contabilita)
- Xero: [Reports API](https://developer.xero.com/documentation/api/accounting/reports) · [No Italia (community)](https://community.xero.com/business/discussion/115295423) — QuickBooks: [Reports API](https://developer.intuit.com/app/developer/qbo/docs/workflows/run-reports) · [FOR:QUICKBOOKS](https://www.quickbooksitalia.com/) · [Ritiro dalla Francia](https://blogs.intuit.com/2023/06/12/discontinuation-of-quickbooks-in-france/) — Holded: [API](https://developers.holded.com/reference)
- ASD/SSD 398/91: [TeamSystem Magazine](https://www.teamsystem.com/magazine/sport-e-wellness/regime-forfettario-asd-ssd/) · [Novità 2026](https://fatturapro.click/associazioni-sportive-dilettantistiche-regime-398-91-e-novita-2026/) · [E-fattura ASD dal 2024](https://www.aruba.it/magazine/fatturazione-elettronica/associazioni-sportive-dilettantistiche-fattura-elettronica-obbligatoria-dal-1-gennaio-2024.aspx)
- Verticali sportivi: [TeamSystem Sportivi in Cloud](https://www.teamsystem.com/sport/sportivi-in-cloud/) · [Wansport](https://wansport.com/gestionale-per-tennis-club/) · [Golee](https://golee.it/gestionale-per-circoli/)
