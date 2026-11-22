---
goal: Migrate all repository reads and writes behind the MessageBus (Phase 3)
version: 0.2
date_created: 2026-09-07
date_updated: 2026-09-08
owner: Joey Smith
status: 'Completed'
tags: [refactor, architecture, message-bus, query, command, testing]
---

# Phase 3: MessageBus read/write migration (proposal)

## Status

| Phase | State |
|---|---|
| Phase 1 - Safety net & coverage (100% line / 100% MSI) | ✅ DONE (PR #17) |
| Phase 2 - Http boundary reorganization | ✅ DONE (PR #18) |
| **Phase 3 - MessageBus read/write migration (this doc)** | **✅ DONE (branch `refactor/phase3-bus-migration`)** |
| Phase 4 - Migrations & CLI | partial (InitDbCommand in PR #17; webware-migration deferred) |

## Resolved conventions (locked 2026-09-07)

1. **Not-found = `Failure` status** (null payload). `Success` only when the query
   produced a result. Actual errors throw (`QueryResult` has no error channel).
   A lookup that legitimately returns nothing (e.g. unknown email) is `Failure`,
   never `Success` + null.
2. Handlers return the concrete `Webware\MessageBus\Query\QueryResult`; the
   payload travels via `getResult()` (e.g. `getResult()` returns an
   `AuthenticationResult` for the authenticate query).
3. Repository stays bus-agnostic and returns RowPrototype (`User`) / typed rows.
   No php-db `ResultSet`/`RowPrototype` types leak past the query handler.
4. **ALL reads and writes go through the MessageBus - no exceptions**, including
   `Admin\Dashboard` (widget aggregation moves to a query too).
5. **Mutations happen in the PSR middleware layer.** RequestHandlers are
   render-only. Rationale: middleware *acts* on incoming data, and behavior placed
   in middleware is composable anywhere in the PSR middleware pipeline; behavior
   placed in a RequestHandler is only composable at the terminal end, which
   provides zero flexibility. (The long-term target is a webware-tools guard rule
   that forbids RequestHandlers from depending on `MessageBusInterface`.)
6. Payloads keep the current `User` entity (RowPrototype) shape - no read-model
   DTOs in this leg.

## Current inventory

### Reads called directly from HTTP classes (to migrate)

| Consumer | Repository call | Replacement |
|---|---|---|
| `Http\Middleware\LoginMiddleware` | `authenticate()` | `AuthenticateUserQuery` |
| `Http\Middleware\IdentityMiddleware` | `checkStatus()` | `CheckUserActiveQuery` |
| `Http\RequestHandler\UserListHandler` | `findAll()` | `FetchUsersQuery` |
| `Http\RequestHandler\ResendVerificationHandler` | `findByEmail()` | `FetchUserByEmailQuery` |
| `Http\RequestHandler\VerifyEmailHandler` | `findByVerificationToken()` | `FetchUserByVerificationTokenQuery` |
| `Http\Admin\RequestHandler\UpdateUserModalHandler` | `findById()` | `FetchUserByIdQuery` |
| `Admin\Dashboard\RegisterWidgetListener` | `findAll()` | `FetchUsersQuery` |

### Writes called directly from HTTP classes (must move to middleware)

| Consumer | Repository call | Replacement |
|---|---|---|
| `Http\RequestHandler\ResendVerificationHandler` | `update()` (regen token) | `RegenerateVerificationToken` command, dispatched from new middleware |
| `Http\RequestHandler\VerifyEmailHandler` | `update()` (activate + clear token) | `ActivateUser` command, dispatched from new middleware |

### Legitimate repository consumers (unchanged)

- `CommandHandler\{CreateUser,ToggleUserActive,UpdateUser}Handler` - `save`/
  `findById`/`update` inside the write-command flow.

### Dead code

- `findRoleIdByName()` - zero callers (`return $roleName;`) → remove.

## Queries + handlers

Each query is a `final readonly class … implements QueryInterface`; each handler
is `final readonly` with a `handle()` returning `QueryResult`; each has a
`Container\*Factory`; all wired under `query_map`.

| Query | Payload via `getResult()` | Not-found |
|---|---|---|
| `Query\FetchUserByIdQuery` (`int $id`) | `User` | `Failure` (null) |
| `Query\FetchUserByEmailQuery` (`string $email`) | `User` | `Failure` (null) |
| `Query\FetchUserByVerificationTokenQuery` (`string $token`) | `User` | `Failure` (null) |
| `Query\FetchUsersQuery` | `list<User>` (adapted from ResultSet) | `Success` + `[]` |
| `Query\AuthenticateUserQuery` (`string $credential`, `#[SensitiveParameter] ?string $password`) | `AuthenticationResult` | `Failure` (null) |
| `Query\CheckUserActiveQuery` (`int $id`) | `bool` | n/a |

## Commands (for the residual writes)

| Command | Effect |
|---|---|
| `Command\ActivateUser` (`int $id`) | `update(['active' => 1, 'verificationToken' => null, 'tokenCreatedAt' => null])` |
| `Command\RegenerateVerificationToken` (`int $id`, `#[SensitiveParameter] string $token`, `string $tokenCreatedAt`) | `update(['verificationToken' => …, 'tokenCreatedAt' => …])` |

Each with a `CommandHandler` + `Container\*Factory`, wired under `command_map`.

## Consumer changes

1. **`LoginMiddleware`**: inject `MessageBusInterface`; dispatch
   `AuthenticateUserQuery(credential: $email, password: $password)`; read the
   `AuthenticationResult` from `getResult()`. Behavior unchanged.
2. **`IdentityMiddleware`**: inject `MessageBusInterface`; dispatch
   `CheckUserActiveQuery(id: $userInfo['id'])`; read `bool` from `getResult()`.
3. **`UserListHandler`**: inject `MessageBusInterface`; dispatch `FetchUsersQuery`;
   render the returned `list<User>`. (Follow-up: render-only via middleware once
   the webware-tools guard rule lands.)
4. **`UpdateUserModalHandler`**: inject `MessageBusInterface`; dispatch
   `FetchUserByIdQuery`; `Failure` → 404, `Success` → render modal.
5. **`VerifyEmailHandler`**: becomes render-only. New
   `ProcessVerifyEmailMiddleware` (in `Http\Middleware\`) does: read token →
   `FetchUserByVerificationTokenQuery` → on `Failure` set error, on `Success` check
   expiry → dispatch `ActivateUser` → set messenger → attach `CommandResult`.
   Handler renders redirect/error.
6. **`ResendVerificationHandler`**: becomes render-only. New
   `ProcessResendVerificationMiddleware` does: read email → `FetchUserByEmailQuery` →
   on `Failure` render "sent" (silent), on `Success` if active redirect to login,
   else dispatch `RegenerateVerificationToken` → send email → render "sent".
7. **`RegisterWidgetListener`**: inject `MessageBusInterface`; dispatch
   `FetchUsersQuery`; iterate `list<User>` to count active/inactive. (No repository
   exception.)
8. **`ConfigProvider`**: add `query_map` + `command_map` entries and all new
   factory DI entries.
9. **Remove `findRoleIdByName`** from `UserRepositoryInterface` + impl.

## Design decisions (resolved)

- **D1 - `authenticate`** → `AuthenticateUserQuery`. Handler delegates to
  `$repo->authenticate()`; returns `QueryResult` whose `getResult()` is the
  `AuthenticationResult`.
- **D2 - `checkStatus`** → dedicated `CheckUserActiveQuery` (keeps the
  lightweight `SELECT active`).
- **D3 - residual writes** → move to **middleware** (policy: mutations happen in
  the middleware layer). Reason: middleware *acts* on incoming data and is
  composable at any point in the PSR middleware pipeline; a RequestHandler is
  terminal and composable only at the end - zero flexibility. Handlers become
  render-only. Future webware-tools guard rule (separate repo) will forbid
  handlers from depending on `MessageBusInterface`.
- **D4 - `RegisterWidgetListener`** → **moves to the bus** (`FetchUsersQuery`). No
  repository one-off; it's a convention violation.
- **D5 - payload type** → maintain current `User` (RowPrototype) entity. A
  follow-up audits all repository return types for non-RowPrototype values.
- **D6 - `webware/webware-message`** → bump `^1.0.0-beta.1` → `^1.0.0-beta.2`.

## Repository perimeter restriction (locks the boundary after migration)

```toml
[[guard.perimeter.restrictions]]
dependency = "Webware\\UserManager\\Repository\\**"
allow-from = [
    "Webware\\*",                                 # root ConfigProvider wiring
    "Webware\\UserManager\\Repository\\**",       # self
    "Webware\\UserManager\\QueryHandler\\**",     # reads
    "Webware\\UserManager\\CommandHandler\\**",   # writes
    "Webware\\UserManager\\Console\\**",         # InitDbCommand uses Schema (table name)
    "WebwareTest\\UserManager\\**",               # unit tests
    "WebwareTestIntegration\\UserManager\\**",    # integration tests
]
```

Note: `Admin\Dashboard\**` is deliberately absent - it moves to the bus (D4).
Add this only after all consumers are migrated, so `mago guard` stays green.

## Follow-ups (separate repos / legs)

- webware-tools guard rule: RequestHandlers must not depend on
  `MessageBusInterface` (render-only enforcement).
- Repository return-type audit: ensure every repository method returns
  RowPrototype (`User`) or typed rows, never arrays/DTOs.

## Sequencing (small sets, verify each: CON-002)

1. Add `Query\` + `QueryHandler\` + factories + `query_map` wiring (no consumer
   changes; tests green).
2. Migrate reads one consumer at a time, updating tests: `UpdateUserModalHandler`
   → `UserListHandler` → `RegisterWidgetListener` → `IdentityMiddleware` →
   `LoginMiddleware`.
3. Add `ActivateUser` + `RegenerateVerificationToken` commands + handlers;
   introduce `ProcessVerifyEmailMiddleware` + `ProcessResendVerificationMiddleware`
   and slim the two handlers to render-only (update routes + tests).
4. Remove `findRoleIdByName`; bump `webware-message`.
5. Add the repository perimeter restriction; confirm `mago guard` green.

## Verification

- `mago format --check`, `mago lint`, `mago analyze`, `mago guard` - clean.
- `composer test` - 199 tests (adjusted, same behavior) green.
- `composer test-integration` - 6 tests green.
- `composer mutation-test` - maintain 100% covered MSI.

## Infection ignores added in this phase (scoped, reviewable)

Three mutator-level ignores were added to `infection.json5.dist`. All are
config-fallback values kept as **private** constructor state on the built
middleware, so they cannot be observed without reflection and therefore cannot
be killed by behaviour tests. Every other mutant in this phase is killed by
assertions (exact email body/recipient/subject, `age == ttl` boundary,
`$updated == 0`, missing-`id` fallback, etc.).

| Mutator | Target (`class::method`) | Ignored site(s) |
|---|---|---|
| `Coalesce` | `Webware\UserManager\Http\Middleware\Container\ProcessResendVerificationMiddlewareFactory::__invoke` | `src/Http/Middleware/Container/ProcessResendVerificationMiddlewareFactory.php` lines 28, 31, 37, 38, 39, 41 |
| `IncrementInteger` | `Webware\UserManager\Http\Middleware\Container\ProcessVerifyEmailMiddlewareFactory::__invoke` | `src/Http/Middleware/Container/ProcessVerifyEmailMiddlewareFactory.php` line 23 (`86_400` → `86_401`) |
| `DecrementInteger` | `Webware\UserManager\Http\Middleware\Container\ProcessVerifyEmailMiddlewareFactory::__invoke` | `src/Http/Middleware/Container/ProcessVerifyEmailMiddlewareFactory.php` line 23 (`86_400` → `86_399`) |

Details:

- **`Coalesce` (6 sites)**: the resend-mail defaults: `$config['user'] ?? []`,
  `$config[MailerInterface::class] ?? []`, and the four mail-config fallbacks
  (`from_email ?? 'noreply@farmers-ims.local'`, `from_name ?? 'Farmers IMS'`,
  `base_url ?? 'http://localhost:8080'`,
  `verification_email_subject ?? 'Verify your account'`). These land in the
  middleware's private `$mailConfig` and are only visible through reflection.
- **`IncrementInteger` / `DecrementInteger` (1 site)**: the `86_400` (24-hour)
  token-TTL default. Mutating it to `86_401`/`86_399` is an unobservable
  1-second difference in a default.

Pre-existing ignores (not added in this phase) remain for
`Webware\UserManager\Console\InitDbCommand::collectUser`
(`TrueValue` line 158, `FalseValue` line 159, `CastString` line 162).
