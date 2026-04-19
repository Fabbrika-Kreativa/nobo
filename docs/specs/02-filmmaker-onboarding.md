# Spec 02 — Filmmaker Onboarding
## Nobo — Aprile 2026

---

## Obiettivo

Definire il flusso completo che porta un `viewer` a diventare un `filmmaker` operativo su Nobo: dalla richiesta di upgrade (con approvazione admin), alla creazione del `filmmaker_profile`, fino al completamento dell'onboarding Stripe Connect per ricevere payout.

Perimetro: `apps/web` (pagine e server functions), `apps/api` (webhook Stripe), `packages/domain` (schemi e tipi), `packages/db` (schema DB già definito).

**Vincoli non negoziabili:**
- Un filmmaker non può caricare film finché l'admin non approva l'upgrade (guard in tutte le server functions filmmaker).
- Un filmmaker non può ricevere payout finché Stripe Connect non è completato e `payout_enabled = true` (guard sul trasferimento in `payment_intent.succeeded`).

---

## Stati del Filmmaker

Il filmmaker attraversa quattro stati, derivati da `users.role` e `filmmaker_profiles`:

```
pending_approval
    │  admin approva → role = 'filmmaker', filmmaker_profile creato
    ▼
active_no_stripe
    │  filmmaker avvia onboarding Stripe Connect
    ▼
active_stripe_pending
    │  webhook account.updated: charges_enabled + payouts_enabled = true
    ▼
active_payout_enabled
```

| Stato | Condizione | Può caricare film? | Può ricevere payout? |
|---|---|---|---|
| `pending_approval` | `role = 'viewer'`, richiesta inviata | No | No |
| `active_no_stripe` | `role = 'filmmaker'`, `stripe_account_id IS NULL` | Sì | No |
| `active_stripe_pending` | `stripe_account_id` presente, `stripe_onboarded = false` | Sì | No |
| `active_payout_enabled` | `stripe_onboarded = true`, `payout_enabled = true` | Sì | Sì |

Lo stato è calcolato a runtime dalla funzione `resolveFilmmakerStatus` — non è un campo esplicito nel DB.

---

## Flussi

### 1. Richiesta upgrade viewer → filmmaker

1. L'utente autenticato con `role = 'viewer'` visita `/creator/onboarding`.
2. Compila il form con motivazione e portfolio URL (opzionale).
3. La pagina chiama `requestFilmmakerUpgrade` (`createServerFn`), validato con `FilmmakerUpgradeRequestSchema`.
4. La server function verifica che l'utente sia `viewer` e non abbia già una richiesta pendente.
5. Viene creato un record in `filmmaker_upgrade_requests` (tabella fuori schema MVP — vedi sezione fuori scope) oppure, nel MVP, viene inviata una notifica email all'admin tramite Resend.
6. L'utente vede la pagina in stato `pending_approval` con banner "Richiesta inviata, riceverai una risposta via email".

**MVP semplificato**: nessuna tabella `filmmaker_upgrade_requests`. L'admin riceve l'email, esegue manualmente:
```sql
UPDATE users SET role = 'filmmaker' WHERE id = ?;
INSERT INTO filmmaker_profiles (id, user_id) VALUES (gen_random_uuid(), ?);
```

La server function `requestFilmmakerUpgrade` invia solo l'email e aggiorna un flag locale (non modifica `role`).

### 2. Creazione `filmmaker_profile`

Il `filmmaker_profile` viene creato dall'admin (MVP: via query diretta o panel `/admin/users`) contestualmente all'upgrade del ruolo. Non esiste auto-creazione lato client.

Guard nella server function `createStripeConnectAccount`: se `filmmaker_profile` non esiste → `FilmmakerProfileNotFoundError`.

### 3. Onboarding Stripe Connect

```
Filmmaker visita /creator/payouts
    │
    ├─ chiama getFilmmakerStripeStatus
    │       └─ stato: active_no_stripe → mostra CTA "Collega conto bancario"
    │
    ├─ click CTA → chiama createStripeConnectAccount
    │       └─ Stripe: stripe.accounts.create({ type: 'express', country: bank_country })
    │       └─ salva stripe_account_id su filmmaker_profiles
    │       └─ Stripe: stripe.accountLinks.create({ type: 'account_onboarding' })
    │       └─ restituisce { onboardingUrl }
    │
    ├─ redirect → Stripe hosted onboarding flow
    │
    └─ Stripe redirect back → /creator/payouts?stripe=return
            └─ pagina chiama getFilmmakerStripeStatus
            └─ se stripe_onboarded = false → banner "Onboarding in corso"
            └─ se stripe_onboarded = true → banner "Conto collegato"
```

### 4. Webhook `account.updated`

`apps/api` riceve `POST /webhooks/stripe`. Il handler filtra l'evento `account.updated`.

```
Stripe → POST /webhooks/stripe
    │
    ├─ verifica firma HMAC (stripe.webhooks.constructEvent)
    │
    ├─ evento: account.updated
    │       └─ cerca filmmaker_profiles WHERE stripe_account_id = event.account
    │       └─ aggiorna:
    │               stripe_onboarded = account.details_submitted
    │               payout_enabled   = account.payouts_enabled
    │               bank_country     = account.country (se presente)
    │
    └─ risponde 200 immediatamente (processing asincrono)
```

Se `stripe_account_id` non corrisponde a nessun `filmmaker_profile` → log warning, risponde 200 (idempotente).

---

## Schema DB

Già definito in `NOBO_TECHNICAL_SPEC.md`. Nessuna modifica richiesta da questa spec.

```sql
filmmaker_profiles
  id              uuid PRIMARY KEY
  user_id         uuid NOT NULL REFERENCES users(id)
  stripe_account_id   text UNIQUE
  stripe_onboarded    boolean DEFAULT false
  bank_country    text
  payout_enabled  boolean DEFAULT false
  total_earned    numeric(10,2) DEFAULT 0
  created_at      timestamptz
  updated_at      timestamptz
```

---

## Zod Schemas — `packages/domain/src/filmmaker/filmmaker.schemas.ts`

```typescript
import { z } from 'zod'

export const FilmmakerUpgradeRequestSchema = z.object({
  motivation: z.string().min(20).max(1000),
  portfolioUrl: z.string().url().optional(),
})
export type FilmmakerUpgradeRequest = z.infer<typeof FilmmakerUpgradeRequestSchema>

export const FilmmakerStatusSchema = z.enum([
  'pending_approval',
  'active_no_stripe',
  'active_stripe_pending',
  'active_payout_enabled',
])
export type FilmmakerStatus = z.infer<typeof FilmmakerStatusSchema>

export const FilmmakerProfileSchema = z.object({
  id: z.string().uuid().brand<'FilmmakerProfileId'>(),
  userId: z.string().uuid().brand<'UserId'>(),
  stripeAccountId: z.string().nullable(),
  stripeOnboarded: z.boolean(),
  bankCountry: z.string().nullable(),
  payoutEnabled: z.boolean(),
  totalEarned: z.string(),
  createdAt: z.date(),
  updatedAt: z.date(),
})
export type FilmmakerProfile = z.infer<typeof FilmmakerProfileSchema>

export const FilmmakerStripeStatusSchema = z.object({
  status: FilmmakerStatusSchema,
  stripeAccountId: z.string().nullable(),
  stripeOnboarded: z.boolean(),
  payoutEnabled: z.boolean(),
  totalEarned: z.string(),
})
export type FilmmakerStripeStatus = z.infer<typeof FilmmakerStripeStatusSchema>

export const CreateStripeConnectAccountInputSchema = z.object({
  bankCountry: z.string().length(2),
})
export type CreateStripeConnectAccountInput = z.infer<typeof CreateStripeConnectAccountInputSchema>
```

Esportati da `packages/domain/src/index.ts`.

---

## Errori — `packages/utils/src/errors.ts`

Si aggiungono alle classi errore esistenti:

```typescript
export class FilmmakerProfileNotFoundError {
  readonly _tag = 'FilmmakerProfileNotFoundError' as const
  readonly message = 'Filmmaker profile not found'
}

export class StripeOnboardingError {
  readonly _tag = 'StripeOnboardingError' as const
  readonly message: string
  constructor(readonly cause?: string) {
    this.message = cause ?? 'Stripe onboarding failed'
  }
}

export class UpgradeAlreadyRequestedError {
  readonly _tag = 'UpgradeAlreadyRequestedError' as const
  readonly message = 'Filmmaker upgrade already requested or active'
}
```

Discriminazione via `_tag` con `ts-pattern`. Mai `instanceof`.

---

## Server Functions — `apps/web/app/features/filmmaker/filmmaker.server.ts`

### `requestFilmmakerUpgrade`

```typescript
export const requestFilmmakerUpgrade = createServerFn({ method: 'POST' })
  .validator(FilmmakerUpgradeRequestSchema)
  .handler(async ({ data }): Promise<ResultShape<void, UnauthenticatedError | UpgradeAlreadyRequestedError | ValidationError>> => {
    const user = await requireUser()
    // guard: solo viewer possono richiedere upgrade
    if (user.role !== 'viewer') return err(new UpgradeAlreadyRequestedError())
    // invia email admin via Resend
    // restituisce ok(undefined)
  })
```

### `createStripeConnectAccount`

```typescript
export const createStripeConnectAccount = createServerFn({ method: 'POST' })
  .validator(CreateStripeConnectAccountInputSchema)
  .handler(async ({ data }): Promise<ResultShape<{ onboardingUrl: string }, FilmmakerProfileNotFoundError | StripeOnboardingError | ForbiddenError>> => {
    const user = await requireRole('filmmaker')
    // guard: filmmaker_profile deve esistere
    const profile = await db.query.filmmakerProfiles.findFirst({
      where: eq(filmmakerProfiles.userId, user.id),
    })
    if (!profile) return err(new FilmmakerProfileNotFoundError())
    // se stripe_account_id già presente, genera solo nuovo link onboarding
    // altrimenti: stripe.accounts.create → salva stripe_account_id
    // stripe.accountLinks.create({ type: 'account_onboarding', return_url, refresh_url })
    // restituisce ok({ onboardingUrl })
  })
```

**Guard pagamento**: il trasferimento verso il filmmaker in `payment_intent.succeeded` (in `apps/api`) controlla `payout_enabled = true` prima di eseguire `stripe.transfers.create`. Se `payout_enabled = false`, il trasferimento viene saltato e loggato come `payout_skipped_stripe_not_ready`.

### `getFilmmakerStripeStatus`

```typescript
export const getFilmmakerStripeStatus = createServerFn({ method: 'GET' })
  .handler(async (): Promise<ResultShape<FilmmakerStripeStatus, FilmmakerProfileNotFoundError | ForbiddenError>> => {
    const user = await requireRole('filmmaker')
    const profile = await db.query.filmmakerProfiles.findFirst({
      where: eq(filmmakerProfiles.userId, user.id),
    })
    if (!profile) return err(new FilmmakerProfileNotFoundError())
    const status = resolveFilmmakerStatus(profile)
    return ok(FilmmakerStripeStatusSchema.parse({ status, ...profile }))
  })
```

### `resolveFilmmakerStatus` (helper puro, non server function)

```typescript
// packages/domain/src/filmmaker/filmmaker.utils.ts
export function resolveFilmmakerStatus(profile: FilmmakerProfile): FilmmakerStatus {
  if (!profile.stripeAccountId) return 'active_no_stripe'
  if (!profile.stripeOnboarded) return 'active_stripe_pending'
  if (profile.payoutEnabled) return 'active_payout_enabled'
  return 'active_stripe_pending'
}
```

Chiamato anche nel componente `/creator/payouts` per decidere quale UI renderizzare.

---

## Pagine

### `/creator/onboarding`

Protetta da `/_filmmaker` layout route — se l'utente è `viewer`, viene reindirizzato da `/_filmmaker.tsx` a `/creator/onboarding` (pagina pubblica per la richiesta di upgrade).

**Nota**: `/creator/onboarding` è accessibile ai `viewer` per inviare la richiesta. Dopo l'invio, l'utente vede lo stato `pending_approval` e non può accedere al resto del creator portal.

Struttura step:

```
Step 1 — Stato pending (viewer non ancora approvato)
  └─ Form: motivazione + portfolio URL
  └─ CTA: "Invia richiesta"
  └─ Dopo invio: banner "Richiesta inviata"

Step 2 — Stato active_no_stripe (filmmaker approvato, nessun conto)
  └─ Banner: "Accesso creator attivato"
  └─ CTA: vai a /creator/payouts per collegare il conto

Step 3 — Onboarding completato
  └─ Redirect a /creator/films
```

La pagina usa `getFilmmakerStripeStatus` (se filmmaker) o controlla `user.role` (se viewer) per determinare lo step corrente.

### `/creator/payouts`

Protetta da `/_filmmaker`. Chiama `getFilmmakerStripeStatus` in `loader`.

| Stato | UI |
|---|---|
| `active_no_stripe` | CTA "Collega conto bancario" + form `bankCountry` |
| `active_stripe_pending` | Banner "Onboarding Stripe in corso" + link "Riprendi onboarding" |
| `active_payout_enabled` | Dashboard: total_earned, prossimo payout, storico transazioni |

Query param `?stripe=return` (redirect da Stripe): la pagina chiama `getFilmmakerStripeStatus` e mostra un banner di conferma o di attesa in base allo stato aggiornato.

Query param `?stripe=refresh` (link scaduto): la pagina chiama `createStripeConnectAccount` automaticamente per generare un nuovo link onboarding.

---

## Fuori Scope

| Funzionalità | Note |
|---|---|
| Tabella `filmmaker_upgrade_requests` | MVP: notifica email all'admin, nessuna tabella. Da implementare in Fase 2 per tracking stato richieste. |
| Dashboard payout con storico dettagliato | Collegato alla spec `purchases` — si lista da `purchases` dove `payout_status = 'paid'` per il filmmaker. Fuori da questa spec. |
| Stripe Express dashboard embed | Non disponibile nel MVP. Link esterno alla Stripe Express dashboard. |
| Notifica email al filmmaker all'approvazione | L'admin notifica manualmente nel MVP. Da automatizzare in Fase 2. |
| Multi-currency (non EUR) | Solo EUR nel MVP. |
| Revoca ruolo filmmaker | Da definire in spec admin. |
| Payout manuale (override admin) | Da definire in spec admin. |
| Upgrade self-service senza approvazione admin | Non previsto nel MVP. |

---

*Spec 02 — v1.0 — Aprile 2026*
