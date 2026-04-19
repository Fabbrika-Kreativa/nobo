# Nobo — Technical Specification
## Aprile 2026 — v0.1

---

## INDICE

1. [Scope e Obiettivi](#1-scope-e-obiettivi)
2. [Stack Tecnologico](#2-stack-tecnologico)
3. [Schema Database](#3-schema-database)
4. [Architettura API](#4-architettura-api)
5. [Video Pipeline](#5-video-pipeline)
6. [Autenticazione e Autorizzazione](#6-autenticazione-e-autorizzazione)
7. [Pagamenti e Payout](#7-pagamenti-e-payout)
8. [Frontend: Struttura e Pagine](#8-frontend-struttura-e-pagine)
9. [Sicurezza e GDPR](#9-sicurezza-e-gdpr)
10. [Deployment e Infrastruttura](#10-deployment-e-infrastruttura)
11. [MVP Scope: Cosa È Fuori](#11-mvp-scope-cosa-è-fuori)

---

## 1. Scope e Obiettivi

Questo documento descrive le specifiche tecniche per il **MVP di Nobo** (Fase 0 + Fase 1, mesi 0-5).

### Utenti del sistema

| Ruolo | Descrizione |
|---|---|
| `viewer` | Utente B2C — sfoglia catalogo, acquista/guarda film |
| `filmmaker` | Creator — carica film, gestisce catalogo, vede analytics e revenue |
| `admin` | Nobo staff — approva film, gestisce piattaforma |
| `b2b_partner` | Festival/istituzione — accede a screening room, richiede licenze |

### Funzionalità MVP (Fase 0-1)

- Filmmaker carica film, inserisce metadata, definisce prezzo
- Admin approva/rifiuta film prima della pubblicazione
- Viewer sfoglia catalogo, acquista singolo titolo (pay-per-view), guarda film
- Payout automatico al filmmaker (70%) a ogni acquisto
- Filmmaker vede dashboard: views, revenue, geografia
- Autenticazione via email/password + OAuth (Google)

### Fuori scope MVP

- Subscription mensile (Fase 2)
- B2B portal (Fase 2)
- App mobile nativa (Fase 3)
- DRM (Fase 3+)
- Sottotitoli multilingua (Fase 3)
- Tipping diretto (Fase 2)

---

## 2. Stack Tecnologico

### Decisioni e motivazioni

| Layer | Tecnologia | Motivo |
|---|---|---|
| Frontend | TanStack Start + TanStack Router | SSR, file-based routing, coerente con setup personale |
| Server calls | `createServerFn` (TanStack Start) | Unico pattern client→server, no tRPC, no raw fetch |
| Data fetching | TanStack Query (`queryOptions` + `useSuspenseQuery`) | Cache, invalidation, coerente con TanStack stack |
| Styling | CSS Modules | Zero Tailwind, zero CSS-in-JS, una `.module.css` per componente |
| Backend | Node.js + Fastify | Leggero, veloce, TypeScript nativo — per webhook e API esterne |
| Database | PostgreSQL 16 | Relazionale, solido, hosted su Railway |
| Cache/Queue | Redis (Upstash) | Rate limiting, job queue |
| ORM | Drizzle ORM | Type-safe, leggero, tipi sempre inferiti mai scritti a mano |
| Auth | Better Auth | Team/org support, bearer token per future mobile, GDPR EU |
| Video | Cloudflare Stream | Encoding incluso, CDN globale, pricing semplice |
| Storage | Cloudflare R2 | S3-compatible, zero egress fee, file originali |
| Pagamenti | Stripe + Stripe Connect | Subscription + one-time + marketplace payout |
| Error handling | neverthrow (`Result` / `ResultAsync`) | Errori come valori, no try/catch per errori di dominio |
| Validation | Zod | Source of truth per tutti i tipi — mai TypeScript scritto a mano |
| Email | Resend | Developer-friendly, template React |
| Monitoring | Sentry + PostHog | Error tracking + product analytics GDPR-safe |
| CI/CD | GitHub Actions | Standard, gratuito per repo privati |
| Deploy | Railway (backend + DB) + Vercel (frontend) | Semplice, economico |

### Monorepo Structure

```
nobo/
├── apps/
│   ├── web/          # TanStack Start app (viewer + filmmaker portal + admin)
│   └── api/          # Fastify backend (webhook receiver: Stripe, Cloudflare, Better Auth)
├── packages/
│   ├── db/           # Drizzle schema + migrations
│   ├── domain/       # Zod schemas, branded types, business rules — framework-agnostic
│   └── utils/        # ResultShape, toShape, unwrapResult, errori condivisi
├── package.json      # Workspace root (pnpm)
└── turbo.json        # Turborepo config
```

---

## 3. Schema Database

### Convenzioni

- UUID v7 come primary key (ordinabili per tempo, no sequential guessing)
- `created_at` / `updated_at` su tutte le tabelle
- Soft delete con `deleted_at` dove serve preservare storia (es. acquisti)
- Nomi in snake_case

### Tabelle

#### `users`
```sql
id              uuid PRIMARY KEY DEFAULT gen_random_uuid()
better_auth_id  text UNIQUE NOT NULL        -- ID Better Auth, source of truth per auth
email           text UNIQUE NOT NULL
display_name    text NOT NULL
role            text NOT NULL DEFAULT 'viewer'  -- viewer | filmmaker | admin | b2b_partner
stripe_customer_id  text UNIQUE             -- creato al primo acquisto
avatar_url      text
bio             text
website_url     text
created_at      timestamptz DEFAULT now()
updated_at      timestamptz DEFAULT now()
deleted_at      timestamptz                 -- soft delete
```

#### `filmmaker_profiles`
```sql
id              uuid PRIMARY KEY DEFAULT gen_random_uuid()
user_id         uuid NOT NULL REFERENCES users(id)
stripe_account_id   text UNIQUE             -- Stripe Connect account
stripe_onboarded    boolean DEFAULT false   -- onboarding Stripe completato
bank_country    text                        -- per Stripe Connect
payout_enabled  boolean DEFAULT false
total_earned    numeric(10,2) DEFAULT 0
created_at      timestamptz DEFAULT now()
updated_at      timestamptz DEFAULT now()
```

#### `films`
```sql
id              uuid PRIMARY KEY DEFAULT gen_random_uuid()
filmmaker_id    uuid NOT NULL REFERENCES users(id)
title           text NOT NULL
slug            text UNIQUE NOT NULL        -- URL-friendly, generato da title
description     text NOT NULL
short_description   text                    -- max 160 char, per card catalogo
genre           text[]                      -- array: ['drama', 'documentary']
duration_seconds    integer NOT NULL
release_year    integer
country         text DEFAULT 'IT'
language        text DEFAULT 'it'
price_eur       numeric(6,2) NOT NULL       -- prezzo pay-per-view
status          text NOT NULL DEFAULT 'draft'
                -- draft | under_review | published | rejected | unpublished
rejection_reason    text                    -- compilato da admin se rejected
thumbnail_url   text                        -- poster principale
trailer_url     text                        -- URL Cloudflare Stream trailer
cf_stream_id    text                        -- Cloudflare Stream video ID
cf_stream_status    text                    -- ready | processing | error
r2_original_key text                        -- chiave file originale su R2
view_count      integer DEFAULT 0
purchase_count  integer DEFAULT 0
created_at      timestamptz DEFAULT now()
updated_at      timestamptz DEFAULT now()
published_at    timestamptz
deleted_at      timestamptz
```

#### `purchases`
```sql
id              uuid PRIMARY KEY DEFAULT gen_random_uuid()
user_id         uuid NOT NULL REFERENCES users(id)
film_id         uuid NOT NULL REFERENCES films(id)
stripe_payment_intent_id    text UNIQUE NOT NULL
amount_eur      numeric(6,2) NOT NULL       -- prezzo pagato
filmmaker_share_eur     numeric(6,2) NOT NULL   -- 70% del netto
platform_share_eur      numeric(6,2) NOT NULL   -- 30% del netto
stripe_fee_eur  numeric(6,2)                -- fee Stripe (sottratta prima della split)
payout_status   text DEFAULT 'pending'      -- pending | paid | failed
payout_transfer_id  text                    -- Stripe Transfer ID quando pagato
ip_country      text                        -- per analytics geografia
created_at      timestamptz DEFAULT now()
deleted_at      timestamptz                 -- mai cancellare acquisti reali
```

#### `watch_sessions`
```sql
id              uuid PRIMARY KEY DEFAULT gen_random_uuid()
user_id         uuid NOT NULL REFERENCES users(id)
film_id         uuid NOT NULL REFERENCES films(id)
purchase_id     uuid REFERENCES purchases(id)   -- null se preview/trailer
started_at      timestamptz DEFAULT now()
ended_at        timestamptz
last_position_seconds   integer DEFAULT 0   -- resume playback
completed       boolean DEFAULT false       -- >85% watched
ip_country      text
device_type     text                        -- desktop | mobile | tablet
```

#### `film_analytics` (aggregated, updated daily)
```sql
id              uuid PRIMARY KEY DEFAULT gen_random_uuid()
film_id         uuid NOT NULL REFERENCES films(id)
date            date NOT NULL
views           integer DEFAULT 0
unique_viewers  integer DEFAULT 0
completions     integer DEFAULT 0           -- sessioni >85%
revenue_eur     numeric(10,2) DEFAULT 0
top_countries   jsonb                       -- {IT: 45, DE: 12, ...}
UNIQUE(film_id, date)
```

#### `admin_reviews`
```sql
id              uuid PRIMARY KEY DEFAULT gen_random_uuid()
film_id         uuid NOT NULL REFERENCES films(id)
reviewed_by     uuid NOT NULL REFERENCES users(id)
action          text NOT NULL               -- approved | rejected
notes           text
created_at      timestamptz DEFAULT now()
```

---

## 4. Architettura API

### Base URL
```
Production: https://api.nobo.film/v1
Development: http://localhost:3001/v1
```

### Convenzioni

- REST con JSON
- Auth via Bearer token (Better Auth JWT) o cookie (web)
- Errori in formato `{ error: { code: string, message: string } }`
- Paginazione cursor-based: `?cursor=<id>&limit=20`
- Rate limiting: 100 req/min per IP, 1000 req/min per utente autenticato

---

### Endpoints

#### Auth (gestito da Better Auth)

```
POST /webhooks/better-auth    # eventi auth: nuovo utente, cambio email, cancellazione
```

---

#### Films — pubblico

```
GET  /films                   # catalogo pubblicato, paginato
     ?genre=drama
     ?sort=newest|popular|price_asc
     ?cursor=<id>&limit=20

GET  /films/:slug             # dettaglio film pubblico
GET  /films/:slug/trailer     # URL firmato trailer (signed URL, 1h TTL)
```

**Response GET /films/:slug**
```json
{
  "id": "...",
  "title": "Il Vento di Marzo",
  "slug": "il-vento-di-marzo",
  "description": "...",
  "short_description": "...",
  "duration_seconds": 5400,
  "release_year": 2025,
  "price_eur": "2.99",
  "genre": ["drama"],
  "thumbnail_url": "https://...",
  "filmmaker": {
    "id": "...",
    "display_name": "Marco Rossi",
    "avatar_url": "https://..."
  },
  "view_count": 142,
  "already_purchased": false   // true se utente autenticato ha già comprato
}
```

---

#### Films — filmmaker (richiede auth + ruolo filmmaker)

```
GET    /filmmaker/films              # lista film del filmmaker autenticato
POST   /filmmaker/films              # crea nuovo film (draft)
GET    /filmmaker/films/:id          # dettaglio film
PATCH  /filmmaker/films/:id          # aggiorna metadata
DELETE /filmmaker/films/:id          # soft delete (solo se draft/rejected)
POST   /filmmaker/films/:id/submit   # invia in review (draft → under_review)
POST   /filmmaker/films/:id/unpublish # ritira dalla pubblicazione
```

**POST /filmmaker/films — body**
```json
{
  "title": "Il Vento di Marzo",
  "description": "...",
  "short_description": "...",
  "genre": ["drama"],
  "duration_seconds": 5400,
  "release_year": 2025,
  "language": "it",
  "price_eur": "2.99"
}
```

---

#### Upload Video (filmmaker)

```
POST /filmmaker/films/:id/upload-url
```

Flow:
1. Filmmaker chiede upload URL
2. API crea upload diretto su Cloudflare Stream, restituisce URL firmato
3. Filmmaker carica direttamente su Cloudflare (no passaggio dal backend)
4. Cloudflare notifica via webhook quando encoding è completo
5. API aggiorna `films.cf_stream_id` e `cf_stream_status = ready`

**Response**
```json
{
  "upload_url": "https://upload.videodelivery.net/...",
  "expires_at": "2026-04-19T12:00:00Z"
}
```

```
POST /webhooks/cloudflare-stream      # webhook encoding completato
```

---

#### Filmmaker Analytics

```
GET /filmmaker/analytics              # aggregato totale
GET /filmmaker/analytics/:film_id     # per singolo film
    ?from=2026-01-01&to=2026-04-19
```

**Response**
```json
{
  "total_views": 342,
  "total_revenue_eur": "156.40",
  "filmmaker_earnings_eur": "109.48",
  "films": [
    {
      "film_id": "...",
      "title": "...",
      "views": 200,
      "completions": 145,
      "completion_rate": 0.725,
      "revenue_eur": "100.00",
      "top_countries": { "IT": 180, "DE": 20 }
    }
  ]
}
```

---

#### Filmmaker Onboarding Stripe Connect

```
POST /filmmaker/stripe/onboard        # crea account Stripe Connect, restituisce link onboarding
GET  /filmmaker/stripe/status         # stato onboarding e payout
```

---

#### Purchases (viewer autenticato)

```
POST /purchases                       # crea payment intent Stripe
GET  /purchases                       # lista acquisti utente
GET  /purchases/:id                   # dettaglio acquisto
```

**POST /purchases — body**
```json
{
  "film_id": "..."
}
```

**Response**
```json
{
  "purchase_id": "...",
  "stripe_client_secret": "pi_..._secret_...",   // usato dal frontend Stripe.js
  "amount_eur": "2.99"
}
```

Dopo conferma pagamento da Stripe webhook → payout automatico al filmmaker.

---

#### Watch (viewer autenticato + acquisto verificato)

```
GET /watch/:film_id                   # URL firmato per streaming (signed URL, 4h TTL)
POST /watch/:film_id/progress         # aggiorna posizione playback
```

**GET /watch/:film_id response**
```json
{
  "stream_url": "https://customer-....cloudflarestream.com/.../manifest/video.m3u8?token=...",
  "last_position_seconds": 0,
  "expires_at": "2026-04-19T16:00:00Z"
}
```

Guard: se l'utente non ha un `purchase` valido per quel film → 403.

---

#### Admin

```
GET   /admin/films/queue              # film in under_review
POST  /admin/films/:id/approve        # pubblica film
POST  /admin/films/:id/reject         # rifiuta con motivo
GET   /admin/users                    # lista utenti
GET   /admin/stats                    # metriche piattaforma
```

---

#### Webhooks

```
POST /webhooks/stripe                 # pagamenti, transfer completati
POST /webhooks/cloudflare-stream      # encoding completato
```

Tutti i webhook verificano la firma del provider prima di processare.

---

## 5. Video Pipeline

### Upload Flow

```
Filmmaker browser
    │
    ├─ 1. POST /filmmaker/films/:id/upload-url
    │       └─ API crea CF Stream upload URL (one-time, 1h TTL)
    │
    ├─ 2. PUT <upload_url> con file video
    │       └─ diretto su Cloudflare Stream (bypass backend)
    │
    └─ 3. Cloudflare encoding asincrono
            └─ webhook POST /webhooks/cloudflare-stream
                    └─ API aggiorna film: cf_stream_id, cf_stream_status=ready
```

### Streaming Flow (viewer)

```
Viewer browser
    │
    ├─ 1. GET /watch/:film_id
    │       └─ API verifica acquisto
    │       └─ genera signed URL Cloudflare Stream (4h TTL)
    │       └─ restituisce HLS manifest URL con token
    │
    └─ 2. Video.js / HLS.js carica manifest
            └─ Cloudflare CDN serve segmenti video
```

### Signed URL

Cloudflare Stream supporta signed URLs nativamente. Il token viene generato lato API con la chiave privata CF e ha TTL configurabile. Impedisce hotlinking e accesso senza acquisto — sufficiente per MVP senza DRM completo.

### Thumbnail e Poster

- Cloudflare Stream genera automaticamente thumbnail dal video
- Filmmaker può caricare poster personalizzato → R2
- Thumbnail URL: `https://customer-xxxx.cloudflarestream.com/<cf_stream_id>/thumbnails/thumbnail.jpg`

---

## 6. Autenticazione e Autorizzazione

### Better Auth

Better Auth gestisce:
- Registrazione email/password
- OAuth Google
- Sessioni via cookie (web) e bearer token (future mobile)
- Email verification

`requireUser()` è l'unico modo per ottenere l'utente autenticato in un `createServerFn`. Mai leggere la sessione direttamente nei componenti.

```typescript
// apps/web/app/server/context.ts
import { auth } from '~/server/auth'
import { ForbiddenError } from '@nobo/utils'

export const requireUser = async () => {
  const session = await auth.getSession()
  if (!session?.user) throw new ForbiddenError('not authenticated')
  return session.user
}
```

### Autorizzazione per ruolo

```typescript
export const requireRole = async (role: UserRole) => {
  const user = await requireUser()
  if (user.role !== role && user.role !== 'admin') {
    throw new ForbiddenError(`requires role ${role}`)
  }
  return user
}
```

Ruoli: `viewer` < `filmmaker` < `b2b_partner` < `admin`

---

## 7. Pagamenti e Payout

### Flow acquisto

```
1. Viewer → POST /purchases { film_id }
2. API:
   - verifica film pubblicato e prezzo
   - calcola split: filmmaker 70%, piattaforma 30%
   - calcola Stripe fee: ~1.4% + €0.25 (EU cards)
   - crea PaymentIntent Stripe con application_fee_amount
   - crea record purchase (status: pending)
   - restituisce client_secret al frontend

3. Frontend: Stripe.js confirma pagamento con client_secret

4. Stripe webhook → POST /webhooks/stripe
   - evento: payment_intent.succeeded
   - API:
     - aggiorna purchase.payout_status = 'pending'
     - crea Stripe Transfer verso filmmaker account (70% del netto)
     - aggiorna purchase.payout_transfer_id
     - aggiorna purchase.payout_status = 'paid'
     - incrementa film.purchase_count
     - incrementa filmmaker_profile.total_earned
```

### Calcolo split

```typescript
const PLATFORM_SHARE = 0.30
const FILMMAKER_SHARE = 0.70
const STRIPE_FEE_PERCENT = 0.014
const STRIPE_FEE_FIXED = 0.25  // €

function calculateSplit(priceEur: number) {
  const stripeFee = priceEur * STRIPE_FEE_PERCENT + STRIPE_FEE_FIXED
  const net = priceEur - stripeFee
  return {
    filmmakerShare: net * FILMMAKER_SHARE,
    platformShare: net * PLATFORM_SHARE,
    stripeFee
  }
}
```

### Stripe Connect

Ogni filmmaker deve completare l'onboarding Stripe Connect (verifica identità, IBAN) prima di poter ricevere payout. Se onboarding non completato, i trasferimenti vengono trattenuti e rilasciati al completamento.

---

## 8. Frontend: Struttura e Pagine

### Route Map

```
/ (homepage)
  ├─ /films (catalogo)
  │    └─ /films/[slug] (pagina film)
  ├─ /filmmaker/[id] (profilo pubblico filmmaker)
  │
  ├─ /watch/[film_id] (player — richiede auth + acquisto)
  │
  ├─ /account (viewer — acquisti, profilo)
  │
  ├─ /creator (filmmaker portal — richiede auth + ruolo filmmaker)
  │    ├─ /creator/films (lista film)
  │    ├─ /creator/films/new (upload nuovo film)
  │    ├─ /creator/films/[id] (edit film)
  │    ├─ /creator/analytics (dashboard analytics)
  │    └─ /creator/payouts (stato pagamenti)
  │
  └─ /admin (admin panel — richiede ruolo admin)
       ├─ /admin/queue (film in review)
       └─ /admin/users
```

### Componenti Chiave

**VideoPlayer** — wrapper attorno a Video.js con HLS.js
- Carica stream URL da `/watch/:film_id`
- Salva posizione ogni 30s via POST `/watch/:film_id/progress`
- Gestisce errori di rete con retry automatico

**FilmCard** — card catalogo
- Thumbnail, titolo, filmmaker, durata, prezzo
- Badge "già acquistato" se viewer ha acquisto

**PurchaseButton** — gestisce intero flow acquisto
- Stato: idle → loading → stripe_modal → success | error
- Stripe Elements embedded (no redirect)

**UploadForm** (creator portal)
- Upload diretto su Cloudflare Stream via signed URL
- Progress bar, retry su failure
- Polling status encoding (ogni 10s finché `cf_stream_status = ready`)

---

## 9. Sicurezza e GDPR

### Sicurezza

- Tutti i secret in variabili d'ambiente (mai in codice)
- Webhook verificati con firma HMAC (Stripe, Cloudflare)
- Signed URL per ogni stream video (TTL 4h, non riusabile)
- Rate limiting su tutti gli endpoint pubblici
- Input validation con Zod su ogni request body
- SQL injection impossibile via Drizzle ORM (query parametrizzate)
- CORS configurato esplicitamente (no wildcard in production)
- Headers sicurezza: CSP, HSTS, X-Frame-Options via TanStack Start middleware

### GDPR

- Consent banner al primo accesso (Better Auth gestisce cookie sessione)
- Dati utente eliminabili via `/account/delete` → soft delete + anonimizzazione
- IP salvati solo per analytics geografia, non per tracciamento utente
- Nessun dato carta su server Nobo — tutto su Stripe (PCI DSS compliant)
- Data residency: tutti i servizi scelti hanno region EU disponibile
  - Railway: Frankfurt
  - Cloudflare: EU data localization
  - Better Auth: self-hosted su Railway Frankfurt
  - Upstash Redis: Frankfurt

---

## 10. Deployment e Infrastruttura

### Ambienti

| Ambiente | URL | Branch | Deploy |
|---|---|---|---|
| Development | localhost | qualsiasi | manuale |
| Staging | staging.nobo.film | `develop` | automatico su push |
| Production | nobo.film / api.nobo.film | `main` | manuale con approvazione |

### CI/CD (GitHub Actions)

```yaml
# .github/workflows/ci.yml
on: [push, pull_request]
jobs:
  test:
    - typecheck (tsc --noEmit)
    - lint (eslint)
    - test (vitest)
    - build

  deploy-staging:
    needs: test
    if: branch == 'develop'
    - deploy api → Railway staging
    - deploy web → Vercel preview

  deploy-production:
    needs: test
    if: branch == 'main'
    environment: production  # richiede approvazione manuale
    - deploy api → Railway production
    - deploy web → Vercel production
```

### Costi Infrastruttura (stima MVP)

| Servizio | Piano | Costo/mese |
|---|---|---|
| Railway (API + PostgreSQL) | Starter | €10-30 |
| Vercel (Frontend) | Pro | €20 |
| Cloudflare Stream | Pay-as-you-go | €50-200 |
| Cloudflare R2 | Pay-as-you-go | €5-20 |
| Better Auth | Self-hosted (su Railway) | €0 |
| Upstash Redis | Pay-as-you-go | €5-15 |
| Resend (email) | Free → Starter | €0-20 |
| Sentry | Free | €0 |
| PostHog | Free (self-host) | €0 |
| **Totale** | | **€90-330/mese** |

---

## 11. MVP Scope: Cosa È Fuori

Queste funzionalità sono **esplicitamente escluse dall'MVP** per mantenere la scope realistica con 1 developer:

| Funzionalità | Motivo esclusione | Quando |
|---|---|---|
| Subscription mensile | Validare prima domanda pay-per-view | Fase 2 (mese 6+) |
| B2B portal / screening room | Sales cycle lungo, priorità B2C validation | Fase 2 (mese 5+) |
| App mobile nativa | Web mobile-responsive sufficiente per MVP | Fase 3 |
| DRM (Widevine/FairPlay) | Costo e complessità sproporzionati per MVP | Fase 3+ |
| Sottotitoli | Scope creep — aggiunto su richiesta filmmaker | Fase 2 |
| Community (reviews, liste) | Non core per validazione business | Fase 3 |
| Tipping diretto | Nice-to-have, non prioritario | Fase 2 |
| Ricerca full-text | Filtri base sufficienti per catalogo piccolo | Quando catalogo >200 titoli |
| Multi-lingua (EN/FR) | Focus Italia per ora | Fase 3 |
| Analytics avanzate (funnel, cohort) | PostHog base sufficiente | Fase 2 |

---

*Versione 0.1 — Aprile 2026*
*Soggetto a revisione dopo feedback utenti beta*
