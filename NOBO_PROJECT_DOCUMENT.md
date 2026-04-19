# NOBO — NoBorders
## Piattaforma di Distribuzione per Cinema Indipendente
### Documento di Progetto — Aprile 2026

---

## INDICE

1. [Executive Summary](#1-executive-summary)
2. [Problema e Opportunità](#2-problema-e-opportunità)
3. [Soluzione: Nobo](#3-soluzione-nobo)
4. [Analisi di Mercato](#4-analisi-di-mercato)
5. [Modello di Business](#5-modello-di-business)
6. [Analisi Competitiva e Rischi](#6-analisi-competitiva-e-rischi)
7. [Architettura Tecnica](#7-architettura-tecnica)
8. [Roadmap e MVP](#8-roadmap-e-mvp)
9. [Struttura del Team](#9-struttura-del-team)
10. [Piano Finanziario](#10-piano-finanziario)
11. [Funding e Grant Disponibili](#11-funding-e-grant-disponibili)
12. [Call to Action](#12-call-to-action)

---

## 1. Executive Summary

**Nobo (NoBorders)** è una piattaforma SaaS di distribuzione per cinema indipendente italiano, con focus iniziale sulla regione Marche. Il progetto nasce dalla necessità reale dei filmmaker indipendenti di raggiungere il pubblico senza passare attraverso i tradizionali intermediari distributivi che trattengono il 40-50% dei ricavi.

**La proposta di valore centrale:** Nobo redistribuisce il 70% dei ricavi ai creator — contro il 50% di MUBI e il 60% di Vimeo OTT — costruendo un ecosistema sostenibile dove il successo della piattaforma dipende direttamente dal successo dei filmmaker.

**Stato attuale:** Fase pre-seed, documento di concept.

**Obiettivo fundraising:** €150.000 per MVP + 12 mesi di operatività.

---

## 2. Problema e Opportunità

### 2.1 Il Problema del Cinema Indipendente Italiano

Il cinema indipendente italiano produce circa **1.200 cortometraggi e 200 lungometraggi** all'anno (fonte: ANICA 2025). Il 94.5% delle società di produzione cinematografica italiane sono micro-imprese (0-9 dipendenti). Questi filmmaker affrontano tre problemi strutturali:

**1. Distribuzione inaccessibile**
I distributori tradizionali richiedono accordi esclusivi e trattengono il 40-65% dei ricavi lordi. Per i cortometraggi, la distribuzione commerciale è quasi inesistente: vengono distribuiti tramite festival (spesso gratuiti) o caricati su YouTube senza monetizzazione strutturata.

**2. Trasparenza zero**
I report di rendiconto dei distributori tradizionali arrivano con 6-18 mesi di ritardo e sono notoriamente opachi. I filmmaker non sanno quante persone hanno visto il loro lavoro, quando, e su quale piattaforma.

**3. Audience frammentata**
Il pubblico del cinema indipendente e d'autore esiste ma è disperso. Non ha una piattaforma dedicata dove cercare attivamente contenuti indipendenti italiani — naviga tra Netflix (che non acquisisce indipendenti small-budget), MUBI (curato, elitario, prevalentemente internazionale), e YouTube (non adatto a distribuzione professionale).

### 2.2 Il Problema dei Festival e delle Istituzioni

Pesaro Film Festival, insieme ad altri 60+ festival italiani, gestisce la propria distribuzione digitale con strumenti legacy, contratti cartacei, e sistemi non integrati. Le cinetecche (Cineteca Nazionale, Fondazione Cineteca Italiana) custodiscono oltre 80.000 titoli ma hanno accesso digitale limitato e frammentato.

---

## 3. Soluzione: Nobo

### 3.1 Vision

> "Una piattaforma dove il cinema indipendente trova il suo pubblico, e il pubblico trova il cinema che la televisione non mostra."

Nobo non è un clone di Netflix. È l'infrastruttura di distribuzione che mancava al cinema indipendente italiano: trasparente, equa verso i creator, costruita per servire sia il pubblico (B2C) che le istituzioni (B2B).

### 3.2 Differenziatori Chiave

| Dimensione | Concorrenti | Nobo |
|---|---|---|
| Revenue share creator | 50-60% | **70%** |
| Trasparenza dati | Report ritardati e opachi | **Dashboard real-time** |
| Focus geografico | Globale/elitario | **Italia, radicato nel territorio** |
| Modello distribuzione | Solo SVOD | **Ibrido: SVOD + TVOD + B2B licensing** |
| Relazione con creator | Transazionale | **Ecosistema collaborativo** |

### 3.3 Prodotto: Funzionalità Core

**Per il pubblico (B2C):**
- Catalogo curato di cortometraggi e lungometraggi indipendenti italiani
- Pay-per-view per singoli titoli (€1,99-3,99) — subscription mensile in Fase 2
- Profilo filmmaker con storia del progetto, note di regia, materiali aggiuntivi

**Per i filmmaker (Creator Portal):**
- Upload e gestione del catalogo
- Dashboard analytics real-time (view, completion rate, geography, revenue)
- Report di rendiconto trasparente e immediato
- Gestione dei diritti e delle finestre di distribuzione

**Per festival e istituzioni (B2B Portal):**
- Licensing di singoli titoli o pacchetti tematici
- Screening room virtuale per selezioni di giuria
- Integrazione API per siti web di festival
- Reports personalizzati per rendiconto istituzionale

---

## 4. Analisi di Mercato

### 4.1 Mercato Italiano del Cinema

- **Box office 2025:** €496,6 milioni (+9% YoY)
- **Biglietti venduti:** 68,4 milioni
- **Film italiani:** 32,7% degli incassi (€162,4 milioni)
- **Industria audiovisiva totale 2024:** €16,8 miliardi

### 4.2 Mercato Indipendente e Indie

Il segmento indie è strutturalmente sottoservito:
- 94,5% delle società di produzione = micro-imprese
- Crescita produzione film italiani, ma ricavi non crescono proporzionalmente
- Accesso agli schermi difficile per distributori indipendenti
- Pubblico cinema d'autore in crescita ma non monetizzato digitalmente

### 4.3 Mercato Target Iniziale: Marche

**Fase 1 (MVP):** Regione Marche
- Popolazione: ~1,5 milioni
- Mercato diretto limitato → **non è il mercato, è il laboratorio**
- Presenza del Pesaro Film Festival (61a edizione, giugno 2025): festival storico, alternativo a Venezia
- Marche Film Commission attiva con budget €1,2M/anno per produzioni
- Network di relazioni esistenti: valore strategico superiore alla dimensione di mercato

**Fase 2 (Mese 12+):** Italia
- Mercato potenziale: 60+ festival, 338 schermi rete cinema indipendente (UniCi)
- Istituzioni: Cineteca Nazionale (60.000 titoli), Fondazione Cineteca Italiana (20.000 titoli)
- Scuole e università: €24M budget MIC per "Cinema e Immagini per la Scuola"

### 4.4 TAM / SAM / SOM

| Mercato | Dimensione | Note |
|---|---|---|
| **TAM** (cinema indipendente EU) | €2,1B | Stima su distribuzione digitale indie europea |
| **SAM** (cinema indie italiano) | €180M | 8,5% del TAM, stima distribuzione digitale |
| **SOM** (target realistico Y3) | €2,5M | 50 festival + 5.000 abbonati + creator fees |

---

## 5. Modello di Business

### 5.1 Perché la Subscription a €3/mese Non Funziona

L'idea iniziale di una subscription a €3/mese è stata analizzata e scartata. I conti non tornano:

| Scenario | Abbonati | Revenue lorda/mese | Revenue netta Nobo (30%) |
|---|---|---|---|
| Ottimistico anno 1 | 500 | €1.500 | €450 — copre 2 giorni di server |
| Realistico anno 1 | 150 | €450 | €135 — sotto i costi minimi |
| Breakeven infrastruttura | 2.300 | €6.900 | €2.070 |
| Breakeven operativo | 15.000+ | €45.000 | €13.500 |

15.000 abbonati in Italia su cinema indipendente regionale non è un obiettivo anno 1 — è un obiettivo anno 5 se tutto va bene. A €3/mese non esiste business case sostenibile.

**Il problema strutturale:** l'italiano medio nel 2026 ha già Netflix, Prime Video, Disney+, RaiPlay. Aggiungere una subscription per contenuti locali sconosciuti richiede un motivo fortissimo. "È etica" è un messaggio, non un motivo per pagare ogni mese.

**La subscription non scompare — viene posticipata alla Fase 2**, quando il catalogo è validato (50+ titoli), la domanda dimostrata (utenti ricorrenti), e il prezzo giustificabile (minimo €5,99/mese).

### 5.2 Strategia di Monetizzazione: B2B First

**Anno 1: sopravvivenza tramite B2B + Pay-per-View**

Il modello ibrido con due gambe reali per l'anno 1:

**Gamba 1: B2B — Licensing Istituzionale (revenue primaria anno 1)**

Festival e istituzioni pagano contratti prevedibili, indipendentemente dal volume utenti B2C. È revenue reale con sales cycle di 3-6 mesi.

- Festival (es. Pesaro Film Festival): €2.000-10.000 per evento — piattaforma di screening + licensing titoli
- Scuole e università: €500-1.500 per titolo/anno (budget MIC €24M per Cinema e Scuola)
- Cinetecche e biblioteche: accordi pluriennali €5.000-20.000
- Revenue stimata anno 1: €20.000-40.000

Il B2B risolve anche il chicken-and-egg: il pubblico arriva tramite i festival, non tramite marketing diretto. 3.000 persone al Pesaro Film Festival vedono il brand Nobo — alcune si registrano, il catalogo cresce organicamente.

**Gamba 2: B2C — Pay-per-View (validazione domanda)**

Nessuna subscription nell'MVP. Ogni titolo ha un prezzo: €1,99-3,99. Abbassa la friction per l'utente (non si "abbona" a qualcosa di sconosciuto), genera revenue immediata anche con pochi utenti, e testa la domanda reale prima di promettere contenuto continuo.

- Pay-per-view: €1,99-3,99 per titolo
- Creator incassa il 70% di ogni transazione — immediato, trasparente
- Revenue stimata anno 1: €5.000-15.000

**Gamba 3 (Fase 2, mese 12+): Subscription**

Quando il catalogo supera i 50 titoli e gli utenti ricorrenti sono dimostrati, si introduce la subscription a **€5,99/mese** (non €3). A quel punto il prezzo è giustificabile dal volume di contenuto e dalla fidelizzazione dimostrata.

### 5.3 Modello "Supporter" come Alternativa Etica

Per rendere concreto il posizionamento etico, si aggiunge un livello opzionale: la piattaforma è navigabile gratuitamente (trailer, info), il pay-per-view è l'accesso al film, e chi vuole supportare il filmmaker può fare tipping diretto oltre al prezzo base.

Questo rende il messaggio di trasparenza e supporto ai creator **operativo**, non solo narrativo.

### 5.4 Revenue Share Creator

| Modello | Revenue share creator |
|---|---|
| Pay-per-view (B2C) | **70%** del netto |
| Licensing B2B (festival/istituzioni) | **60%** del netto (negoziato) |
| Subscription Fase 2 | **70%** del netto |

Nessuna doppia tassazione: non esiste fee mensile al filmmaker **e** revenue share. Si sceglie uno dei due modelli.

### 5.5 Proiezioni Finanziarie (Conservative)

| | Anno 1 | Anno 2 | Anno 3 |
|---|---|---|---|
| B2B Revenue | €30.000 | €100.000 | €250.000 |
| Pay-per-View (B2C) | €10.000 | €50.000 | €120.000 |
| Subscription (da mese 12) | — | €30.000 | €150.000 |
| **Totale Revenue** | **€40.000** | **€180.000** | **€520.000** |
| Costi operativi | €100.000 | €170.000 | €280.000 |
| **EBITDA** | **-€60.000** | **+€10.000** | **+€240.000** |

*Anno 1 in perdita coperto dal seed round. Breakeven operativo previsto a mese 20-22.*

---

## 6. Analisi Competitiva e Rischi

### 6.1 Competitor Diretti

| Piattaforma | Punti di forza | Punti deboli vs Nobo |
|---|---|---|
| **MUBI** | Brand forte, qualità curatoriale | 50/50 split, no italiano, elitario |
| **Vimeo OTT** | Infrastruttura robusta, API | Non curato, no community italiana |
| **Shift72** | Ottimo per festival | No B2C, no creator portal |
| **RaiPlay** | Reach enorme | No indipendenti, no creator monetization |

### 6.2 Rischi Principali e Mitigazioni

#### Rischio 1: Mercato B2C troppo piccolo — ALTO
**Scenario negativo:** Marche non genera abbonati sufficienti per coprire i costi fissi di streaming.

**Mitigazione:**
- B2B (festival, istituzioni) genera revenue indipendentemente dal volume abbonati
- Focus iniziale su Marche come laboratorio e PR, non come mercato primario
- Espansione Italia a mese 12 con relazioni già costruite

#### Rischio 2: Subscription fatigue — MEDIO
**Scenario negativo:** Utenti già abbonati a Netflix, Disney+, MUBI non aggiungono un'altra subscription.

**Mitigazione:**
- Pay-per-view come entry point: €1,99 è low-commitment
- Posizionamento come "supporto ai creator" non come entertainment puro
- Opzione di "tipping" diretto senza subscription

#### Rischio 3: Chicken-and-egg (contenuto vs. utenti) — ALTO
**Scenario negativo:** Nessun creator carica senza utenti, nessun utente si abbona senza contenuto.

**Mitigazione:**
- Partnership pre-lancio con 10-20 filmmaker marchigiani (già contatti esistenti)
- Accordo con Pesaro Film Festival per accesso al catalogo storico
- B2B (festival, scuole) funziona indipendentemente dal volume B2C

#### Rischio 4: Costi tecnici (streaming) — MEDIO
**Scenario negativo:** I costi di CDN e storage superano i ricavi nella fase early.

**Mitigazione:**
- Cloudflare Stream per MVP: costo fisso basso, nessun setup complesso
- Encoding on-demand, non in batch — si paga solo quando si usa
- Budget tecnico allocato nel seed round (vedi sezione 10)

#### Rischio 5: Concorrenza da piattaforme grandi — BASSO (breve termine)
Netflix, Amazon, MUBI non sono interessati al segmento micro-indie italiano. Il rischio aumenta se Nobo scala a 100k+ utenti — un problema che vale la pena avere.

### 6.3 Il Dubbio Fondamentale (Onestà verso gli Investitori)

Creare una piattaforma streaming è costoso, complesso, e il mercato è saturo. Il vero rischio non è la concorrenza dei big player, ma la **mancanza di domanda sufficiente** nel segmento target.

**La risposta onesta:** Nobo non è un business di scala immediata. È un'infrastruttura culturale che richiede 2-3 anni per validare il modello B2C e 1 anno per il modello B2B. Il seed round non deve finanziare la crescita — deve finanziare la **validazione**.

---

## 7. Architettura Tecnica

### 7.1 Principi di Design

- **API-first** — ogni funzionalità è accessibile via API per integrazioni B2B e futura espansione su altri device (mobile, TV)
- **GDPR-compliant by design** — data residency EU su tutti i servizi
- **Mobile-first frontend** — il pubblico del cinema indie usa smartphone
- **Astratto dal device** — la logica di business è separata dal layer presentazione, pronta per web, mobile e TV

### 7.2 Stack Tecnologico

| Layer | Tecnologia | Motivo |
|---|---|---|
| Frontend | TanStack Start + TanStack Router | SSR, routing tipizzato, API-first |
| Styling | CSS Modules | Performance, nessuna dipendenza runtime |
| Auth | Better Auth | Cookie web + bearer token per mobile futuro, self-hosted EU |
| Database | PostgreSQL + Redis | Relazionale + cache, hosted su Railway Frankfurt |
| ORM | Drizzle | Type-safe, zero runtime overhead |
| Video | Cloudflare Stream | Encoding incluso, CDN globale, pricing pay-as-you-go |
| Storage | Cloudflare R2 | Zero egress fee, S3-compatible |
| Pagamenti | Stripe + Stripe Connect | Pay-per-view + payout automatico ai creator |
| Monitoring | Sentry + PostHog | Error tracking + analytics GDPR-safe, self-hostable |
| Deploy | Railway (backend + DB) + Vercel (frontend) | Semplice, economico, EU region |
| CI/CD | GitHub Actions | Standard, gratuito |

### 7.3 Architettura a Alto Livello

```
┌─────────────────────────────────────────────────────────────┐
│                        UTENTI                               │
│    B2C (Web/Mobile/TV*)   │   B2B (Festival/Istituzioni)    │
└──────────────┬────────────┴──────────────┬──────────────────┘
               │                            │
               ▼                            ▼
┌─────────────────────────────────────────────────────────────┐
│              NOBO WEB APP (TanStack Start)                   │
│    Public Site │ Viewer App │ Creator Portal │ Admin         │
└────────────────────────┬────────────────────────────────────┘
                         │ createServerFn
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   NOBO API LAYER                             │
│   Auth │ Catalog │ Video │ Payments │ Analytics │ Webhooks   │
└───┬──────────────────┬─────────────────┬────────────────────┘
    │                  │                 │
    ▼                  ▼                 ▼
┌──────────┐  ┌─────────────────┐  ┌──────────────────────┐
│PostgreSQL│  │Cloudflare Stream│  │Stripe Connect        │
│+ Redis   │  │+ R2 Storage     │  │(Payments + Payouts)  │
└──────────┘  └─────────────────┘  └──────────────────────┘

* TV (Apple TV, Android TV) pianificata in Fase 3 via Expo
```

### 7.4 Sicurezza e Compliance

- **GDPR:** Consent management, right to erasure, tutti i servizi con data residency EU
- **Protezione contenuti (MVP):** Signed URL con TTL (impedisce hotlinking — sufficiente per MVP)
- **DRM completo (Fase 3+):** Widevine + FairPlay se richiesto da contenuti premium
- **Pagamenti:** Nessun dato carta su server Nobo — tutto su Stripe (PCI DSS compliant)

### 7.5 Stima Costi Tecnici

| Voce | MVP (mese 1-12) | Growth (mese 12-24) |
|---|---|---|
| Video hosting (Cloudflare Stream) | €100-300/mese | €500-1.500/mese |
| Database e backend (Railway) | €50-150/mese | €200-500/mese |
| CDN e assets | €20-50/mese | €100-300/mese |
| Monitoring e tools | €50/mese | €150/mese |
| **Totale infrastruttura** | **€220-550/mese** | **€950-2.450/mese** |

---

## 8. Roadmap e MVP

### 8.1 Definizione di MVP

L'MVP di Nobo non è "una piattaforma streaming completa". È il **minimo necessario per validare tre ipotesi:**

1. I filmmaker marchigiani caricano contenuti su Nobo se la revenue share è 70%
2. Il pubblico locale paga (anche €1,99) per guardare cinema indipendente locale
3. Un festival o istituzione firma un accordo B2B con Nobo entro 6 mesi

### 8.2 Fasi di Sviluppo

#### Fase 0 — Fondamenta (Mesi 0-2) | Budget: €15.000
- Setup infrastruttura cloud (Cloudflare, Railway, Stripe)
- Backend API core: auth, catalog, user management
- Schema database
- Creator portal: upload video, metadata, gestione titoli
- **Deliverable:** Filmmaker può caricare un film e definire prezzo

#### Fase 1 — MVP Pubblico (Mesi 2-5) | Budget: €40.000
- Frontend pubblico: homepage, catalogo, pagina film, pagina filmmaker
- Video player con streaming adattivo
- Sistema pagamenti: **pay-per-view only** (no subscription nell'MVP)
- Payout automatico ai creator via Stripe Connect
- Dashboard analytics creator (views, revenue, geografia)
- **Deliverable:** 10 filmmaker caricati, 100 utenti beta, primo film venduto

#### Fase 2 — B2B e Subscription (Mesi 5-10) | Budget: €35.000
- B2B Portal: screening room, licensing request, API per festival
- Miglioramenti UX basati su feedback utenti
- **Introduzione subscription €5,99/mese** — solo se catalogo >50 titoli e utenti ricorrenti dimostrati
- Integrazioni: Marche Film Commission, Pesaro Film Festival
- **Deliverable:** 1 accordo B2B firmato, 500 utenti registrati, subscription live

#### Fase 3 — Espansione (Mesi 10-18) | Budget: €60.000
- Espansione a livello nazionale
- Sottotitoli e localizzazione
- Funzionalità community (reviews, liste, seguire filmmaker)
- **Deliverable:** Presenza a livello nazionale, 5.000 utenti

### 8.3 KPI di Validazione

| KPI | Obiettivo Mese 6 | Obiettivo Mese 12 |
|---|---|---|
| Titoli in catalogo | 30 | 100 |
| Utenti registrati | 200 | 1.000 |
| Acquisti pay-per-view | 50 | 300 |
| Revenue mensile | €500 | €3.000 |
| Accordi B2B | 1 | 3 |
| NPS filmmaker | >7 | >8 |

---

## 9. Struttura del Team

### 9.1 Team Attuale

Il progetto parte con due attori:

| Ruolo | Persona | Responsabilità | Impegno |
|---|---|---|---|
| **CTO / Lead Developer** | Valerio Narcisi | Architettura, sviluppo full-stack, infrastruttura, product | Part-time → full-time post-seed |
| **Commercial / BD** | Fabbrika Kreativa | Relazioni filmmaker, B2B sales, eventi, comunicazione | Commerciale |

**Gap riconosciuti esplicitamente:**
- Nessuna figura marketing/growth dedicata (allocata nel budget come hiring mese 12)
- Curation editoriale gestita da Fabbrika nella fase iniziale — serve processo definito entro mese 3
- Nessun legale interno — esternalizzato (budget allocato)

Questa è la realtà operativa dell'anno 1. Non gonfiare il team su carta per impressionare investitori: chi ha esperienza lo vede subito e perde fiducia.

### 9.2 Hiring Plan (Post-Seed, subordinato ai ricavi)

| Timing | Ruolo | Trigger |
|---|---|---|
| Mese 12 | Marketing / Growth manager | Quando B2B genera >€3.000/mese ricorrenti |
| Mese 18 | Junior developer (frontend) | Quando la roadmap Fase 3 inizia |
| Mese 24 | Business development (B2B nazionale) | Quando si espande fuori Marche |

---

## 10. Piano Finanziario

### 10.1 Utilizzo del Seed Round (€150.000)

| Categoria | Importo | % | Note |
|---|---|---|---|
| Compenso CTO/Dev (12 mesi) | €0 | 0% | Fondatore — coperto da equity |
| Infrastruttura tecnica (12 mesi) | €15.000 | 10% | Server, CDN, video hosting, tools |
| B2B Sales & eventi (Fabbrika) | €20.000 | 13% | Festival, presentazioni, PR |
| Marketing e acquisizione utenti | €35.000 | 23% | Contenuto, social, presenza fisica festival |
| Legale, GDPR, contratti creator | €15.000 | 10% | Onboarding legale, template contratti |
| Consulente grant (MIC/Creative Europe) | €10.000 | 7% | Riduce fabbisogno equity futura |
| Riserva operativa | €27.000 | 18% | Buffer + freelancer frontend se roadmap si ingolfa |
| Strumenti SaaS e licenze | €10.000 | 7% | Stripe fees, Sentry, PostHog, etc. |
| **Totale** | **€132.000** | **88%** | Margine €18k per opportunità non previste |

### 10.2 Breakeven

Con i ricavi proiettati (tabella sezione 5.4), il breakeven operativo è previsto a **mese 20-24**, assumendo crescita lineare conservativa.

Il modello B2B (festival, istituzioni) è il driver principale per raggiungere il breakeven — è revenue prevedibile e non dipendente dal volume di utenti B2C.

---

## 11. Funding e Grant Disponibili

### 11.1 Grant Pubblici Accessibili

Nobo come piattaforma (non come produzione cinematografica) può accedere a:

**Regione Marche:**
- Budget annuale audiovisivo: €1,2M — principalmente per produzioni, non piattaforme
- Possibile accesso come "progetto di promozione del cinema regionale"
- Contatto: Marche Film Commission (Ancona)

**MIC — Ministero della Cultura:**
- Budget 2025 cinema e audiovisivo: €696M totali
- Piano Nazionale Cinema e Immagini per la Scuola: €24M — potenziale partner come piattaforma distributiva per scuole
- Contatto: cinema.cultura.gov.it

**EU — Creative Europe Media:**
- European Film Distribution: €34M (2026) — per distributori che portano film europei in altri paesi UE
- **Nobo può qualificarsi** come distributore digitale se porta contenuti italiani a un pubblico europeo
- Deadline tipiche: Q1/Q2 annuale

**SIMEST / CDP (Cassa Depositi e Prestiti):**
- Finanziamenti agevolati per PMI innovative
- Nobo come startup tech-culturale può qualificarsi

### 11.2 Raccomandazione

Prima di chiudere il seed round, ingaggiare un consulente specializzato in grant MIC e Creative Europe. I grant pubblici possono ridurre del 30-40% il fabbisogno di equity.

---

## 12. Call to Action

### Per gli Investitori

Nobo cerca **€150.000 in seed financing** per:
- Costruire e validare l'MVP in 12 mesi
- Stabilire 3+ partnership B2B con festival e istituzioni italiane
- Raggiungere 1.000 utenti registrati con un NPS > 7

**Struttura proposta:** SAFE note o equity minima (10-15%) con cap di valutazione a €1M.

**Cosa offriamo oltre al ritorno economico:** la possibilità di partecipare alla costruzione di un'infrastruttura culturale per il cinema indipendente italiano — con trasparenza totale su impatto, revenue, e direzione del progetto.

### Per i Filmmaker e Registi

Nobo non è un distributore che vi chiede i diritti in cambio di una promessa. È una piattaforma dove voi mantenete il controllo del vostro lavoro, incassate il 70% di ogni visione, e avete accesso in tempo reale ai dati del vostro pubblico.

Se avete un cortometraggio o un lungometraggio indipendente e volete essere tra i filmmaker fondatori della piattaforma, contattateci.

### Per i Festival e le Istituzioni

Nobo può diventare l'infrastruttura digitale del vostro festival: screening room virtuale, licensing semplificato, analytics del pubblico, e integrazione con il vostro sito web via API. Parliamoci prima dell'estate.

---

## Appendice: Glossario

- **SVOD:** Subscription Video on Demand — abbonamento mensile per accesso illimitato
- **TVOD:** Transactional Video on Demand — pagamento per singolo titolo (pay-per-view)
- **CDN:** Content Delivery Network — infrastruttura globale per delivery veloce dei video
- **DRM:** Digital Rights Management — protezione tecnologica contro la copia non autorizzata
- **HLS/DASH:** Protocolli di streaming adattivo — regolano automaticamente la qualità video in base alla connessione
- **Seed Round:** Primo round di finanziamento istituzionale per una startup
- **SAFE:** Simple Agreement for Future Equity — strumento finanziario per investimenti early-stage
- **NPS:** Net Promoter Score — metrica di soddisfazione utenti (0-10)

---

*Documento redatto in Aprile 2026*
*Fabbrika Kreativa / Nobo — NoBorders*
*Contatto: valerio.narcisi@gmail.com*
