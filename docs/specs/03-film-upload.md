# Spec 03 — Film Upload e Gestione Film
## Nobo — Aprile 2026

---

## Obiettivo

Definire il flusso completo di creazione e pubblicazione di un film: dalla creazione del draft, alla compilazione dei metadati, all'upload del video su Cloudflare Stream, all'upload della thumbnail su R2, fino alla sottomissione in review da parte del filmmaker.

Perimetro: `apps/web` (pagine e server functions), `apps/api` (webhook Cloudflare Stream), `packages/domain` (schemi Zod e tipi), `packages/db` (schema già definito in `NOBO_TECHNICAL_SPEC.md`).

**Vincolo non negoziabile**: solo un filmmaker con `payout_enabled = true` (Stripe Connect completato, spec `02-filmmaker-onboarding.md`) può caricare film. Un filmmaker in stato `active_no_stripe` o `active_stripe_pending` riceve `PayoutNotEnabledError` su tutte le operazioni di upload e submit.

---

## Stati Film

```
draft
  │  filmmaker compila metadati + carica video + thumbnail
  │  filmmaker chiama submitFilmForReview
  ▼
under_review
  │  admin approva
  │                    admin rifiuta
  ▼                         ▼
published              rejected
  │                         │  filmmaker corregge + ri-submette
  │                         ▼
  │                        draft
  │
  │  filmmaker ritira
  ▼
unpublished
```

| Stato | Descrizione | Chi può agire |
|---|---|---|
| `draft` | Film in lavorazione, non visibile pubblicamente | Filmmaker (edit, upload, submit, delete) |
| `under_review` | In attesa di revisione admin | Admin (approve, reject) |
| `published` | Visibile e acquistabile nel catalogo pubblico | Filmmaker (unpublish), Admin (unpublish, reject) |
| `rejected` | Rifiutato da admin con motivazione | Filmmaker (corregge e ri-submette → torna `draft`) |
| `unpublished` | Ritirato dal filmmaker dopo pubblicazione | Filmmaker (re-submit → `under_review`) |

### Transizioni valide

| Da | A | Trigger |
|---|---|---|
| `draft` | `under_review` | `submitFilmForReview` (filmmaker) |
| `under_review` | `published` | approvazione admin |
| `under_review` | `rejected` | rifiuto admin |
| `rejected` | `draft` | `revertFilmToDraft` (filmmaker) |
| `published` | `unpublished` | `unpublishFilm` (filmmaker o admin) |
| `unpublished` | `under_review` | `submitFilmForReview` (filmmaker) |

Qualsiasi altra transizione → `FilmStatusError`.

---

## Flussi

### 1. Creazione film (draft)

1. Filmmaker visita `/creator/films/new`.
2. Compila il form iniziale (titolo, descrizione, prezzo, generi, durata, anno).
3. La pagina chiama `createFilm` (`createServerFn`), validato con `CreateFilmSchema`.
4. La server function:
   - verifica ruolo `filmmaker` e `payout_enabled = true`
   - genera lo slug da `title` (funzione `generateSlug`, gestisce conflitti)
   - crea il record `films` con `status = 'draft'`
   - restituisce il `film.id`
5. Redirect a `/creator/films/[id]` (pagina di edit del film).

### 2. Aggiornamento metadati

1. Filmmaker modifica i campi su `/creator/films/[id]`.
2. Chiama `updateFilmMetadata` (`createServerFn`), validato con `UpdateFilmMetadataSchema`.
3. La server function verifica che il film appartenga al filmmaker (`FilmNotOwnedError` altrimenti).
4. Aggiornamento consentito solo se `status IN ('draft', 'rejected')` — non durante `under_review` o `published`.
5. Se `title` cambia, lo slug viene ricalcolato (stesso meccanismo anti-conflitto).

### 3. Upload video (Cloudflare Stream)

```
/creator/films/[id]  — componente UploadForm
    │
    ├─ 1. chiama getFilmUploadUrl (createServerFn)
    │       └─ verifica filmmaker + payout_enabled
    │       └─ verifica film owned + status = draft | rejected
    │       └─ Cloudflare Stream API: crea TUS upload diretto (one-time URL, 1h TTL)
    │       └─ salva cf_stream_id su films (restituito da CF nella creazione)
    │       └─ imposta cf_stream_status = 'processing'
    │       └─ restituisce { uploadUrl, cfStreamId, expiresAt }
    │
    ├─ 2. UploadForm carica il file direttamente su uploadUrl (TUS protocol)
    │       └─ progress bar calcolata dagli eventi TUS
    │       └─ in caso di interruzione: resume automatico (TUS è resumable)
    │
    └─ 3. Polling cf_stream_status ogni 10s
            └─ chiama getFilmmakerFilm per leggere cf_stream_status corrente
            └─ loop finché status = 'ready' | 'error'
            └─ se 'ready': mostra banner "Video pronto"
            └─ se 'error': mostra errore + pulsante "Riprova upload"
```

Il video non passa mai per il backend di Nobo — il file va direttamente su Cloudflare Stream.

### 4. Webhook encoding completato

```
Cloudflare Stream → POST /webhooks/cloudflare-stream  (apps/api)
    │
    ├─ verifica firma HMAC (header Webhook-Signature)
    │
    ├─ evento: stream.live.input (ignorato)
    │
    └─ evento: stream.video.finished
            └─ estrae uid (cf_stream_id) e status (ready | error)
            └─ cerca films WHERE cf_stream_id = uid
            └─ se non trovato → log warning, risponde 200
            └─ aggiorna films:
                    cf_stream_status = status  -- 'ready' | 'error'
                    updated_at = now()
            └─ risponde 200
```

Il webhook gestisce solo `stream.video.finished`. Tutti gli altri eventi vengono ignorati con risposta 200.

### 5. Upload thumbnail (Cloudflare R2)

1. Filmmaker seleziona un'immagine da `/creator/films/[id]`.
2. Il componente chiama `getThumbnailUploadUrl` (`createServerFn`).
3. La server function:
   - verifica filmmaker + `payout_enabled = true`
   - verifica film owned + `status IN ('draft', 'rejected')`
   - genera chiave R2: `thumbnails/<film_id>/<uuid>.jpg`
   - crea presigned PUT URL su R2 (TTL 15 minuti)
   - restituisce `{ uploadUrl, r2Key, thumbnailUrl }`
4. Il componente carica direttamente su R2 via `PUT <uploadUrl>` (singola request, no TUS).
5. Dopo upload completato, chiama `confirmThumbnailUpload` (`createServerFn`) con `r2Key`.
6. La server function aggiorna `films.thumbnail_url` con l'URL pubblico CDN.

Il file originale su R2 è privato. `thumbnail_url` punta all'URL CDN pubblico Cloudflare (non all'URL R2 diretto).

### 6. Submit in review

1. Filmmaker clicca "Invia in revisione" su `/creator/films/[id]`.
2. La pagina chiama `submitFilmForReview` (`createServerFn`).
3. La server function esegue le seguenti guard in ordine:
   - `requireRole('filmmaker')` → `ForbiddenError`
   - `payout_enabled = true` → `PayoutNotEnabledError`
   - film esiste → `FilmNotFoundError`
   - film owned → `FilmNotOwnedError`
   - `status IN ('draft', 'unpublished')` → `FilmStatusError` altrimenti
   - `cf_stream_status = 'ready'` → `FilmNotReadyError` altrimenti
   - `thumbnail_url IS NOT NULL` → `FilmNotReadyError` (thumbnail obbligatoria)
4. Se tutte le guard passano:
   - `films.status = 'under_review'`
   - `films.updated_at = now()`
5. Email notifica all'admin (Resend) — fuori scope di questa spec.

---

## Slug Generation

File: `packages/domain/src/film/film.utils.ts`

```typescript
export function generateSlug(title: string): string {
  return title
    .toLowerCase()
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')  // rimuove accenti
    .replace(/[^a-z0-9\s-]/g, '')
    .trim()
    .replace(/\s+/g, '-')
    .replace(/-+/g, '-')
}
```

**Gestione conflitti**: se lo slug è già in uso nel DB, si appende un suffisso numerico incrementale (`il-vento-di-marzo-2`, `il-vento-di-marzo-3`, ecc.). La verifica avviene nella server function, non nel DB (non c'è race condition rilevante a questo volume).

```typescript
async function resolveUniqueSlug(baseSlug: string, db: Db, excludeId?: string): Promise<string> {
  let slug = baseSlug
  let counter = 1
  while (true) {
    const existing = await db.query.films.findFirst({
      where: and(eq(films.slug, slug), excludeId ? not(eq(films.id, excludeId)) : undefined),
    })
    if (!existing) return slug
    slug = `${baseSlug}-${++counter}`
  }
}
```

---

## Schema e Tipi

File: `packages/domain/src/film/film.schemas.ts`

```typescript
import { z } from 'zod'

export const FilmIdSchema = z.string().uuid().brand<'FilmId'>()
export type FilmId = z.infer<typeof FilmIdSchema>

export const FilmStatusSchema = z.enum([
  'draft',
  'under_review',
  'published',
  'rejected',
  'unpublished',
])
export type FilmStatus = z.infer<typeof FilmStatusSchema>

export const CfStreamStatusSchema = z.enum(['processing', 'ready', 'error'])
export type CfStreamStatus = z.infer<typeof CfStreamStatusSchema>

// Schema per creazione film
export const CreateFilmSchema = z.object({
  title: z.string().min(1).max(200),
  description: z.string().min(10).max(5000),
  shortDescription: z.string().max(160).optional(),
  genre: z.array(z.string()).min(1).max(5),
  durationSeconds: z.number().int().positive(),
  releaseYear: z.number().int().min(1900).max(2100).optional(),
  country: z.string().length(2).default('IT'),
  language: z.string().length(2).default('it'),
  priceEur: z.string().regex(/^\d+(\.\d{1,2})?$/).refine(v => parseFloat(v) >= 0.99),
})
export type CreateFilmInput = z.infer<typeof CreateFilmSchema>

// Schema per aggiornamento metadati (tutti i campi opzionali)
export const UpdateFilmMetadataSchema = CreateFilmSchema.partial().extend({
  filmId: FilmIdSchema,
})
export type UpdateFilmMetadataInput = z.infer<typeof UpdateFilmMetadataSchema>

// Schema risposta film (filmmaker view — include campi interni)
export const FilmFilmmakerViewSchema = z.object({
  id: FilmIdSchema,
  filmakerId: z.string().uuid(),
  title: z.string(),
  slug: z.string(),
  description: z.string(),
  shortDescription: z.string().nullable(),
  genre: z.array(z.string()),
  durationSeconds: z.number(),
  releaseYear: z.number().nullable(),
  country: z.string(),
  language: z.string(),
  priceEur: z.string(),
  status: FilmStatusSchema,
  rejectionReason: z.string().nullable(),
  thumbnailUrl: z.string().nullable(),
  cfStreamId: z.string().nullable(),
  cfStreamStatus: CfStreamStatusSchema.nullable(),
  viewCount: z.number(),
  purchaseCount: z.number(),
  createdAt: z.date(),
  updatedAt: z.date(),
  publishedAt: z.date().nullable(),
})
export type FilmFilmmakerView = z.infer<typeof FilmFilmmakerViewSchema>

// Schema per lista film filmmaker (subset campi)
export const FilmListItemSchema = FilmFilmmakerViewSchema.pick({
  id: true,
  title: true,
  slug: true,
  status: true,
  cfStreamStatus: true,
  thumbnailUrl: true,
  priceEur: true,
  viewCount: true,
  purchaseCount: true,
  createdAt: true,
})
export type FilmListItem = z.infer<typeof FilmListItemSchema>

// Schema risposta upload URL video
export const FilmUploadUrlResponseSchema = z.object({
  uploadUrl: z.string().url(),
  cfStreamId: z.string(),
  expiresAt: z.date(),
})
export type FilmUploadUrlResponse = z.infer<typeof FilmUploadUrlResponseSchema>

// Schema risposta upload URL thumbnail
export const ThumbnailUploadUrlResponseSchema = z.object({
  uploadUrl: z.string().url(),
  r2Key: z.string(),
  thumbnailUrl: z.string().url(),
})
export type ThumbnailUploadUrlResponse = z.infer<typeof ThumbnailUploadUrlResponseSchema>

// Schema body webhook Cloudflare Stream
export const CfStreamWebhookSchema = z.object({
  uid: z.string(),
  status: z.object({
    state: z.string(),
    pctComplete: z.string().optional(),
    errorReasonCode: z.string().optional(),
    errorReasonText: z.string().optional(),
  }),
})
export type CfStreamWebhook = z.infer<typeof CfStreamWebhookSchema>
```

Esportati da `packages/domain/src/index.ts`.

---

## Server Functions

File: `apps/web/app/features/film/film.server.ts`

Tutte le server functions che modificano dati usano `method: 'POST'`. Le funzioni di lettura usano `method: 'GET'`.

### `createFilm`

```typescript
export const createFilm = createServerFn({ method: 'POST' })
  .validator(CreateFilmSchema)
  .handler(async ({ data }): Promise<ResultShape<{ filmId: FilmId }, ForbiddenError | PayoutNotEnabledError | ValidationError>> => {
    const user = await requireRole('filmmaker')
    // guard: payout_enabled
    const profile = await getFilmmakerProfile(user.id)
    if (!profile?.payoutEnabled) return err(new PayoutNotEnabledError())
    // genera slug unico
    const slug = await resolveUniqueSlug(generateSlug(data.title), db)
    // insert film
    const [film] = await db.insert(films).values({ ...data, filmakerId: user.id, slug }).returning({ id: films.id })
    return ok({ filmId: FilmIdSchema.parse(film.id) })
  })
```

### `updateFilmMetadata`

```typescript
export const updateFilmMetadata = createServerFn({ method: 'POST' })
  .validator(UpdateFilmMetadataSchema)
  .handler(async ({ data }): Promise<ResultShape<void, ForbiddenError | FilmNotFoundError | FilmNotOwnedError | FilmStatusError | ValidationError>> => {
    const user = await requireRole('filmmaker')
    const film = await getFilmOrError(data.filmId)
    if (!film) return err(new FilmNotFoundError())
    if (film.filmakerId !== user.id) return err(new FilmNotOwnedError())
    if (!['draft', 'rejected'].includes(film.status)) return err(new FilmStatusError(film.status))
    // ricalcola slug solo se title cambia
    const slug = data.title && data.title !== film.title
      ? await resolveUniqueSlug(generateSlug(data.title), db, data.filmId)
      : undefined
    await db.update(films).set({ ...data, ...(slug ? { slug } : {}), updatedAt: new Date() }).where(eq(films.id, data.filmId))
    return ok(undefined)
  })
```

### `getFilmUploadUrl`

```typescript
export const getFilmUploadUrl = createServerFn({ method: 'POST' })
  .validator(z.object({ filmId: FilmIdSchema }))
  .handler(async ({ data }): Promise<ResultShape<FilmUploadUrlResponse, ForbiddenError | PayoutNotEnabledError | FilmNotFoundError | FilmNotOwnedError | FilmStatusError>> => {
    const user = await requireRole('filmmaker')
    const profile = await getFilmmakerProfile(user.id)
    if (!profile?.payoutEnabled) return err(new PayoutNotEnabledError())
    const film = await getFilmOrError(data.filmId)
    if (!film) return err(new FilmNotFoundError())
    if (film.filmakerId !== user.id) return err(new FilmNotOwnedError())
    if (!['draft', 'rejected'].includes(film.status)) return err(new FilmStatusError(film.status))
    // Cloudflare Stream: crea TUS direct upload
    const cfUpload = await cloudflareStream.videos.directUpload.create({
      maxDurationSeconds: 14400,  // 4h max
      expiry: new Date(Date.now() + 60 * 60 * 1000).toISOString(),
    })
    // salva cf_stream_id e imposta processing
    await db.update(films)
      .set({ cfStreamId: cfUpload.uid, cfStreamStatus: 'processing', updatedAt: new Date() })
      .where(eq(films.id, data.filmId))
    return ok(FilmUploadUrlResponseSchema.parse({
      uploadUrl: cfUpload.uploadURL,
      cfStreamId: cfUpload.uid,
      expiresAt: new Date(cfUpload.expiry),
    }))
  })
```

### `getThumbnailUploadUrl`

```typescript
export const getThumbnailUploadUrl = createServerFn({ method: 'POST' })
  .validator(z.object({ filmId: FilmIdSchema, mimeType: z.enum(['image/jpeg', 'image/png', 'image/webp']) }))
  .handler(async ({ data }): Promise<ResultShape<ThumbnailUploadUrlResponse, ForbiddenError | PayoutNotEnabledError | FilmNotFoundError | FilmNotOwnedError | FilmStatusError>> => {
    const user = await requireRole('filmmaker')
    const profile = await getFilmmakerProfile(user.id)
    if (!profile?.payoutEnabled) return err(new PayoutNotEnabledError())
    const film = await getFilmOrError(data.filmId)
    if (!film) return err(new FilmNotFoundError())
    if (film.filmakerId !== user.id) return err(new FilmNotOwnedError())
    if (!['draft', 'rejected'].includes(film.status)) return err(new FilmStatusError(film.status))
    const ext = data.mimeType.split('/')[1]
    const r2Key = `thumbnails/${data.filmId}/${crypto.randomUUID()}.${ext}`
    const uploadUrl = await r2.createPresignedUrl('PUT', r2Key, { expiresIn: 900 })
    const thumbnailUrl = `${env.R2_PUBLIC_URL}/${r2Key}`
    return ok({ uploadUrl, r2Key, thumbnailUrl })
  })
```

### `confirmThumbnailUpload`

```typescript
export const confirmThumbnailUpload = createServerFn({ method: 'POST' })
  .validator(z.object({ filmId: FilmIdSchema, r2Key: z.string() }))
  .handler(async ({ data }): Promise<ResultShape<void, ForbiddenError | FilmNotFoundError | FilmNotOwnedError>> => {
    const user = await requireRole('filmmaker')
    const film = await getFilmOrError(data.filmId)
    if (!film) return err(new FilmNotFoundError())
    if (film.filmakerId !== user.id) return err(new FilmNotOwnedError())
    const thumbnailUrl = `${env.R2_PUBLIC_URL}/${data.r2Key}`
    await db.update(films).set({ thumbnailUrl, r2OriginalKey: data.r2Key, updatedAt: new Date() }).where(eq(films.id, data.filmId))
    return ok(undefined)
  })
```

### `submitFilmForReview`

```typescript
export const submitFilmForReview = createServerFn({ method: 'POST' })
  .validator(z.object({ filmId: FilmIdSchema }))
  .handler(async ({ data }): Promise<ResultShape<void, ForbiddenError | PayoutNotEnabledError | FilmNotFoundError | FilmNotOwnedError | FilmStatusError | FilmNotReadyError>> => {
    const user = await requireRole('filmmaker')
    const profile = await getFilmmakerProfile(user.id)
    if (!profile?.payoutEnabled) return err(new PayoutNotEnabledError())
    const film = await getFilmOrError(data.filmId)
    if (!film) return err(new FilmNotFoundError())
    if (film.filmakerId !== user.id) return err(new FilmNotOwnedError())
    if (!['draft', 'unpublished'].includes(film.status)) return err(new FilmStatusError(film.status))
    if (film.cfStreamStatus !== 'ready') return err(new FilmNotReadyError('video not ready'))
    if (!film.thumbnailUrl) return err(new FilmNotReadyError('thumbnail missing'))
    await db.update(films).set({ status: 'under_review', updatedAt: new Date() }).where(eq(films.id, data.filmId))
    return ok(undefined)
  })
```

### `listFilmmakerFilms`

```typescript
export const listFilmmakerFilms = createServerFn({ method: 'GET' })
  .handler(async (): Promise<ResultShape<FilmListItem[], ForbiddenError>> => {
    const user = await requireRole('filmmaker')
    const rows = await db
      .select()
      .from(films)
      .where(and(eq(films.filmakerId, user.id), isNull(films.deletedAt)))
      .orderBy(desc(films.createdAt))
    return ok(rows.map(r => FilmListItemSchema.parse(r)))
  })
```

### `getFilmmakerFilm`

```typescript
export const getFilmmakerFilm = createServerFn({ method: 'GET' })
  .validator(z.object({ filmId: FilmIdSchema }))
  .handler(async ({ data }): Promise<ResultShape<FilmFilmmakerView, ForbiddenError | FilmNotFoundError | FilmNotOwnedError>> => {
    const user = await requireRole('filmmaker')
    const film = await db.query.films.findFirst({
      where: and(eq(films.id, data.filmId), isNull(films.deletedAt)),
    })
    if (!film) return err(new FilmNotFoundError())
    if (film.filmakerId !== user.id) return err(new FilmNotOwnedError())
    return ok(FilmFilmmakerViewSchema.parse(film))
  })
```

---

## Webhook Cloudflare Stream

File: `apps/api/src/webhooks/cloudflare-stream.ts`

Endpoint: `POST /webhooks/cloudflare-stream`

Cloudflare Stream invia il webhook con l'header `Webhook-Signature` (HMAC-SHA256). La chiave è `CF_STREAM_WEBHOOK_SECRET` (variabile d'ambiente).

```typescript
async function handleCfStreamWebhook(req: FastifyRequest, reply: FastifyReply) {
  const signature = req.headers['webhook-signature']
  if (!verifyCfStreamSignature(req.rawBody, signature, env.CF_STREAM_WEBHOOK_SECRET)) {
    return reply.status(401).send()
  }

  const parsed = CfStreamWebhookSchema.safeParse(req.body)
  if (!parsed.success) return reply.status(200).send()  // ignora payload malformati

  const { uid, status } = parsed.data
  if (status.state !== 'ready' && status.state !== 'error') {
    return reply.status(200).send()  // stati intermedi ignorati
  }

  const cfStreamStatus = status.state === 'ready' ? 'ready' : 'error'

  const film = await db.query.films.findFirst({
    where: eq(films.cfStreamId, uid),
  })

  if (!film) {
    logger.warn({ cfStreamId: uid }, 'cloudflare-stream webhook: film not found')
    return reply.status(200).send()
  }

  await db.update(films)
    .set({ cfStreamStatus, updatedAt: new Date() })
    .where(eq(films.cfStreamId, uid))

  logger.info({ filmId: film.id, cfStreamId: uid, cfStreamStatus }, 'cloudflare-stream status updated')
  return reply.status(200).send()
}
```

Il webhook risponde sempre `200` (idempotente). Errori interni vengono loggati, mai restituiti a Cloudflare.

---

## Pagine e Componenti

### Route

```
/_filmmaker (layout — richiede ruolo filmmaker)
  /creator/films                  → listFilmmakerFilms
  /creator/films/new              → form creazione → createFilm → redirect /creator/films/[id]
  /creator/films/$id              → getFilmmakerFilm
```

### `/creator/films`

Carica `listFilmmakerFilms` nel loader. Mostra una tabella con:
- Titolo, stato (badge colorato), `cf_stream_status`, data creazione, views, acquisti
- CTA "Nuovo film" → `/creator/films/new`
- Click su riga → `/creator/films/[id]`

Se `payout_enabled = false`: banner di avviso "Completa l'onboarding Stripe per poter caricare film" con link a `/creator/payouts`. I film in lista restano visibili ma la CTA "Nuovo film" è disabilitata.

### `/creator/films/new`

Form con i campi `CreateFilmSchema`. Alla submit chiama `createFilm`. In caso di successo, redirect a `/creator/films/[id]`.

Se `payout_enabled = false`: redirect immediato a `/creator/payouts` con banner.

### `/creator/films/$id`

Pagina principale di gestione film. Carica `getFilmmakerFilm` nel loader.

Layout a sezioni:

**Sezione Metadati**
- Form con tutti i campi `UpdateFilmMetadataSchema`
- Pulsante "Salva" → `updateFilmMetadata`
- Abilitato solo se `status IN ('draft', 'rejected')`

**Sezione Video**
- Componente `UploadForm` (dettaglio sotto)
- Se `cf_stream_status = 'ready'`: badge "Video pronto" + preview thumbnail automatica da CF
- Se `cf_stream_status = 'processing'`: badge "Encoding in corso" + spinner + polling attivo
- Se `cf_stream_status = 'error'`: badge "Errore encoding" + pulsante "Riprova"
- Se nessun video ancora: pulsante "Carica video"

**Sezione Thumbnail**
- Input file (`.jpg`, `.png`, `.webp`, max 5MB)
- Se thumbnail presente: anteprima + pulsante "Sostituisci"
- Upload: `getThumbnailUploadUrl` → PUT diretto su R2 → `confirmThumbnailUpload`

**Sezione Stato e Azioni**
- Badge stato corrente
- Se `status = 'rejected'`: banner con `rejection_reason` + pulsante "Correggi e risubmetti" → `revertFilmToDraft` → `submitFilmForReview`
- Se `status = 'draft'` e film pronto (video ready + thumbnail presente): pulsante "Invia in revisione" → `submitFilmForReview`
- Se `status = 'under_review'`: banner "In attesa di revisione"
- Se `status = 'published'`: pulsante "Ritira dalla pubblicazione"

### Componente `UploadForm`

File: `apps/web/app/features/film/components/UploadForm.tsx`

```
stato interno: idle | requesting_url | uploading | polling | ready | error
```

Comportamento:

1. **`idle`**: pulsante "Carica video" (accetta `.mp4`, `.mov`, `.avi`, `.mkv`, max 50GB).
2. **`requesting_url`**: chiama `getFilmUploadUrl`, spinner.
3. **`uploading`**: upload TUS verso `uploadUrl`. Progress bar (percentuale da eventi TUS). Pulsante "Annulla" (annulla l'upload, non elimina il film).
4. **`polling`**: upload completato, encoding in corso. Chiama `getFilmmakerFilm` ogni 10s. Spinner con messaggio "Encoding in corso...".
5. **`ready`**: `cf_stream_status = 'ready'`. Mostra badge + eventuale anteprima.
6. **`error`**: `cf_stream_status = 'error'` oppure upload fallito. Messaggio di errore specifico + pulsante "Riprova" (riparte da `idle`).

Il componente usa una libreria TUS client (da approvare prima dell'implementazione) per il resume automatico in caso di interruzione di rete.

Il polling usa `setInterval` con cleanup su unmount. Intervallo: 10 secondi. Timeout massimo: 30 minuti (dopo cui mostra errore "Encoding troppo lungo, contatta il supporto").

---

## Errori

File: `packages/utils/src/errors.ts` — aggiunti agli errori esistenti.

```typescript
export class FilmNotFoundError {
  readonly _tag = 'FilmNotFoundError' as const
  readonly message = 'Film not found'
}

export class FilmNotOwnedError {
  readonly _tag = 'FilmNotOwnedError' as const
  readonly message = 'Film does not belong to this filmmaker'
}

export class FilmNotReadyError {
  readonly _tag = 'FilmNotReadyError' as const
  readonly message: string
  constructor(readonly reason: 'video not ready' | 'thumbnail missing') {
    this.message = `Film not ready for review: ${reason}`
  }
}

export class FilmStatusError {
  readonly _tag = 'FilmStatusError' as const
  readonly message: string
  constructor(readonly currentStatus: string) {
    this.message = `Operation not allowed in status: ${currentStatus}`
  }
}

export class PayoutNotEnabledError {
  readonly _tag = 'PayoutNotEnabledError' as const
  readonly message = 'Stripe Connect onboarding not completed'
}
```

Discriminazione via `_tag` con `ts-pattern`. Mai `instanceof`.

---

## Variabili d'Ambiente

Aggiunte a quelle esistenti:

| Variabile | Package | Descrizione |
|---|---|---|
| `CF_STREAM_ACCOUNT_ID` | `apps/api`, `apps/web` | Account ID Cloudflare |
| `CF_STREAM_API_TOKEN` | `apps/api`, `apps/web` | Token API Cloudflare Stream |
| `CF_STREAM_WEBHOOK_SECRET` | `apps/api` | Secret per verifica firma webhook |
| `R2_ACCOUNT_ID` | `apps/web` | Account ID Cloudflare R2 |
| `R2_ACCESS_KEY_ID` | `apps/web` | Access key R2 |
| `R2_SECRET_ACCESS_KEY` | `apps/web` | Secret key R2 |
| `R2_BUCKET_NAME` | `apps/web` | Nome bucket R2 |
| `R2_PUBLIC_URL` | `apps/web` | URL CDN pubblico R2 (es. `https://cdn.nobo.film`) |

---

## Fuori Scope

| Funzionalità | Note |
|---|---|
| Approvazione / rifiuto film da parte dell'admin | Spec `04-admin-review.md` |
| Visualizzazione film pubblicati nel catalogo pubblico | Spec `05-catalog.md` |
| Acquisto e streaming film (viewer) | Spec `06-purchases.md` |
| Trailer upload | Stessa infrastruttura del video principale — da aggiungere in Fase 2 |
| Sottotitoli | Fase 2 |
| Modifica film in stato `published` | Richiede ritiro preventivo (`unpublish`) — già modellato negli stati |
| Notifica email all'admin al submit | Da definire in spec `04-admin-review.md` |
| Notifica email al filmmaker all'approvazione/rifiuto | Da definire in spec `04-admin-review.md` |
| Eliminazione film (soft delete) | Consentita solo in `draft` e `rejected` — implementazione banale, non richiede spec separata |
| Analytics film (views, revenue) | Spec `07-analytics.md` |
| Re-upload video (sostituzione) | Stesso flusso di `getFilmUploadUrl` — il vecchio `cf_stream_id` viene sovrascritto |
| DRM (Widevine/FairPlay) | Fase 3+ |

---

*Spec 03 — v1.0 — Aprile 2026*
