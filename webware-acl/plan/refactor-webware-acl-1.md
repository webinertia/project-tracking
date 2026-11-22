---
goal: Reorganize webware-acl (Http boundary namespace, MessageBus read migration, migrations/CLI ownership)
version: 0.2
date_created: 2026-08-23
last_updated: 2026-08-28
owner: Joey Smith
status: 'In Progress'
tags: [refactor, architecture, namespace, message-bus, testing]
---

# Introduction

![Status: In Progress](https://img.shields.io/badge/status-In%20Progress-blue)

Reorganize the webware-acl component to remove ambiguity between PSR (HTTP) middleware/request-handlers and MessageBus middleware/handlers, enforce the read/write boundary through the MessageBus, and move DB migration/CLI assets out of IMS into the component that owns them. This establishes the reference pattern that will be replicated (as speckit tasks) across the remaining webware components, including UserManager.

## 1. Requirements & Constraints

- **CON-001**: ZERO behavior changes outside what is required by a dependency change. None.
- **CON-002**: When a set of classes is moved, their tests are updated immediately and the test suite is run to verify before the next move.
- **CON-003**: Mitigate as many Mago analysis/lint issues as possible BEFORE starting, to widen the safety net (baselines: `lint-baseline.toml`, `analysis-baseline.toml`).
- **CON-004**: RequestHandlers are only responsible for building and returning a response. All reads and writes occur in Middleware (via MessageBus). Enforce with Mago Guard where possible.
- **CON-005**: PHPUnit 13, strict mode; `#[CoversClass]` metadata required by `phpunit.xml.dist`; **target: 100% code coverage AND 100% mutation coverage (MSI/MCC)**: user directive 2026-08-24, supersedes earlier 95+ gate.
- **CON-006**: Mago must stay clean: `mago format`, `mago lint`, `mago analyze`, `mago guard` green.
- **CON-007**: Validation runs in the containerized dev environment (`docker compose` services: `tooling`, `mysql` for integration tests) for cross-platform compatibility.
- **CON-008**: No changes to `docs/**` without explicit user approval - user is reviewing docs closely.
- **CON-009**: ~~Pre-existing skipped tests (5 unit, 7 integration) are intentional; remain skipped unless a move directly addresses them.~~ **Superseded 2026-08-24:** 100% coverage target means the 12 skipped tests must be investigated, un-skipped (fixing fixtures/schema where needed) or replaced with working equivalents.
- **CON-010**: Do not remove IMS originals in Step 3 - IMS remains the blueprint for the DB state each component requires.

## 2. Implementation Steps

### Implementation Phase 1: Safety net: tests + Mago mitigation (CON-003, CON-002)

- GOAL-001: Characterize existing behavior with integration/unit tests and clear as many Mago issues as possible before any file moves.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-001 | Run full validation suite (mago format/lint/analyze/guard, unit, integration, mutation) to establish baseline state | ✅ | 2026-08-24 |
| TASK-002 | Inventory current Mago baseline entries (`lint-baseline.toml`, `analysis-baseline.toml`); fix what can be fixed safely, regenerate baselines for the rest | ✅ | 2026-08-24 |
| TASK-003 | Write characterization/safety-net tests for every behavior the moves must preserve (see Section 6) | ✅ | 2026-08-25 |
| TASK-004 | Encode Guard rules for the new layout using `[[guard.perimeter.rules]]` + `[[guard.perimeter.restrictions]]`: (a) PSR `RequestHandlerInterface` usable only from `Http\RequestHandlers\` (+ `Http\Admin\RequestHandlers\`); (b) PSR `MiddlewareInterface` usable only from `Http\Middleware\` (+ `Http\Admin\Middleware\`); (c) vendor `MessageBus\QueryHandlerInterface` \| `MessageBus\CommandHandlerInterface` usable only from the MessageBus-side handler namespace (TBD, Step 2); (d) `Repository\*` denied from Http RequestHandler namespaces (reads via bus only). Verify empirically that `implements` counts as dependency use (Guard docs only show `use`-statement examples) | ✅ | 2026-08-28 |

### Implementation Phase 2: Step 1: Http boundary reorganization

- GOAL-002: Move all HTTP-boundary classes into the `Http` namespace tree:
  - `Http\RequestHandlers` (non-admin)
  - `Http\Middleware` (non-admin)
  - `Http\Admin\RequestHandlers`
  - `Http\Admin\Middleware`
  - Ambiguity between PSR middleware/handlers and MessageBus middleware/handlers must be eliminated.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-005 | Namespace map (decided): `Admin\Middleware\*` → `Http\Admin\Middleware\*`; `Admin\RequestHandler\*` → `Http\Admin\RequestHandlers\*`; root `Middleware\*` (AclMiddleware, AuthorizationMiddleware) → `Http\Middleware\*`; root `RequestHandler\ForbiddenHandler` → `Http\RequestHandlers\*`. Classification verified by implemented interface (current dirs already align). Class NAMES unchanged - namespaces only. `Http\RouteResource*` stays under `Http\` alongside the boundary. BC policy: no aliases, track every move in IMS issue #30 | ✅ | 2026-08-27 |
| TASK-006 | Move classes in small sets; update tests per set and run suite (CON-002) | ✅ | 2026-08-27 |
| TASK-007 | **Resolved:** `Admin\Dashboard` (Widget, RegisterWidgetListener) stays in place - admin UI integration, not PSR boundary | ✅ | 2026-08-23 |

### Implementation Phase 3: Step 2: MessageBus changes

- GOAL-003: Verify the existing messagebus change set; migrate all reads to the MessageBus (queries) so Repositories are completely decoupled from the Middleware boundary. All read/write remains in Middleware; RequestHandlers only build/return responses (CON-004).

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-007 | MessageBus verification (Step 2): UNWIRE `AclMessageHandlerMiddleware` from `getBusConfig()` pipeline + remove its factory wiring from `getDependencies()`; keep all acl-local classes in place; keep command map as-is | ✅ | 2026-08-28 |
| TASK-008 | Migrate reads to query handlers using the vendor query bus: `query_map` sub-key in `getBusConfig()`, `Query\QueryInterface` queries + `QueryHandlerInterface` handlers (optionally `Strategy\ClassnameStrategy`); targets: `Acl::load()`, `OverviewMiddleware`, `AddRoleModalHandler`, `EditRoleModalHandler`, `RoleListHandler` | ✅ | 2026-08-28 |
| TASK-009 | Migrate reads to query handlers: `Acl::load()` (decision: everything through the bus - resolve DI cycle strategy first, RISK-006), `OverviewMiddleware`, `AddRoleModalHandler`, `EditRoleModalHandler`, `RoleListHandler` | ✅ | 2026-08-28 |

**Phase 3 - read-migration decisions (locked 2026-08-27; full convention in webware-ecosystem-memory):**

- Handlers implement `QueryHandlerInterface`, method `handle()` with NO `#[Override]`, and return the concrete `Query\QueryResult` (`new QueryResult($query, MessageStatus::Success, $payload)`). Repositories stay bus-agnostic - never implement `QueryResultInterface` (`@internal`). Payload is component-owned (arrays/read-models), not php-db result sets.
- 4 granular queries: `FetchAllRulesQuery`, `FetchDistinctResourceIdsQuery`, `FetchAclRoleRegistryQuery`, `FetchAllRolesQuery` - namespace `Webware\Acl\Query\*` / `Webware\Acl\QueryHandler\*` (final location TBD).
- Scope: ALL reads AND the `Acl::addRole()` write go through the bus. `Acl::addRole()` persist dispatches `Admin\Command\SaveRoleCommand` - NOT a direct `RoleRepository::save()`. `Acl` keeps NO repository dependency.
- DI: `Acl` + `OverviewMiddleware` + the 3 role handlers inject `MessageBusInterface` only; `Repository\*` imports removed from them (RISK-006 resolved - no cycle).
- Drift found (audit, fix in this pass): `SaveRuleHandler` injects `RoleRepository` but never uses it (dead dep); `SaveRoleHandler` ignores `$command->id` (dead field). `UpdateRuleTypeHandler` reads repos directly in its cascade (`fetchDirectChildren`, `findByRoleAndResource`) - flag: strict "reads via query handlers" would move these too.

### Implementation Phase 4: Step 3: Migrations & CLI scripts

- GOAL-004: Move the required CLI scripts and migration classes from IMS into the components that should own them (IMS originals stay). BLOCKED until WSL code is pushed to a branch.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-010 | Migrate acl migration classes into webware-acl: port `Migration016AclRole`, `Migration017AclRule` (from `src/ims-migration/` on the IMS branch) per the migrations-layer design in `.github/migrations-layer-plan.md` (lightweight PHP system: `MigrationInterface` with getVersion/getDescription/up/down, `schema_migrations` tracking table, `bin/migrate` runner, `Migration{NNN}{PascalDescription}` naming). IMS originals stay. CLI seed scripts + `data/schema/*.sql` ownership: scope TBD | ⬜ |  |

**Phase 4 - architecture decisions (locked 2026-08-28):**

- Migration tooling is extracted from IMS into two shared components - `webware/webware-migration` (all migration logic, contracts, and CLI commands) and `webware/webware-console` (the TUI surfacing Webware + Mezzio CLI commands). Each is spec-kit planned in its own repo; their constitutions/specs are the source of truth for their internals.
- acl consumes them as its tooling: it ships `Migration016AclRole` + `Migration017AclRule` and seeds the base roles (`Guest`, `Member`, `Administrator`) into the database via its migration/seed workflow. IMS builds its roles/rules on those; IMS originals stay (CON-010).

## 3. Alternatives

- **ALT-001**: TBD

## 4. Dependencies

- **DEP-001**: `webware/message-bus: ^2.0.0-beta.1` - already required; ships `Query/QueryInterface`, `Query/QueryResult`, `QueryHandlerInterface`, `Command/CommandResult`, `MessageStatus`, `Middleware/MessageHandlerMiddleware` (needed for Step 2).
- **DEP-002**: `webware/webware-admin: 0.1.x-dev` (locked `468e8b4`) - dashboard widget/event integration.
- **DEP-003**: `webware/webware-tools` - provides shared `mago.toml` (guard structural rules) and container validation.
- **DEP-004**: IMS WSL working code pushed - branch `override-user-manager-update-user-via-ims-store` @ `6f119a0` (2026-08-24). **Working branch, mid-migration: may be broken; treat as reference only.** IMS originals must stay (CON-010). Contains the Step 3 blueprint: `src/ims-migration/` (Migration001-017 incl. `Migration016AclRole`, `Migration017AclRule`), `data/schema/*.sql`, `.github/migrations-layer-plan.md` (migrations layer design), `docs/module/component-migration-plan.md` (deep audit - authoritative messagebus migration delta + query bus adoption notes).
- **DEP-005**: `roave/backward-compatibility-check` (require-dev) - will flag moved/removed classes; policy needed (open question).

## 5. Files

- **FILE-001**: `src/Admin/Middleware/*` → target TBD (Phase 2)
- **FILE-002**: `src/Admin/RequestHandler/*` → target TBD (Phase 2)
- **FILE-003**: `src/Middleware/*` (AclMiddleware, AuthorizationMiddleware) → `Http/Middleware` (Phase 2)
- **FILE-004**: `src/RequestHandler/*` (ForbiddenHandler) → `Http/RequestHandlers` (Phase 2)
- **FILE-005**: `src/Http/*` (RouteResource) - stays under `Http\` alongside the boundary (decided)
- **FILE-006**: `src/Admin/Command*`, `src/Admin/CommandHandler/*` - move to MessageBus-side namespace in Phase 3 (exact name TBD); `src/Admin/Dashboard/*` stays (decided)
- **FILE-007**: `src/MessageBus/*` - acl-local classes vs vendor package (Phase 3)
- **FILE-008**: `src/ConfigProvider.php`, `src/RouteProvider.php` - wiring/keys updated as classes move

## 6. Testing

- **TEST-001**: Update `test/unit/**` and `test/integration/**` imports immediately after each move set (CON-002).
- **TEST-002**: Integration coverage of admin flows (ProcessRoleMiddlewareTest, ProcessRuleMiddlewareTest, MessageHandlerMiddlewareIntegrationTest) is the primary behavior lock - extend before moving.
- **TEST-003**: New unit tests for moved classes (CommandHandlers, Middleware, RequestHandlers) where gaps exist.
- **TEST-004**: Post-move validation: `composer test`, `composer test-integration`, `composer mutation-test` (MSI 95+), mago suite.

## 7. Risks & Assumptions

- **RISK-001**: **Resolved:** `RouteResource*` classes remain under `Webware\Acl\Http\` alongside the new boundary subnamespaces (`Http\Middleware`, `Http\RequestHandlers`, `Http\Admin\...`).
- **RISK-002**: **Decision:** namespace moves change DI keys (FQCNs) in ConfigProvider/RouteProvider and consumer configs (IMS pipeline, bus pipeline) - no aliases shipped; every moved class tracked in IMS known-changes issue #30.
- **RISK-003**: Existing boundary violations: `AddRoleModalHandler`, `EditRoleModalHandler`, `RoleListHandler` (RequestHandlers) read `RoleRepository` directly; `OverviewMiddleware` reads `RuleRepository` directly. These are the exception the user predicted; they must be migrated in Phase 3, not left.
- **RISK-004**: **Decision:** keep the acl-local `MessageBus` classes in place, but UNWIRE from `getBusConfig()` in Step 2 (`AuthorizableCommandInterface`, `CommandResult`, `CommandStatus`, `Middleware\MessageHandlerMiddleware`, `Container\MessageHandlerMiddlewareFactory`, dead `Container\CommandHandlerMiddlewareFactory` - the latter references nonexistent `Webware\Acl\CommandBus\Middleware\CommandHandlerMiddleware`; keep per decision). No command implements `AuthorizableCommandInterface`, so the middleware is currently a no-op pass-through.
- **RISK-005**: **Resolved:** Mago Guard perimeter restrictions (`[[guard.perimeter.restrictions]]` - `dependency` → `allow-from`/`deny-from`) provide dependency-boundary enforcement (docs: https://mago.carthage.software/1.47.3/en/tools/guard/usage/#restricting-where-a-dependency-may-be-used). Structural guard has no "must implement X" rule, but restrictions achieve the same effect. Caveat: docs only demonstrate `use` statements - Phase 1 must verify empirically that `implements` clauses count as dependency use.
- **RISK-006**: **Decision made:** ALL reads go through the bus, including `Acl::load()`/`Acl`. **Resolved:** with the ACL-enforcing middleware unwired from the bus pipeline (RISK-004), `Acl` dispatching load queries creates NO DI cycle. Revisit only if the middleware is ever re-wired.
- **RISK-007**: Known-possibly-fixed bugs (`ProcessRuleMiddleware` processPatch/processDelete CommandResult violations; RuleFilter readonly assignments) live in classes being moved. **Decision:** move classes intact (CON-001); Phase 1 characterization tests verify whether the bugs still exist (tracking may be stale); any confirmed bugs are fixed in a separate, clearly-marked post-refactor effort.
- **ASSUMPTION-001**: "No behavior changes" = no functional/behavioral changes; namespace moves + config key updates are the intended change surface.
- **ASSUMPTION-002**: Phase 4 (Step 3) does not block Phases 1-3.

## 8. Related Specifications / Further Reading

- [ACL Work Todo - docs/planning/todo-2026-05-31.md](../docs/planning/todo-2026-05-31.md)
- [Deprecations - docs/deprecations.md](../docs/deprecations.md)
- [Architecture Blueprint - docs/architecture/blueprint.md](../docs/architecture/blueprint.md)
- [IMS known-changes tracking issue - tyrsson/inventory-management-system #30](https://github.com/tyrsson/inventory-management-system/issues/30)
- [IMS working branch (WSL) - override-user-manager-update-user-via-ims-store](https://github.com/tyrsson/inventory-management-system/tree/override-user-manager-update-user-via-ims-store)
- [IMS migrations layer plan - .github/migrations-layer-plan.md](https://github.com/tyrsson/inventory-management-system/blob/override-user-manager-update-user-via-ims-store/.github/migrations-layer-plan.md)
- [IMS component migration deep audit - docs/module/component-migration-plan.md](https://github.com/tyrsson/inventory-management-system/blob/override-user-manager-update-user-via-ims-store/docs/module/component-migration-plan.md)
- [webware/message-bus](https://github.com/webinertia/message-bus)
