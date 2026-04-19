# 07 — Player e Tracking Visione

## Obiettivo

Permettere ai viewer con acquisto valido di guardare un film tramite un player video sicuro, tracciare il progresso della visione, e aggregare analytics giornaliere.

---

## Flussi

### Avvio visione

1. Viewer autenticato con acquisto valido accede a `/watch/$filmId`.
2. Loader chiama `getWatchStream({ filmId })`:
   - Verifica che il film sia `published` → altrimenti `FilmNotPublishedError`.
   - Verifica che esista un acquisto valido `(userId, filmId)` con `deleted_at IS NULL` → altrimenti `FilmNotPurchasedError`.
   - Controlla se esiste una `watch_session` aperta per `(userId, filmId)` (senza `ended_at`):
     - Se sì: riutilizza la sessione esistente e restituisce `last_position_seconds`.
     - Se no: crea una nuova `watch_session` con `started_at = now()`.
   - Genera un signed JWT per Cloudflare Stream lato server con TTL 4h firmato con la chiave privata CF.
   - Restituisce `signedUrl` e `lastPositionSeconds`.
3. La pagina monta il componente `VideoPlayer` con `src = signedUrl` e `initialPosition = lastPositionSeconds`.
4. Il player riprende automaticamente dalla posizione salvata.

### Tracking progresso

1. Il componente `VideoPlayer` invia `updateWatchProgress` ogni 30 secondi e all'evento `pause`/`ended`.
2. Server function `updateWatchProgress({ filmId, positionSeconds })`:
   - Recupera la `watch_session` attiva per `(userId, filmId)`.
   - Aggiorna `last_position_seconds = positionSeconds`.
   - Se `positionSeconds / film.duration_seconds > 0.85` → imposta `completed = true`.
   - Se è il primo aggiornamento della sessione (la sessione è appena stata creata) → incrementa `films.view_count` di 1.
   - Se `completed = true` → imposta `ended_at = now()`.
3. Al termine naturale del video (`ended` event), il player invia un ultimo aggiornamento con la posizione finale.

### Signed URL Cloudflare Stream

- Generato esclusivamente lato server, mai esposto il token master CF.
- JWT firmato con la chiave privata CF (`CF_STREAM_PRIVATE_KEY` env var).
- Payload JWT: `{ sub: videoId, kid: keyId, exp: now + 4h, iat: now }`.
- L'URL ha la forma: `https://customer-<code>.cloudflarestream.com/<token>/manifest/video.m3u8`.
- Il TTL di 4h copre la visione di un intero film con margine.
- Se la chiave CF non è disponibile o la firma fallisce → `StreamUrlError`.

---

## Schema DB

```sql
-- watch_sessions
id                      uuid PRIMARY KEY DEFAULT gen_random_uuid()
user_id                 uuid NOT NULL REFERENCES users(id)
film_id                 uuid NOT NULL REFERENCES films(id)
purchase_id             uuid NOT NULL REFERENCES purchases(id)
started_at              timestamptz NOT NULL DEFAULT now()
ended_at                timestamptz
last_position_seconds   integer NOT NULL DEFAULT 0
completed               boolean NOT NULL DEFAULT false
ip_country              text
device_type             text CHECK (device_type IN ('desktop', 'mobile', 'tablet'))

-- film_analytics (aggregata daily, non scritta da queste server functions)
id              uuid PRIMARY KEY DEFAULT gen_random_uuid()
film_id         uuid NOT NULL REFERENCES films(id)
date            date NOT NULL
views           integer NOT NULL DEFAULT 0
unique_viewers  integer NOT NULL DEFAULT 0
completions     integer NOT NULL DEFAULT 0
revenue_eur     numeric(10,2) NOT NULL DEFAULT 0
top_countries   jsonb
UNIQUE(film_id, date)

-- films (campo rilevante)
view_count  integer NOT NULL DEFAULT 0
```

---

## Tipi (Zod — `packages/domain`)

```ts
export const GetWatchStreamInputSchema = z.object({
  filmId: z.string().uuid(),
})
export type GetWatchStreamInput = z.infer<typeof GetWatchStreamInputSchema>

export const WatchStreamResultSchema = z.object({
  signedUrl: z.string().url(),
  expiresAt: z.coerce.date(),
  lastPositionSeconds: z.number().int().min(0),
  watchSessionId: z.string().uuid(),
})
export type WatchStreamResult = z.infer<typeof WatchStreamResultSchema>

export const UpdateWatchProgressInputSchema = z.object({
  filmId: z.string().uuid(),
  positionSeconds: z.number().int().min(0),
})
export type UpdateWatchProgressInput = z.infer<typeof UpdateWatchProgressInputSchema>

export const WatchSessionSchema = z.object({
  id: z.string().uuid(),
  userId: z.string().uuid(),
  filmId: z.string().uuid(),
  purchaseId: z.string().uuid(),
  startedAt: z.coerce.date(),
  endedAt: z.coerce.date().nullable(),
  lastPositionSeconds: z.number().int(),
  completed: z.boolean(),
  ipCountry: z.string().nullable(),
  deviceType: z.enum(['desktop', 'mobile', 'tablet']).nullable(),
})
export type WatchSession = z.infer<typeof WatchSessionSchema>
```

---

## Server Functions (`apps/web` via `createServerFn`)

### `getWatchStream(input: GetWatchStreamInput)`

- **Guard**: utente autenticato → altrimenti `ForbiddenError`.
- Fetcha il film; se `status !== 'published'` → `FilmNotPublishedError`.
- Controlla `purchases` per `(userId, filmId)` con `deleted_at IS NULL` → se non esiste → `FilmNotPurchasedError`.
- Cerca `watch_session` aperta: `WHERE user_id = ? AND film_id = ? AND ended_at IS NULL ORDER BY started_at DESC LIMIT 1`.
- Se non esiste: `INSERT INTO watch_sessions (user_id, film_id, purchase_id, device_type, ip_country)`.
- Genera signed JWT per Cloudflare Stream con TTL 4h. Se la firma fallisce → `StreamUrlError`.
- **Return**: `ResultAsync<WatchStreamResult, FilmNotPublishedError | FilmNotPurchasedError | StreamUrlError>`.

### `updateWatchProgress(input: UpdateWatchProgressInput)`

- **Guard**: utente autenticato.
- Recupera la `watch_session` attiva per `(userId, filmId)`.
- Aggiorna `last_position_seconds`.
- Se `positionSeconds / film.duration_seconds > 0.85`: `completed = true`, `ended_at = now()`.
- Se è il primo aggiornamento (la sessione ha `last_position_seconds = 0` prima dell'update): `UPDATE films SET view_count = view_count + 1 WHERE id = filmId`.
- Il controllo "primo aggiornamento" usa il valore di `last_position_seconds` prima dell'aggiornamento, non dopo.
- **Return**: `ResultAsync<void, FilmNotPurchasedError>`.

---

## Job Giornaliero — Aggregazione Analytics

Il job viene descritto qui per completezza architetturale; l'implementazione è fuori scope di questa spec.

Il job gira ogni notte (es. 02:00 UTC) e per ogni film con `watch_sessions` nel giorno precedente:

1. Conta `views` = numero di sessioni con `started_at::date = yesterday`.
2. Conta `unique_viewers` = `COUNT(DISTINCT user_id)` sulle sessioni del giorno.
3. Conta `completions` = sessioni con `completed = true` del giorno.
4. Calcola `revenue_eur` = `SUM(purchases.amount_eur)` per gli acquisti del giorno legati al film.
5. Calcola `top_countries` = `jsonb` con i top 5 paesi per numero di sessioni.
6. `INSERT INTO film_analytics ... ON CONFLICT (film_id, date) DO UPDATE SET ...`.

---

## Pagine e Componenti

### `/watch/$filmId`

- Layout minimal: no header principale, no footer.
- Loader chiama `getWatchStream`. Se fallisce con `FilmNotPurchasedError` → redirect a `/films/$slug` con query param `?error=not_purchased`.
- Mostra: titolo film in alto, componente `VideoPlayer`, link "← Torna al film".

### Componente `VideoPlayer`

- Stack: **Video.js** con plugin **HLS.js** per la riproduzione HLS.
- Props: `src: string`, `initialPosition: number`, `filmId: string`, `onComplete?: () => void`.
- **Resume automatico**: all'evento `loadedmetadata`, se `initialPosition > 0`, imposta `player.currentTime(initialPosition)` prima dell'autoplay.
- **Salvataggio posizione**: listener su `timeupdate` con throttle a 30 secondi. Invia `updateWatchProgress`.
- **Salvataggio su pausa**: listener su `pause` — invia `updateWatchProgress` immediatamente.
- **Completamento**: listener su `ended` — invia `updateWatchProgress` con posizione finale, chiama `onComplete`.
- **Retry su errore di rete**: Video.js `error` event — attende 3 secondi e tenta di ripartire dallo stesso punto (max 3 retry prima di mostrare errore all'utente).
- Nessun controllo qualità manuale esposto all'utente (HLS sceglie automaticamente il bitrate).

---

## Errori

```ts
export class FilmNotPurchasedError {
  readonly _tag = 'FilmNotPurchasedError'
  constructor(
    public readonly userId: string,
    public readonly filmId: string,
  ) {}
}

export class FilmNotPublishedError {
  readonly _tag = 'FilmNotPublishedError'
  constructor(
    public readonly filmId: string,
    public readonly currentStatus: string,
  ) {}
}

export class StreamUrlError {
  readonly _tag = 'StreamUrlError'
  constructor(public readonly reason: string) {}
}
```

---

## Fuori Scope

- DRM (Widevine, FairPlay).
- Download offline.
- Selezione manuale della qualità video.
- Chromecast / AirPlay / casting.
- Live streaming.
- Sottotitoli e tracce audio multiple.
- Player embeddabile in siti terzi.
