---
goal: Decouple webware-admin from webware-acl via laminas-permissions-acl type hints
version: 1.0
date_created: 2026-08-23
last_updated: 2026-08-23
owner: Joey Smith
status: 'In progress'
tags: [refactor, architecture, decoupling, acl, admin]
---

# Introduction

![Status: In progress](https://img.shields.io/badge/status-In%20progress-yellow)

Eliminates the circular dependency between `webware-admin` and `webware-acl`.
Today admin hard-couples to `Webware\Acl\AclInterface` (4 source files + 1 test)
while acl ships an admin UI and a dashboard widget listener. After this
refactor, admin type-hints only `Laminas\Permissions\Acl\AclInterface` (already
a direct dependency) as a decoupling seam; in practice the implementation is
always webware-acl's `Acl`. Dependency direction becomes one-way: acl
**requires** admin (its management UI), admin only **suggests** acl.

Full design context: <https://github.com/webinertia/webware-admin/issues/12>

## 1. Requirements & Constraints

- **REQ-001**: No `Webware\Acl\*` reference remains anywhere in webware-admin (src, test, composer metadata).
- **REQ-002**: `webware/webware-acl` moves from `require` to `suggest` in admin's composer.json; its VCS repository entry is removed.
- **REQ-003**: Dashboard widget filtering MUST stay in admin. The dashboard can never render unfiltered - the pipeline fails closed: no authenticated role-aware user ⇒ zero widgets; no `AclInterface` service in the container ⇒ container resolution fails ⇒ dashboard errors.
- **REQ-004**: Widgets MUST be filtered in every environment, including tests. Integration tests exercise the real filter path using a real `Laminas\Permissions\Acl\Acl` instance, with no database requirement.
- **REQ-005**: Only `WidgetInterface` instances may reach `AclWidgetFilterIterator` - enforced structurally: `WidgetContainer` accepts widgets solely through its typed constructor and `addWidget()`, and there is no other mutation path.
- **REQ-006**: Widget system classes live in the component root namespace (`Webware\Admin\WidgetInterface`, `Webware\Admin\WidgetContainer`, `Webware\Admin\AclWidgetFilterIterator`) - no `Widget\` sub-namespace duplicating the component name.
- **CON-001**: `laminas/laminas-permissions-acl: ^2.16` is already a direct admin dependency - the runtime decoupling seam adds no new packages.
- **CON-002**: `Webware\UserManager\UserInterface` is dropped from admin; the user is read from the standard `Mezzio\Authentication\UserInterface::class` request attribute (`mezzio/mezzio-authentication: ^1.13` is already a direct dependency). For PHPUnit testing without a database, `mezzio/mezzio-authentication-session` is added as a `require-dev` package (session-backed authentication adapter).
- **CON-003**: PHP `~8.4.1 || ~8.5.0`; PHPUnit 13 strict mode (coverage metadata, mock/stub separation) per the webware constitution.
- **CON-004**: acl→admin is already event-decoupled (`RegisterWidgetEvent` + listener); this refactor only removes the admin→acl hard references.
- **CON-005**: Line endings LF only; Mago format/lint/analyze/guard green; coverage and infection thresholds unchanged.

## 2. Implementation Steps

### Implementation Phase 1: Admin composer metadata

- GOAL-001: Remove the hard composer dependency so admin resolves without acl.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-001 | `composer.json`: move `"webware/webware-acl": "0.1.x-dev"` from `require` (line 34) to `suggest` with the description `"Provides the ACL implementation and ACL configuration consumed by the dashboard widget filtering, plus the ACL management UI."`. | ✅ | 2026-08-23 |
| TASK-002 | `composer.json`: remove the `webware/webware-acl` VCS entry from `repositories` (lines 62-65). | ✅ | 2026-08-23 |
| TASK-003 | `composer.json`: add `mezzio/mezzio-authentication-session: ^1.11` to `require-dev` (CON-002). Regenerate `composer.lock` (`composer update`) so the lockfile no longer resolves acl and gains the session adapter. | | |

### Implementation Phase 2: Admin source decoupling

- GOAL-002: Re-type the ACL seam to laminas while keeping widget filtering in admin (REQ-003).

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-004 | `src/Container/DashboardMiddlewareFactory.php`: replace `use Webware\Acl\AclInterface;` with `use Laminas\Permissions\Acl\AclInterface;`. Container lookup stays `$container->get(AclInterface::class)` (host app aliases it to its ACL implementation, in practice `Webware\Acl\Acl`). | ✅ | 2026-08-23 |
| TASK-005 | `src/Middleware/DashboardMiddleware.php`: (a) type-hint the constructor `acl` argument as `Laminas\Permissions\Acl\AclInterface`; (b) read the user from `Mezzio\Authentication\UserInterface::class` instead of `Webware\UserManager\UserInterface`; (c) keep filtering in admin - pass `$user instanceof RoleInterface ? $user : null` to the iterator so non-role-aware users fail closed. | ✅ | 2026-08-23 |
| TASK-006 | `src/ConfigProvider.php`: keep `getAclConfig()` but re-shape it for hosts that use laminas-permissions-acl - emit the default ACL rules under the `mezzio-authorization-acl` config key, matching the structure required by mezzio's integration package (`roles`: `User` + `Administrator => ['User']`; `resources`: the admin dashboard route name `webware.admin.dashboard.read`; `allow`: `Administrator`). Remove only the `Webware\Acl\AclInterface` import and the old `AclInterface::class` config key. webware-acl never consumes this config (it is DB-driven). | ✅ | 2026-08-23 |
| TASK-007 | `src/Widget/AclWidgetFilterIterator.php`: re-type to `Laminas\Permissions\Acl\AclInterface` and `?Laminas\Permissions\Acl\Role\RoleInterface`; null user denies every widget. The iterator STAYS in admin (REQ-003). | ✅ | 2026-08-23 |
| TASK-008 | `docs/dashboard-widget-system.md`: update the dependencies table (laminas-permissions-acl is the required `AclInterface`, webware-acl is the suggested implementation), the Configuration section (host app MUST provide an `AclInterface` service), and the security note (filtering fails closed). | ✅ | 2026-08-23 |

### Implementation Phase 3: WidgetContainer type enforcement

- GOAL-003: Enforce by construction that only `WidgetInterface` instances can reach `AclWidgetFilterIterator`, and flatten the `Widget\*` classes into the package root.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-015 | Move `src/Widget/WidgetInterface.php` → `src/WidgetInterface.php` and `src/Widget/AclWidgetFilterIterator.php` → `src/AclWidgetFilterIterator.php` (namespace `Webware\Admin\Widget` → `Webware\Admin`); delete the empty `src/Widget/` directory. | ✅ | 2026-08-23 |
| TASK-016 | New `src/WidgetContainer.php`: `final class WidgetContainer implements IteratorAggregate`; `__construct(?WidgetInterface $widget = null)` delegates to `addWidget(WidgetInterface $widget): void`; `addWidget()` appends and `usort`s ascending by `$order` (the ONLY mutation path); `getIterator()` returns a fresh `ArrayIterator` per call. | ✅ | 2026-08-23 |
| TASK-017 | `src/Event/RegisterWidgetEvent.php`: hold a `WidgetContainer` (constructor-injected, default `new WidgetContainer()`); `registerWidget()` delegates to `addWidget()`; expose `getWidgetContainer(): WidgetContainer`. | ✅ | 2026-08-23 |
| TASK-018 | `src/AclWidgetFilterIterator.php`: constructor takes `WidgetContainer` directly and passes `$widgets->getIterator()` to `parent::__construct()`; drop the `instanceof WidgetInterface` guard (type safety is structural now); keep null-user fail closed. `DashboardMiddleware` passes `$event->getWidgetContainer()`. | ✅ | 2026-08-23 |
| TASK-019 | Docs + plan updated for the new API; `analysis-baseline.toml` regenerated (the `IteratorIterator` false positive should disappear). | ✅ | 2026-08-23 |

### Implementation Phase 4: Webware-acl companion (separate repo, separate PR)

- GOAL-004: acl drops its bogus suggest entry and formally requires admin; admin keeps ownership of the laminas-acl default config (acl is DB-driven and never consumes it).

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-009 | acl `ConfigProvider`: NO config changes - acl's rules live in its database and are not derived from admin's `mezzio-authorization-acl` defaults. Admin keeps `getAclConfig()` specifically for hosts that use laminas-permissions-acl. | | |
| TASK-010 | acl `composer.json`: replace the bogus `suggest` entry (lines 41-43, referencing nonexistent `RegisterAclWidgetListener`) with `require: "webware/webware-admin": "0.1.x-dev"` and add the admin VCS repository entry. | | |
| TASK-011 | acl: ensure its `Acl` implementation satisfies `Laminas\Permissions\Acl\AclInterface` so host apps can alias `Laminas\Permissions\Acl\AclInterface::class => Webware\Acl\Acl::class`. Existing `RegisterWidgetListener` already bridges the event seam - no delegator or extra middleware needed. | | |

### Implementation Phase 5: Validation

- GOAL-005: Prove the refactor green and the decoupling real.

| Task | Description | Completed | Date |
|------|-------------|-----------|------|
| TASK-012 | Admin: `composer test`, `composer test-coverage`, Mago format/lint/analyze/guard, infection - all green with acl absent from admin's dependency tree. | | |
| TASK-013 | Integration check (acl repo): with admin installed as acl's dependency, dispatch the dashboard route; the ACL management widget appears and `AclWidgetFilterIterator` filters widgets per the settled ACL config. | | |
| TASK-014 | Fail-closed smoke check: admin without an `AclInterface` service cannot resolve `DashboardMiddleware` (container error); admin with an unauthenticated request renders zero widgets. | | |

## 3. Alternatives

- **ALT-001**: `mezzio-authorization` (mezzio's laminas-acl integration) as the filtering seam. Rejected - its string-based `isGranted()` fights webware-acl's object-centric `UserInterface`/assertion design (research in issue #12). Its **config structure** IS adopted for admin's shipped defaults (`mezzio-authorization-acl` key: roles/resources/allow).
- **ALT-002**: Move `AclWidgetFilterIterator` to webware-acl and have acl re-filter via a delegator or extra middleware. Rejected - filtering must stay in admin so the dashboard fails closed by construction (REQ-003); a delegator also silently breaks admin's fail-closed guarantee if acl is absent.
- **ALT-003**: Admin-defined `WidgetFilterInterface` service seam with acl providing the implementation. Rejected as over-engineering - the user's directive is a plain laminas type hint.
- **ALT-004**: WidgetManager extending `AbstractSingleInstancePluginManager` (laminas-servicemanager) with `$instanceOf = WidgetInterface::class` guarding validation. Rejected - the initial IMS rebuild around `/vendor` would absorb a huge refactor surface for marginal gain; plain typed mutation paths achieve the same guarantee.
- **ALT-005**: Hand-rolled `Iterator` (vs `IteratorAggregate`) on `WidgetContainer`. Rejected - a hand-rolled iterator shares one cursor across nested iterations; `IteratorAggregate` returning a fresh `ArrayIterator` per call avoids the aliasing trap.
- **ALT-006**: Keep the `instanceof WidgetInterface` guard inside `AclWidgetFilterIterator::accept()`. Rejected - with the constructor taking `WidgetContainer`, type safety is structural and the runtime check is dead weight.

## 4. Dependencies

- **DEP-001**: `laminas/laminas-permissions-acl: ^2.16` - present in admin (runtime seam).
- **DEP-002**: `mezzio/mezzio-authentication: ^1.13` - present in admin (user attribute contract).
- **DEP-003**: `mezzio/mezzio-authentication-session: ^1.11` - NEW require-dev (DB-free session authentication for tests, CON-002); brings `mezzio/mezzio-session` transitively.
- **DEP-004**: Companion PR in webware-acl (Phase 3) - merge order: acl PR first or coordinated, since admin drops the dependency acl's UI requires.
- **DEP-005**: The user under the `Mezzio\Authentication\UserInterface::class` attribute must implement laminas `RoleInterface` for object-role `isAllowed()` calls; admin fails closed otherwise (no runtime dependency on webware-usermanager).

## 5. Files

- **FILE-001**: `composer.json` (+ `composer.lock`) - require→suggest, VCS repo removal, mezzio-authentication-session require-dev.
- **FILE-002**: `src/ConfigProvider.php` - re-shaped `getAclConfig()` (mezzio-authorization-acl structure, roles `User`/`Administrator`, resource = admin dashboard route name); `AclInterface::class` key and `Webware\Acl` import removed.
- **FILE-003**: `src/Middleware/DashboardMiddleware.php` - laminas type hint, mezzio user attribute, fail-closed RoleInterface bridge, keeps filtering.
- **FILE-004**: `src/Container/DashboardMiddlewareFactory.php` - laminas import.
- **FILE-005**: `src/Widget/AclWidgetFilterIterator.php` - moved to `src/AclWidgetFilterIterator.php` (root namespace) and re-typed to laminas `AclInterface` + `?RoleInterface`; constructor now takes `WidgetContainer`; stays in admin.
- **FILE-005a**: `src/Widget/WidgetInterface.php` - moved to `src/WidgetInterface.php` (root namespace).
- **FILE-005b**: `src/WidgetContainer.php` - NEW: typed widget collection (TASK-016).
- **FILE-005c**: `src/Event/RegisterWidgetEvent.php` - holds `WidgetContainer`; `registerWidget()`/`getWidgetContainer()` API.
- **FILE-006**: `test/unit/AclWidgetFilterIteratorTest.php` - re-enabled with laminas stubs + `GenericRole` user.
- **FILE-007**: `test/unit/DashboardMiddlewareTest.php` - NEW: allowed/denied/missing-user/non-role-aware-user cases.
- **FILE-008**: `test/integration/DashboardMiddlewareIntegrationTest.php` - NEW: real PhpSession + real laminas Acl, no DB (CON-002).
- **FILE-009**: `test/integration/ConfigProviderIntegrationTest.php` - assert the `mezzio-authorization-acl` defaults (roles/resources/allow for the dashboard route) and no laminas `AclInterface` service definition.
- **FILE-010**: `docs/dashboard-widget-system.md` - dependency table, configuration note, fail-closed security note.
- **FILE-011** (acl): no config changes - DB-driven rules only.
- **FILE-012** (acl): `composer.json` - require admin, VCS repo.

## 6. Testing

- **TEST-001**: `AclWidgetFilterIteratorTest` (re-enabled, no skips): laminas `AclInterface` stubs + `Laminas\Permissions\Acl\Role\GenericRole` user - accepts when allowed, filters partially allowed, rejects when denied, denies all with a null user (fail closed). Non-widget rejection is now enforced by the `WidgetContainer` constructor instead of an `instanceof` guard.
- **TEST-002**: `DashboardMiddlewareTest` (new): allowed widgets attached to the `RegisterWidgetEvent::class` attribute; denied widgets filtered; missing user ⇒ empty iterator; non-`RoleInterface` user (`Mezzio\Authentication\DefaultUser`) ⇒ empty iterator (fail closed).
- **TEST-003**: `DashboardMiddlewareIntegrationTest` (new, DB-free): real `PhpSession` adapter authenticates a session-bound dual-interface user (Mezzio `UserInterface` + laminas `RoleInterface`), then a real `Laminas\Permissions\Acl\Acl` drives `DashboardMiddleware` - Administrator sees the widget, Member sees none.
- **TEST-004**: `ConfigProviderIntegrationTest`: assert the `mezzio-authorization-acl` defaults (roles `User`/`Administrator`, resource/allow = `webware.admin.dashboard.read`), assert every getter and `__invoke()` return their exact expected shape via `TestAssets\ExpectedConfig` (kills `ArrayItemRemoval` mutants), and assert that no `Laminas\Permissions\Acl\AclInterface::class` service definition is registered.
- **TEST-005**: Full CI matrix (mago, unit lowest/locked/latest, coverage, infection) green in both repos.
- **TEST-006**: `WidgetContainerTest` (new): empty container, constructor seeds one widget, `addWidget()` maintains ascending `$order`, `getIterator()` returns a fresh iterator per call. `RegisterWidgetEventTest` updated for the container API (delegation + stable container instance).

## 7. Risks & Assumptions

- **RISK-001**: The fail-closed design requires every host app to provide a laminas `AclInterface` service; apps that forget get a container-resolution error rather than a dashboard. Accepted: REQ-003 forbids unfiltered access.
- **RISK-002**: `mezzio-authentication-session`'s `PhpSession` expects the mezzio-session `SessionInterface` request attribute; the integration test supplies an in-memory session double - behavior on real `LazySession` persistence is covered by mezzio's own suite.
- **RISK-003**: Merge-order window where acl requires an admin version that no longer requires acl - coordinate PRs or land Phase 3 first.
- **ASSUMPTION-001**: The object under the mezzio authentication attribute implements laminas `RoleInterface` in practice (webware-usermanager/webware-acl users do); any other user type fails closed by design.
- **ASSUMPTION-002**: Admin's default ACL config targets laminas-permissions-acl hosts via the mezzio-authorization-acl config structure; webware-acl (DB-driven) ignores it. The route name is the resource: `webware.admin.dashboard.read` (prefix constant + `dashboard.read`, as registered by admin's `RouteProvider`).

## 8. Related Specifications / Further Reading

- [webware-admin issue #12 - Decouple admin from webware-acl](https://github.com/webinertia/webware-admin/issues/12)
- [webware-acl integration guide](https://github.com/webinertia/webware-acl/blob/0.1.x/docs/integration-guide.md)
- [webware-admin dashboard widget system docs](docs/dashboard-widget-system.md)
