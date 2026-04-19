# 04 — Coda di Review Admin

## Obiettivo

Permettere agli admin di revisionare i film in stato `under_review`, approvarli o rifiutarli con una nota, e notificare il filmmaker via email.

---

## Flussi

### Approvazione

1. Admin accede a `/admin/queue` e vede la lista dei film in stato `under_review`.
2. Admin seleziona un film e clicca "Approva".
3. Server function `approveFilm`:
   - Verifica che il film esista e sia in stato `under_review` → altrimenti `FilmStatusError`.
   - Aggiorna `films.status = 'published'` e `films.published_at = now()`.
   - Crea un record in `admin_reviews` con `action = 'approved'`.
   - Invia email al filmmaker via Resend.
4. La UI rimuove il film dalla coda e mostra un toast di conferma.

### Rifiuto

1. Admin seleziona un film e clicca "Rifiuta".
2. Si apre un dialog che richiede una nota obbligatoria.
3. Server function `rejectFilm`:
   - Verifica esistenza e stato `under_review` → altrimenti `FilmStatusError`.
   - Aggiorna `films.status = 'rejected'` e `films.rejection_reason = note`.
   - Crea un record in `admin_reviews` con `action = 'rejected'` e `notes = note`.
   - Invia email al filmmaker con la motivazione del rifiuto via Resend.
4. La UI rimuove il film dalla coda e mostra un toast di conferma.

---

## Schema DB

```sql
-- admin_reviews
id          uuid PRIMARY KEY DEFAULT gen_random_uuid()
film_id     uuid NOT NULL REFERENCES films(id)
reviewed_by uuid NOT NULL REFERENCES users(id)
action      text NOT NULL CHECK (action IN ('approved', 'rejected'))
notes       text
created_at  timestamptz NOT NULL DEFAULT now()

-- films (campi rilevanti)
status           text NOT NULL  -- draft | under_review | published | rejected | unpublished
published_at     timestamptz
rejection_reason text
```

---

## Tipi (Zod — `packages/domain`)

```ts
export const AdminReviewActionSchema = z.enum(['approved', 'rejected'])
export type AdminReviewAction = z.infer<typeof AdminReviewActionSchema>

export const AdminReviewSchema = z.object({
  id: z.string().uuid(),
  filmId: z.string().uuid(),
  reviewedBy: z.string().uuid(),
  action: AdminReviewActionSchema,
  notes: z.string().nullable(),
  createdAt: z.coerce.date(),
})
export type AdminReview = z.infer<typeof AdminReviewSchema>

export const ApproveFilmInputSchema = z.object({
  filmId: z.string().uuid(),
})
export type ApproveFilmInput = z.infer<typeof ApproveFilmInputSchema>

export const RejectFilmInputSchema = z.object({
  filmId: z.string().uuid(),
  notes: z.string().min(10).max(1000),
})
export type RejectFilmInput = z.infer<typeof RejectFilmInputSchema>

export const AdminStatsSchema = z.object({
  underReviewCount: z.number().int(),
  approvedCount: z.number().int(),
  rejectedCount: z.number().int(),
  publishedCount: z.number().int(),
})
export type AdminStats = z.infer<typeof AdminStatsSchema>
```

---

## Server Functions (`apps/web` via `createServerFn`)

### `listFilmsUnderReview`

- **Guard**: utente autenticato con ruolo `admin` → altrimenti `ForbiddenError`.
- **Query**: `SELECT films.*, users.email, users.name FROM films JOIN users ON films.filmmaker_id = users.id WHERE films.status = 'under_review' ORDER BY films.submitted_at ASC`.
- **Return**: `ResultAsync<FilmUnderReview[], ForbiddenError>`.

### `approveFilm(input: ApproveFilmInput)`

- **Guard**: ruolo `admin` → altrimenti `ForbiddenError`.
- Fetcha il film; se non esiste → `FilmNotFoundError`.
- Se `film.status !== 'under_review'` → `FilmStatusError`.
- Transazione:
  1. `UPDATE films SET status = 'published', published_at = now() WHERE id = filmId`.
  2. `INSERT INTO admin_reviews (film_id, reviewed_by, action) VALUES (...)`.
- Invia email al filmmaker con template "Film approvato" via Resend.
- **Return**: `ResultAsync<void, FilmNotFoundError | FilmStatusError | ForbiddenError>`.

### `rejectFilm(input: RejectFilmInput)`

- **Guard**: ruolo `admin` → altrimenti `ForbiddenError`.
- Fetcha il film; se non esiste → `FilmNotFoundError`.
- Se `film.status !== 'under_review'` → `FilmStatusError`.
- Transazione:
  1. `UPDATE films SET status = 'rejected', rejection_reason = notes WHERE id = filmId`.
  2. `INSERT INTO admin_reviews (film_id, reviewed_by, action, notes) VALUES (...)`.
- Invia email al filmmaker con template "Film rifiutato" + motivazione via Resend.
- **Return**: `ResultAsync<void, FilmNotFoundError | FilmStatusError | ForbiddenError>`.

### `getAdminStats`

- **Guard**: ruolo `admin` → altrimenti `ForbiddenError`.
- **Query**: `SELECT status, COUNT(*) FROM films GROUP BY status`.
- **Return**: `ResultAsync<AdminStats, ForbiddenError>`.

---

## Pagine e Componenti

### `/admin/queue`

- Loader chiama `listFilmsUnderReview`.
- Lista film con: titolo, filmmaker name, data sottomissione, thumbnail.
- Per ogni film: pulsante "Approva" e pulsante "Rifiuta".
- Rifiuto: modale con textarea per la nota (min 10 caratteri, obbligatoria).
- Sidebar o header con `AdminStats` (count per stato).

### `/admin/users`

- Lista utenti con ruolo, data registrazione, numero film.
- Fuori scope per questa spec: modifica ruoli, ban.

### Guard di navigazione

- Tutte le route `/admin/*` sono protette da un layout route che verifica il ruolo `admin` lato server.
- Se l'utente non è admin → redirect a `/` con messaggio di errore.

### Email templates (Resend)

- **Approvazione**: oggetto "Il tuo film è stato pubblicato", corpo con titolo film e link alla pagina pubblica.
- **Rifiuto**: oggetto "Il tuo film non è stato approvato", corpo con titolo film e motivazione del rifiuto.

---

## Errori

```ts
export class FilmNotFoundError {
  readonly _tag = 'FilmNotFoundError'
  constructor(public readonly filmId: string) {}
}

export class FilmStatusError {
  readonly _tag = 'FilmStatusError'
  constructor(
    public readonly filmId: string,
    public readonly currentStatus: string,
    public readonly expectedStatus: string,
  ) {}
}

export class ForbiddenError {
  readonly _tag = 'ForbiddenError'
  constructor(public readonly reason?: string) {}
}
```

---

## Fuori Scope

- Approvazione/rifiuto in bulk.
- Assegnazione review a un reviewer specifico.
- Commenti multipli o thread di revisione.
- Storico audit log visualizzabile dall'admin.
- Notifiche in-app (solo email).
