# 05 — Catalogo Pubblico

## Obiettivo

Esporre il catalogo dei film pubblicati con filtri e paginazione, le pagine di dettaglio film, e i profili pubblici dei filmmaker. Nessuna autenticazione richiesta per la navigazione.

---

## Flussi

### Browse catalogo

1. Utente (autenticato o no) accede a `/films`.
2. Server function `listPublishedFilms` recupera i film con filtri e paginazione cursor-based.
3. La UI mostra la griglia dei film con titolo, thumbnail, prezzo, genere e nome filmmaker.
4. Filtri disponibili: genere, ordinamento (`newest` | `popular` | `price_asc`).
5. Paginazione: il client invia il cursor dell'ultimo film ricevuto; il server restituisce il batch successivo.

### Pagina singolo film

1. Utente accede a `/films/$slug`.
2. Server function `getFilmBySlug`:
   - Se il film non esiste → `FilmNotFoundError`.
   - Se `film.status !== 'published'` → `FilmNotPublishedError`.
   - Se l'utente è autenticato, controlla se ha già acquistato il film (`already_purchased`).
3. Il trailer è accessibile tramite signed URL con TTL 1h generato da `getFilmTrailerUrl`.
4. La CTA mostra "Acquista" se non ancora acquistato, "Guarda" se già acquistato, oppure solo il trailer se non autenticato.

### Profilo pubblico filmmaker

1. Utente accede a `/filmmaker/$id`.
2. Server function `getPublicFilmmakerProfile` restituisce dati pubblici del filmmaker e lista film pubblicati.

---

## Schema DB (campi rilevanti)

```sql
-- films
id           uuid PRIMARY KEY
slug         text UNIQUE NOT NULL
title        text NOT NULL
description  text
genre        text
price_eur    numeric(6,2)
status       text NOT NULL  -- solo 'published' è visibile nel catalogo
view_count   integer DEFAULT 0
published_at timestamptz
filmmaker_id uuid REFERENCES users(id)

-- filmmaker_profiles
user_id      uuid PRIMARY KEY REFERENCES users(id)
display_name text
bio          text
avatar_url   text
```

---

## Tipi (Zod — `packages/domain`)

```ts
export const FilmListQuerySchema = z.object({
  genre: z.string().optional(),
  sort: z.enum(['newest', 'popular', 'price_asc']).default('newest'),
  cursor: z.string().uuid().optional(),
  limit: z.number().int().min(1).max(50).default(20),
})
export type FilmListQuery = z.infer<typeof FilmListQuerySchema>

export const FilmPublicSchema = z.object({
  id: z.string().uuid(),
  slug: z.string(),
  title: z.string(),
  description: z.string().nullable(),
  genre: z.string().nullable(),
  priceEur: z.number(),
  viewCount: z.number().int(),
  publishedAt: z.coerce.date(),
  thumbnailUrl: z.string().url().nullable(),
  filmmaker: z.object({
    id: z.string().uuid(),
    displayName: z.string(),
    avatarUrl: z.string().url().nullable(),
  }),
  alreadyPurchased: z.boolean().optional(),
})
export type FilmPublic = z.infer<typeof FilmPublicSchema>

export const FilmListResultSchema = z.object({
  items: z.array(FilmPublicSchema),
  nextCursor: z.string().uuid().nullable(),
  total: z.number().int(),
})
export type FilmListResult = z.infer<typeof FilmListResultSchema>

export const FilmmakerPublicProfileSchema = z.object({
  id: z.string().uuid(),
  displayName: z.string(),
  bio: z.string().nullable(),
  avatarUrl: z.string().url().nullable(),
  publishedFilms: z.array(FilmPublicSchema),
})
export type FilmmakerPublicProfile = z.infer<typeof FilmmakerPublicProfileSchema>
```

---

## Server Functions (`apps/web` via `createServerFn`)

### `listPublishedFilms(query: FilmListQuery)`

- **Auth**: non richiesta.
- **Query**: `SELECT ... FROM films WHERE status = 'published' [AND genre = ?] ORDER BY [sort] LIMIT limit + 1`.
- Paginazione cursor-based: se `cursor` è presente, aggiunge `WHERE id > cursor` (o equivalente per il sort scelto).
- Se vengono restituiti `limit + 1` elementi, il cursor è l'id dell'ultimo elemento e viene rimosso dalla lista.
- **Return**: `ResultAsync<FilmListResult, never>`.

### `getFilmBySlug(slug: string, userId?: string)`

- **Auth**: opzionale. Se `userId` è presente, controlla `purchases` per impostare `alreadyPurchased`.
- Se non esiste → `FilmNotFoundError`.
- Se `status !== 'published'` → `FilmNotPublishedError`.
- **Return**: `ResultAsync<FilmPublic & { alreadyPurchased: boolean }, FilmNotFoundError | FilmNotPublishedError>`.

### `getFilmTrailerUrl(filmId: string)`

- **Auth**: non richiesta.
- Genera un signed URL verso il provider di storage (es. Cloudflare R2 o S3) con TTL di 1 ora.
- Se il film non ha un trailer → `FilmNotFoundError`.
- **Return**: `ResultAsync<{ url: string; expiresAt: Date }, FilmNotFoundError>`.

### `getPublicFilmmakerProfile(filmMakerId: string)`

- **Auth**: non richiesta.
- Fetcha `filmmaker_profiles` + lista film `published` del filmmaker.
- Se non esiste → `FilmNotFoundError`.
- **Return**: `ResultAsync<FilmmakerPublicProfile, FilmNotFoundError>`.

---

## Pagine e Componenti

### `/films`

- Loader chiama `listPublishedFilms` con query params dalla URL.
- Griglia di card film: thumbnail, titolo, genere, prezzo, nome filmmaker.
- Filtri nella sidebar o header: select genere, select ordinamento.
- Paginazione: pulsante "Carica altri" che appende il batch successivo (no pagine numerate).
- Aggiornamento URL con i filtri attivi (query string) per URL condivisibile.

### `/films/$slug`

- Loader chiama `getFilmBySlug` e `getFilmTrailerUrl`.
- Sezioni: trailer (video embed), titolo, descrizione, genere, filmmaker (link a profilo), prezzo.
- CTA:
  - Non autenticato: "Accedi per acquistare".
  - Autenticato, non acquistato: `PurchaseButton` (spec 06).
  - Autenticato, già acquistato: pulsante "Guarda" → `/watch/$filmId`.
- Meta tags SEO: `<title>`, `<meta name="description">`, `og:title`, `og:description`, `og:image` (thumbnail).
- Structured data JSON-LD: schema `Movie` con `name`, `description`, `image`, `author`.

### `/filmmaker/$id`

- Loader chiama `getPublicFilmmakerProfile`.
- Header: avatar, display name, bio.
- Griglia film pubblicati del filmmaker (stessa card di `/films`).

---

## SEO

- Ogni pagina film genera meta tags specifici per il film.
- `og:image` punta alla thumbnail del film (URL pubblico).
- JSON-LD con tipo `Movie`:
  ```json
  {
    "@context": "https://schema.org",
    "@type": "Movie",
    "name": "...",
    "description": "...",
    "image": "...",
    "author": { "@type": "Person", "name": "..." }
  }
  ```
- Le pagine del catalogo (`/films`) non richiedono structured data avanzato.

---

## Errori

```ts
export class FilmNotFoundError {
  readonly _tag = 'FilmNotFoundError'
  constructor(public readonly identifier: string) {}
}

export class FilmNotPublishedError {
  readonly _tag = 'FilmNotPublishedError'
  constructor(
    public readonly filmId: string,
    public readonly currentStatus: string,
  ) {}
}
```

---

## Fuori Scope

- Ricerca full-text.
- Raccomandazioni personalizzate.
- Watchlist / preferiti.
- Filtri per prezzo (range).
- Recensioni e valutazioni degli utenti.
