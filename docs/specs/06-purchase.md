# 06 — Acquisto Pay-Per-View

## Obiettivo

Implementare il flusso completo di acquisto pay-per-view: creazione del PaymentIntent Stripe, conferma lato client, gestione del webhook, e trasferimento della quota al filmmaker tramite Stripe Connect.

---

## Flussi

### Acquisto

1. Utente autenticato clicca `PurchaseButton` sulla pagina del film.
2. Frontend chiama `createPurchase({ filmId })`.
3. Server function:
   - Verifica che il film sia `published` → altrimenti `FilmNotPublishedError`.
   - Verifica idempotenza: se esiste già un acquisto completato per `(userId, filmId)` → `AlreadyPurchasedError`.
   - Calcola lo split con `calculateSplit(film.priceEur)`.
   - Crea un `PaymentIntent` Stripe con `amount = priceEur * 100` (centesimi) e `metadata.filmId`, `metadata.userId`.
   - Inserisce un record in `purchases` con `stripe_payment_intent_id` e `payout_status = 'pending'`.
   - Restituisce il `client_secret` del PaymentIntent.
4. Frontend usa Stripe.js per aprire il modale di pagamento con il `client_secret`.
5. L'utente inserisce i dati carta e conferma.
6. Stripe invia il webhook `payment_intent.succeeded` al backend.
7. Webhook handler:
   - Verifica la firma Stripe (`stripe.webhooks.constructEvent`).
   - Recupera il `purchase` dalla tabella tramite `stripe_payment_intent_id`.
   - Crea un `Transfer` Stripe verso `filmmaker_profile.stripe_account_id` per `filmmaker_share_eur`.
   - Aggiorna `purchases.payout_transfer_id` e `purchases.payout_status = 'paid'` al ricevimento del webhook `transfer.paid`.
8. Frontend polling o Stripe.js redirect → mostra stato success.

### Calcolo split

```
stripe_fee = (price_eur * 0.014) + 0.25        -- ~1.4% + €0.25
net_after_fee = price_eur - stripe_fee
filmmaker_share = net_after_fee * 0.70
platform_share = net_after_fee * 0.30
```

La funzione `calculateSplit` è pura, senza effetti collaterali, e vive in `packages/domain`. Tutti i valori sono arrotondati a 2 decimali con `Math.round(value * 100) / 100`.

---

## Schema DB

```sql
-- purchases
id                          uuid PRIMARY KEY DEFAULT gen_random_uuid()
user_id                     uuid NOT NULL REFERENCES users(id)
film_id                     uuid NOT NULL REFERENCES films(id)
stripe_payment_intent_id    text UNIQUE NOT NULL
amount_eur                  numeric(6,2) NOT NULL
filmmaker_share_eur         numeric(6,2) NOT NULL
platform_share_eur          numeric(6,2) NOT NULL
stripe_fee_eur              numeric(6,2) NOT NULL
payout_status               text NOT NULL DEFAULT 'pending'  -- pending | paid | failed
payout_transfer_id          text
ip_country                  text
created_at                  timestamptz NOT NULL DEFAULT now()
deleted_at                  timestamptz
```

---

## Tipi (Zod — `packages/domain`)

```ts
export const CreatePurchaseInputSchema = z.object({
  filmId: z.string().uuid(),
})
export type CreatePurchaseInput = z.infer<typeof CreatePurchaseInputSchema>

export const PurchaseSplitSchema = z.object({
  amountEur: z.number(),
  stripeFeeEur: z.number(),
  filmmakerShareEur: z.number(),
  platformShareEur: z.number(),
})
export type PurchaseSplit = z.infer<typeof PurchaseSplitSchema>

export const PurchaseSchema = z.object({
  id: z.string().uuid(),
  userId: z.string().uuid(),
  filmId: z.string().uuid(),
  stripePaymentIntentId: z.string(),
  amountEur: z.number(),
  filmmakerShareEur: z.number(),
  platformShareEur: z.number(),
  stripeFeeEur: z.number(),
  payoutStatus: z.enum(['pending', 'paid', 'failed']),
  payoutTransferId: z.string().nullable(),
  ipCountry: z.string().nullable(),
  createdAt: z.coerce.date(),
})
export type Purchase = z.infer<typeof PurchaseSchema>

export const CreatePurchaseResultSchema = z.object({
  clientSecret: z.string(),
  purchaseId: z.string().uuid(),
})
export type CreatePurchaseResult = z.infer<typeof CreatePurchaseResultSchema>
```

### Funzione pura `calculateSplit` (`packages/domain`)

```ts
export function calculateSplit(priceEur: number): PurchaseSplit {
  const stripeFeeEur = round2(priceEur * 0.014 + 0.25)
  const netAfterFee = round2(priceEur - stripeFeeEur)
  const filmmakerShareEur = round2(netAfterFee * 0.7)
  const platformShareEur = round2(netAfterFee * 0.3)
  return { amountEur: priceEur, stripeFeeEur, filmmakerShareEur, platformShareEur }
}

function round2(value: number): number {
  return Math.round(value * 100) / 100
}
```

---

## Server Functions (`apps/web` via `createServerFn`)

### `createPurchase(input: CreatePurchaseInput)`

- **Guard**: utente autenticato → altrimenti `ForbiddenError`.
- Fetcha il film; se non esiste → `FilmNotFoundError`.
- Se `film.status !== 'published'` → `FilmNotPublishedError`.
- Controlla idempotenza: `SELECT id FROM purchases WHERE user_id = ? AND film_id = ? AND deleted_at IS NULL` → se esiste → `AlreadyPurchasedError`.
- Calcola split con `calculateSplit(film.priceEur)`.
- Crea PaymentIntent Stripe: `stripe.paymentIntents.create({ amount: priceEur * 100, currency: 'eur', metadata: { filmId, userId } })`.
- Inserisce record in `purchases` con `payout_status = 'pending'`.
- **Return**: `ResultAsync<CreatePurchaseResult, FilmNotFoundError | FilmNotPublishedError | AlreadyPurchasedError | PaymentError>`.

### `listUserPurchases`

- **Guard**: utente autenticato.
- **Query**: `SELECT purchases.*, films.title, films.slug, films.thumbnail_url FROM purchases JOIN films ON purchases.film_id = films.id WHERE purchases.user_id = ? AND purchases.deleted_at IS NULL ORDER BY purchases.created_at DESC`.
- **Return**: `ResultAsync<PurchaseWithFilm[], ForbiddenError>`.

### `getPurchase(purchaseId: string)`

- **Guard**: utente autenticato; l'acquisto deve appartenere all'utente → altrimenti `ForbiddenError`.
- **Return**: `ResultAsync<Purchase, FilmNotFoundError | ForbiddenError>`.

### Webhook handler (`apps/api` — route HTTP POST `/webhooks/stripe`)

Non è una `createServerFn` ma una route HTTP dedicata in `apps/api`.

- Legge il body raw (necessario per la verifica firma).
- Verifica firma con `stripe.webhooks.constructEvent(body, sig, webhookSecret)`.
- Switch sull'event type:
  - `payment_intent.succeeded`: crea Transfer Stripe verso `stripe_account_id` del filmmaker.
  - `transfer.paid`: aggiorna `purchases.payout_status = 'paid'` e `purchases.payout_transfer_id`.
  - `transfer.failed`: aggiorna `purchases.payout_status = 'failed'`, logga il motivo.
- Risponde sempre `200 OK` rapidamente; la logica pesante è asincrona.

---

## Pagine e Componenti

### `/account/purchases`

- Loader chiama `listUserPurchases`.
- Lista acquisti con: thumbnail, titolo, data acquisto, importo, stato payout.
- Link "Guarda" per ogni film acquistato → `/watch/$filmId`.

### Componente `PurchaseButton`

Stato macchina:

```
idle → loading → stripe_modal → success
                             ↘ error
```

- **idle**: mostra pulsante "Acquista per €X.XX".
- **loading**: spinner, pulsante disabilitato. Chiama `createPurchase`.
- **stripe_modal**: Stripe Payment Element modale. L'utente inserisce i dati.
- **success**: messaggio "Acquisto completato!", link "Guarda ora".
- **error**: messaggio di errore, pulsante "Riprova".

Il componente riceve `filmId`, `priceEur` e `alreadyPurchased` come props. Se `alreadyPurchased = true`, mostra direttamente "Guarda" senza passare per gli stati di acquisto.

---

## Errori

```ts
export class FilmNotPublishedError {
  readonly _tag = 'FilmNotPublishedError'
  constructor(public readonly filmId: string) {}
}

export class AlreadyPurchasedError {
  readonly _tag = 'AlreadyPurchasedError'
  constructor(
    public readonly userId: string,
    public readonly filmId: string,
  ) {}
}

export class PaymentError {
  readonly _tag = 'PaymentError'
  constructor(
    public readonly message: string,
    public readonly stripeCode?: string,
  ) {}
}

export class FilmmakerPayoutError {
  readonly _tag = 'FilmmakerPayoutError'
  constructor(
    public readonly purchaseId: string,
    public readonly reason: string,
  ) {}
}

export class ForbiddenError {
  readonly _tag = 'ForbiddenError'
  constructor(public readonly reason?: string) {}
}
```

---

## Fuori Scope

- Rimborsi.
- Gift card e codici promozionali.
- Modello subscription.
- Pagamento con metodi diversi da carta (es. PayPal, SEPA).
- Fatturazione / emissione ricevute fiscali.
