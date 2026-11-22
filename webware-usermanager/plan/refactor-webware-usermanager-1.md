---
goal: Reorganize webware-usermanager (Http boundary namespace, MessageBus read migration, migrations/CLI ownership) - replicate the webware-acl reference pattern
version: 0.1
date_created: 2026-09-05
owner: Joey Smith
status: 'In Progress'
tags: [refactor, architecture, namespace, message-bus, testing]
---

# Refactor webware-usermanager: Implementation Plan

## Current status (2026-09-08)

- Phase 1 (safety net & coverage) - DONE (PR #17, 100% line / 100% MSI).
- Phase 2 (Http boundary reorganization) - DONE (PR #18).
- Phase 3 (MessageBus read/write migration) - DONE (branch
  `refactor/phase3-bus-migration`); see
  `plan/refactor-webware-usermanager-2-phase3-read-migration.md`.
- Phase 4 (migrations & CLI) - partial: `InitDbCommand` + `UserSchema` in PR #17;
  `webware/webware-migration` consumption deferred (package unreleased).

## Introduction

Replicate the reference pattern established in `webware-acl`
(`plan/refactor-webware-acl-1.md`, which explicitly names usermanager as the next
component). Three goals:

1. Remove ambiguity between PSR (HTTP) middleware/request-handlers and MessageBus
   middleware/handlers.
2. Enforce the read/write boundary through the MessageBus - repositories are pure
   persistence; reads go through query handlers, writes through commands.
3. Move DB migration/CLI assets out of IMS into the component that owns them.

## Current State (already done)

| Item | Status |
|---|---|
| Contracts (`UserInterface`, `AclInterface`) moved to `webware-core` | ✅ PR #6 |
| webware-tools alignment (RowPrototype for `Entity\User`, composer, mago, CI) | ✅ PR #2 |
| Doc corrections, `.gitignore`, accidental speckit-skills removal, `$commandBus` → `$messageBus` | ✅ PR #10, #11, #12 |
| CommandBus → MessageBus namespace migration | ✅ complete - zero `Webware\CommandBus\` references; `composer.lock` resolves `webware/message-bus` |
| CommandHandlers aligned to MessageBus 2.0 | ✅ verified (`CommandHandlerInterface` marker, no `#[Override]` on `handle()`, `MessageStatus`, `CommandResult`) |

## Research Findings (gaps to close)

- **Tests**: only 2 files exist today:
  - `test/unit/Command/ToggleUserActiveCommandTest.php`
  - `test/integration/CommandHandler/CreateUserHandlerTest.php`
  vs webware-acl's 244 unit + 11 integration at 100% line and 100% mutation coverage.
- **Http boundary not reorganized**: `Middleware/`, `RequestHandler/`,
  `Admin/RequestHandler/` sit at the package root (acl moved these under `Http/`).
- **Boundary violations**: RequestHandlers reading the repository directly:
  - `Admin/RequestHandler/UpdateUserModalHandler.php`
  - `RequestHandler/ResendVerificationHandler.php`
  - `RequestHandler/UserListHandler.php`
  - `RequestHandler/VerifyEmailHandler.php`
  - Middleware: `LoginMiddleware.php` (`authenticate`), `IdentityMiddleware.php` (verify).
- **Repository reads to migrate to queries** (`UserRepositoryInterface`):
  `authenticate`, `checkStatus`, `findAll`, `findByEmail`, `findById`,
  `findByVerificationToken`, `findRoleIdByName`.
  Writes stay as commands: `insert`, `save`, `update`.
- **InputFilter stateful API**: `ProcessUpdateUserMiddleware.php` and
  `RegistrationMiddleware.php` use `setData()`/`isValid()`/`getValues()`/
  `getMessages()`/`setValidationGroup()` → migrate to laminas-inputfilter 3.0
  stateless `validate()` + `Webware\Core\InputFilter\SystemMessageTrait`
  (core trait already updated per the 2026-09-03 webware-core handoff).
- **SystemMessage usage**: 6 files: `InputFilter/UserDataFilter.php`,
  `Middleware/LoginMiddleware.php`, `Middleware/ProcessUpdateUserMiddleware.php`,
  `Middleware/RegistrationMiddleware.php`, `RequestHandler/LoginHandler.php`,
  `RequestHandler/VerifyEmailHandler.php`. Verify alignment with the centralized
  SystemMessage pattern adopted in webware-acl (issue #22).
- **Composer drift vs webware-acl**: `webware/webware-message ^1.0.0-beta.1`
  (acl is `^1.0.0-beta.2`); no direct `php-db/phpdb` requirement (acl has
  `php-db/phpdb ^0.6.x-dev`); no `webware/webware-console` in require-dev (acl added it).

## Requirements & Constraints

- **CON-001**: Zero behavior changes beyond what a dependency change requires.
- **CON-002**: Move classes in small sets; update tests immediately and run the
  suite before the next move.
- **CON-003**: Mitigate Mago analysis/lint issues before starting (widen the safety net).
- **CON-004**: RequestHandlers only build/return responses; all reads and writes
  happen in middleware (via MessageBus). Enforce with Mago Guard.
- **CON-005**: PHPUnit 13 strict mode; `#[CoversClass]` metadata required; target
  **100% line AND 100% mutation coverage (MSI/MCC)**.
- **CON-006**: Mago stays green: `format`, `lint`, `analyze`, `guard`.
- **CON-007**: Validation runs in the containerized dev env (`tooling`, `mysql`).
- **CON-008**: No `docs/**` changes without explicit approval (user reviews docs closely).
- **CON-010**: Do not remove IMS originals - IMS remains the blueprint for DB state.

## Implementation Phases

### Phase 1: Safety net & coverage (dominant effort)

- **TASK-001** Baseline validation: mago format/lint/analyze/guard + unit + integration + mutation.
- **TASK-002** Inventory `lint-baseline.toml` / `analysis-baseline.toml`; fix what is safe, regenerate the rest.
- **TASK-003** Characterization/safety-net tests for: `Entity\User`, `InputFilter\UserDataFilter`,
  all middleware, request handlers, command handlers, `UserRepository`.
- **TASK-004** Mago Guard perimeter rules: PSR `RequestHandlerInterface` / `MiddlewareInterface`
  only usable from `Http\…`; `Repository\*` denied from `Http\RequestHandler\…`.

### Phase 2: Http boundary reorganization

Namespace moves (class names unchanged; no BC aliases; track every move):

- `Middleware/*` → `Http/Middleware/*`
- `RequestHandler/*` → `Http/RequestHandler/*`
- `Admin/RequestHandler/*` → `Http/Admin/RequestHandler/*`

Stays put: `Admin/Dashboard`, `Auth`, `Command`, `CommandHandler`, `Repository`,
`Event`, `Listener`, `InputFilter`, `View`, `ConfigProvider`, `RouteProvider`.

Update DI keys in `ConfigProvider`, `RouteProvider`, factories, and test imports per move set.

### Phase 3: MessageBus read migration (repository gateway)

- Add `Query/` + `QueryHandler/` namespaces; wire `query_map` under
  `MessageBusInterface::class` in `ConfigProvider`.
- Candidate queries: `FetchUserById`, `FetchUserByEmail`, `FetchUserByVerificationToken`,
  `FetchUsers` (and decide grouping for `authenticate` / `checkStatus` / `findRoleIdByName`).
- Handlers return the concrete `Query\QueryResult`; repositories stay bus-agnostic;
  payload is component-owned (arrays/read-models) - no php-db result sets leak.
- Writes already flow through `CreateUserCommand` / `ToggleUserActiveCommand` / `UpdateUserCommand`.

### Phase 4: Migrations & CLI

- Port the usermanager DB migration(s) (user table) from IMS `src/ims-migration/`.
- Consume `webware/webware-migration` + `webware/webware-console`; add console to
  require-dev; add an init/seed workflow analogous to `acl:init-db`.

## Cross-cutting

- **Repository Schema + TableGateway IoC** (in progress) - adopt `Webware\Core\SchemaFactory`
  for `Repository\Schema` (drop the `table()` method) and move `TableGateway` construction
  out of the repository constructor into the factory (mirror webware-acl's
  `RoleRepositoryFactory` / `RuleRepositoryFactory`). Consumers: `UserRepository`,
  `UserRepositoryFactory`.
- Bump `webware/webware-message` → `^1.0.0-beta.2` (done via `composer update`).
- Add direct `php-db/phpdb: ^0.6.x-dev` (RowPrototype contract).
- InputFilter `validate()` migration (core `SystemMessageTrait` already updated).
- Centralized SystemMessage notification alignment (acl issue #22 pattern).
- Mago Guard perimeter rules (Phase 1, TASK-004).

## Sequencing & Dependencies

1. **Phase 1 first**: it is the safety net and the largest gap (2 test files today).
2. **Phase 2 before Phase 3**: query handlers + middleware need the settled namespaces.
3. **Phase 3** depends on the `webware/message-bus` query bus (already required) and the
   ecosystem query convention (see webware-ecosystem memory).
4. **Phase 4** depends on `webware/webware-migration` + `webware/webware-console`
   (separate repos) - can run last, in parallel with acl's Phase 4.

## Risks & Open Questions

- **RISK-001**: The test-suite gap dominates Phase 1 - expect the bulk of effort here.
- **RISK-002**: `Webware\MessageBus\Command\CommandResultInterface` is `@internal`
  (same as the query side). Handlers currently declare it (usermanager and acl alike);
  the locked query convention returns the concrete `QueryResult`. Decide whether command
  handlers should likewise return the concrete `CommandResult`.
- **RISK-003**: Handler param typing differs from acl - usermanager uses
  `CommandInterface` + `assert()`, acl uses the concrete command type. Optional alignment.
- **RISK-004**: `docs/plan-admin-edit-user.md` has design drift (`SaveUserCommand` /
  `ProcessUserMiddleware` / `EditUserModalHandler` vs the actual `UpdateUserCommand` /
  `ProcessUpdateUserMiddleware` / `UpdateUserModalHandler`) - needs a separate docs pass.

## Reference Files

- webware-acl `plan/refactor-webware-acl-1.md` + `plan/SESSION-HANDOFF-*.md`
- webware-tools `NOTES-spec-kit-alignment.md` + `presets/webware-alignment/` (templates & artifacts)
- IMS `docs/module/component-migration-plan.md` (authoritative MessageBus migration delta)
- User memory: `webware-ecosystem`, `phpdb-memory`, `webware-dev-env`, `mago-memory`
