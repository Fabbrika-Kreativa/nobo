# Spec 01 — Autenticazione e Autorizzazione
## Nobo — Aprile 2026

---

## Obiettivo

Definire i flussi di autenticazione e autorizzazione per l'MVP di Nobo usando Better Auth come provider, con sincronizzazione verso la tabella `users` interna, gestione ruoli, protezione route e error handling tipizzato.

Perimetro: web app (`apps/web`). Il bearer token per mobile è predisposto nell'architettura ma non implementato in questa spec.

---

## Flussi Utente

### 1. Registrazione Email/Password

1. L'utente compila il form su `/register` (email, password, display_name).
2. Il form chiama `registerUser` (`createServerFn`), che valida l'input con `RegisterSchema`.
3. `registerUser` chiama `auth.api.signUpEmail` di Better Auth.
4. Better Auth crea il record nella propria tabella interna e invia l'email di verifica tramite Resend.
5. L'hook `onUserCreate` (webhook Better Auth → `apps/api`) crea il record nella tabella `users` con `better_auth_id`, `email`, `display_name`, `role = 'viewer'`.
6. L'utente viene reindirizzato a `/verify-email` con banner "Controlla la tua email".

L'accesso alla piattaforma è consentito anche prima della verifica email, ma alcune azioni (es. acquisto) richiedono email verificata.

### 2. Login Email/Password

1. L'utente compila il form su `/login`.
2. Il form chiama `loginUser` (`createServerFn`), validato con `LoginSchema`.
3. `loginUser` chiama `auth.api.signInEmail` di Better Auth.
4. Better Auth imposta il cookie di sessione (`HttpOnly`, `Secure`, `SameSite=Lax`).
5. `requireUser()` risolve la sessione dal cookie a ogni richiesta server successiva.
6. Redirect a `/account/profile` oppure all'URL di destinazione originale (query param `redirect`).

### 3. OAuth Google

1. L'utente clicca "Accedi con Google" su `/login` o `/register`.
2. La pagina reindirizza a `auth.api.signInOAuth({ provider: 'google' })`.
3. Better Auth gestisce il flow OAuth (redirect → callback → sessione).
4. Al callback, se è il primo accesso, Better Auth emette l'evento `onUserCreate`: l'hook crea il record in `users` con `display_name` ricavato dal profilo Google e `email_verified = true` (Google garantisce la verifica).
5. Se l'utente esiste già (stessa email), Better Auth collega l'account OAuth a quello esistente.
6. Redirect a `/account/profile`.

### 4. Logout

1. L'utente chiama `logoutUser` (`createServerFn`).
2. `logoutUser` chiama `auth.api.signOut`, che invalida la sessione e cancella il cookie.
3. Redirect a `/`.

### 5. Verifica Email

1. L'utente riceve l'email con link di verifica generato da Better Auth.
2. Il click sul link porta a `/verify-email?token=<token>`.
3. La pagina chiama `verifyEmail` (`createServerFn`) con il token.
4. Better Auth valida il token e marca l'email come verificata nella propria tabella.
5. L'hook `onEmailVerified` aggiorna `users.email_verified = true` (se si decide di denormalizzarlo — vedi note sotto).
6. Redirect a `/account/profile` con banner di conferma.

> **Nota**: `email_verified` non è nello schema `users` attuale. Se serve per gate delle funzionalità (es. acquisto), va aggiunto. Questa spec lo lascia fuori scope ma lo segnala come campo candidato da aggiungere prima di implementare il gate sull'acquisto.

---

## Sincronizzazione Better Auth → tabella `users`

Better Auth mantiene la propria tabella interna (`user`, `session`, `account` come da schema Better Auth). La tabella `users` di Nobo è la source of truth per i dati di dominio (ruolo, profilo, Stripe).

### Strategia: webhook interno

`apps/api` (Fastify) espone `POST /webhooks/better-auth`. Better Auth è configurato per emettere eventi verso questo endpoint.

| Evento Better Auth | Azione su `users` |
|---|---|
| `user.created` | `INSERT` con `better_auth_id`, `email`, `display_name`, `role = 'viewer'` |
| `user.updated` (email change) | `UPDATE email` |
| `user.deleted` | `UPDATE deleted_at = now()` (soft delete) |

Il webhook verifica la firma HMAC prima di processare qualsiasi evento.

### Lookup per `better_auth_id`

Ogni volta che `requireUser()` risolve la sessione Better Auth, il `better_auth_id` viene usato per recuperare il record `users` corrispondente:

```typescript
// apps/web/app/server/context.ts
const user = await db
  .select()
  .from(users)
  .where(eq(users.betterAuthId, session.user.id))
  .limit(1)
```

`requireUser()` restituisce il record `users` di Nobo, non il record Better Auth. Questo garantisce che `role` e tutti i campi di dominio siano quelli controllati da Nobo, non da Better Auth.

---

## Schema e Tipi

### Zod schemas — `packages/domain/src/auth/auth.schemas.ts`

```typescript
export const LoginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})
export type LoginInput = z.infer<typeof LoginSchema>

export const RegisterSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8).max(100),
  displayName: z.string().min(2).max(80),
})
export type RegisterInput = z.infer<typeof RegisterSchema>

export const UserRoleSchema = z.enum(['viewer', 'filmmaker', 'admin', 'b2b_partner'])
export type UserRole = z.infer<typeof UserRoleSchema>

export const AuthUserSchema = z.object({
  id: z.string().uuid().brand<'UserId'>(),
  betterAuthId: z.string(),
  email: z.string().email(),
  displayName: z.string(),
  role: UserRoleSchema,
  avatarUrl: z.string().url().nullable(),
  createdAt: z.date(),
})
export type AuthUser = z.infer<typeof AuthUserSchema>
```

### Branded types

```typescript
type UserId = Brand<string, 'UserId'>
```

Definito in `packages/domain/src/shared/branded.ts`, riesportato da `packages/domain`.

---

## Server Functions

Tutte le server functions relative all'auth vivono in `apps/web/app/features/auth/auth.server.ts`.

### `loginUser`

```typescript
export const loginUser = createServerFn({ method: 'POST' })
  .validator(LoginSchema)
  .handler(async ({ data }): Promise<ResultShape<AuthUser, UnauthenticatedError | ValidationError>> => {
    // chiama auth.api.signInEmail
    // in caso di credenziali errate → err(new UnauthenticatedError())
    // in caso di successo → fetchUserByBetterAuthId → ok(user)
  })
```

### `registerUser`

```typescript
export const registerUser = createServerFn({ method: 'POST' })
  .validator(RegisterSchema)
  .handler(async ({ data }): Promise<ResultShape<void, ValidationError | ConflictError>> => {
    // chiama auth.api.signUpEmail
    // se email già esistente → err(new ConflictError('email'))
  })
```

### `logoutUser`

```typescript
export const logoutUser = createServerFn({ method: 'POST' })
  .handler(async (): Promise<ResultShape<void, never>> => {
    // chiama auth.api.signOut
  })
```

### `verifyEmail`

```typescript
export const verifyEmail = createServerFn({ method: 'POST' })
  .validator(z.object({ token: z.string() }))
  .handler(async ({ data }): Promise<ResultShape<void, UnauthenticatedError>> => {
    // chiama auth.api.verifyEmail({ token })
  })
```

---

## `requireUser()` e `requireRole()`

File: `apps/web/app/server/context.ts`

```typescript
import { auth } from '~/server/auth'
import { db } from '@nobo/db'
import { users } from '@nobo/db/schema'
import { eq } from 'drizzle-orm'
import { UnauthenticatedError, ForbiddenError } from '@nobo/utils'
import type { AuthUser, UserRole } from '@nobo/domain'

export async function requireUser(): Promise<AuthUser> {
  const session = await auth.api.getSession({ headers: getRequestHeaders() })
  if (!session?.user) throw new UnauthenticatedError()

  const [user] = await db
    .select()
    .from(users)
    .where(eq(users.betterAuthId, session.user.id))
    .limit(1)

  if (!user || user.deletedAt !== null) throw new UnauthenticatedError()

  return AuthUserSchema.parse(user)
}

export async function requireRole(role: UserRole): Promise<AuthUser> {
  const user = await requireUser()
  if (user.role !== role && user.role !== 'admin') {
    throw new ForbiddenError(`requires role: ${role}`)
  }
  return user
}
```

`requireUser()` e `requireRole()` lanciano (`throw`) perché TanStack Start gestisce i redirect da errori non-`Result` nel layer server. Gli errori di auth non vengono mai wrappati in `Result` — causano redirect immediato gestito dal router.

---

## Ruoli

| Ruolo | Descrizione | Come si ottiene |
|---|---|---|
| `viewer` | Default per tutti gli utenti registrati | Assegnato alla creazione |
| `filmmaker` | Accesso al creator portal, upload film | Upgrade manuale tramite admin, o self-service con verifica (fuori scope MVP) |
| `admin` | Accesso completo, gestione piattaforma | Assegnato manualmente da altro admin via DB o panel admin |
| `b2b_partner` | Accesso a screening room e licensing (Fase 2) | Assegnato da admin a seguito di accordo B2B |

### Upgrade a `filmmaker`

MVP: un admin esegue `UPDATE users SET role = 'filmmaker' WHERE id = ?` direttamente o tramite il panel admin (`/admin/users`). Nessun flusso self-service nel MVP.

Fase 2: flusso self-service con form di richiesta, review admin, notifica email.

---

## Error Types

File: `packages/utils/src/errors.ts`

```typescript
export class UnauthenticatedError {
  readonly _tag = 'UnauthenticatedError' as const
  readonly message = 'Authentication required'
}

export class ForbiddenError {
  readonly _tag = 'ForbiddenError' as const
  readonly message: string
  constructor(reason?: string) {
    this.message = reason ?? 'Insufficient permissions'
  }
}

export class ValidationError {
  readonly _tag = 'ValidationError' as const
  readonly message: string
  constructor(readonly fields: Record<string, string[]>) {
    this.message = 'Validation failed'
  }
}

export class ConflictError {
  readonly _tag = 'ConflictError' as const
  readonly message: string
  constructor(readonly field: string) {
    this.message = `${field} already in use`
  }
}
```

Tutti gli errori usano `_tag` per la discriminazione con `ts-pattern`. Mai `instanceof`.

---

## Route Protette

### Strategia: `beforeLoad` su TanStack Router

Le route protette definiscono un `beforeLoad` che chiama `requireUser()` o `requireRole()`. Se la funzione lancia un errore, il router gestisce il redirect a `/login`.

```typescript
// apps/web/app/routes/_authenticated.tsx
// Layout route per tutte le route che richiedono auth
export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ location }) => {
    const user = await requireUser().catch(() => null)
    if (!user) {
      throw redirect({
        to: '/login',
        search: { redirect: location.href },
      })
    }
    return { user }
  },
})
```

```typescript
// apps/web/app/routes/_filmmaker.tsx
// Layout route per il creator portal
export const Route = createFileRoute('/_filmmaker')({
  beforeLoad: async () => {
    const user = await requireRole('filmmaker').catch(() => null)
    if (!user) {
      throw redirect({ to: '/' })
    }
    return { user }
  },
})
```

### Albero route e protezione

```
/ (pubblico)
/films (pubblico)
/films/$slug (pubblico)
/login (pubblico — redirect a /account/profile se già autenticato)
/register (pubblico — redirect a /account/profile se già autenticato)
/verify-email (pubblico)

/_authenticated (layout — richiede auth)
  /account/profile
  /watch/$filmId

/_filmmaker (layout — richiede ruolo filmmaker)
  /creator
  /creator/films
  /creator/films/new
  /creator/films/$id
  /creator/analytics
  /creator/payouts

/_admin (layout — richiede ruolo admin)
  /admin
  /admin/queue
  /admin/users
```

Il query param `redirect` su `/login` viene letto dopo il login per reindirizzare l'utente alla destinazione originale.

---

## Pagine

### `/login`

- Form: email + password
- CTA OAuth: "Accedi con Google"
- Link: "Non hai un account? Registrati"
- Link: "Password dimenticata?" (fuori scope MVP)
- In caso di errore credenziali: messaggio generico "Email o password non corretti" (non specificare quale dei due è sbagliato).
- Redirect a `/account/profile` se già autenticato (controllato in `beforeLoad`).

### `/register`

- Form: email + password + display_name
- CTA OAuth: "Registrati con Google"
- Link: "Hai già un account? Accedi"
- Dopo registrazione: redirect a `/verify-email`.
- Se email già in uso: errore inline sul campo email.

### `/verify-email`

- Stato 1 (senza token): banner "Controlla la tua email" + pulsante "Invia nuovamente".
- Stato 2 (con token da URL): chiama `verifyEmail`, poi redirect a `/account/profile` con banner di conferma.
- Stato 3 (token scaduto o invalido): messaggio di errore + pulsante "Invia nuovamente".

### `/account/profile`

- Richiede auth (`/_authenticated`).
- Mostra: display_name, email, avatar, bio, website_url.
- Form di modifica profilo (server function `updateProfile` — fuori scope di questa spec, definita in `02-profile.md`).
- Non mostra il ruolo all'utente (campo interno).

---

## Fuori Scope di Questa Spec

| Funzionalità | Spec |
|---|---|
| Modifica profilo utente (bio, avatar, website) | `02-profile.md` |
| Reset password via email | Da definire in spec separata |
| Upgrade self-service a `filmmaker` | Da definire (Fase 2) |
| B2B partner onboarding | Da definire (Fase 2) |
| Bearer token per mobile (Expo) | Da definire (Fase 3) |
| Cancellazione account (GDPR right to erasure) | Da definire in spec separata |
| 2FA / MFA | Non nel MVP |
| Magic link login | Non nel MVP |
| Rate limiting su endpoint auth | Implementazione in `apps/api`, fuori da questa spec |

---

*Spec 01 — v1.0 — Aprile 2026*
