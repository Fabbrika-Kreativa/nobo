# Nobo — CLAUDE.md

Before touching any code, read these files in order:

1. `NOBO_TECHNICAL_SPEC.md` — stack, schema database, API, architettura
2. `NOBO_PROJECT_DOCUMENT.md` — contesto business, modello, roadmap
3. This file

---

## Workflow

1. Ogni feature approvata produce una spec in `docs/specs/NN-feature-name.md` prima che inizi qualsiasi codice
2. Leggi lo schema database e il domain package prima di toccare qualsiasi feature
3. Valida tutti gli input con Zod prima che giri qualsiasi logica
4. Ogni logica server va in `createServerFn` — mai in componenti React, mai in raw fetch
5. Testa con Vitest (logica pura) e Playwright (flussi browser)
6. Code-review della diff prima di ogni commit — `/caveman-review` sulla change, risolvi critici, poi commit
7. Commit seguendo le convenzioni Git sotto

When in doubt about an architectural decision, stop and ask.

---

## Stack

| Layer | Tool |
|---|---|
| Framework | TanStack Start (SSR, file-based routing) |
| Routing | TanStack Router |
| Server calls | `createServerFn` — unico modo per chiamare server logic |
| Data fetching | TanStack Query (`queryOptions` + `useSuspenseQuery`) |
| ORM | Drizzle + PostgreSQL |
| Auth | Better Auth (bearer token + cookie, niente hard-coupling a cookies) |
| Styling | CSS Modules — zero Tailwind, zero CSS-in-JS |
| Validation | Zod — source of truth per tutti i tipi |
| Error handling | neverthrow — `Result` e `ResultAsync` |
| Video | Cloudflare Stream |
| Storage | Cloudflare R2 |
| Pagamenti | Stripe + Stripe Connect |
| Testing | Vitest (unit) + Playwright (E2E) |

---

## Struttura Monorepo

```
nobo/
├── apps/
│   ├── web/               # TanStack Start (viewer + filmmaker portal + admin)
│   └── api/               # Fastify (webhook: Stripe, Cloudflare, Better Auth)
├── packages/
│   ├── db/                # Drizzle schema + migrations
│   ├── domain/            # Zod schemas, branded types, business rules — framework-agnostic
│   └── utils/             # ResultShape, toShape, unwrapResult, errori condivisi
├── docs/
│   └── specs/             # NN-feature-name.md — una spec per feature
├── package.json
└── turbo.json
```

---

## Domain Boundaries

- `features/auth/` — sessione, utente, ruoli
- `features/catalog/` — film, slug, metadata, stato pubblicazione
- `features/filmmaker/` — profilo, upload, analytics, payout
- `features/viewer/` — acquisti, watch session, progress
- `features/payments/` — Stripe payment intent, transfer, split calcolo
- `features/admin/` — review queue, approvazione/rifiuto film
- `features/video/` — upload URL Cloudflare, signed URL streaming, webhook encoding

---

## Server Functions

Ogni client→server interaction va attraverso `createServerFn`. Mai tRPC, mai raw fetch a endpoint interni.

```typescript
import { createServerFn } from '@tanstack/start'
import { ok, err, ResultAsync } from 'neverthrow'
import { toShape } from '@nobo/utils'
import type { ResultShape } from '@nobo/utils'
import { requireUser } from '~/server/context'
import { getDb } from '~/server/db'

export const getFilm = createServerFn({ method: 'GET' })
  .validator(z.object({ slug: z.string() }))
  .handler(async ({ data }): Promise<ResultShape<Film, NotFoundError | DbError>> => {
    const db = await getDb()
    return toShape(await findFilmBySlug(db, data.slug))
  })
```

Ogni `createServerFn` che muta dati richiede:
- Zod validation via `.validator()`
- Auth check via `requireUser()` (o `requireRole()`)
- Permission check sul resource
- `toShape()` al boundary di ritorno

---

## Error Handling

Errori sono valori. Mai `throw` per errori di dominio.

```typescript
// Mai questo per errori di dominio
if (!film) throw new Error('Not found')

// Sempre questo
if (!film) return err(new NotFoundError('film', id))
```

Errori condivisi (`ForbiddenError`, `DbError`, `NotFoundError`) in `packages/utils/src/errors.ts`.
Errori di dominio specifici in `features/X/X.errors.ts`, riesportano quelli condivisi.

Tutti gli error class usano `_tag` per discriminazione con `ts-pattern`, non `instanceof`.

```typescript
export class NotFoundError {
  readonly _tag = 'NotFoundError' as const
  readonly message: string
  constructor(readonly entity: string, readonly id: string) {
    this.message = `${entity} not found: ${id}`
  }
}
```

---

## Tipi

Zod è l'unica source of truth. Mai scrivere tipi TypeScript a mano.

```typescript
// Mai
type Film = { id: string; title: string; priceEur: number }

// Sempre
export const FilmSchema = z.object({
  id: z.string().uuid(),
  title: z.string().min(1).max(200),
  priceEur: z.number().positive()
})
export type Film = z.infer<typeof FilmSchema>
```

Tipi Drizzle sempre inferiti:
```typescript
type FilmRow = typeof films.$inferSelect
type NewFilm = typeof films.$inferInsert
```

Branded types per tutti gli ID:
```typescript
type FilmId = Brand<string, 'FilmId'>
type UserId = Brand<string, 'UserId'>
```

---

## CSS

- CSS Modules — un `.module.css` per componente
- Custom properties per ogni valore — mai hex hardcoded, mai px magici
- `--radius-md` come default border-radius
- Flexbox come default, grid solo per layout 2D espliciti
- Container queries, non media queries per i componenti
- Proprietà logiche (`margin-inline-end` non `margin-right`)
- CSS nesting nativo — max 2 livelli
- Animazioni solo CSS, mai Framer Motion — sempre con `prefers-reduced-motion`
- Class names in camelCase

---

## React Patterns

State con `useReducer` + `ts-pattern`:

```typescript
const reducer = (state: State, action: Action): State =>
  match(action)
    .with({ type: 'set/film' }, ({ payload }) => ({ ...state, film: payload }))
    .exhaustive()
```

Action creators come pure functions:
```typescript
const actions = {
  setFilm: (payload: Film): Action => ({ type: 'set/film', payload }),
} as const
```

---

## Testing

- Vitest per logica pura: parser, transformer, validazione, error handling
- Playwright per flussi browser: auth, acquisto, upload, watch
- Co-locate test Vitest: `film.server.test.ts` accanto a `film.server.ts`
- Test Playwright in `tests/` con tag `[NOBO-001]`
- Ogni mutation ha almeno un test

---

## Platform Reach

Target futuro: web (primario MVP), mobile + Apple TV + Android TV (Fase 3 via Expo).

**Implicazione pratica oggi:** tutta la logica di dominio vive in `packages/domain` — zero import React, zero import TanStack, zero browser API. Se una funzione trasforma dati o applica una regola di business, deve poter girare su Expo senza modifiche.

Non scrivere codice Expo ora. Astrarre in modo che aggiungerlo non richieda rewrite.

---

## Never Do

- **Mai usare tRPC** — solo `createServerFn`
- **Mai query DB sul client** — tutte le chiamate Drizzle dentro `createServerFn`
- **Mai `try/catch`** per errori attesi — usa `ResultAsync.fromPromise` e typed errors
- **Mai mutare** state, array, oggetti — sempre nuovi valori
- **Mai scrivere tipi TypeScript a mano** — inferisci da Zod o Drizzle
- **Mai Tailwind**, utility classes, CSS-in-JS
- **Mai `null` e `undefined` misti** nello stesso tipo — usa `null` per assenza intenzionale
- **Mai esporre** chiavi API sul client
- **Mai loggare** token, password, API keys
- **Mai firme AI** nei commit
- **Mai hard-coupling auth a cookie** — Better Auth deve poter emettere bearer token per mobile futuro
- **Mai importare** React, TanStack, Monaco in `packages/domain` o `packages/utils`

---

## Git

Commit format: `[NOBO] type: description`

- `feat:` nuova funzionalità
- `fix:` bug fix
- `chore:` manutenzione, dipendenze
- `refactor:` nessun cambio di comportamento
- `test:` aggiunta o modifica test

Nessuna firma AI nei commit.
